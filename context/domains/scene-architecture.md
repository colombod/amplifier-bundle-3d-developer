# Scene architecture — domain reference

## Evidence status (read before citing anything below)

- **Sourced, engine-specific facts.** Babylon.js `TransformNode`,
  `AssetContainer`, thin instances, octree acceleration; Unreal's
  Actor/Component model, World Partition, HLOD, Nanite. Each has dedicated
  engine documentation. Treat as reliable.
- **Analytical generalization (GENERALIZATION).** The cross-engine "principles"
  layer — separate the spatial index from the scene graph, tier LOD, apply
  switch hysteresis, budget streaming — is **not asserted by any source in the
  research set**. It is inference from the two engines' documented behavior.
  Present it as design judgment, never as a sourced fact.
- **Explicitly unsupported; do not assert.** (a) That Babylon thin-instance
  collision uses a single bounding volume rather than per-instance tests — the
  research dropped this as having no valid source. (b) Unreal's
  `InstancedStaticMeshComponent` / `HierarchicalInstancedStaticMeshComponent` —
  standard practice, but unsourced; flag it as outside knowledge. (c) That
  Unreal `SceneComponent` carries transforms while `PrimitiveComponent` carries
  renderable geometry — reasonable elaboration, not directly sourced.
- **No numeric budgets are sourced.** Every number here is a working default to
  measure on target hardware, not a citation.

## The five layers

GENERALIZATION. Large-scene organization decomposes into five structures often
conflated into one — which makes each pay the others' update cost.

| Layer | Question it answers | Typical structure | Changes |
|---|---|---|---|
| 1. Transform hierarchy | What moves with what? | Parent chain of transform nodes | Per frame, for animated subtrees |
| 2. Spatial index | What is near/inside/hit by this? | Octree, BVH, uniform grid | On rebuild/refit cadence |
| 3. LOD tiers | How detailed should this be right now? | Per-object tier table | Per frame, threshold-driven |
| 4. Instancing/batching | How few draws can express N copies? | Instance buffers, merged meshes | On population change |
| 5. Residency/streaming | What is in memory at all? | Cells/tiles + eviction policy | On camera movement |

## Layer 1 — transform hierarchy

Babylon: `TransformNode` is a non-rendered transform object used to parent
meshes, lights and cameras — the documented way to build group nodes without
paying for an empty renderable mesh. Unreal: an `AActor` is a placeable world
object composing `UActorComponent`s, with a documented component registration
path (`RegisterComponent`, `BeginPlay`) and actor lifecycle.

GENERALIZATION: depth costs per-frame matrix work on active nodes. Build the
hierarchy around *what moves, toggles or streams together*; keep semantic
taxonomy ("all pumps in Building 3") in a side map. Culling removes draw work,
not transform propagation of nodes you kept active.

## Layer 2 — spatial index (keep it out of the hierarchy)

Babylon exposes scene-level octrees via `createOrUpdateSelectionOctree` and
`OctreeSceneComponent`, plus submesh-level octrees on `AbstractMesh` for picking
and collision acceleration on meshes with many submeshes. Babylon's
optimization, picking, and collision docs frame octrees as accelerating
visibility/picking/collision queries — a query structure distinct from the
parent hierarchy. Unreal exposes no equivalent application-facing spatial
structure in the research set; its per-system structures (World Partition grid,
Nanite clusters) are handled per system.

GENERALIZATION — structure choice:

| Structure | Best when | Build | Query | Under motion |
|---|---|---|---|---|
| None (linear scan) | Few thousand candidates, rare queries | free | O(n) | free |
| Uniform grid | Uniform object size/density, bounded extent | O(n) | O(1) cell + contents | cheap: reassign cell |
| Octree | Strongly non-uniform density, large empty volume | O(n log n) | O(log n) | poor for fast movers; needs rebuild/refit |
| BVH | Irregular geometry needing tight bounds; ray queries | O(n log n) | O(log n) | refit cheap, quality decays |

Practical rule (GENERALIZATION): keep static and dynamic objects in separate
indexes — build the static one once, keep the dynamic set small enough to scan
linearly. Below a few thousand tested objects, an index often loses to a linear
scan.

## Layer 3 — LOD tiering and hysteresis

Unreal supplies two documented and *different* mechanisms:

- **HLOD** (World Partition): groups static actors into HLOD layers and
  generates proxy meshes/materials for unloaded cells — explicitly framed around
  **reducing draw calls** in large open worlds. Generated via the World
  Partition builder commandlet; requires static actors assigned to an HLOD layer.
- **Nanite**: imports meshes into hierarchical triangle clusters and selects
  detail dynamically at render time ("virtualized geometry"), documented for
  foliage and landscapes. This attacks **triangle count**.

Babylon has no automatic clustering or aggregate-proxy system in the research
set; tiering there is application-built.

GENERALIZATION — tiering template (tune, do not copy the numbers):

| Tier | Representation | Switch in (screen coverage) | Switch out |
|---|---|---|---|
| L0 | Full geometry | > 15% | — |
| L1 | Reduced mesh | 4–15% | above 18% |
| L2 | Impostor / billboard | 0.5–4% | above 5% |
| L3 | Aggregate proxy per cell | < 0.5% | above 0.7% |
| L4 | Not resident | outside radius | inside radius minus band |

Two rules: switch on projected screen coverage, not raw distance (distance
mis-tiers objects of differing size); and give every threshold a hysteresis band
— a separate switch-in and switch-out value — or boundary objects flip every
frame.

## Layer 4 — instancing

| Need | Technique | Cost / trade |
|---|---|---|
| Per-copy lifecycle, picking, collision | Ordinary instances — Babylon documents these separately, for copies needing independent behavior | Per-copy object and per-copy CPU update |
| Transform-only variation, very large N | Thin instances — per-instance transforms in packed matrix data, documented as avoiding per-instance JS object overhead, intended for large near-identical populations | Add/remove rewrites buffer data; do **not** assume per-instance picking/collision — unsourced |
| Static, never individually addressed | Merge into one mesh | One draw; loses per-object culling and picking; full rebuild on any change |
| Millions of identical primitives | Single procedural mesh, GPU-side placement (GENERALIZATION) | Cheapest draw-wise; CPU addressability gone |

A Babylon forum comparison of clone vs instance vs container exists, but is
community material, not spec.

## Layer 5 — residency and streaming

Babylon: `AssetContainer` / `loadAssetContainerAsync` load assets **without**
automatically adding them to the active scene; the application controls
activation and removal. Community threads report friction around memory
retention, moving already-attached content into a container, and multi-GLB
loading. Read that as: the container is a **staging primitive, not a residency
manager**. You own eviction and disposal.

Unreal: **World Partition** divides a persistent world into a streaming grid of
cells loaded and unloaded based on streaming sources, with data layers and an
offline builder commandlet. Unreal's asset/package system is outside the
research set.

GENERALIZATION — derive the streaming unit from a byte budget: memory ceiling
divided by worst-case (not average) per-cell footprint. The unit of load must
equal the unit of unload. Give the load radius its own hysteresis band so a
camera hovering on a boundary does not thrash, and amortize activation across
frames — a cell that becomes resident in one frame is a visible hitch.

## Platform pointers (hand-off, not a tutorial)

| Technique | Babylon.js | Unreal |
|---|---|---|
| Group/transform node | `TransformNode` | `AActor` + components |
| Off-scene staging | `AssetContainer`, `loadAssetContainerAsync` | not in research set |
| Coarse world partition | application-built cells | World Partition grid, streaming sources, data layers |
| Distant aggregate proxy | application-built | HLOD layers + builder commandlet |
| Fine geometric LOD | application-built tiers | Nanite hierarchical clusters |
| Repeated objects | thin instances; ordinary instances | instanced static mesh components (unsourced) |
| Spatial query acceleration | scene octree (`createOrUpdateSelectionOctree`, `OctreeSceneComponent`), submesh octree on `AbstractMesh` | no application-facing equivalent sourced |

## Symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Hitch when the camera crosses a boundary | Streaming unit too large, or activation done synchronously in one frame |
| Objects pop at a fixed distance | Single LOD threshold, no hysteresis band |
| Memory climbs monotonically while navigating | No eviction path; staged containers never disposed |
| Added an octree, no speedup | Index rebuilt every frame under dynamic content, count below the crossover, or queries not routed through it |
| Triangles down after LOD work, frame time flat | You were draw-call bound; need aggregation/proxies or instancing |
| CPU pegged with almost nothing visible | Transform propagation over the whole active world; culling does not skip it |
| Picking slow at scale | Scanning the full mesh list instead of a spatial or submesh octree |

## Pitfalls

- Mirroring semantic taxonomy into the transform hierarchy, then paying for its
  depth every frame.
- Rebuilding the whole spatial index because a handful of objects moved.
- Tuning LOD by world distance when objects vary in size.
- Merging geometry for draw-call wins and silently losing per-object culling and
  picking.
- Treating `AssetContainer` as a memory manager rather than a staging primitive.
- Assuming Nanite or HLOD removes the architecture decision — they solve
  triangle count and draw count respectively, and HLOD requires static actors
  assigned to an HLOD layer: a content constraint, not a runtime toggle.
- Designing the load path and never the unload path.
- Assuming thin instances support per-instance picking or collision without
  verifying — that behavior is explicitly unsourced.

## Known open questions (unsettled in the research)

- Babylon octree defaults (leaf capacity, depth) and dynamic-content handling
  across versions.
- Whether classic Unreal instancing components interoperate with World
  Partition and HLOD as commonly assumed — unsourced.
- Whether the cross-engine principles layer reflects documented consensus or
  only generalization — no cross-engine source existed.

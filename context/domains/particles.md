# Particle systems and real-time VFX — domain reference

## Evidence status legend

Preserve this separation when you answer.

- **[DOC]** — official Babylon.js or Unreal/Niagara documentation.
- **[COMM]** — community forums/blogs only; plausible, not official.
- **[PRACTICE]** — standard VFX authoring practice, **not** established by the
  documentation surveyed. Sound craft; cite it as craft.

Unverified by the research: soft particles/depth fade by name; any canonical
explosion recipe; curl noise, vector fields, attractors, SDF flow; and whether
Niagara Fluids or Babylon's Fluid Renderer use SPH, FLIP or grid solvers. There
is no cross-engine benchmark in the sources, so numbers here are heuristics.

## 1. Simulation architecture: where the particle solve runs

| System | Simulation | Best for | Known limits |
|---|---|---|---|
| Babylon `ParticleSystem` | CPU sim, GPU render [DOC] | Moderate counts, sub-emitters, update callbacks [DOC] | CPU tick scales with live count |
| Babylon `GPUParticleSystem` | Sim on GPU [DOC] | High counts, cheap uniform math | No sub-emitters; manual-emission and gradient restrictions; falls back to CPU when unavailable [DOC] |
| Babylon `SolidParticleSystem` | One mesh, many instances; you write `updateParticle` + `setParticles()` [DOC] | Mesh debris, chunks, solid crowds | No emitter/recycler/physics layer [DOC]; mesh depth sorting is community-discussed only [COMM] |
| Niagara emitter | Per-emitter CPU-or-GPU target [DOC] | Everything; target must match the spawn/update scripts [DOC] | GPU Simulation Stages add iterative compute over grids/render targets [DOC] |

**Crossover heuristic [PRACTICE]:** CPU below roughly a few thousand live
particles per system, or whenever you need sub-emitters, collision events or
per-particle reads back into application logic; GPU above that, or when the math
is uniform and self-contained. The decisive question is not count but *what the
particle needs to know*.

**Overhead that does not move [DOC]:** Epic's scalability guidance states every
Niagara simulation carries CPU cost for system/emitter management even when the
solve runs on the GPU. Ten GPU systems of 100 particles are worse than one system
of 1,000.

## 2. Emitter and module anatomy

Babylon's `ParticleSystem` exposes emission, lifetime, colour, size, gravity,
direction and sprite-sheet properties as flat settings on one object, extended by
update callbacks [DOC]; a graph-based Node Particle Editor is newer [DOC].
Niagara's model is an **ordered module stack** per emitter (Spawn/Update stages
at System, Emitter and Particle scope, plus Events and Simulation Stages), and
**module order is functionally significant** — a force applied before versus
after a velocity clamp produces different motion [DOC]. Babylon ships composable
primitives; Niagara ships a data-driven authoring framework [DOC].

Portable vocabulary [PRACTICE]: emitter shape, spawn mode (rate vs burst),
lifetime and jitter, initial velocity cone and speed range, forces (gravity,
drag, turbulence, attraction), and **over-life curves** for size, colour, alpha
and rotation. Readability lives in the curves, not the counts.

## 3. Effect recipe table

All rows are [PRACTICE] unless noted; timings are starting points, not
measurements.

| Effect | Layered components (onset order) | Key parameters | Dominant cost |
|---|---|---|---|
| **Smoke plume** | dense core puffs (alpha) · outer wispy dissipators (alpha, low opacity) · ground-spread ring · optional heat haze | Size grows monotonically (smoke never shrinks); slow rotation-over-life; high drag; fast alpha fade-in, slow fade-out; colour darkens then lightens; spawn 5–30/s, lifetime 2–6 s | Overdraw from large alpha quads, plus the alpha sort |
| **Explosion** | t=0 core flash (additive, 1–3 frames, 1–2 huge particles) · t=0 shockwave (one expanding distortion ring) · t=0–0.1 fireball (additive flipbook, fast growth then decelerate) · t=0.05–0.3 debris (mesh particles, gravity + trails) · t=0.1–2.5 smoke column (alpha, slowest, outlives all) · transient light keyed to the flash | Staggered onsets; per-layer size curve with strong deceleration; light intensity/radius matched to the fireball; debris speed cone + drag | Peak overdraw in the first ~3–6 frames; the transient light; debris draw calls |
| **Energy flow / beam** | additive core ribbon or trail along a spline · outer glow sheath · sparse travelling motes · endpoint impact burst | Lifetime = path length / speed (particles die at the end); trail segment count; emissive intensity set against the bloom threshold | Trail vertex count and the bloom interaction, rarely particle count |
| **Fluid / liquid** | particle positions (scripted, force-driven or solver) · screen-space surface reconstruction from depth + thickness + diffuse targets [DOC: Babylon Fluid Renderer] · optional foam/spray layer | Particle radius; depth-blur iterations; thickness-to-absorption; refraction | Extra full-screen passes and the blur — *not* particle count |
| **Dust / motes** | one low-rate emitter in a camera-attached volume, respawning near the viewer | Alpha 0.02–0.15; lifetime 5–20 s; tiny size; slight turbulence; near-zero gravity | Near-zero; fails by being invisible or reading as sensor noise |
| **Sparks / embers** | velocity-stretched additive billboards · short bounce/secondary spawn · slow drifting embers with long fade | Stretch ∝ speed; initial speed 3–15 m/s in a tight cone; gravity + drag; lifetime jitter ±50%; alpha flicker | Cheapest family; cost is per-system overhead and any per-spark light |

Two rules behind the table [PRACTICE]: **the slowest layer is the longest lived**
(smoke outlives fire, fire outlives flash), and **each layer gets its own blend
mode** — emissive layers additive, occluding layers alpha.

## 4. Blending, sorting and depth

- **Additive is order-independent.** Use it for anything emissive; it removes the
  sort problem. Cost: it cannot darken, and stacked layers saturate to white —
  severe with HDR plus bloom [PRACTICE].
- **Alpha needs an order.** Babylon exposes blend-mode options on the standard
  property set [DOC]; Niagara documents `sort_mode` on sprite and mesh renderers
  and `TranslucencySortPriority` at system/component level [DOC].
- **Open disagreement, do not resolve it:** official Niagara docs present sorting
  as a controllable feature [DOC]; community sources report recurring
  translucency-sorting problems in practice [COMM]. Design so a sort failure
  degrades gracefully.
- **Depth fade / soft particles.** Not named in either engine's surveyed docs;
  what exists is Babylon developers sampling a depth texture (`depthSampler`)
  from a custom `ShaderMaterial` [COMM]. Fading alpha by (scene depth − particle
  depth) to kill the hard intersection line is standard [PRACTICE] and **depends
  on the pass layout** supplying a scene depth texture to particle materials.

## 5. Flipbooks versus volumetric

Babylon documents sprite-sheet / animated-billboard properties [DOC]. Niagara has
a **Flipbook Baker** converting a simulation — including volumetric smoke/gas —
into a tiled texture played back cheaply by a sprite emitter [DOC], and Niagara
Fluids provides 2D/3D templates for fire, smoke and gas [DOC].

The asymmetry is source-supported: **Niagara has a documented volumetric-smoke
pipeline; Babylon's documented tools do not** — its Fluid Renderer is explicitly
a liquid-surface renderer [DOC]. On the web, budget for flipbook smoke and design
the effect to read without volumetric parallax. The trade is VRAM for simulation
time: an 8x8 atlas at 1024x1024 RGBA8 is ~4 MB uncompressed — state atlas
dimensions and format in any cost claim [PRACTICE].

## 6. Flow fields and energy motion

No surveyed source names curl noise, vector fields, attractors or SDFs for either
engine — treat all as undocumented [PRACTICE]. The *infrastructure* is
documented: Babylon CPU particles accept arbitrary per-particle update logic
[DOC]; GPU particles have a narrower set plus shader hooks [DOC]; Niagara GPU
Simulation Stages iterate over grids and render targets, exactly what a flow
field needs [DOC]. "You can build it" is documented; "it is built in" is not.

## 7. Cost control

Niagara ships a scalability document covering quality levels, culling,
significance and platform overrides [DOC]. Babylon's is thinner: GPU particle
docs cover capacity, emission and disposal, and note that **stopping emission
does not remove already rendered particles — explicit disposal is required**
[DOC]. The only benchmark in the sources compares Babylon SPS to Unity WebGL
[COMM] — make no Babylon-vs-Niagara performance claim.

Cut order when over budget [PRACTICE]: scale spawn rate → shrink particle size
(direct overdraw reduction) → drop the least-narrative layer (outer wisps, then
debris) → bake a flipbook → distance-cull the system.

## 8. Symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Explosion reads as one flat puff | One emitter doing five jobs; no staggered onsets; no per-layer blend modes |
| Hard line where smoke meets the floor | No depth fade; quad intersects geometry |
| Particles pop in front of each other while orbiting | Alpha layers depending on per-particle sorting; move emissive layers to additive and set explicit sort priority |
| Frame collapses only when the camera is close | Overdraw — screen coverage rose, count did not |
| Effect washes out to white | Additive stacking in HDR feeding bloom |
| GPU sim enabled but CPU time unchanged | Per-system management overhead is still CPU-side; too many separate systems |
| Particles persist after the effect ends | Emission stopped, system never disposed |
| Fluid surface looks like blobs | Depth blur too weak, or particle radius small relative to spacing |

## 9. Platform pointers (hand-off, not tutorial)

- **Babylon.js:** `ParticleSystem` (CPU) · `GPUParticleSystem` (GPU, no
  sub-emitters) · `SolidParticleSystem` for mesh debris · blend-mode and
  sprite-sheet properties · Fluid Renderer for screen-space liquid · Node
  Particle Editor · depth texture via custom `ShaderMaterial` for depth fade.
- **Unreal / Niagara:** System → Emitter → Particle module stacks (Spawn/Update)
  · Events for cross-emitter triggering · GPU Simulation Stages · sprite and mesh
  renderers with `sort_mode` · `TranslucencySortPriority` · Niagara Fluids
  templates · Flipbook Baker · the scalability document (quality levels, culling,
  significance).

Name the construct, then route API-level work to the platform specialist.

## 10. Pitfalls

1. **Budgeting in particle counts** instead of covered pixels times overlapping
   blended layers.
2. **Assuming GPU simulation is free** — management overhead stays CPU-side, and
   the feature loss (sub-emitters, manual emission) is real.
3. **Alpha by default**, which buys a sorting problem the layer did not need —
   then trusting that sorting.
4. **Forgetting disposal.** Stopping emission is not cleanup.
5. **Shrinking smoke.** Dissipating volumes expand and thin.
6. **Uniform timing.** Layers that start and end together read as one object.
7. **Claiming a solver class.** Screen-space fluid rendering is a renderer, not a
   simulation.
8. **Many small systems instead of one large one.**
9. **Presenting craft as documentation** — explosion recipes, soft particles and
   flow fields have no documentation trail here.

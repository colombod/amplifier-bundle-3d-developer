# Unreal Engine — domain to construct map

Reference for implementing an already-designed 3D visualization in Unreal
Engine 5. Organised as *domain → Unreal construct*, with an evidence tag on
each claim.

## Evidence tags

| Tag | Meaning |
|---|---|
| `[documented]` | First-party Epic documentation. |
| `[community]` | Real and widely used, but supported only by forum threads, third-party guides or bug trackers. |
| `[4.27-era]` | The supporting page is UE 4.27. The concept persists into 5.x; continuity of names, UI and behaviour is **unconfirmed**. |
| `[unverified]` | Outside knowledge with **no supporting source**. Say so or drop it. |

Two cautions everywhere. **Locale duplicates are not corroboration** — the
Lumen Performance Guide, Spring Arm guide, anti-aliasing and material pages
each appear in several languages; one source, repeated. **Renames masquerade as
different features**: `Skeletal Controls` → `Animation Blueprint Skeletal
Controls`; `Control Rig Blueprints` → `Animating with Control Rig`;
`Sequencer Overview` → `Sequencer Cinematic Editor`.

## 1. Scene — actors, partitioning, LOD, instancing

| Concern | Construct | Tag |
|---|---|---|
| World object | `AActor` composing `UActorComponent`s; lifecycle via `RegisterComponent`, `BeginPlay` | `[documented]` |
| Transform vs geometry | `USceneComponent` carries transform, `UPrimitiveComponent` renderable geometry | `[unverified]` — no reference page in the research; elaboration, not fact |
| Coarse streaming | World Partition: a streaming grid of cells loaded/unloaded by *streaming sources*; an offline builder commandlet processes the world | `[documented]` |
| Distant proxies | HLOD layers group **static** actors into generated proxy meshes/materials for unloaded cells, framed around draw-call reduction; actors must be static and in an HLOD layer, built by the commandlet | `[documented]` |
| Fine geometric LOD | Nanite: hierarchical triangle clusters, detail selected at render time ("virtualized geometry"); foliage and landscape covered separately | `[documented]` |
| Repeated objects | `InstancedStaticMeshComponent` / `HierarchicalInstancedStaticMeshComponent` | `[unverified]` — standard practice, but **no source documents these classes** |
| Spatial index | None application-facing; no octree/BVH equivalent, acceleration is renderer-internal per system | `[documented]` as an absence |

**Nanite eligibility.** Nanite's *exclusion* rules are **not documented
anywhere in the research**. The commonly held constraints — skeletal/deforming
geometry, world-position-offset materials, translucency, with per-version
relaxations across 5.0–5.7 — are `[unverified]` here. Never state one as fact:
state it, tag it, name the current Nanite page to check. Whether classic
instancing interoperates with World Partition/HLOD is likewise unsettled.

## 2. Rendering — paths, Nanite, Lumen, culling

- **Path choice.** The supported-features-by-rendering-path page is the
  authority on forward vs deferred; mobile is separate. Lumen and Nanite do not
  change the path — they add work on top of it. `[documented]`
- **Translucency** in the deferred renderer runs through a forward-style pass
  and gets **no** deferred lighting, Nanite opaque geometry or not.
  `[documented]`
- **Nanite and draw calls**: cluster selection and visibility are GPU-driven
  rather than per-object CPU draw submission; non-Nanite meshes still depend on
  classic instancing/merging. `[documented]`
- **Culling**: occlusion is the most expensive stage, applied after cheaper
  distance/frustum rejection. `[documented]`
- **The one published budget number**: Lumen documents roughly **4 ms GPU at
  1080p** for a 60 fps console target `[documented]` — but whether that covers
  screen-probe gather, hardware RT and surface-cache update is **not
  disambiguated**. **No per-stage frame budget is published at all**; any table
  splitting the 16.6 ms is synthesis, labelled `[unverified]`.
- **Threads**: game, draw/render and GPU time are reported independently, never
  summed into one FPS number. `[documented]`; some pages `[4.27-era]`.

## 3. Materials and shading

| Concern | Construct | Tag |
|---|---|---|
| Inputs | Base Color, Metallic, Specular, Roughness, Normal, AO, Emissive, Opacity; metallic/roughness `[0,1]`; colour textures sRGB, data textures linear | `[documented]` |
| Cutout vs fractional | **Opacity Mask** is a binary discard; **Opacity** is fractional transparency — different inputs, not two settings | `[documented]` |
| Hand-written HLSL | Custom Material Expression node, named inputs, declared output type; Epic advises keeping it compact, not replacing the graph | `[documented]` |
| Generated code | Material Editor → Window → Shader Code → HLSL Code | `[documented]` |
| Reuse and variants | Material Instances for parameter override; Material Functions for shared logic; **static switches compile features out and generate permutations** — Epic recommends minimising unused combinations | `[documented]` |
| Packed data | Virtual texturing supports packed mask channels | `[documented]` |
| Per-element IDs | No "ID material input" exists. Encode via vertex attributes or data textures: linear, no sRGB, no unintended interpolation | `[unverified]` by name, supported by mechanism |
| Transparency order | Source-over blend, order-sensitive; **Translucency Sort Priority** is the control | `[documented]` |
| OIT | **No built-in general OIT is documented** — a real asymmetry with Babylon.js, which ships one-flag dual depth peeling | `[documented]` as an absence |
| Shader cost | Shader Complexity view mode for per-pixel instruction cost | `[documented]`; its approximation caveat (translucency cost also depends on overdraw) and the ProfileGPU + Material Editor stats pairing are `[4.27-era]` |

Cost ordering supported by the evidence: cut passes, overdraw and translucent
screen coverage first; then redundant texture samples; then permutation count;
only then instruction-level arithmetic.

## 4. Interaction, picking and input

- **Hit testing** is line traces: `LineTraceByChannel` (with `Trace Complex`),
  `MultiLineTraceByChannel`, `BreakHitResult` for the fields. `Trace Complex`
  means per-triangle collision and is an expensive explicit opt-in — keep it
  off for routine hover and picking. `[documented]`
- **Enhanced Input** is the documented device-abstraction layer.
  **Touch-gesture handling within it is not documented** — a real gap.
- **Gizmos**: Interactive Tools Framework — `UInteractiveGizmo`,
  `UTransformGizmo` against a `UTransformProxy`, `UCombinedTransformGizmo`,
  `UInteractiveGizmoManager` `[documented]`. **Whether these are supported for
  runtime gameplay rather than editor tooling is not stated by any source** —
  the Editor/Runtime module split makes this a cook-boundary question.
  `WidgetInteractionComponent` raycasts a `WidgetComponent` for world-space UI.
- **Selection outlines** via custom depth/stencil + post-process material rest
  on a third-party guide, forum threads and a bug tracker `[community]` — treat
  stencil conventions as unverified. **No gameplay multi-selection API is
  documented**; any selection-set architecture is `[unverified]`.
- **Accessibility**: screen-reader support, `FScreenReaderUser`, CommonUI and
  `UWidget` are documented **for UMG/Slate only** — nothing covers the 3D
  viewport or gizmo interaction.

## 5. Camera

| Need | Construct | Tag |
|---|---|---|
| Follow / third-person | Spring Arm + attached Camera Component: target arm length, collision probe, camera and rotation lag with **lag substepping** for stability under variable frame rate | `[documented]` |
| View parameters | `UCameraComponent`; `FMinimalViewInfo` / `MakeMinimalViewInfo` carry near/far clip. No standalone `CameraActor` page in the research | `[documented]` |
| Cinematic | Cine Camera Actor / `UCineCameraComponent`, with extra clipping controls | `[documented]` |
| Orthographic | Explicit near/far clip, ortho planes auto-calculable from ortho width, projection mode settable from Blueprint | `[documented]` |
| Authored moves | Sequencer `CameraCutTrack` binds a camera to a timeline section and cuts between cameras | `[documented]` |
| Overview / minimap | `USceneCaptureComponent2D` renders a secondary view to a texture a widget can display | `[documented]` |
| Runtime fit-to-view | **Not documented** — application-level from actor bounds, projected size and FOV | gap |

Depth precision is governed by the **far/near ratio**, not absolute far
distance; a zero or needlessly small near plane is the primary z-fighting
cause. The Spring Arm collision probe stops the camera entering geometry — it
is **not** a depth-precision mechanism.

## 6. Post-processing

- **Authoring surface**: Post Process Volumes and `FPostProcessSettings` — DOF,
  bloom, exposure/tone mapper, colour grading and LUTs. `[documented]`
- **The distinction that trips people up**: post-process materials separate an
  **HDR "before tonemapper"** input from an **LDR "after tonemapper"** input.
  HDR-dependent work (bloom extraction, DOF, motion blur) sits before the
  tonemapper; stylised grading and outline overlays that expect
  display-referred values sit after. Sampling the wrong one is the single most
  common post-process material bug. `[documented]`
- **Anti-aliasing**: FXAA, MSAA, TAA and TSR are **mutually exclusive
  final-stage choices**. MSAA is forward desktop/console and mobile only —
  **not deferred**. FXAA is unsupported on mobile-forward. TSR is the
  first-class temporal upscaler and has `stat tsr`. `[documented]`
- **Fog**: `ExponentialHeightFogComponent` with height falloff; Volumetric Fog
  layers onto it with albedo/scattering/emissive controls. `[documented]`
- **Absent from the evidence**: GTAO by name, any dedicated SSR page, and **any
  quantified GPU cost for any effect** — cost rankings are `[unverified]`. The
  4.27 pages describe a feature set with **no TSR in it**. Scalability controls
  are the documented quality lever.

## 7. Labels and callouts

- `WidgetComponent` has two modes with a load-bearing difference: **World
  Space** participates in the depth buffer (so it can be occluded) and offers
  Geometry Mode **Plane** or **Cylinder**; **Screen Space** renders outside the
  3D world and is **never depth-occluded**. `[documented]`
- `ModifyProjectedLocalPosition` adjusts a widget's projected screen position —
  the integration point for a custom decluttering pass. `[documented]`
- Cost control for many widgets: Slate/UMG invalidation,
  `bUseInvalidationInWorldSpace`, Invalidation Boxes, Retainer Boxes/Panels
  (render children to a texture, update at a reduced rate), UMG optimization
  guidelines. `[documented]`
- **SDF text** gives resolution-independent scaling and cheap outlines, with an
  explicit caveat: it is an approximation, loses quality at very small sizes
  and on thin glyphs, and lacks hinting. `[documented]`
- **Depth-occluding a screen-space widget is unsolved and community-improvised**
  (line traces, custom-depth checks), and **no leader-line control exists** —
  that needs a custom UMG paint pass. `[community]`

## 8. Animation and sequencing

- **Animation Blueprint AnimGraph**: blend nodes, additive, overrides, per-bone
  blending; state machines; Blend Spaces for continuous parameters; **Layered
  Blend Per Bone**; Blend Masks/Profiles; Virtual Bones. `[documented]`
- **Skeletal Control nodes** (Two Bone IK, FABRIK, Look At, Transform/Modify
  Bone, CCD IK, Spline IK) work largely in component space `[documented]`, with
  the 4.27/5.6 rename noted above.
- **Physics blending**: the Rigid Body node simulates a skeletal-mesh segment
  inside the AnimGraph; blend weight runs 0.0 (keyframed) to 1.0 (simulated),
  via `Set All Bodies Below Simulate Physics` / `... Physics Blend Weight`.
  `[documented]`
- **Control Rig** when procedural logic needs a reusable rig, constraints and
  Sequencer-authorable controls. `[documented]`, title drift 4.27 → 5.6.
- **Sequencer** carries camera shots, transforms, animation, Control Rig and
  Niagara tracks; a Slot node plus track weight blends sequence animation with
  live Anim BP output; `UMovieSceneSequencePlayer` plays/pauses/scrubs at
  runtime, and a Python surface builds sequences programmatically.
  `[documented]`
- **Determinism** — the lever a visualization actually needs: Sequencer keeps
  display rate, tick resolution, play rate and time dilation separate on a
  rational-frame-rate tick model; display-rate defaults are per project and per
  sequence; Cinematic Playback Rate propagates time scaling to materials and
  particles; a custom time-step class fixes the engine tick rate for genlocked
  or reproducible output. `[documented]`
- **Morph targets**: no Unreal morph-target documentation appears in the
  research; any guidance on naming layers, correctives or capping active
  targets is `[unverified]`.

## 9. Niagara

- **Simulation target is per emitter** (CPU or GPU) and must be consistent with
  that emitter's spawn/update/simulation scripts. `[documented]`
- **Module stack**: ordered per emitter — System, Emitter and Particle Spawn
  and Update stages, extended by Events and Simulation Stages. **Module order
  is functionally significant.** `[documented]`
- **GPU Simulation Stages** do iterative computation over particle arrays,
  grids and render targets — the infrastructure a flow-field or grid solver
  needs. `[documented]`
- **Sorting**: `sort_mode` on sprite and mesh renderers plus
  `TranslucencySortPriority` at system/component level `[documented]` — **but
  independent community reports describe Niagara sorting behaving unreliably**,
  and the evidence does not resolve whether that is version- or
  configuration-specific. Surface the tension. `[community]`
- **Flipbook Baker** bakes a simulation (including volumetric smoke/gas) into a
  tiled flipbook for cheap sprite playback — no Babylon equivalent.
  **Niagara Fluids** gives 2D/3D fire/smoke/gas templates and is the only
  documented volumetric smoke path in either engine, though **whether it is
  SPH, FLIP or grid-based is unconfirmed**. `[documented]`, `[unverified]`
  solver.
- **Mesh renderer motion vectors** are configurable — needed for correct motion
  blur on GPU mesh particles. `[documented]`
- **Scalability** is first-class: quality levels, culling, significance,
  platform overrides. Its load-bearing warning: **every Niagara simulation
  carries CPU overhead for system and emitter management even when the
  simulation runs on the GPU** — GPU moves part of the cost, not all.
  `[documented]`
- **Not named anywhere in the evidence**: soft particles / depth fade, curl
  noise, vector fields, SDF attractors, any explosion recipe, any fixed
  timestep mode. All `[unverified]`.

## 10. Profiling — the commands that back a cost claim

| Question | Tool | Tag |
|---|---|---|
| Where is the frame going? | `stat unit` (game / draw / GPU), then `stat gpu` | `[documented]`, some pages `[4.27-era]` |
| Pass-level GPU breakdown | `ProfileGPU` / `Stat GPU` — what the Lumen guide recommends for Lumen timing | `[documented]` |
| Temporal upscaler cost | `stat tsr` | `[documented]` |
| CPU/GPU timeline, thread stalls | Unreal Insights | `[documented]` |
| Per-pixel shader cost | Shader Complexity view mode | `[documented]` (5.6); caveat `[4.27-era]` |
| Frame capture / replay | RenderDoc (incl. a Meta fork for Quest and UE mobile VR); PIX on Windows/Xbox | `[documented]` |
| Mobile overdraw | Epic's mobile perf docs flag masked and translucent overdraw, with diagnostic views | `[documented]` |
| Golden-image regression | **No Unreal equivalent to Unity's Graphics Test Framework exists in the evidence** | gap |
| Frame-budget methodology | **Not documented by anyone** — tools exist, a budget process is not a named artifact | gap |

VR has its own performance documentation and a different frame budget — never
reuse a 60 fps desktop number for it.

## 11. Where Unreal's game defaults fight a visualization

Analytical, not sourced as an Unreal claim — but each row names a real default.

| Default | Why it corrupts a visualization | Lever |
|---|---|---|
| Auto-exposure | Apparent brightness of an encoded surface changes between frames and viewpoints, so values stop being comparable | Fix exposure in the Post Process Volume |
| TAA / TSR | Smears thin geometry and small moving features — what plot lines, glyphs and sparse points are made of | Try a non-temporal AA path; verify thin-geometry legibility |
| Cinematic post stack | Blooms bright data into neighbouring data; blurs the positions the reader must compare | Disable per-effect in the volume |
| Lumen GI | Bounces coloured light onto surfaces whose colour *is* the value — shading and data colour become confounded | Constrain bounce colour, or unlit shading for encoded surfaces |
| Translucency defaults | Order-sensitive source-over with no OIT yields sorting artifacts that read as data | Opaque, then masked; set Translucency Sort Priority |

## 12. Pitfalls — symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Post-process material washed out or clipped | Sampling the LDR after-tonemapper input when the effect needs HDR scene colour, or the reverse |
| MSAA option does nothing | Project is deferred; MSAA is forward/mobile only |
| Thin geometry shimmers or ghosts on camera motion | TAA/TSR accumulation on sub-pixel features |
| Same colour reads differently in two screenshots | Auto-exposure, not a material or colormap bug |
| Translucent elements pop past each other while orbiting | Source-over ordering; set Translucency Sort Priority, or make the surface masked |
| Selection outline works in editor, not packaged | Custom depth/stencil off in that config, or an editor-only module dependency |
| Picking cost spikes on hover | `Trace Complex` left on — per-triangle collision every pointer move |
| Labels draw over everything regardless of depth | Screen-space `WidgetComponent`; never depth-occluded by design |
| GPU particle system still costs CPU | Niagara system/emitter management overhead is paid regardless of sim target |
| HLOD proxies missing | Actors not static, or not in an HLOD layer; commandlet not re-run |
| Material change triggers a long stall | Static-switch permutation explosion — each combination is a separate shader |
| Editor frame time unlike the packaged build | PIE includes editor overhead and skips cooked paths — never quote PIE as shipping |

## 13. Where the research is thin or version-suspect

Say so rather than filling these in:

- **`[4.27-era]`**: profiling terminology, Shader Complexity guidance,
  ProfileGPU/Material Editor stats, older post-process pages (pre-TSR). Names
  and UI moved across 5.0–5.7.
- **`[community]` only**: selection outlining, custom depth, screen-space
  widget occlusion, Niagara sorting reliability.
- **Undocumented**: Nanite eligibility; ISM/HISM; runtime fit-to-view;
  multi-selection; leader lines; morph targets; touch in Enhanced Input; GTAO;
  SSR docs; per-effect GPU costs; golden-image regression; frame budgets.
- **Which release is "current" is unresolved** — 5.5, 5.6 and 5.7 all appear.
  Ask which version the caller is on.

# Babylon.js — domain to construct map

Translating an engine-agnostic design into named Babylon.js constructs, with an
evidence grade on each and the browser cost attached.

## How to read the evidence tags

- `[documented]` — named in official Babylon.js docs or typedoc.
- `[community]` — forums, DeepWiki mirrors, blogs; real, API possibly unsettled.
- `[unverified]` — practice or inference, not sourced; a hypothesis to check.

A tag is provenance, not truth. `[documented]` still drifts: the research could
not establish which release is current (9.0 was announced, with nothing ruling
out a later one), so even a documented property name is a snapshot. Verify
anything load-bearing before shipping.

---

## 1. Scene and asset structure

| Need | Construct | Tag |
|---|---|---|
| Group/parent node, no render cost | `TransformNode` | `[documented]` |
| Load without adding to the scene | `AssetContainer`, `loadAssetContainerAsync` | `[documented]` |
| Many near-identical meshes, minimum JS overhead | Thin instances (packed matrix data) | `[documented]` |
| Copies needing own lifecycle, picking or collision | Ordinary instances / `InstancedMesh` | `[documented]` |
| Spatial acceleration for visibility, picking, collision | `scene.createOrUpdateSelectionOctree()`, `OctreeSceneComponent`; submesh octree on `AbstractMesh` for meshes with many submeshes (method name not captured) | `[documented]` |
| Bounding extents for instanced-set culling/LOD | `computeMaxExtents` | `[documented]` |

`AssetContainer` is a **staging primitive, not a residency manager** — forum
threads report friction the docs do not resolve (memory retention after
clearing, several GLBs into one container, moving already-attached content in)
`[community]`. Budget lifecycle code; the container does not reclaim for you.

Babylon has **no** documented equivalent to World Partition (streaming cells),
HLOD (distant proxies), or Nanite (automatic geometric clustering). Cell
streaming, LOD tiering and hysteresis are application-level work — say so
explicitly to anyone porting an Unreal-shaped design.

Claim to avoid repeating: that thin-instance collision uses a single bounding
volume rather than per-instance tests. No source; dropped `[unverified]`.

---

## 2. Render loop, draw calls, culling

| Need | Construct | Tag |
|---|---|---|
| Draw-call diagnostic | `drawCallsCounter` (research attributes it to `EngineInstrumentation`/scene counters; which class owns it is ambiguous) | `[documented]` / name `[unverified]` |
| Separate CPU vs GPU timing | `EngineInstrumentation` — scene evaluation, render submission, GPU frame time, render-target time | `[documented]` |
| Custom pass ordering per rendering group | `Scene.setRenderingOrder` | `[documented]` |
| Transparent draw order | Sorted by `alphaIndex`, then camera distance | `[documented]` |
| Frustum culling tuning | `AbstractMesh.cullingStrategy` — bounding-sphere-only is the fast default; bbox and optimistic-inclusion variants cost more | `[documented]` |
| GPU occlusion queries | `Mesh.occlusionType` | `[documented]` |
| Many lights without per-pixel loops | Clustered lighting, both backends, contingent on float colour-buffer support | `[documented]` |
| Cut CPU submission on WebGPU | Snapshot rendering (CPU submission only, **not** GPU cost) | `[documented]` |
| Declarative pass graph | Frame graph tasks, e.g. `FrameGraphGeometryRendererTask` | `[documented]` |

Two costs that surprise people: **thin instances are not independently culled**
— if the source mesh is visible, the whole buffer draws; and **occlusion queries
are asynchronous**, typically consuming the previous frame's result, so budget a
frame of latency and the popping it causes `[documented]`.

Babylon publishes **no numeric GPU budget** for any feature — every millisecond
figure you give is synthesis, and must be labelled so.

---

## 3. Materials and shading

| Need | Construct | Tag |
|---|---|---|
| Standard PBR surface | `PBRMaterial`, `PBRMetallicRoughnessMaterial` — base colour, metallic, roughness, normal, occlusion, emissive | `[documented]` |
| Hand-written shader | `ShaderMaterial` — binds attributes (`position`, `uv`), uniforms (`worldViewProjection`), samplers; prefer the shader-store/include mechanism over large inline strings | `[documented]` |
| Artist-facing graph material | `NodeMaterial` + Node Material Editor (compiles to generated shader code) | `[documented]` |
| Per-vertex mask/scalar | `useVertexColors` / `useVertexAlpha` on the mesh | `[documented]` |
| Dense scalar field | Data texture / `RenderTargetTexture` sampled in the material | `[documented]` |
| Order-independent transparency | `scene.useOrderIndependentTransparency = true` → dual depth peeling via `scene.depthPeelingRenderer`; default passes cover ~ten layers | `[documented]` |

Colour-space discipline: base-colour textures are sRGB; roughness, metallic,
normal, AO and mask textures are linear. glTF requires base colour decoded from
sRGB to linear before shading, and packs metallic in B, roughness in G
`[documented]`. Encoding a data value into a channel the shader gamma-decodes is
a silent correctness bug, not a look bug.

**GLSL and WGSL are not interchangeable.** GLSL targets the WebGL path, WGSL the
WebGPU path; bindings, entry points and buffer layout differ `[documented]`. A
`ShaderMaterial` does not port between backends by find-and-replace.

A graph is not cheaper than hand-written code — both compile to shader code, and
instruction count, texture fetches and permutations remain the cost drivers
`[documented]`.

OIT cost: depth peeling re-renders transparent meshes multiple times, raising
CPU cost and, to a lesser degree, GPU cost `[documented]`. Order opaque →
alpha-test → blended → OIT; reach for the last only when sorting artifacts are
genuinely unacceptable.

---

## 4. Interaction and picking

| Need | Construct | Tag |
|---|---|---|
| Ray pick | `scene.pick`, `scene.pickWithRay`; results surfaced through `PointerInfo` / `PointerInfoPre` on the pointer observable | `[documented]` |
| ID/colour picking | `GPUPicker` | `[community]` |
| Declarative per-mesh triggers | `ActionManager` with `OnPickTrigger`; `OnPickDownTrigger` for down-state | `[documented]` |
| Glow highlight | `HighlightLayer` (base class `EffectLayer`) | `[documented]` |
| Editor-style outline plus selection set | `SelectionOutlineLayer` with `addSelection` / `clearSelection`; frame-graph variant `FrameGraphSelectionOutlineLayerTask` | `[documented]` |
| Transform manipulators | `GizmoManager` and the gizmo feature page (attach to meshes, bones, transform-bearing objects) | `[documented]`, property surface `[unverified]` |
| XR input | `WebXRInputSource`, `WebXRNearInteraction`, WebXR controller support | `[documented]` |
| Screen-reader accessibility | Accessibility layer generating HTML "twin" elements from `ActionManager` pick triggers | `[documented]`, thin |

`OnPickTrigger` no longer fires for a drag/swipe — use `OnPickDownTrigger` for
down-state `[documented]`. The only sourced fact about click-versus-drag here.

`GPUPicker` maturity is unsettled: an announcement thread, then a request to
expose it as a scene-level picking option, then a follow-up. It is not confirmed
as a drop-in with `scene.pick` ergonomics. Throttling readback rather than
running it per pointer-move is sound but is inference `[unverified]`. A
synchronous GPU readback stalls the main thread — always a frame-cost line item.

Unsourced though ordinary: `isPickable` filtering and bounding pre-tests as a
performance recommendation, and constrained-drag maths `[unverified]`.

---

## 5. Camera and depth

| Need | Construct | Tag |
|---|---|---|
| Orbit / turntable | `ArcRotateCamera` — `alpha`, `beta`, `radius`, `target`, elevation limits to prevent pole flips | `[documented]` |
| Fly / first-person | `UniversalCamera` (older material lists `FreeCamera` separately; unresolved) | `[documented]`, terminology unsettled |
| Third-person follow | `FollowCamera` — `radius`, `heightOffset`, `rotationOffset`; approaches its goal via acceleration and max speed | `[documented]` |
| Fit to view | `ArcRotateCamera`'s `zoomOn`-style operation (min distance at which the supplied meshes are fully visible); `FramingBehavior` for an animated fit, configurable duration, stop-on-user-zoom | `[documented]` |
| Planet-scale navigation | Geospatial camera path | `[documented]` |
| XR / stereo variants | `WebXRCamera`; `StereoscopicArcRotateCamera`, `AnaglyphArcRotateCamera`, `ArcFollowCamera`, `TouchCamera` | `[documented]` |

**Depth precision is governed by the far/near ratio, not the absolute far
distance** `[documented graphics references]`. `ArcRotateCamera` ships concrete
`minZ`/`maxZ` defaults and warns that a distant far plane causes depth fighting
because the buffer is finite `[documented]`. Raise `minZ` before lowering `maxZ`.

An opt-in logarithmic depth buffer exists for Standard Materials via a material
flag (name not captured `[unverified]`), falling back to linear when the browser
lacks the extension `[documented]`. Not a free fix — depth flickering with log
buffers is reported in the wild, and reverse depth is an open GitHub issue, not
a feature `[community]`. No Babylon FlyCamera and no minimap pattern appear in
this evidence `[unverified]`.

---

## 6. Post-processing

| Effect | Construct | Tag |
|---|---|---|
| Bloom | `DefaultRenderingPipeline.bloomEnabled` + threshold / weight / kernel / scale | `[documented]` |
| Depth of field | `DefaultRenderingPipeline.depthOfFieldEnabled` + `focusDistance`, `focalLength`, `fStop`, blur level; `LensRenderingPipeline` for lens effects | `[documented]` |
| Tone mapping / grading | `imageProcessingEnabled`, `toneMappingEnabled`, `toneMappingType` | `[documented]` |
| FXAA | `DefaultRenderingPipeline.fxaaEnabled` | `[documented]` |
| MSAA | `DefaultRenderingPipeline` option, contingent on backend/target support | `[documented]` |
| SSAO | `SSAO2RenderingPipeline` (no GTAO anywhere in the evidence) | `[documented]` |
| Screen-space reflections | `SSRRenderingPipeline` (replaced an older SSR post-process) | `[documented]` |
| Custom chain | `PostProcess` (each consumes the prior output), `PostProcessRenderPipeline` | `[documented]` |
| Motion blur | `StandardRenderingPipeline` — **deprecated** in favour of `DefaultRenderingPipeline`, whose motion blur is not documented in the evidence | `[documented]` / gap |

Gaps to state rather than paper over: **no TAA/TSR equivalent**; **no height or
volumetric fog pipeline** (scene fog modes are documented only generically); and
**no quantified GPU cost** for any effect — every cost ranking is synthesis.

---

## 7. Labels, GUI and callouts

| Need | Construct | Tag |
|---|---|---|
| Screen-space label tracking a mesh | `AdvancedDynamicTexture` fullscreen mode + `linkWithMesh`, `linkOffsetX` / `linkOffsetY` | `[documented]` |
| Leader line | GUI `Line` control with `connectedControl` | `[documented]` |
| Overlap avoidance | `moveToNonOverlappedPosition()` + `overlapGroup` — requires manual per-frame invocation, e.g. from a render observer | `[documented]` |
| Scale-independent text | MSDF text add-on (billboarded and instanced paragraphs) | `[documented]` |
| Rich HTML content in-scene | `HtmlMesh` add-on — a real scene mesh (so occludable) or an overlay; for rich content, not many simple labels | `[documented]` |
| 3D GUI containers | `Container3D` and the 3D GUI controls (billboard-mode property semantics not captured) | `[documented]` / `[unverified]` |

**Fullscreen GUI is not depth-occluded.** The only primitives are
`Mesh.occlusionType` and occlusion queries — order-dependent, needing correct
render-group and depth-state handling, not a turnkey label-occlusion system.
Babylon users repeatedly ask how to hide a linked label behind geometry and get
no first-party answer `[documented + community]`. Never claim Babylon "handles
label occlusion". No label LOD/fade thresholds or density policies are sourced
anywhere; any such table is `[unverified]`.

---

## 8. Animation

| Need | Construct | Tag |
|---|---|---|
| Keyframes | `BABYLON.Animation` — target property, `setKeys`, `ANIMATIONTYPE_*`, `ANIMATIONLOOPMODE_*` | `[documented]` |
| Easing | `setEasingFunction` with e.g. `CubicEase` and `EasingFunction.EASINGMODE_EASEINOUT` | `[documented]` |
| Timeline / coordination | `AnimationGroup` — `start`, `pause`, `restart`, `stop`, `goToFrame`, `speedRatio`; glTF imports arrive as animation groups | `[documented]` |
| Cross-fade between groups | Ramping per-group weights (`setWeightForAllAnimatables`) | `[unverified]` |
| Skeletal | `Skeleton` with bones / linked transform nodes; retargeting matches bone targets by name, morph targets by mesh and morph name | `[documented]` |
| Morph targets | Target must have exactly the same vertex count as the base mesh; final geometry sums weighted deltas; influence animates as a float | `[documented]` |
| Per-frame procedural motion | `scene.onBeforeRenderObservable` | `[documented]` |
| Baked deformation at scale | Baked vertex animation — precomputed into a texture sampled in the vertex shader, no per-instance CPU skinning | workflow `[documented]`, `BakedVertexAnimationManager` shape `[unverified]` |
| Trails / thick lines | `TrailMesh`, `GreasedLineBaseMesh` | `[documented]` |
| Frame-rate-independent timing | Delta time and a constant-delta / deterministic-step option on `Scene` | `[documented]`, exact property names `[unverified]` |

Use quaternions, not Euler angles, for rotation channels; the keyframe rate
passed to `Animation` is not the display rate — playback runs against elapsed
scene time `[documented]`. Per-object `Animation` instances do not scale to
large datasets; a shared time parameter in a shader or GPU buffer is the
alternative `[unverified — synthesis]`.

---

## 9. Particles and fluids

| System | What it is | Tag |
|---|---|---|
| `ParticleSystem` | CPU-simulated, GPU-rendered; flat property set for emission, lifetime, colour, size, gravity, direction, sprite-sheet, blend mode | `[documented]` |
| `GPUParticleSystem` | Simulation on the GPU; **narrower feature set** — no sub-emitters, restrictions on manual emission and gradients; CPU fallback when unavailable | `[documented]` |
| `SolidParticleSystem` | One mesh holding many particle instances; **no built-in emitter, recycler or physics** — you write `updateParticle` and call `setParticles()` | `[documented]` |
| Node Particle Editor | Graph-based particle authoring, newer addition | `[documented]` |
| Fluid Renderer | **Screen-space rendering**, not a solver: takes particle positions and renders depth/thickness/diffuse textures composited into a fluid surface | `[documented]` |

Stopping emission does **not** remove already-rendered particles — disposal is
required `[documented]`; a common leak.

Absent from the evidence, therefore `[unverified]` if claimed: soft particles as
a named feature (only depth-texture sampling via a custom `ShaderMaterial` and
`depthSampler`, which is `[community]`); curl noise, vector fields or SDF flow
fields; SPH or FLIP solvers; volumetric smoke; simulation-to-flipbook baking.
SPS depth sorting is `[community]` only.

---

## 10. Profiling and instrumentation

`EngineInstrumentation` and scene counters expose draw calls, active-mesh
evaluation time, render time, GPU frame time and render-target time as separate
streams `[documented]`. CPU and GPU work overlap across frames, so one FPS
number cannot identify the limiter.

Sourced browser tooling: WebGPU Inspector, Chrome DevTools tracing and
dropped-frame analysis, `about:tracing` for WebGL `[documented]`. **Spector.js
is absent from the research** `[unverified]`, and no golden-image or
perceptual-diff framework is documented for Babylon.

---

## 11. WebGL2 versus WebGPU — the capability fork

| Concern | WebGL2 | WebGPU |
|---|---|---|
| Shader language | GLSL | WGSL — different bindings, entry points, buffer layout `[documented]` |
| Baseline | UBOs, instancing, MSAA `[documented]` | Superset plus compute; a documented limitations list applies `[documented]` |
| CPU submission | No snapshot mechanism | Snapshot rendering cuts CPU submission cost, not GPU cost `[documented]` |
| Availability | Broad | Not universal — the fallback path is a product decision |

Clustered lighting is contingent on floating-point colour-buffer support on both
`[documented]`. A WGSL cache to cut load-time compilation is still an open issue
`[community]`.

---

## 12. Web-specific pitfalls

| Symptom | Likely cause |
|---|---|
| Frame spikes uncorrelated with scene complexity | GC from per-frame allocation — `new Vector3`/`Matrix` inside `onBeforeRenderObservable`. Preallocate and reuse. |
| Frame rate halves on a "good" laptop | Device pixel ratio: DPR 2 is 4x the pixels and 4x every fill-rate cost. Budget at the real DPR. |
| Stutter during interaction, GPU idle | Main-thread occupancy — scene update, app logic and render submission share one thread with the whole page. |
| Long freeze on first interaction | Shader compilation, or a synchronous GPU readback. |
| Fast for you, unusable for the user | The user's GPU is unknown, unqueryable and undriven by you. Capability-detect; never assume. |
| Blank canvas mid-session | WebGL/WebGPU context loss — real, and needs a recovery path `[unverified — not in this research]`. |
| Long time to first frame | Download and decode are separate costs — GLB payload size plus mesh-compression and texture-transcode CPU time `[unverified — Draco/KTX2 specifics are not in this research]`. |
| Data read back wrong from a texture | sRGB/linear confusion — data maps must be linear `[documented]`. |

---

## 13. Where the research is thin — highest hallucination risk

In these eight, name the class and point at its doc page — never write a method
signature:

1. **Which release is current** — unresolved; every name is a snapshot.
2. **`GPUPicker`** — forum-grade; integration level unsettled.
3. **`BakedVertexAnimationManager`** — workflow sourced, no class reference.
4. **GPU profiling** — no named timing API confirmed; `drawCallsCounter`'s
   owning class is ambiguous.
5. **Asset pipeline** — Draco, KTX2, transcoding, loader options: unsourced.
6. **Label occlusion, decluttering, legibility thresholds** — any number stated
   is invented.
7. **Particle techniques by name** — soft particles, flow fields, volumetric
   smoke, fluid solvers.
8. **Constrained-drag maths and gizmo drag semantics** — practice only.

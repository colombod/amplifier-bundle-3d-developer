# The rendering pipeline: frame cost and draw order

Reference knowledge for frame-budget reasoning. Read the evidence markers
literally: **[documented]** means an engine vendor publishes it; **[synthesis]**
means it is practitioner generalization with no engine document behind it;
**[unverified]** means the underlying source is unsettled.

## 1. The budget arithmetic

Frame interval is arithmetic, not an engine claim.

| Target | Frame interval | Notes |
|---|---|---|
| 30fps | 33.3ms | Common ceiling for heavy offline-quality visualization |
| 60fps | 16.6ms | Desktop/web default target |
| 90fps | 11.1ms | XR; per-eye work roughly doubles pixel cost at the same budget |
| 120fps | 8.3ms | High-refresh displays |

Two facts constrain how this interval is spent:

- **CPU and GPU overlap across frames; they do not sum serially.** Both
  Babylon.js and UE document CPU (game/logic/submission) and GPU (rendering)
  work as separate, pipelined streams, and both instrument them separately
  rather than reporting one FPS number. [documented]
- **No engine publishes a per-stage millisecond allocation.** There is no
  official Epic or Babylon.js document splitting 16.6ms across culling,
  geometry, shading, transparency and post. The one concrete engine-published
  GPU figure in this domain is **Lumen's approximately 4ms GPU target at 1080p
  for a 60fps console target** [documented]. Whether that 4ms includes screen
  probe gather, hardware RT and surface-cache update, or only a subset, is not
  disambiguated by the performance guide alone. [unverified scope]

Any per-stage table is therefore a target to test against captured data, not a
specification.

A workable starting split for a 16.6ms desktop frame, offered explicitly as
**[synthesis]**: visibility/culling CPU 1–2ms; draw submission CPU 2–4ms; shadow
and depth GPU 2–4ms; opaque base pass GPU 3–5ms; lighting GPU 2–4ms;
transparency GPU 1–3ms; post-process and resolve GPU 1–3ms. These are starting
hypotheses to falsify with a capture, never conclusions.

## 2. Determining bound-ness first

| Symptom | Likely bound | Confirming measurement |
|---|---|---|
| Frame time flat as resolution drops | CPU (submit or logic) | UE `stat unit`: game/draw thread ms unchanged with `r.ScreenPercentage` lowered; Babylon: render time high, GPU frame time low |
| Frame time scales with resolution | GPU fill / fragment | Halve render resolution, expect near-proportional GPU time drop |
| Draw calls in the thousands, low pixel work | CPU submit | Babylon `drawCallsCounter`; UE draw thread ms |
| GPU time dominated by one pass | GPU stage-specific | UE `stat gpu` / `ProfileGPU`; Babylon GPU frame time plus render-target time |
| High active-mesh evaluation time | CPU scene evaluation | Babylon active-meshes evaluation counter |
| Stutter, not sustained slowness | Stalls/sync, not throughput | Unreal Insights timeline showing thread stalls |

The practical rule: change one variable that only affects one stream
(resolution for GPU fill, object count for CPU submit) and see which number
moves.

## 3. Draw calls and batching

| Technique | Draw-call effect | Cost / trade-off |
|---|---|---|
| Group by shared material/texture/state | Fewer state changes and calls | Constrains authoring and material variation [documented as guidance] |
| Hardware instancing (`InstancedMesh`) | One draw call per instanced set | Instances share geometry and material; per-instance data via buffers [documented] |
| Thin instances | Lowest per-object CPU overhead at very large counts | **Not independently culled** — if the source mesh is visible, every instance in the buffer draws [documented] |
| Static merging of meshes | Large reduction | Coarsens culling: one visible fragment forces the whole merged object to render |
| Nanite (UE5, suitable static meshes) | GPU-driven cluster selection replaces per-object CPU submission | Applies to static meshes meeting Nanite's constraints; does not cover translucency [documented] |

The standing tension, both engines: **aggressive merging reduces draw calls but
weakens culling granularity.** Any batching proposal must state what it costs in
culling precision. Babylon's `drawCallsCounter` is the primary diagnostic for
whether batching work is paying off.

## 4. Render pass ordering

The conceptual order both engines describe:

1. Visibility determination: distance rejection, frustum culling, then
   occlusion.
2. Depth prepass and shadow map rendering.
3. Opaque base pass (GBuffer fill under deferred; direct shading under forward).
4. Lighting / global illumination (Lumen inserts Lumen Scene Lighting and
   Screen Probe Gather here in UE5).
5. Alpha-tested geometry.
6. Alpha-blended (translucent) geometry, sorted.
7. Post-processing, upscaling/AA (TSR in UE5), tone mapping, present.

Neither engine's documentation claims one ordering is universally correct;
both frame ordering as something to profile per render-graph configuration.
[documented] Babylon exposes per-rendering-group custom ordering through
`Scene.setRenderingOrder`.

## 5. Transparency and depth sorting

- Babylon sorts alpha-blended meshes by `alphaIndex`, then camera distance.
  Custom opaque / alpha-test / transparent sort functions are available per
  rendering group. [documented]
- Blending is documented as inherently more expensive than alpha-test or opaque
  paths; the guidance is to minimize blended surface area. [documented]
- In UE5's deferred renderer, **translucency is handled through a forward-style
  pass, not the opaque GBuffer path** — translucent materials do not receive
  the same deferred lighting treatment, regardless of whether the opaque
  geometry uses Nanite. [documented]
- Consequence for budgeting: transparency cost scales with blended pixel area
  and overdraw depth, not object count. Ten large overlapping blended quads cost
  far more than a thousand small ones. [synthesis]
- Per-object sorting cannot resolve intersecting or concave transparent
  geometry; that is a correctness limit of the technique, not a tuning problem.

## 6. Culling

| Stage | Relative cost | Notes |
|---|---|---|
| Distance / detail rejection | Cheapest | Run first |
| Frustum culling | Cheap | Babylon `AbstractMesh.cullingStrategy`: bounding-sphere-only is the fast default; bounding-box and optimistic-inclusion variants trade cost for accuracy [documented] |
| Occlusion | Most expensive | Applied after cheaper rejection [documented] |
| Backface culling | Per-triangle, GPU-side | Free-ish; correctness matters for single-sided authoring |

Babylon exposes GPU occlusion queries via `Mesh.occlusionType`. These are
**asynchronous and typically consume the previous frame's result** — budget for
one frame of latency and the popping it can produce. [documented]

UE5's Nanite performs cluster-level visibility selection on the GPU rather than
relying solely on CPU per-object culling; UE5.4 added statistics reflecting
indirect-argument results after GPU culling. [documented]

## 7. Shading path choice

| Path | Scales badly with | Scales well with | Notes |
|---|---|---|---|
| Forward | Lights per pixel, overdraw | Transparency, MSAA, low light counts | Babylon's default material path evaluates lighting during material shading [documented] |
| Clustered | Cluster-binding overhead | Many local lights | Babylon documents clustered lighting on both WebGL2 and WebGPU, contingent on floating-point color-buffer support [documented] |
| Deferred | GBuffer bandwidth, resolution, translucency | Many lights over opaque geometry | UE's supported-features-by-rendering-path doc is authoritative for what each path supports on desktop [documented] |

Lumen and Nanite **add work on top of** whichever path is selected; they do not
replace the forward/deferred decision. [documented]

## 8. Profiling: which tool answers which question

**Babylon.js** — `EngineInstrumentation` and scene counters expose
`drawCallsCounter`, active-mesh evaluation time, render time, GPU frame time and
render-target time. WebGPU-specific: snapshot rendering reduces **CPU submission
cost, not GPU cost** — do not expect it to move a fill-bound frame. WebGL2 vs
WebGPU numbers are not directly comparable without accounting for documented
backend limitations and non-compatibility-mode implications. A WGSL shader cache
to cut load-time compilation is an **open GitHub issue, not shipped documented
practice** — re-check before citing. [unverified]

**UE5** — start with `stat unit` (game/draw/GPU separation) and `stat gpu`;
escalate to Unreal Insights for CPU/GPU timeline capture including thread
stalls; `ProfileGPU` for pass-level timing; `stat tsr` for TSR cost; the ray
tracing performance guide for RT-specific cost. Note version drift: some
profiling documentation is 4.27-era — the concepts carry forward but tool names
and UI changed across 5.0–5.6. [documented]

## 9. Platform pointers

| Technique | Babylon.js | Unreal Engine 5 |
|---|---|---|
| Instancing | `InstancedMesh`, thin instances | Instanced static meshes; Nanite for suitable static geometry |
| Draw-call diagnostic | `drawCallsCounter` | `stat unit` draw thread, `stat rhi` |
| Custom pass order | `Scene.setRenderingOrder`, rendering groups | Render graph / translucency sort policy settings |
| Transparency sort | `alphaIndex` then camera distance | Forward-style translucency pass under deferred |
| Culling controls | `AbstractMesh.cullingStrategy`, `Mesh.occlusionType` | Nanite GPU cluster culling; per-object visibility settings |
| Many-light path | Clustered lighting feature | Deferred path; Lumen for GI/reflections |
| Resolution scaling | Engine hardware scaling level | Screen percentage plus TSR upscaling |
| Deep profiler | Engine instrumentation counters | Unreal Insights, `ProfileGPU` |

Exact class names, flags, console variables and call signatures belong to the
platform specialists — hand off rather than guessing.

## 10. Pitfalls

- **Optimizing a stream that was never the bottleneck.** Batching a
  fill-bound frame, or dropping resolution on a submit-bound one, moves the
  number by nothing.
- **Treating average FPS as the metric.** Frame-time percentiles expose the
  stutter that an average hides; a 60fps average with 40ms spikes fails in XR.
- **Merging geometry until culling stops working.** Draw calls fall, GPU time
  rises, and the regression is hard to attribute later.
- **Assuming thin instances are culled.** They are not; a visible source mesh
  draws the entire buffer.
- **Expecting same-frame results from occlusion queries.** They lag a frame by
  design, producing pop-in that looks like a streaming bug.
- **Blending everything "because it looks better".** Blended pixel area and
  overdraw depth, not object count, drive the cost — and sorted per-object
  transparency cannot resolve intersecting surfaces.
- **Expecting Nanite or Lumen to remove a budget line.** They shift where the
  cost sits; Lumen's own documented target is roughly 4ms of GPU at 1080p.
- **Comparing WebGPU and WebGL2 timings naively.** Different backend
  limitations and submission models make raw comparison misleading.
- **Citing a synthesized per-stage table as engine guidance.** Neither vendor
  publishes one.

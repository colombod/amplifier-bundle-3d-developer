# Auditing a frame budget: tools, evidence, and anti-patterns

Reference for reviewing a performance claim against what can be measured.

## 1. Evidence status — read this before quoting a number

The research base behind this document covers **profiling tools** thoroughly and
**budget methodology** barely at all. Preserve that distinction when reviewing:

| Area | Status |
|---|---|
| GPU/CPU profiling tools (RenderDoc, PIX, Unreal Insights + `stat`, Unity Profiler/Frame Timing Manager, Android GPU Inspector, Xcode Metal tools, Chrome/WebGPU tracing) | Vendor-documented |
| Mobile overdraw risk from masked and translucent materials; prefer-opaque guidance | Vendor-documented (Epic mobile performance docs; Android materials/shaders guide) |
| Image-comparison regression testing | Vendor-documented, but **Unity-specific** (Graphics Test Framework, `ImageComparisonSettings`) |
| sRGB conversion, blend factors, premultiplied alpha at API boundaries | Specification-level (Khronos GL/GLES/WebGL) |
| Named frame-budget methodology; per-stage millisecond allocations | **Not vendor-documented.** Every per-stage ms split in circulation is practitioner synthesis |
| A general cross-platform rendering anti-pattern checklist | **Not a documented industry artifact.** Section 5 is engineering generalization, not sourced fact |
| Graphics-specific code-review checklists; depth precision (reverse-Z, near/far); normal/tangent-space conventions | **No evidentiary support** — treat any claim as unverified |
| Spector.js, Nsight Graphics, Radeon GPU Profiler, Snapdragon Profiler, PresentMon | Absent from the evidence base. Widely referred to in practice, but **unverified here** — do not cite them as documented capability |

Practical consequence: you can almost always name a **tool and counter** with
confidence, and almost never a **vendor-blessed millisecond target**. Lean on
the former.

## 2. Which profiler answers which question

| Platform / target | Tool | What it actually answers |
|---|---|---|
| Any D3D/Vulkan/GL capture | **RenderDoc** | Per-draw-event inspection of a captured frame; Performance Counter Viewer samples GPU counters per draw. Its Python API fetches counter data programmatically, so per-draw counters can be exported for regression tracking |
| Windows / Xbox | **PIX** | GPU captures for per-draw GPU work; **timing captures** pair CPU sampling with GPU timing — the documented route for diagnosing CPU frame-time spikes |
| Unreal | **`stat` commands** | Live breakdown; the fast first pass for splitting game thread vs draw/render thread vs GPU |
| Unreal | **Unreal Insights** | Recorded cross-thread timeline; the tool for intermittent spikes and variance that a live `stat` readout averages away |
| Unity | **Profiler / GPU Usage module / Frame Timing Manager** | Per-frame CPU and GPU timing; Frame Timing Manager is the documented CPU-vs-GPU source |
| Android | **Android GPU Inspector (AGI)** | Frame trace capture and GPU analysis on device |
| iOS / macOS | **Xcode Metal capture + Metal debugger** | Capturing and debugging Metal workloads; capture can be triggered programmatically, which is how you catch a transient spike |
| Web / WebGPU | **WebGPU Inspector** extension; **Chrome DevTools** | GPU-side inspection and CPU-side profiling; DevTools dropped-frame analysis is the documented route for "frames are being missed" |
| Web / WebGL (legacy) | **`about:tracing`** | Older but documented WebGL frame profiling technique |

Selection rule: **a capture tool proves per-draw cost; a timeline tool proves
variance; a live counter proves steady state.** A spike claim backed only by a
steady-state average is unsupported.

## 3. Deciding CPU-bound vs GPU-bound

CPU and GPU work overlap across frames rather than summing serially, so a single
frame-rate number cannot name the limiter — which is why every engine's
instrumentation reports the streams separately. Procedure:

1. **Separate the three streams** — game/simulation CPU, render-submit (draw)
   CPU, GPU. Unreal reports these independently via `stat` and Insights; Unity
   via the Frame Timing Manager; PIX timing captures pair CPU sampling with GPU
   timing.
2. **The stream that tracks frame time is the limiter.** GPU time ≈ frame time
   with CPU streams well below → GPU-bound; render-submit time tracking frame
   time → submit-bound.
3. **Split GPU-bound with a resolution test.** Drop render resolution
   substantially: frame time falling roughly in proportion means
   fill/bandwidth-bound; barely moving means geometry/vertex/state-bound.
   (Generalization, not vendor text — label it as such in a review.)
4. **Split submit-bound with a draw-call test.** Frame time scaling with
   draw-call count rather than pixel work means the cost is submission.
5. **Nothing saturated, yet frames missed** → present/sync, vsync pacing, or a
   stall (synchronous readback, shader compile, allocation). Chrome DevTools
   dropped-frame analysis and PIX timing captures are the documented ways to see
   this on their platforms.

A bottleneck asserted before step 1 is a guess, however plausible.

## 4. Budget arithmetic — what is division and what is invention

Dividing the target refresh rate gives the frame interval: 16.6ms at 60fps,
11.1ms at 90fps (typical XR), 8.3ms at 120fps. That is arithmetic, safe to
assert.

Splitting that interval across stages — shadows, base pass, lighting,
transparency, post — is **not** vendor-published. No source defines an
allocation convention, a category taxonomy, or an enforcement process. So:

- A design that states its own per-stage split is doing the right thing, even
  though the split is synthesis. Audit it for **internal consistency and target
  realism**, not against an imaginary standard table.
- Never manufacture a per-stage millisecond figure and then grade the design
  against that invention. Where no sourced figure exists, name the measurement
  that would produce one.
- Budgets must be stated against a **named target device class and resolution**
  (per-eye for XR). A cost with no target is half a claim.

## 5. Anti-pattern catalog — symptom, likely cause, confirming measurement

Only the mobile overdraw and prefer-opaque entries are vendor-documented; the
rest is engineering generalization. The "confirming measurement" column is the
falsifier — it converts a suspicion into a finding.

| Symptom | Likely cause | Confirming measurement |
|---|---|---|
| Frame time scales with object count, GPU underutilised | Submission bound; too many unique material/state changes | Draw-event count in a RenderDoc capture; render-thread time in `stat`/Insights tracking frame time |
| Frame time falls sharply with resolution | Fill-rate or bandwidth bound; overdraw | Re-measure at half resolution; per-draw pixel-shader counters in RenderDoc's Performance Counter Viewer |
| Mobile-only collapse, desktop fine | Masked/translucent material overdraw — called out in Epic's mobile performance guidance; prefer opaque materials (Android guidance) | AGI frame trace; on-device shader-complexity/overdraw views |
| Periodic multi-frame spike, good average | Stall: synchronous readback, shader/PSO compile, streaming hitch, GC | Unreal Insights timeline or PIX timing capture over the spike — an averaged counter hides this by construction |
| First-run or first-look-at stutter only | Shader/pipeline-state compilation on demand | Timeline capture correlating the spike with compile events |
| Cost proportional to screen area covered by transparent geometry | Blended overdraw depth; sorted per-draw, no early-Z rejection | Overdraw visualisation; per-draw pixel counters on the transparent pass |
| Merging geometry cut draw calls but frame time got worse | Batching coarsened culling granularity — one visible fragment renders the whole merged object | Compare rendered-primitive counts before/after in a capture, not draw-call counts |
| Many small passes / render targets, each individually cheap | Per-pass fixed overhead and bandwidth (especially tiled mobile GPUs) | Pass list and per-pass GPU time in a capture; bandwidth counters where exposed |
| VRAM pressure, intermittent hitching | Render-target and texture footprint at the stated resolution/format | Memory figures per target; Unreal memory-optimisation guidance on Android |

## 6. Auditing a claimed optimisation

- **Same scene, camera path, resolution and device on both sides**, or the delta
  is unattributable.
- **Averages hide the thing that ruins the experience.** Ask for the frame-time
  distribution or a timeline capture, not mean FPS — FPS is a reciprocal and
  compresses exactly the region where the regression lives.
- **Check the counter matches the claim.** "We reduced draw calls" is confirmed
  by draw-event count, not frame time.
- **Ask whether the lever matched the bottleneck.** Resolution scaling and
  overdraw reduction only help a fill-bound frame; batching and instancing only
  a submit-bound one. A win from a mismatched lever is usually noise.

## 7. Platform pointers

Enough to hand off — not an API tutorial.

- **Unreal** — the `stat unit` / `stat gpu` / `stat scenerendering` family for
  the first-pass split; Unreal Insights for recorded timelines; on-device
  shader-complexity views for overdraw; Epic's mobile performance and debugging
  docs for the masked/translucent guidance. Exact command and cvar names belong
  to `unreal-specialist`.
- **Babylon.js / web** — engine instrumentation counters separating scene
  evaluation, render submission and GPU frame time; browser side is Chrome
  DevTools (CPU, dropped frames) plus the WebGPU Inspector extension. (Spector.js
  is common in practice for WebGL capture but unverified here.) Exact class and
  property names belong to `babylonjs-specialist`.
- **Cross-engine** — RenderDoc works at the API level, so it is the common
  denominator for per-draw evidence when engine counters are too coarse; its
  Python API supports scripted counter extraction for regression tracking.

## 8. Pitfalls when reviewing

- **Treating a missing budget as a PASS.** Silence is not evidence of low cost;
  it is absence of evidence, and it resolves late.
- **Grading against an invented millisecond table.** There is no vendor
  allocation standard. Say "unmeasured" instead.
- **Accepting FPS as the metric.** Frame time is linear; FPS is not.
- **Accepting desktop measurements for a mobile target.** Tiled GPUs and thermal
  throttling change the answer; sustained load differs from a short capture.
- **Mistaking a capture for steady state.** One captured frame proves per-draw
  cost, not typical cost.
- **Confusing abstention with a blocker.** "No performance surface" and "will
  miss frame time" are opposite verdicts; never phrase them alike.
- **Padding findings.** Speculation trains the owner to discount the anchored
  findings too.

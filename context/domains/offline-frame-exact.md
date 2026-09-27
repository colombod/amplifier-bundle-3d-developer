# Frame-exact offline rendering — seek-driven capture to video

Reference for the delivery target where the output is a **video file**, and a
renderer outside the scene decides which instant each frame shows. Real-time
advice (frame interval, variable delta, dropped frames) does not transfer
directly; the failure modes here are different and mostly silent.

## Evidence legend

Unlike the rest of this bundle, most of this doc comes from one real project
(Unfold, an LLM-driven explainer-video tool, 2026-09), not a deep-research run.

- **[MEASURED]** — measured in that project. Rig: Linux, NVIDIA T600 4 GB,
  headless Chrome, HyperFrames 0.8.33, Babylon.js 9.28 (the late-appearance
  runs in §3 and §6 used HyperFrames' own downloaded chrome-headless-shell
  152.0.7977.30, software GL). One project, one rig; quote the rig with the
  number.
- **[SOURCE]** — read in the renderer's or engine's source at the named version.
  Stronger than docs for that version, and silent about the next one.
- **[ASSESSED]** — established during that project's engine evaluation
  (install, licence and capability checks); not re-verified here.
- **[PRACTICE]** — used in that project and visually checked, not measured.
- **[RECOMMENDED]** — this doc's advice, derived from the above but not itself
  exercised in the project. Cite it as advice.

## 1. Why this is a separate delivery target

| | Real-time interactive | Offline, seek-driven |
|---|---|---|
| Who owns time | The render loop | The capture renderer, via `seek(t)` |
| Frame order | Strictly increasing | Any; frames split across independent workers |
| Budget unit | ms per frame (16.6 at 60 fps) | **seconds per frame per worker**, wall-clock per clip |
| Typical failure | Dropped frame, stutter | **Wrong** frame, seam at a chunk boundary, blank video |
| GPU | The user's, unknown | Yours — often a CPU rasterizer without you noticing (§7) |

The same discipline applies to any headless capture pipeline: Remotion
(`@remotion/three`, `useCurrentFrame`), Theatre.js (`sequence.position`), and
hand-rolled Puppeteer/CDP capture [RECOMMENDED]. Whatever supplies "the current
frame", the scene must be a function of it.

## 2. The renderer contract (HyperFrames 0.8.33 as the worked example)

[SOURCE] HyperFrames renders an HTML page in headless Chrome frame by frame:

- freezes `performance.now` and `Date.now`;
- replaces `requestAnimationFrame` with a queue flushed **once per seek**;
- dispatches a window `hf-seek` event `{time, waitUntil(promise)}` every frame;
- captures with CDP `HeadlessExperimental.beginFrame`;
- splits the frame range across **parallel, independent browser workers**,
  each seeking forward through its own chunk.

The consequence that drives everything below: a worker starting at frame 91
has never seen frames 0–90. Any state carried from a previous frame is state
that worker does not have.

## 3. The determinism rule: frame = renderAt(t)

Each frame must be a pure function of `t` (plus a fixed seed). [MEASURED] With
the checklist below, a 6 s, 180-frame 1080p30 scene — PBR node materials,
shadows, 3,000 particles, a 400-boid flock, bloom + ACES + DOF + chromatic
aberration + grain — rendered with 1 worker and with 2 workers produced
**180/180 bit-identical decoded frames** (ffmpeg `framemd5`; PSNR inf),
including chunk-boundary frames 90/91.

**That result proves determinism for that scene only.** It had no mesh that
first became visible mid-timeline. A later 2 s, 60-frame scene whose CPU-side
values were all pure functions of `t` still **diverged** between 2-worker and
1-worker renders from frame ~15–16 on (frames 0–14 identical) [MEASURED]: a
mesh hidden at start was never compiled by the readiness gate, its shader then
linked asynchronously, and the first frame it appeared in depended on
wall-clock time (§5). Items 5, 8 and 9 below fixed it; the scene now passes the
§6 test [MEASURED]. Purity of `t` on the CPU is necessary, not sufficient — **GPU
program compilation is a second clock.**

1. **No engine-owned loop.** No `runRenderLoop`; render exactly once inside the
   seek handler's `waitUntil`.
2. **Disable the engine's animation clock** and drive clips by explicit frame
   (Babylon: `scene.animationsEnabled = false`, `AnimationGroup.goToFrame`).
3. **Seeded PRNG, never `Math.random`.** mulberry32 keyed by seed + element id,
   so an element's randomness does not depend on creation order. Audit engine
   features that call `Math.random` internally (animated film grain, GPU
   particle random textures — see the Babylon platform doc).
4. **Fixed-step simulation with checkpoints.** 1/60 s steps; checkpoint every
   0.5 s; on a backward or cold seek, restore the nearest earlier checkpoint and
   replay forward. Start each integrator's clock at **its effect's own start**,
   so replay is bounded by the effect's age, not the clip's.
5. **Register the timeline only after the readiness gate** (§5), and warm that
   gate across the whole timeline, not just `t = 0`.
6. **No temporal accumulation.** TAA, motion blur, history-based shadows and
   any "previous frame" buffer are history the next worker lacks. Motion blur is
   seek-safe only by rendering `t − 1/fps` first to seed the previous matrices.
7. **Build the pipeline once; animate uniforms only.** Toggling an effect's
   enable flag rebuilds shaders mid-render. For a focus pull, keep DOF on and
   animate `fStop` (e.g. 22 → 1.4) instead of toggling DOF [SOURCE].
8. **No asynchronous shader compilation.** Where the engine links programs in
   the background (WebGL `KHR_parallel_shader_compile`, which headless Chrome
   exposes), a mesh is silently skipped until linking finishes — a wall-clock
   event. Force synchronous linking (Babylon recipe in the platform doc)
   [SOURCE 9.28], [MEASURED].
9. **Never toggle visibility for timeline-driven appearance.** A mesh that is
   not drawn is not compiled. Hide it by scaling to 0 (still drawn, so compiled
   from `t = 0`) or keep it drawn [MEASURED]. Do not fake it with alpha
   either: changing a material's transparency can select a different program
   [RECOMMENDED].

## 4. Stateless vs stateful — the split that decides the cost

The real split is **closed-form vs integrated**, not CPU vs GPU.

| Class | Examples | Seek cost | Rule |
|---|---|---|---|
| **Stateless** — closed-form `f(seed, t, path)` | stream-along-path, orbit, burst, ambient field, travelling token | O(1) per frame; any worker evaluates any frame directly | Prefer this form whenever the look allows |
| **Stateful** — an integrator | boids/flocking, advection, rigid-body physics | Replay from nearest checkpoint: ≤ 0.5 s = ≤ 30 steps at 1/60 s (arithmetic) | Fixed step + checkpoints (§3.4) |

A ballistic burst with linear drag `k` stays closed-form:
`p(t) = p0 + v0·(1 − e^(−kt))/k` plus the matching gravity term [PRACTICE]. Many
"physics" looks are closed-form once written down; reach for an integrator only
when particles genuinely interact.

## 5. The readiness gate

The first frame the renderer captures must be the fully compiled frame.
Lazily created resources — a depth renderer for DOF is created on first render
— are not covered by a naive "scene ready" check [SOURCE, Babylon 9.28]. The
Babylon procedure is in the platform doc: `whenReadyAsync(true)`, warm-render
until `isReady(true)`, then reset any render-dedupe so the renderer's own `t = 0`
seek produces frame 0.

**A readiness gate only readies what it renders.** Anything that first draws
after `t = 0` — a mesh made visible, an effect that starts late — has not been
compiled when the gate passes [MEASURED]. Warm the gate by rendering at several
probe times across the timeline (e.g. 0, 0.25, 0.5, 0.75 and 1 × duration)
until the engine reports ready, then rewind; checkpointed simulations make
those warm renders harmless [MEASURED].

**Blank video is the silent failure.** [MEASURED] A page-killing exception
before the timeline registered (calling `Engine.IsSupported()` — a property in
9.x, not a function) produced a 180-frame render that was entirely blank: a
16 KB file and one `sub_timeline_readiness_timeout` warning. The renderer
reported success. Gate every render on its output — file size, non-background
pixel coverage of a sampled frame, or per-frame hash variance [RECOMMENDED].

## 6. The standard determinism test: 1 worker vs N workers

```bash
# same clip, same machine, same GL implementation
render --workers 1 -o one.mp4
render --workers N -o many.mp4
ffmpeg -v error -i one.mp4  -f framemd5 - > one.md5
ffmpeg -v error -i many.mp4 -f framemd5 - > many.md5
diff one.md5 many.md5       # expect no output
```

[MEASURED] method; the `render` line stands for your renderer's own CLI. Look
first at the frames either side of each chunk boundary — carried state shows up
there. A divergence that starts **away** from a chunk boundary points at a
wall-clock dependency instead, typically asynchronous shader compilation.

**The scene must exercise the hazard.** Include at least one element that first
appears after `t = 0`. The measured failure surfaced at frame ~15, inside the
first worker's chunk, so any mid-timeline first appearance exercises it; a
scene where everything is visible from frame 0 cannot catch it [MEASURED — the
180/180 scene had no such element]. A pass proves determinism for
the code paths the scene exercised, not for the checklist in general.

Identical hashes also hold **within one GL implementation only**: hardware and
software GL produce different pixels [MEASURED], so a reference render is valid
only for the rasterizer that made it. The test is still far cheaper than any
golden-image harness.

## 7. Headless GPU realities

[MEASURED] on the rig above:

- Headless Chrome's default WebGL is **SwiftShader (CPU)**.
- `--use-angle=gl-egl` fell back to **Mesa llvmpipe (CPU)**.
- Vulkan and WebGPU: **no adapter**.
- Cause: NVIDIA's "headless"/compute-only driver packages
  (`nvidia-headless-no-dkms-*-server`, `libnvidia-compute`) ship **no
  `libnvidia-gl`** — no EGL vendor file, empty `/etc/vulkan/icd.d`. Hardware
  WebGL/WebGPU needs `libnvidia-gl-<ver>-server`.
- Blender Cycles (CUDA) **does** work on the compute-only stack.

**The renderer's own browser download** [MEASURED]: HyperFrames fetches its own
chrome-headless-shell and needs an unzip tool to unpack it. A host that scrubs
environment variables (as Unfold does) never passes `HYPERFRAMES_BROWSER_PATH`
through, so that override does not reach the renderer; installing the
optional npm dependency `yauzl` in the renderer backend is a working fallback
with no system change.

Before quoting any throughput, log the page's unmasked renderer string
(`WEBGL_debug_renderer_info`) and put it next to the number [RECOMMENDED]. "The box
has a GPU" is not evidence the browser is using it.

## 8. Frame-cost claim for offline work

State: seconds per frame per worker, worker count, wall-clock per clip,
resolution, rasterizer (hardware or which software GL), and the worst-case
replay length of any stateful system.

[MEASURED] software GL, 1080p, full post stack, 180 frames:

| Workers | Wall-clock | Per output frame |
|---|---|---|
| 1 | 423 s | ~2.35 s |
| 2 | 263–284 s | ~1.5 s |

Derived from those numbers: 2 workers are 1.49–1.61× faster, not 2×; each
worker slowed to ~2.9–3.2 s per frame, consistent with workers sharing one CPU
rasterizer. Do not extrapolate to N workers without measuring.

## 9. Which engine produces the clip (automated pipelines)

| Engine | Verdict | Evidence |
|---|---|---|
| Babylon.js / three.js inside a seek-driven renderer | Default for explainer 3D; deterministic as shown in §3 | [MEASURED] |
| Blender headless (`bpy`, Cycles/EEVEE) | Offline clip producer: baked geometry-node / mantaflow sims; Film → Transparent for alpha; CUDA works on compute-only drivers | [ASSESSED], CUDA [MEASURED] |
| Unreal 5 Movie Render Queue | Viable only on an 8 GB+ RTX host; licensing and install cost detailed in the Unreal platform doc | [ASSESSED] |
| Unity Recorder | Editor/Play-mode bound; no true batchmode rendering | [ASSESSED] |

Offline clips composite frame-exactly into the seek-driven page: HyperFrames
pre-extracts `<video>` frames with FFmpeg (VP9 or ProRes alpha → PNG) rather
than playing the video [SOURCE]. Check that the producer writes real alpha:
additive Niagara sprites wrote near-zero alpha, so their glow vanishes on
composite [ASSESSED]; check any producer's additive effects the same way
[RECOMMENDED].

## 10. Symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Blank video, tiny file, only a readiness-timeout warning | Page threw before the timeline registered (e.g. `Engine.IsSupported()`) |
| 1-worker and N-worker renders diverge from a chunk boundary on | State carried across frames: unreplayed integrator, `Math.random`, engine clock |
| Renders diverge from the first frame a late-appearing mesh shows, not at a chunk boundary | Its shader was never warmed and linked asynchronously; visibility toggled instead of scale 0 |
| Determinism test passes, production scenes still diverge | Test scene had nothing that first appears after `t = 0` |
| Frame 0 missing an effect the rest have | Readiness gate missed a lazily compiled shader, or a dedupe swallowed the `t = 0` seek |
| One-frame hitch or flash mid-clip | A pipeline enable flag toggled mid-render, rebuilding shaders |
| Grain differs between two renders of the same clip | Animated grain reseeding from `Math.random` |
| Animation runs at the wrong speed at 30 fps | Engine constant-delta mode fixed at 16 ms (60 fps) |
| Render far slower than planned on a GPU host | Browser on SwiftShader/llvmpipe; compute-only NVIDIA driver |
| Composited clip loses its glow | Producer's additive sprites wrote near-zero alpha (seen with Niagara) |

## 11. Platform pointers

Construct-level detail lives with the specialists: the Babylon seek-safe/ban
table, `IsSupported` and parallel-shader-compile traps, readiness procedure and
CSP notes in
`babylonjs-specialist`'s platform doc; Movie Render Queue's host, install and
licensing verdict in `unreal-specialist`'s. Stateless particle forms are in the
particles domain doc.

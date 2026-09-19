# Visual quality: legibility and visual regression

Reference knowledge for judging whether a 3D visualization reads correctly to a human
eye, and whether a regression would be detected.

Evidence status is marked throughout. **[Sourced]** = the research documents it.
**[Analytical]** = engineering generalization consistent with the evidence, not
established by it. **[Unverified]** = the research found no supporting source. Never
promote one tier to another.

## 1. The legibility checklist

Run in order. Stop and record a finding wherever a step fails; do not skip ahead.

1. **Name the intended reading in one sentence** ("the viewer should see which cluster
   is largest"). If the artifact cannot supply it, that is finding one and everything
   downstream is unjudgeable.
2. **Static or interactive?** An interactive viewer can rotate out of an occlusion; a
   static export or thumbnail cannot. Judge the worst reachable state, not the demo pose.
3. **Depth-cue inventory.** Count independent cues: occlusion order, shading / ambient
   occlusion, cast shadows, perspective size gradient, texture gradient, motion
   parallax, stereo, explicit scaffolding (reference plane, drop lines, axis box).
   Depth is reconstructed from multiple, often conflicting cues **[Sourced]**; one cue
   alone is ambiguous **[Analytical]**.
4. **Occlusion audit.** Estimate what fraction of marks is hidden. Occlusion hides data
   and privileges whatever is frontmost **[Sourced]** — in a 3D pie the foreground slice
   measurably obscures the rest **[Sourced]**.
5. **Projection versus the judgment asked.** Perspective foreshortening means equal data
   distances are not equal screen distances, a documented mechanism behind reduced
   accuracy in position and magnitude judgments **[Sourced]**. Magnitude comparison
   asked under perspective is a defect.
6. **Colormap against the shading, not on a swatch.** Shading-driven luminance must stay
   distinguishable from data-driven colour, so colormaps should hold luminance high
   enough not to be confused with shading **[Sourced]**; rainbow-style maps create false
   boundaries from uneven perceptual lightness **[Sourced]**. Check under
   colour-vision-deficiency simulation; confirm a calibrated legend **[Sourced]**.
7. **Encoding channel versus precision demanded.** Position and length on a common scale
   beat area and volume, which beat colour and shading **[Sourced]**. A precise
   comparison encoded in volume or hue is a defect even if it is pretty.
8. **Local contrast of the smallest meaningful feature** — judged against the background
   immediately behind *it*. A bright point is invisible over a bright fog band whatever
   the global histogram says.
9. **Label legibility at the stated display size** — pixel height, overlap, and whether
   each label's attachment to its referent is unambiguous.
10. **Is the effect stack subtracting information?** See section 4.

## 2. Looks wrong → likely cause

| Symptom | Likely cause | Evidence |
|---|---|---|
| Flat; cannot tell what is in front | Single depth cue; no AO, shadows or drop lines | Analytical |
| Bars/points do not compare correctly | Perspective foreshortening; gratuitous 3rd dimension on a statistical chart | Sourced |
| Foreground element dominates, rest unreadable | Occlusion privileging the frontmost mark | Sourced |
| False bands on a smooth surface | Rainbow / non-perceptually-uniform colormap | Sourced |
| Data colour vanishes in shadowed regions | Colormap luminance colliding with shading range | Sourced |
| Edges crawl or shimmer between frames | Insufficient AA; specular aliasing | Analytical |
| Whole frame washed out or crushed | Tone mapping/exposure, or sRGB applied twice — correctness-critic | Sourced (sRGB is spec-documented) |
| Transparent overlays fringe dark or bright | Straight vs premultiplied alpha — correctness-critic | Sourced |
| Text unreadable at small size | Billboarded labels without a minimum pixel size | Analytical |
| Golden test passes but the view is wrong | Tolerance too loose, or reference captured after the defect landed | Analytical |

Effect-stack symptoms — grey far field, smeared highlights, white haze — see section 4.

## 3. Golden-image and perceptual-diff testing

**What the evidence establishes.** Unity's Graphics Test Framework is a documented,
versioned image-comparison regression system with a configurable
`ImageComparisonSettings` API **[Sourced]** — the only named golden-image system in the
evidence set. No equivalent documented framework for Unreal, and no engine-agnostic
perceptual-diff standard (SSIM, ΔE), appears in the sources **[Unverified]**. The
technique is real and widely practised; its *standardization* is what is unsupported.

**What a golden-image test must name to be a test at all** (all **[Analytical]**):

| Element | Why it is load-bearing |
|---|---|
| Scene, camera pose, frame index | Without a pinned pose every run is a different image |
| Output resolution | A 1080p reference cannot validate a 4K or mobile render |
| Device / driver / API tier | The reference is only valid where it was captured |
| Comparison metric and tolerance | "Compare the images" is not a threshold |
| Determinism controls | Temporal AA off, fixed seed, fixed time step, no wall-clock, streaming/LOD settled |
| Diff artifact storage | A failure with no stored diff image cannot be triaged |

**Honest limitations — state these whenever you recommend one** (all **[Analytical]**):

- Threshold sensitivity: tight tolerance produces flake, loose tolerance silently passes
  real regressions. There is no safe default.
- GPU and driver variance changes pixels for identical inputs, so a reference from one
  vendor's driver needs a raised tolerance elsewhere — reintroducing the problem above.
- Antialiasing is not bit-deterministic across runs or hardware, and temporal methods
  (TAA, TSR, upscalers) accumulate across frames, so a capture depends on history.
  Disable temporal AA for captures or accept permanent noise.
- Stochastic content — particles, GPU noise, jittered soft shadows — needs a seeded RNG
  or it can never be compared. Async streaming and LOD mean the same frame index is a
  different image unless you wait for settle. Text rasterization differs by platform.
- A tolerance loosened to stop a flaky test is a test turned off without anyone saying
  so. Call that out by name.

**Fallbacks when a golden image is impractical** **[Analytical]**: assert cheap derived
invariants — mean-luminance or histogram bounds, a coverage mask of non-background
pixels, the pixel-space bounding box of a known marker, or a structural check on the
geometry rather than the pixels. Weaker than a golden image, far stronger than nothing,
much less flaky.

## 4. Effects that most often destroy legibility

| Effect | How it removes information |
|---|---|
| Aggressive depth of field | Deletes everything off the focal plane. In an interactive view the user picks the subject, so the blur lands on what they wanted to read. |
| Heavy bloom | Smears the small bright features that are often the data; low thresholds bleed into neighbours and destroy local contrast. |
| Fog eating the far field | Improves near-field depth reading while flattening the far field to one value; data past the fog knee is unreadable. |
| Low-contrast or rainbow colormaps | Uneven perceptual lightness manufactures boundaries and hides real ones **[Sourced]**. |
| Additive particle wash | Overlapping sprites saturate toward white; worst where density is highest, usually the interesting region. |
| Excessive SSAO | Over-darkened contact regions read as data-encoded darkness, not geometry. |
| Vignette and chromatic aberration | Add zero information; vignette suppresses the corners of the data extent. |

The rule separating defect from taste: a cinematic effect is a **legibility tax**
whenever it removes a reading the artifact claims to support. If the artifact never
claimed that reading, it is taste — advisory only.

## 5. Cross-device validation

The research documents platform *tooling* — Android GPU Inspector, Xcode's Metal capture
and debugger, RenderDoc, PIX **[Sourced]** — and Epic's mobile documentation flags
overdraw risk from masked and translucent materials, pointing to diagnostic views for
costly mobile shading **[Sourced]**.

The research does **not** document a device/API matrix methodology, a tiering practice,
or a thermal-state protocol as a named industry artifact **[Unverified]**. A tier ladder
is reasonable practice; present it as your own inferred structure, not an industry
standard. A usable minimum **[Analytical]**: one low-tier mobile device (tile-based GPU,
low bandwidth), one mid integrated GPU, one high discrete GPU, plus each target
browser/API backend on the web — with golden images captured per tier, since one
reference cannot serve all of them.

## 6. Platform pointers (handoff targets, not API instruction)

- **Babylon.js** — `DefaultRenderingPipeline` (bloom, DOF, image processing, grading),
  `SSAO2RenderingPipeline`, scene fog (`fogMode`, `fogDensity`, `fogStart`/`fogEnd`),
  `Tools.CreateScreenshotUsingRenderTarget` for a resolution-pinned capture,
  `engine.setHardwareScalingLevel` for render resolution, and the null engine for
  headless runs. Name these to the `babylonjs-specialist`; do not assert signatures.
- **Unreal Engine** — Post Process Volume (DOF, bloom, exposure, film tonemapper),
  `ExponentialHeightFog`, `r.ScreenPercentage`, anti-aliasing switches for deterministic
  captures, `HighResShot`, the automation screenshot-comparison path. Epic's profiling
  and mobile-performance docs are **[Sourced]**; the screenshot harness is a **handoff
  target for the `unreal-specialist` to confirm**, not an evidenced framework
  **[Unverified]**.

## 7. Pitfalls

- **Reviewing the hero shot.** The demo pose is chosen to look good. Judge the worst
  reachable state.
- **Judging the colormap on a legend strip.** Its failure mode appears only when
  composited against the shading on the actual surface.
- **Trusting global contrast statistics.** An excellent overall histogram can hide every
  small feature locally.
- **Accepting "it looks fine on my machine".** One device, driver, display and colour
  profile — usually the developer's high-tier one.
- **Capturing the reference after the bug shipped.** The golden image enshrines the
  defect and the test defends it forever.
- **Muting a flaky comparison instead of making the capture deterministic.** The flake
  is almost always temporal AA, an unseeded RNG, or unsettled streaming — fixable at the
  capture, not the threshold.
- **Testing only the default camera.** Regressions concentrate in states nobody
  captured: extreme zoom, empty dataset, single datum, max density, selection active.

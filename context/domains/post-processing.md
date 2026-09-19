# Post-processing and screen-space effects

Reference knowledge for whole-frame passes: what each one does, what it consumes,
what it costs, and where it belongs in the chain.

## Evidence status (read this before quoting anything below as fact)

The underlying research covers real Babylon.js classes and real Unreal Engine
post-process infrastructure (UE 4.27–5.7). Three things are **not** sourced and
must be marked as inference whenever used:

- **A canonical cross-effect pass order.** No source states one. The order below is
  standard deferred-renderer practice synthesized from constraints, not a citation.
- **GTAO as a named algorithm.** Absent from every source. Babylon documents SSAO2;
  Unreal exposes generic "ambient occlusion". Treat GTAO naming as unverified here.
- **Quantified GPU costs.** No source gives benchmark numbers. Every cost class below
  is generalization; profile before promising a millisecond figure.

Also unverified: a Babylon.js equivalent of Unreal's TSR (a genuine capability gap in
the evidence), and official Epic guidance on outlines (community sources only).

## The cost law

Post-processing is **fill-rate work proportional to output pixels**, independent of
scene complexity: the same chain costs the same on an empty scene and a dense one.

| Output | Megapixels | Relative cost |
|---|---|---|
| 1280x720 | 0.92 | 0.44x |
| 1920x1080 | 2.07 | 1.0x (baseline) |
| 2560x1440 | 3.69 | 1.78x |
| 3840x2160 | 8.29 | 4.0x |
| Stereo XR, 2x eye buffers with typical supersample | — | roughly 2.5–3x the mono equivalent |

So the first lever for an over-budget chain is *resolution*, not *removing effects*.
Half-resolution execution is standard for the low-frequency passes (AO, bloom, DOF
blur, volumetric fog) and unacceptable for the high-frequency ones (outlines, AA,
sharpening).

## Pass order (inference — see Evidence status)

Sourced constraints: Babylon post-processes chain, each consuming the previous
output; Unreal post-process materials distinguish an HDR "before tonemapper" input
from an LDR "after tonemapper" input, fixing which side of the tonemapper an effect
may live on; AA/upscaling is a set of mutually exclusive terminal choices, not
interleavable effects.

```
1.  G-buffer: depth / normals / velocity / custom depth-stencil   (scene passes)
2.  Ambient occlusion (SSAO)
3.  Screen-space reflections
4.  Fog / volumetric fog
--- scene-linear HDR through here ---
5.  Bloom extraction + blur pyramid
6.  Depth of field
7.  Motion blur
8.  Exposure / tone mapping             <-- THE BOUNDARY
--- LDR from here ---
9.  Colour grading / LUT
10. Outlines, highlights, stylized overlays
11. TAA / TSR resolve, or temporal upscale
12. FXAA and/or sharpen
13. UI / HUD / labels
```

Resolve-vs-grade order is itself a tradeoff: resolving first keeps history in a
stable colour space; grading first stops grading amplifying temporal noise.

## Buffer dependencies

| Effect | Depth | Normals | Velocity | Prev-frame colour | Custom depth/stencil |
|---|---|---|---|---|---|
| SSAO | yes | yes | — | — | — |
| SSR | yes | yes | helps | yes | — |
| Fog (depth-based) | yes | — | — | — | — |
| Volumetric fog | yes | — | — | — | — |
| DOF | yes | — | — | — | — |
| Motion blur (camera) | yes | — | preferred | — | — |
| Motion blur (per-object) | yes | — | required | — | — |
| TAA / TSR | yes | — | required | yes (history) | — |
| Outline (post route) | yes | optional | — | — | yes |
| Bloom, tone map, grade, vignette | — | — | — | — | — |

The bottom row is why bloom and tone mapping work everywhere, forward and mobile
included, while SSAO/SSR/TAA quietly impose a deferred-ish architecture.

## Effect table

Cost classes are generalization, not measured.

| Effect | Typical approach | Cost class | Main failure mode |
|---|---|---|---|
| Bloom | Threshold, downsample pyramid, blur, additive recombine | low–medium | flickers on small bright pixels; smears fine bright detail |
| Tone mapping | Curve applied to scene-linear HDR | very low | skipping it = clipping; "washed out" is bad exposure, not a missing pass |
| Colour grading | 3D LUT or curve set | very low | LUT applied in the wrong colour space |
| Vignette / chromatic aberration | Screen mask / channel offset | very low | zero information; disguises data as lens artefact |
| FXAA | Single-pass edge-detect smoothing on LDR | low | blurs text, thin lines, 1px data features |
| MSAA | Hardware multisample on geometry edges | medium, memory-heavy | deferred-unavailable; misses shading and alpha-test aliasing |
| TAA / TSR | Jittered accumulation across frames | medium (also upscales) | ghosting, smearing; needs velocity |
| SSAO | Depth+normal hemisphere sampling | medium | haloing, banding, noise without a denoise blur |
| DOF | CoC from depth, separable/bokeh blur | medium–high | foreground bleeding at the focal edge |
| Motion blur | Velocity-directed line sampling | medium | trails across occlusion boundaries |
| Fog (exponential / height) | Analytic function of depth and altitude | very low | best value-per-cycle depth cue available |
| Volumetric fog | Froxel scattering volume + raymarch | high | froxel aliasing; near-plane artefacts |
| SSR | Ray-march the depth buffer, fall back to probe | high | reflections vanish at screen edges and behind objects |
| Outline (stencil/post) | Custom depth-stencil pass, edge detect, composite | low–medium + a geometry pass | fixed pixel width makes distant objects look equally important |

## Anti-aliasing decision

Sourced: Unreal restricts **MSAA** to forward desktop/console and mobile paths (not
deferred), documents **FXAA** as unsupported on mobile-forward, and treats **TSR** as
first-class across 5.x. Babylon's `DefaultRenderingPipeline` exposes **MSAA**
(backend-dependent) and **FXAA**; no source documents a TAA or TSR switch there.

| Situation | Choose | Why |
|---|---|---|
| Deferred, has velocity, desktop | TAA or TSR | only method that antialiases shading and specular; TSR also upscales |
| Forward, CAD/technical, thin lines matter | MSAA | sharp edges, no temporal smearing or ghosting |
| Mobile or WebGL, low budget | MSAA if the backend offers it, else FXAA | one cheap pass; accept the blur |
| Stereo / XR | MSAA, forward path | temporal methods ghost under head motion |
| Static camera, screenshot quality | temporal + supersample | history converges when nothing moves |

## When a visualization reads "video-gamey"

| Symptom | Usual cause | Fix |
|---|---|---|
| Glowing haze, small features lost | bloom threshold too low / weight too high | raise threshold above the data's brightness range |
| Data off the focal plane unreadable | DOF in an interactive view | disable DOF; use AO + fog instead |
| Flat, no sense of contact | no AO | SSAO at half resolution, moderate radius |
| Dark screen edges, fringed colour | vignette + chromatic aberration | remove both |
| Ghost trails behind moving elements | bad or missing velocity | fix velocity, or switch to MSAA/FXAA |
| Text and leader lines fuzzy | AA applied over UI | composite UI after AA |
| Grey and low-contrast | wrong exposure or no tone mapping | fix exposure; do not compensate with grading |
| Reflections cut off at screen edge | SSR's inherent limitation | fall back to a probe/cubemap off-screen |

## Platform pointers (for handoff, not implementation)

**Babylon.js.** `DefaultRenderingPipeline` is the main surface: bloom
(`bloomEnabled`, threshold/weight/kernel/scale), DOF (`depthOfFieldEnabled`,
`focusDistance`, `focalLength`, `fStop`, blur level), image processing
(`imageProcessingEnabled`, `toneMappingEnabled`, `toneMappingType`, colour grading),
`fxaaEnabled`, and backend-dependent MSAA. `SSAO2RenderingPipeline` for AO;
`SSRRenderingPipeline` for reflections (it replaced an older SSR post-process).
`StandardRenderingPipeline` is deprecated and is where the documented motion blur
lived — current default-pipeline motion blur is not in the evidence. Scene fog modes
are generic; no height-fog or volumetric-fog pipeline appears. Custom chains use
`PostProcessRenderPipeline` / `PostProcess` with `RenderTargetTexture`;
`HighlightLayer` is the outline surface.

**Unreal Engine.** Post Process Volumes and `FPostProcessSettings` author DOF, bloom,
AO, exposure/tonemapper, colour grading and motion blur. Post-process materials
attach at explicit **before-tonemapper (HDR)** or **after-tonemapper (LDR)** points —
the mechanism enforcing the ordering constraint above.
`ExponentialHeightFogComponent` provides exponential height fog, with volumetric fog
enabled on that same component (albedo, scattering distribution, emissive).
AA/upscaling is a project/view setting — FXAA, MSAA, TAA, TSR — with a platform
support matrix and scalability controls. Outlines use the Custom Depth/Stencil pass
sampled from a post-process material; community-documented only, so verify against
the engine version in use.

## Pitfalls

- **Costing post at internal resolution when the display is higher.** With an
  upscaler, some passes run pre-upscale and some post-upscale; a chain quoted at one
  resolution is wrong for the other.
- **Compositing UI before AA.** Text is not geometry; FXAA and temporal methods smear
  it. UI goes last.
- **Grading before tone mapping.** Grading an HDR signal with an LDR LUT clips
  silently and is hard to diagnose after the fact.
- **Half-resolution AO without depth-aware upsampling.** A naive bilinear upsample
  produces halos along depth discontinuities that look like a bug in the AO itself.
- **Enabling SSR to fix material response.** SSR cannot show what is off-screen or
  occluded; reliable reflections are a probe/cubemap decision in the material.
- **Temporal AA without correct velocity for skinned, instanced, or GPU-displaced
  geometry.** Those objects ghost while everything else looks fine, sending debugging
  in the wrong direction.
- **Per-view post in a multi-viewport tool.** Four viewports with the same chain is
  four times the fill rate; share passes or degrade inactive views.

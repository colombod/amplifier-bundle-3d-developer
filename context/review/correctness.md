# Numerical and rendering-correctness pitfalls

Reference for auditing a rendering artifact for demonstrable wrongness. The
organising idea: the dangerous defects do not crash and do not look broken — they
look like a decision somebody made.

## Evidence standing (read before asserting anything)

| Area | Standing |
|---|---|
| sRGB texture and framebuffer conversion | **Specification-level.** Khronos required-image-format docs and the OpenGL ES 3.0 / 3.2 specs define sRGB-to-linear on sample, linear-to-sRGB on write. Assert flatly. |
| Blend factors, straight vs premultiplied alpha | **Specification-level.** `glBlendFunc` refpages (ES 3.0/3.1) and the Khronos "Blending" article define factor mechanics; the WebGL 1.0 spec defines premultiply/unpremultiply conversion at canvas and texture-upload boundaries. |
| Depth precision, reverse-Z, near plane | **Not sourced.** Widely used practice, no usable documentation in this lens's research. Label findings engineering judgment. |
| Normal/tangent space, handedness, TBN | **Not sourced at all.** The mathematics (short normal gives smaller N·L; negative determinant flips winding) is assertable as mathematics. Which convention a given engine or format uses must be verified in its docs, never asserted from memory. |
| Graphics code-review checklists | **Not sourced.** No documented industry artifact; the checklist below is constructed for this lens. Cite it as nothing. |
| Image-comparison regression testing | **Documented, Unity-specific.** Unity's Graphics Test Framework and `ImageComparisonSettings` give versioned golden-image comparison with configurable thresholds. No equivalent for other engines appears in the research. |

## The pitfall catalog

| Pitfall | How it looks | Why it is silent | The check that catches it |
|---|---|---|---|
| **Colour texture not flagged sRGB** | Mid-tones washed out; artists compensate with extra saturation and light intensity | Nothing errors; the compensation becomes the look, and later assets are matched to it | Inspect the real format in a frame capture, not the import dialog. Mid-grey 0.5 sRGB must sample to ~0.216 linear |
| **Data texture flagged sRGB** (normal, roughness, metallic, AO, mask, height) | Roughness response skewed dark, normal maps too weak or strong, AO too heavy | The map still works, values just shift smoothly — reads as authoring choice | Same capture check, inverted: data must be linear/UNORM. A flat normal-map region must decode to (0,0,1) |
| **Blending in gamma space** (blend on a non-sRGB target holding encoded values, or compositing after encode) | Cross-fades and overlays muddy through the middle; 50/50 saturated complements go grey | Shows only where operands differ strongly, and only mid-range; on greys it is invisible | Composite red over green at 50%: linear-correct gives a distinctly brighter mid. Confirm the target format is sRGB-capable or the values linear |
| **Tone mapping in the wrong place** (bloom/DOF/motion blur after the tonemapper, or tonemapping twice) | Highlights clip flat, bloom fails on bright sources, grading fights itself | The maths still runs, on already-compressed values — reads as "a subdued grade" | Mark the tonemapper boundary: radiance operations before, display-value operations (vignette, UI, outlines) after |
| **Mips or filtering in the wrong space** (averaging sRGB-encoded texels; bilinear over packed normals, ID buffers, encoded depth) | Textures darken as they minify; seams and spurious values at texel boundaries | Darkening tracks distance so it reads as aerial perspective; filtering always yields *a* value, wrong only at edges | Sample a flat 50%-grey sRGB texture at mip 0 and a high mip — equal expected, a drop means encoded-space averaging. Data that is not linearly meaningful needs point sampling or decode-before-interpolate |
| **Premultiplied vs straight alpha mismatch** | Signature: **dark fringe / black halo** on cut-out edges (foliage, sprites, UI, text). Reverse mismatch gives a bright halo | Reads as a deliberate outline or rim, and at small sizes is barely perceptible | Match factors to the data: premultiplied blends `ONE, ONE_MINUS_SRC_ALPHA`; straight blends `SRC_ALPHA, ONE_MINUS_SRC_ALPHA`. Check every boundary where the convention can flip — texture upload, canvas compositing, readback, library defaults |
| **Colour bleed from transparent texels** | Fringe appears only after mipping or filtering | Transparent texels carry colour nobody inspects until a filter mixes it in | Verify they were flood-filled/dilated, or the asset premultiplied before mipping |
| **Depth precision: near plane far too close** | Z-fighting shimmer on near-coplanar surfaces, worse with distance; flickering decals and overlays | Reads as an aliasing artifact, and it moves with the camera so it is dismissed as transient | Compute far/near; raise the near plane to the largest value the content tolerates before adding depth bits. If that fixes the flicker it was precision; if the flicker is geometry-locked at any range the surfaces are truly coplanar and need an offset, stencil or merged mesh. *(Engineering judgment — unsourced here)* |
| **Normal not renormalized after interpolation, skinning or morphing** | Dark, soft shading in triangle interiors, worst on low-poly curves; deforming meshes darken in bent regions as they move | N·L with a short N is simply smaller — dimmer but perfectly smooth, so it reads as "soft lighting" or as a lighting response to the pose | `normalize()` in the fragment stage on every interpolated normal, tangent and bitangent, and on skinning/morph output. Neither a lerp nor a weighted sum of unit vectors is unit |
| **Normal not transformed by the inverse-transpose** | Lighting skews on any non-uniformly scaled object | Still lit, just lit as a different shape | Non-uniform scale anywhere in the chain requires the inverse-transpose. Counter-case: scale one axis by 3 |
| **Tangent handedness / green-channel mismatch** | Detail reads inverted — bumps become dents — under some light directions | The map still produces detail, and an inverted bevel is often accepted as design | Light a known-convex feature from a known direction and check it reads convex. Verify the engine's convention against the exporter's, in their docs |
| **Winding reversed / negative-determinant transform** (mirrored instance, negative scale) | Faces vanish under backface culling, or interiors show through; mirrored copies break while the original is fine | With culling off it renders plausibly while the normals point inward | Check the world-matrix determinant: negative flips winding, so cull mode or normals must flip with it. Counter-case: mirror a working asset on one axis |
| **Unit/scale mismatch at import** (cm vs m, DCC unit vs world unit) | Attenuation, fog and DOF ranges unusable; shadow bias needs absurd values; physics feels wrong | Every dial can be re-tuned, each compensation individually reasonable | Measure a known object in world units against its real dimension. A scene needing odd attenuation, fog and bias is a scale mismatch until proven otherwise |
| **Coordinate/handedness mismatch** (Y-up vs Z-up, left- vs right-handed, UV origin) | Assets arrive rotated 90 degrees or mirrored | A per-instance corrective rotation fixes one asset and breaks the next | Name both conventions and the single conversion point. Per-asset corrections signal a missing global conversion |
| **Quantity-to-geometry encoding error — the data lie** | A bar, radius or offset that no longer matches the number it encodes: nonzero baseline, value mapped to radius while the eye reads area, log data on a linear axis, perspective making equal quantities different sizes | The chart is beautiful and internally consistent; nothing renders wrong. The *claim about the world* is false | Measure three data points' rendered extents and compare the ratios to the source numbers; check axis and legend match the actual mapping |

## Silent vs visible — the ranking rule

Rank silent above visible, always.

- **Silent** (wrong values, plausible image): sRGB flag errors, gamma-space
  blending, mips in the wrong space, unnormalized normals, missing
  inverse-transpose, unit/scale mismatch, quantity-to-geometry errors. These get
  compensated for, and the compensation entrenches them.
- **Visible** (artifacts): black fringes, z-fighting shimmer, missing faces,
  inverted bumps, seams. Someone eventually reports these.

## Graphics code-review checklist

Constructed for this lens; the research documents no industry-standard graphics
code-review checklist. Use it as a coverage list, cite it as nothing.

1. **Colour** — every texture classified colour (sRGB) or data (linear); working
   space named; tonemapper boundary marked; blends on linear values.
2. **Alpha** — convention declared per asset and boundary; blend factors match it;
   transparent texels dilated before mipping.
3. **Depth** — near/far and their ratio stated; buffer format stated; coplanar
   surfaces have an explicit offset or stencil strategy.
4. **Normals** — `normalize()` after interpolation, skinning and morphing;
   inverse-transpose under non-uniform scale; tangent convention matched to the
   authoring tool.
5. **Orientation** — winding and cull mode consistent under mirrored/negative
   transforms; determinant sign checked.
6. **Units** — source unit, conversion factor, world unit and handedness stated
   once, globally; no per-asset corrective transforms.
7. **Encoding fidelity** (visualization only) — sampled points' rendered extents
   match their source ratios; axis and legend match the actual mapping.
8. **Regression guard** — is there a golden-image or value-level test for each
   item, or does it rely on someone noticing?

## Platform pointers

Enough to hand off; confirm names with the platform specialist.

- **Babylon.js / WebGL / WebGPU** — sRGB handling is per-texture (colour flagged,
  data not). The WebGL 1.0 spec defines `UNPACK_PREMULTIPLY_ALPHA_WEBGL` and the
  canvas `premultipliedAlpha` attribute: the two places the alpha convention
  silently flips. Camera `minZ`/`maxZ` set the depth range. Inspect real state
  with WebGPU Inspector or Chrome DevTools tracing, not declared intent.
- **Unreal Engine** — texture compression settings and the sRGB checkbox separate
  colour from data; post-process materials expose explicit before- and
  after-tonemapper placement, where chain-order correctness is decided. Confirm
  real state with Unreal Insights, `stat` commands, RenderDoc (including the Meta
  fork for Quest), PIX, Android GPU Inspector, or Xcode's Metal debugger.
- **Regression tooling** — Unity's Graphics Test Framework with
  `ImageComparisonSettings` is the one documented golden-image system here. For
  other engines, treat an equivalent as a gap to name, not a capability to assume.

## Pitfalls in the review itself

- **Accepting intent as state.** "We use a linear workflow" is a sentence, not a
  texture flag. Anchor to the declaration or capture.
- **Asserting an unsourced convention.** Tangent handedness and depth-precision
  practice are undocumented here — label them judgment or verify in engine docs.
- **Reporting taste as error.** If it cannot be phrased "computes X, correct is Y",
  it belongs to the visual-quality critic.
- **A finding with no counter-case.** Unreproducible means indefensible. Downgrade it.
- **Confusing the two z-fighting causes.** Precision is distance-driven and moves
  with the camera; true coplanarity is geometry-locked. Different fixes.
- **Treating an abstention as a blocker.** No correctness surface is a real
  outcome, not a failure.

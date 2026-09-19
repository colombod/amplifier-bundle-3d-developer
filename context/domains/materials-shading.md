# Materials and shading — domain reference

Evidence grade: **medium confidence**. Engine facts below are documented in
Babylon.js and Unreal sources; anything marked *(analytical)* is generalization
or arithmetic, not a sourced engine claim. Version caveats: the research could
not resolve which Babylon.js release is current (9.0 was announced), nor whether
Unreal 5.6 or 5.7 is — both are documented. Several profiling claims trace to
Unreal **4.27** docs; continuity into 5.x is *not* source-confirmed. Never quote
a version-specific behaviour as settled.

## 1. PBR parameter conventions

| Aspect | Babylon.js | Unreal Engine |
|---|---|---|
| Model | `PBRMaterial` / `PBRMetallicRoughnessMaterial`: base color, metallic, roughness, normal, occlusion, emissive | Standard Material inputs: Base Color, Metallic, Specular, Roughness, Normal, AO, Emissive, Opacity |
| Base color | Dielectric diffuse colour; for metals it carries reflectance/F0 colour | Diffuse portion of reflected light, *excluding* specular |
| Metallic | Scalar `[0,1]`; ~0 dielectric, ~1 metal | Same `[0,1]` convention |
| Roughness | `0` smooth → `1` rough | Same convention |
| Opacity | Meaning depends on blend mode | **Opacity** = fractional transparency; **Opacity Mask** = binary discard |

**Metallic-roughness vs specular-glossiness.** Metallic-roughness is the glTF
interchange convention and the default in both engines' documented inputs. Pick
specular-glossiness only when an asset library forces it; it costs a channel and
does not interchange cleanly *(analytical)*.

**Colour space is a correctness rule, not a preference.** Base-colour textures
are sRGB and must be decoded to linear before shading math (glTF requires this).
Roughness, metallic, AO, masks and any data channel are **linear**. A data value
sampled through an sRGB-decoding path is silently non-linear — the most common
data-viz shading bug.

**glTF channel packing:** occlusion/roughness/metallic packed as R = AO,
**G = roughness, B = metallic**. Follow it when assets cross between engines.

**Do not multiply unrelated effects into Base Color / Metallic / Roughness
merely because the input exists.** Opacity, Emissive or a dedicated channel is
more honest.

## 2. Authoring path — decision table

| Situation | Path | Why |
|---|---|---|
| Standard lit surface, artist-tunable | Stock PBR material + parameters | Cheapest to maintain, no permutation risk |
| Family of surfaces differing only in values/textures | **Material instances** of one parent | Reuses the compiled shader |
| Reusable graph logic across materials | Babylon node-material sub-graphs / Unreal Material Functions | One definition, many consumers |
| Compact specialized math or sampling | Custom HLSL node (Unreal) | Documented guidance: keep Custom nodes small and contained |
| Custom BRDF, procedural surface, full-screen effect, simulation-style data processing | Babylon `ShaderMaterial` (hand-written GLSL/WGSL) | Generated-code control; the case node graphs cannot express |
| Feature that must be *compiled out* entirely | Static switch / static parameter | But every combination is a new permutation |

A graph is **not** automatically cheaper than hand-written code — it compiles to
shader code, so instruction count, texture fetches and permutation count remain
the cost drivers regardless of authoring method.

GLSL (WebGL) and WGSL (WebGPU) are **not** interchangeable syntaxes for the same
binary: bindings, entry points and buffer layouts differ. Use shader-store /
include mechanisms rather than large inline shader strings.

## 3. Instancing and permutation explosion

- **Instance parameter** (scalar, vector, texture override) → same compiled
  shader, new constant data. Effectively free at compile time.
- **Static parameter / static switch** → a *new compiled permutation*. Unreal's
  documentation explicitly recommends minimizing unused static combinations.
- Permutations multiply: *n* independent static switches → up to 2ⁿ variants
  *(analytical — the multiplication is arithmetic, the recommendation to
  minimize them is sourced)*. Ten switches is a thousand shaders to compile,
  ship and keep in memory.

Budget rule *(analytical)*: treat permutation count as a line item alongside
VRAM — it appears in no frame graph, so it gets discovered at package time.

## 4. Data-driven materials — encoding non-visual values

Governing rule from the evidence: **encode according to the data's semantics,
not according to whichever material input is convenient.**

| Data | Babylon.js mechanism | Unreal mechanism |
|---|---|---|
| Per-vertex mask / weight | Vertex colour / vertex alpha (`useVertexColors` / `useVertexAlpha`) | Vertex Color input |
| Dense scalar field | Data texture or render-target texture sampled by the material | Texture, virtual texture, render target |
| Packed masks | Channel packing (metallic/roughness-style convention) | Virtual texturing supports packed mask channels alongside Base Color/Normal/Roughness/Specular |

**Categorical IDs: unsupported-by-name, supported-by-mechanism.** No source
documents a dedicated "ID material input" in either engine. The defensible path
is to treat IDs as a data-texture or vertex-attribute encoding problem under the
mask discipline: linear format, no sRGB, nearest sampling, no unintended
interpolation. Any stronger claim about an engine-blessed ID system is
unverified — route it to the platform specialist.

Encoding checklist *(analytical)*: nearest filtering and no mipmaps for
categorical data (mip averaging fabricates categories that do not exist);
linear or unsigned-integer formats; vertex attributes interpolate — fine for
continuous fields, wrong for categories.

## 5. Transparency

Ascending cost order, and the documented practice:

1. **Opaque** — always first choice.
2. **Masked / alpha-tested cutout** — binary discard (Unreal: Opacity Mask).
   Order-independent by construction, no blending.
3. **Blended translucency** — Unreal's documented translucent blend is standard
   source-over: `source·opacity + destination·(1−opacity)`, which is inherently
   order-sensitive. Unreal exposes **Translucency Sort Priority** to manage draw
   order.
4. **Order-independent transparency** — reserve for cases where sorting
   artifacts are unacceptable.

**The cross-engine asymmetry is real.** Babylon.js documents a scene-level
built-in: `scene.useOrderIndependentTransparency = true` enables a
dual-depth-peeling implementation (`scene.depthPeelingRenderer`), with a default
pass count sufficient for roughly **ten transparency layers**. Babylon's docs
warn this **re-renders transparent meshes multiple times**, raising CPU cost and,
less so, GPU cost. The Unreal sources describe **no equivalent universal OIT
toggle** — only sort priority and masked cutouts. Assuming "enable OIT" is
portable overstates Unreal's documented capability.

## 6. Cost ordering and profiling

Optimize in this order (cross-engine, supported by the evidence):

1. Unnecessary passes, overdraw, and translucent screen coverage
2. Redundant texture samples
3. Permutation count
4. Instruction-level arithmetic

Bandwidth, overdraw and pass count typically dominate instruction cost.

**Unreal tooling (documented, but version-mixed):** Shader Complexity view mode
visualizes per-pixel instruction cost; Epic's own guidance calls it an
*approximation* and notes translucency cost also depends on scene overdraw
(4.27-sourced). `ProfileGPU` plus Material Editor statistics are recommended
alongside it (also 4.27-sourced). Continuity into 5.6/5.7 is not confirmed.

**Babylon tooling:** the sources describe **no unified profiling suite** and
**no named GPU-timing API** — only inspecting generated shader source and the
render-target / frame-graph pass infrastructure. Do not assert a named Babylon
profiling command.

VRAM arithmetic *(analytical)*:
`bytes ≈ width × height × bytes-per-pixel × 1.33 (full mip chain) × count`.
A 2048² RGBA8 texture ≈ 22 MB with mips; the same at BC/ASTC compression is
roughly a quarter to an eighth of that. Multiply by the number of distinct
materials — instancing shares shaders, not textures.

## 7. Symptom → likely cause

| Symptom | Likely cause |
|---|---|
| Transparent surfaces pop/swap as camera orbits | Per-object sort with mutually overlapping geometry; needs sort priority, masked cutout, or OIT |
| Data values look washed out or gamma-bent | Data texture flagged sRGB, or a value routed through Base Color |
| Categories bleed into each other at distance | Mipmapping or linear filtering on an ID/category texture |
| Metal looks like grey plastic | Metallic near 1 with no environment/IBL to reflect |
| Everything looks correct but the frame collapses when zoomed in | Translucent screen coverage / overdraw, not instruction cost |
| Package or cook time exploding; shader memory high | Static-switch permutation multiplication |
| Edges shimmer on cutout foliage-style assets | Alpha-tested cutout without appropriate coverage/AA handling |
| Shader Complexity view looks green but it is still slow | Complexity view does not account for overdraw |

## 8. Platform pointers (handoff, not tutorial)

**Babylon.js:** `PBRMaterial` / `PBRMetallicRoughnessMaterial` (standard PBR) ·
`NodeMaterial` + Node Material Editor (graph authoring, compiles to generated
shader code) · `ShaderMaterial` (hand-written GLSL/WGSL; binds attributes like
`position`/`uv`, uniforms like `worldViewProjection`, and samplers) · shader
store / include mechanisms · `useVertexColors` / `useVertexAlpha` on meshes ·
render-target textures and frame-graph tasks for data passes ·
`scene.useOrderIndependentTransparency` + `scene.depthPeelingRenderer`.

**Unreal Engine:** Material Editor + Material Expressions into the Main Material
node · generated HLSL via Window → Shader Code → HLSL Code · Material Functions
(reusable logic) · Material Instances (parameter override) · static parameters /
static switches (permutations) · Custom Material Expression (arbitrary HLSL,
named inputs, declared output type) · Opacity vs Opacity Mask · Translucency
Sort Priority · Virtual Texturing (packed channels) · Shader Complexity view
mode · `ProfileGPU`.

## 9. Pitfalls

- Marking a data/mask texture as sRGB — silently non-linear values.
- Encoding a categorical ID with linear filtering or mipmaps, producing
  in-between categories that do not exist.
- Assuming an engine-agnostic design can "just turn on OIT" — Unreal has no
  documented universal toggle.
- Enabling Babylon's OIT without budgeting the multi-pass re-render of every
  transparent mesh (CPU cost, documented) or checking the ~10-layer default.
- Using a static switch where an instance parameter would do, multiplying
  compiled permutations for no runtime gain.
- Replacing a material graph with one giant Custom HLSL block, defeating
  inspection and permutation reasoning.
- Treating GLSL and WGSL as the same shader with different spelling.
- Reading Shader Complexity as total cost rather than a per-pixel instruction
  approximation, and missing overdraw entirely.
- Counting texture memory without the ~1.33× mip factor, or assuming material
  instances save texture memory the way they save shaders — they do not.

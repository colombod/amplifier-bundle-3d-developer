# Research: materials-shading

- run id: `dr-cbdd8ca8`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 73

## Confidence note (verbatim from the run's brief)

Medium confidence. The named subjects — Babylon's Node Material Editor, ShaderMaterial, PBRMaterial, GLSL/WGSL, and Unreal's Material Editor, Material Instances, shader permutations, Custom HLSL nodes — are all directly documented in the provided sources, so the core practices below are genuinely supported, not adjacent guesses. Confidence is capped at medium for two reasons: the sources disagree on which release is actually 'current' (Babylon 9.0 announced but not clearly the latest; UE 5.6 and 5.7 release notes both present with no resolution), and several performance/profiling claims trace to Unreal Engine 4.27 documentation rather than confirmed 5.6/5.7 pages. Babylon documents a built-in, scene-level OIT technique (dual depth peeling); Unreal's documented translucency pipeline has no equivalent universal built-in OIT — this is a real cross-engine asymmetry, not a smoothed-over detail. Every citation below traces to a source in the supplied list; no claim is attached to an id outside it.

## Report

# As of the current stable releases of Babylon.js and Unreal Engine, what are the documented and practitioner-recommended best practices for real-time materials and shading, specifically: (1) physically based rendering parameter conventions, (2) custom shader authoring workflows, (3) node-based/shader-graph material authoring, (4) data-driven materials for encoding non-visual values (e.g. IDs, scalar fields, masks), (5) transparency handling including order-independent transparency (OIT) techniques, and (6) shader performance costing/profiling practices — covering Babylon.js's Node Material Editor, ShaderMaterial, PBRMaterial, and WGSL/GLSL shader code, and Unreal Engine's Material Editor, Material Instances, shader permutations, and Custom HLSL nodes?

## 1. What the evidence actually covers, and a version-currency caveat

The question names specific, checkable things — Babylon.js's Node Material Editor, `ShaderMaterial`, `PBRMaterial`, WGSL/GLSL shader code; Unreal's Material Editor, Material Instances, shader permutations, and Custom HLSL nodes. All of these appear as dedicated subjects in the provided sources: `ShaderMaterial` [s2][s9][s59], `PBRMaterial` [s5][s11][s54][s62][s63], `NodeMaterial`/Node Material Editor [s12][s14][s15], Unreal Material Editor concepts and generated HLSL [s19][s25], Material Instances and instancing/permutation behavior [s23][s26], and Custom Material Expressions/Custom HLSL [s16][s24][s27]. The subject matter is genuinely present in the evidence, not merely adjacent to it.

What is **not** cleanly settled by the evidence is which release each engine's "current stable" actually is. Babylon.js 9.0 was announced in the sources [s1][s8], but the source list gives no signal that a later Babylon release has since superseded it, nor confirmation that 9.0 is still current at the time of writing. For Unreal, the sources contain documentation for 5.5 [s36][s37], 5.6 (the bulk of the material-system pages: [s16][s19][s23][s25][s26][s33][s38][s39][s42][s44][s60][s63][s64][s66]), and 5.7 release notes in multiple locales and outlets [s17][s18][s20][s28][s30]. The evidence does not establish which of 5.6 or 5.7 was the current stable release at the time in question — both are represented. This is treated in Section 8 as an open disagreement rather than resolved silently.

Separately, a meaningful share of the performance/profiling guidance is sourced to Unreal Engine **4.27** documentation [s29][s32][s35][s41][s45], not 5.x. The concepts (ProfileGPU, Shader Complexity view mode, Material Editor stats) are longstanding and appear consistent with the 5.6 pages that do exist [s33][s38][s44], but no source in the list confirms these tools are unchanged in 5.6/5.7. This is flagged, not smoothed over.

## 2. Physically based rendering parameter conventions

| Aspect | Babylon.js | Unreal Engine |
|---|---|---|
| Model | `PBRMaterial` / `PBRMetallicRoughnessMaterial` with base color, metallic, roughness, normal, occlusion, emissive [s63] | Standard Material inputs: Base Color, Metallic, Specular, Roughness, Normal, AO, Emissive, Opacity [s65] |
| Base color | Dielectric diffuse color; for metals, reflectance/F0 color [s63] | Diffuse portion of reflected light, excluding specular [s68] |
| Metallic | Scalar `[0,1]`, near 0 for dielectrics, near 1 for metals [s63][s65] | Same `[0,1]` convention [s65] |
| Roughness | `0` smooth, `1` rough; metallic/roughness packing uses B for metallic, G for roughness [s63] | Same `0`–`1` smooth-to-rough convention [s65][s67] |
| Color space | Base-color textures are sRGB; roughness/metallic/normal/AO/masks are linear; glTF requires base-color decoded from sRGB to linear before shading [s66] | Same general convention (color textures sRGB, data textures linear), consistent with Unreal's material-input documentation [s65] |
| Opacity vs. Opacity Mask | Alpha channel meaning depends on selected material/blend mode | Opacity is fractional transparency; Opacity Mask is a binary discard [s39] |

Practical guidance drawn from the sources: keep scalar/data maps (roughness, metallic, AO, masks, IDs) in linear-encoded formats, decode base-color textures from sRGB before lighting math [s66], follow glTF's metallic(B)/roughness(G) packing convention when moving assets between the two engines [s63][s73], and avoid multiplying unrelated effects into Base Color/Metallic/Roughness simply because the input exists — Opacity, Emissive, or a dedicated channel is more honest to the data.

## 3. Custom shader authoring workflows

**Babylon.js.** `ShaderMaterial` is the direct programmable path: it binds attributes (e.g., `position`, `uv`), uniforms (e.g., `worldViewProjection`), and samplers to hand-written vertex/fragment code [s59][s9][s2]. Recommended use cases drawn from the documentation set: full-screen effects, procedural surfaces, custom BRDFs, and cases the standard pipeline can't express, using the shader-store/include mechanisms rather than large inline strings [s10][s58]. GLSL targets WebGL paths; WGSL targets WebGPU-compatible paths — the sources do not support treating these as interchangeable syntaxes for the same binary; bindings, entry points, and buffer layout differ.

**Unreal Engine.** The Material Editor is the normal path, connecting Material Expressions to the Main Material node, with generated HLSL viewable via Window → Shader Code → HLSL Code [s19]. Custom Material Expression nodes allow arbitrary HLSL with named inputs and a declared output type [s16][s24][s27]. The documented guidance favors using Custom nodes for compact, well-contained logic (specialized sampling, small math functions) rather than as a substitute for the whole material graph, because large Custom blocks are harder to inspect, permutation-reason about, and review than native graph logic [s16].

## 4. Node-based / shader-graph material authoring

Babylon's `NodeMaterial` is documented as a material assembled from shader blocks into generated code [s14], with editor tooling described in the Node Material history and community documentation [s12][s15]. It is suited to artist-facing parameterization, procedural masks, and reusable graph sections, while `ShaderMaterial` remains preferable when generated-code control or simulation-style data processing is required [s10].

Unreal's Material Editor plays the equivalent role: parameters are exposed for override, Material Functions hold reusable logic, and static switches are reserved for features that should be compiled out entirely [s19][s25]. In both engines, a graph is not automatically cheaper than hand-written code — it is compiled into shader code, so instruction count, texture fetches, and permutation count remain the actual cost drivers regardless of authoring method.

## 5. Data-driven materials for non-visual values (IDs, scalar fields, masks)

The evidence supports a single governing rule: encode a value according to its data semantics, not according to whichever material input is convenient.

| Data type | Babylon.js mechanism (from sources) | Unreal Engine mechanism (from sources) |
|---|---|---|
| Per-vertex mask | Vertex color / vertex alpha, documented via `useVertexColors`/`useVertexAlpha` on meshes [s56] | Vertex Color inputs [s39] |
| Dense scalar field | Data/render-target texture, sampled by a material [s51][s55] | Texture, virtual texture, render target [s52][s60] |
| Packed masks | Metallic/roughness-style channel packing conventions [s63] | Virtual texturing supports packed mask channels alongside Base Color/Normal/Roughness/Specular [s52] |

For categorical IDs specifically, none of the provided sources document a dedicated "ID material input" in either engine. The safest documented path is to treat IDs as a data-texture or vertex-attribute encoding problem using the same linear-format, no-sRGB, no-unintended-interpolation discipline that governs masks and scalar fields — but this is inference from adjacent documented mechanisms (vertex attributes, data textures, packed channels), not a source describing an "ID system" by name. That should be read as unsupported-by-name, supported-by-mechanism.

## 6. Transparency handling and order-independent transparency

**Babylon.js** documents a scene-level, built-in OIT feature: setting `scene.useOrderIndependentTransparency = true` enables a dual-depth-peeling implementation (`scene.depthPeelingRenderer`), with a default pass count sufficient for roughly ten transparency layers; Babylon's own documentation warns this re-renders transparent meshes multiple times, raising CPU cost and, to a lesser degree, GPU cost [s31].

**Unreal Engine's** documented translucent blend is the standard source-over compositing equation (source·opacity + destination·(1−opacity)) [s44], which is inherently order-sensitive. Unreal exposes **Translucency Sort Priority** to manage draw order [s42], and Masked blend mode with Opacity Mask for binary cutouts [s39]. None of the provided Unreal sources describe a built-in, general-purpose OIT toggle equivalent to Babylon's `useOrderIndependentTransparency`.

General practice supported across both engines: prefer opaque, then masked/alpha-tested cutouts, then ordinary blended translucency, and reserve full OIT for cases where visible sorting artifacts are unacceptable, because every documented OIT-adjacent technique adds passes, re-renders, or sorting overhead.

## 7. Shader performance costing and profiling

**Babylon.js.** The sources do not describe a single unified profiling suite; the closest documented mechanisms are inspecting generated shader source (Node Material / custom shaders) and Babylon's own render-target/frame-graph and pass infrastructure [s43][s51]. No source in the list names a specific Babylon GPU-timing API by name, so claims about built-in GPU instrumentation should be treated as general engine capability rather than a specifically documented, named feature.

**Unreal Engine** has more directly documented tooling:
- **Shader Complexity** view mode, invoked via viewport mode controls, visualizes per-pixel instruction cost [s33].
- Epic's own guidance treats Shader Complexity as an approximation, noting translucency cost also depends on scene overdraw [s32] (sourced to 4.27 documentation).
- **ProfileGPU** and Material Editor statistics are recommended together with Shader Complexity for performance validation [s35] (also 4.27-sourced).
- Static-parameter combinations generate additional compiled permutations, and Epic's documentation explicitly recommends minimizing unused combinations [s26][s38].

The consistent, cross-engine ordering supported by the evidence: reduce unnecessary passes/overdraw and translucent screen coverage first, then redundant texture samples, then permutation count, and only then micro-optimize arithmetic — because bandwidth, overdraw, and pass count typically dominate over instruction-level cost.

## 8. Disagreements and asymmetries in the evidence

This section exists because smoothing these over would misrepresent what the sources actually show.

1. **Which release is "current" is not resolved by the evidence.** Babylon.js 9.0 is documented as announced [s1][s8], with no source confirming or ruling out a later stable release. Unreal is documented at both 5.6 (the majority of material-system pages used above) and 5.7 (release notes in English, Japanese, and secondary news coverage: [s17][s18][s20][s28][s30]). Anyone treating 5.6 as definitively "current" for this comparison is making a choice the evidence does not fully support.
2. **OIT is not a symmetric best practice across engines.** Babylon documents a built-in, one-flag, engine-level OIT technique (dual depth peeling) [s31]. The Unreal sources describe only sort-priority-based ordering for standard translucency [s42][s44], with no built-in universal OIT toggle documented. Treating "use OIT when needed" as an equally available option in both engines would overstate Unreal's documented capability.
3. **Duplicated/localized sources inflate apparent source count without adding independent evidence.** Several Unreal topics appear multiple times as language or version variants of essentially the same page — Custom Material Expressions in English, Chinese, and Japanese [s16][s24][s27]; Physically Based Materials across US, Polish, Spanish (two versions), and 4.27 English [s61][s63][s64][s68][s73]. These should be read as one underlying source repeated, not as five independent confirmations.
4. **Profiling-tool claims mix engine versions.** Shader Complexity and general performance guidance draw partly on 4.27 documentation [s32][s35][s45] and partly on 5.6 pages [s33][s38]. The sources do not confirm these tools are identical across that version gap, only that the same named concepts persist in both eras of documentation.

## 9. What remains unsettled, and what would settle it

- **Unsettled:** which Babylon.js release and which Unreal Engine release (5.6 vs. 5.7) should be treated as "current stable" at the time this question was asked. This would be settled by a dated release-note or changelog source explicitly marking one as the actively supported stable branch at the query date.
- **Unsettled:** whether Babylon.js exposes a named, documented GPU profiling/instrumentation API comparable to Unreal's ProfileGPU. The sources describe generated-shader inspection and frame-graph/render-target constructs [s43][s51] but no named Babylon profiling command. A Babylon.js performance-documentation page (not present in this source list) would settle this.
- **Unsettled:** whether Unreal's Shader Complexity/ProfileGPU workflow, as documented for 4.27 [s32][s35][s45], is unchanged in 5.6/5.7. A 5.x-specific performance/profiling documentation page would settle this instead of inferring continuity from older docs.
- **Unsettled:** how categorical, non-interpolated IDs are formally intended to be encoded in either engine's documented material system — the sources support adjacent mechanisms (vertex attributes, data textures, packed channels) but no source names an "ID encoding" convention directly. A dedicated engine document on GPU-side object/primitive ID passes would resolve this rather than leaving it as an inference.

## Sources

1. [Announcing Babylon.js 9.0](https://blogs.windows.com/windowsdeveloper/2026/03/26/announcing-babylon-js-9-0/) — other
2. [ShaderMaterial | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.ShaderMaterial) — other
3. [Babylon.js - Wikipedia](https://en.wikipedia.org/wiki/Babylon.js) — other
4. [Babylon.js - Open Source Web Engine for Gaussian Splats](https://radiancefields.com/platforms/babylon-js) — other
5. [PBRMaterial](https://doc.babylonjs.com/typedoc/classes/BABYLON.PBRMaterial) — other
6. [Build React Native applications with the power of Babylon ...](https://github.com/BabylonJS/BabylonReactNative) — docs
7. [Babylon.js: Powerful, Beautiful, Simple, Open - Web-Based 3D ...](https://www.babylonjs.com/) — other
8. [Part 3 – Babylon.js 9.0: OpenPBR and additional engine updates](https://blogs.windows.com/windowsdeveloper/2026/04/02/part-3-babylon-js-9-0-openpbr-and-additional-engine-updates/) — other
9. [Shader Materials | Babylon.js Documentation](https://doc.babylonjs.com/communityExtensions/Unity/03_ShaderMaterials/) — other
10. [Custom Shader Material & Node Material | BabylonJS/Babylon ...](https://deepwiki.com/BabylonJS/Babylon-Lite/3.4-custom-shader-material-and-node-material) — other
11. [Mastering PBR Materials | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/using/masterPBR) — other
12. [Creating the Babylon.js Node Material - Medium](https://babylonjs.medium.com/creating-the-babylon-js-node-material-6893b3abe4df) — other
13. [Announcements - Babylon.js Forum](https://forum.babylonjs.com/c/announcements/7) — other
14. [NodeMaterial | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.NodeMaterial) — other
15. [Materials & Textures | BabylonJS/Editor | DeepWiki](https://deepwiki.com/BabylonJS/Editor/5.3-materials-and-textures) — other
16. [Custom Material Expressions in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/custom-material-expressions-in-unreal-engine) — other
17. [Unreal Engine 5.7 Release Notes | Unreal Engine 5.7 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes) — other
18. [Unreal Engine 5.7 リリース ノート](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/unreal-engine-5-7-release-notes) — other
19. [Essential Unreal Engine Material Concepts - Epic Games Developersdev.epicgames.com › documentation › en-us › essential-unreal-engine-mat...](https://dev.epicgames.com/documentation/en-us/unreal-engine/essential-unreal-engine-material-concepts) — other
20. [Unreal Engine 5.7 Release Notes](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-7-release-notes) — other
21. [NISHIWAKI - Game Development News Digest (2026/05/29)](https://note.com/lumidina/n/n8f8da9daa8ad?hl=en) — other
22. [Unreal Engine 5: updates, reviews & deals - Infinite Detail](https://www.infinitedetail.xyz/tools/unreal-engine) — other
23. [Creating and Using Material Instances in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-and-using-material-instances-in-unreal-engine) — other
24. [虚幻引擎自定义材质表达式 | 虚幻引擎 5.5 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/custom-material-expressions-in-unreal-engine) — other
25. [Unreal Engine Materials | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-materials) — other
26. [Instanced Materials in Unreal Engine - Epic Games Developersdev.epicgames.com › documentation › en-us › instanced-materials-in-unre...](https://dev.epicgames.com/documentation/en-us/unreal-engine/instanced-materials-in-unreal-engine) — other
27. [Unreal Engine の Custom 表現式 | Unreal Engine 5.6 ドキュメンテーション | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/custom-material-expressions-in-unreal-engine) — other
28. [Unreal Engine 5.7 Now Available](https://www.awn.com/news/unreal-engine-57-now-available) — other
29. [In-Camera VFX Best Practices | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/in-camera-vfx-best-practices?application_version=4.27) — other
30. [Epic Games Launches Unreal Engine 5.7 With Major Upgrades to Open Worlds, Rendering, and MetaHuman Tools](https://www.invenglobal.com/articles/19890/epic-games-launches-unreal-engine-57-with-major-upgrades-to-open-worlds-rendering-and-metahuman-tools) — other
31. [Transparent Rendering | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/advanced/transparent_rendering) — other
32. [Getting Results | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/getting-results?application_version=4.27) — other
33. [Viewport Modes in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/viewport-modes-in-unreal-engine) — other
34. [Unreal Engine 5.0 Release Notes | Unreal Engine 5.0 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.0-release-notes?application_version=5.0) — other
35. [Performance Guidelines for Artists and Designers | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-guidelines-for-artists-and-designers?application_version=4.27) — other
36. [Unreal Engine 5.5 Release Notes | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-5-release-notes) — other
37. [Scalability Reference for Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/unreal-engine/scalability-reference-for-unreal-engine) — other
38. [Scalability Reference for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/scalability-reference-for-unreal-engine) — other
39. [Material Inputs in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/material-inputs-in-unreal-engine) — other
40. [Utiliser la transparence dans les matériaux | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/fr-fr/unreal-engine/using-transparency-in-unreal-engine-materials) — other
41. [Material Properties | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/material-properties?application_version=4.27) — other
42. [Using Transparency in Unreal Engine Materials | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-transparency-in-unreal-engine-materials) — other
43. [Class FrameGraphGeometryRendererTask - Babylon.js docs](https://doc.babylonjs.com/typedoc/classes/BABYLON.FrameGraphGeometryRendererTask) — other
44. [Unreal Engine Material Properties | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-material-properties) — other
45. [Performance and Profiling Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-and-profiling-overview?application_version=4.27) — other
46. [Material | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.Material) — other
47. [Babylon.js Specifications](https://www.babylonjs.com/specifications/) — other
48. [Getting Started - Chapter 2 - Texture | Babylon.js Documentation](https://doc.babylonjs.com/features/introductionToFeatures/chap2/material/) — other
49. [Materials and Vertices | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/advanced/facet_uvs/) — other
50. [Using Materials | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/using/) — other
51. [Documentation/content/features/featuresDeepDive/postProcesses/renderTargetTextureMultiPass.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/postProcesses/renderTargetTextureMultiPass.md) — docs
52. [Virtual Texturing Reference | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-texturing-reference?application_version=4.27) — other
53. [How do I iterate with texture in ShaderMaterial, as in the following ...](https://forum.babylonjs.com/t/how-do-i-iterate-with-texture-in-shadermaterial-as-in-the-following-example/40692) — other
54. [Documentation/content/features/featuresDeepDive/materials/using/masterPBR.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/materials/using/masterPBR.md) — docs
55. [Hot to use RenderTargetTexture to get result of custom material](https://forum.babylonjs.com/t/hot-to-use-rendertargettexture-to-get-result-of-custom-material/45979) — other
56. [occlusionType](https://doc.babylonjs.com/typedoc/classes/BABYLON.Mesh) — other
57. [Colors and Textures Materials - BabylonJS Guide](https://babylonjsguide.github.io/basics/Materials) — other
58. [BabylonJS Guide - GitHub Pages](https://babylonjsguide.github.io/advanced/Custom.html) — other
59. [Shader Material | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/shaders/shaderMaterial) — other
60. [Virtual Texturing Settings and Properties in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-texturing-settings-and-properties-in-unreal-engine) — other
61. [Physically Based Materials | Unreal Engine 4.27 ...](https://dev.epicgames.com/documentation/en-us/unreal-engine/physically-based-materials?application_version=4.27) — other
62. [Introduction to Physically Based Rendering (PBR) - Babylon.js docs](https://doc.babylonjs.com/features/featuresDeepDive/materials/using/introToPBR) — other
63. [Physically Based Materials in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/pl-pl/unreal-engine/physically-based-materials-in-unreal-engine%3Fapplication_version=5.2%3Fapplication_version=5.2) — other
64. [Physically Based Materials in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/physically-based-materials-in-unreal-engine) — other
65. [1. Foreword](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html) — other
66. [Using Materials and Textures in Unreal Engine for Maya Users | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-materials-and-textures-in-unreal-engine-for-maya-users) — other
67. [GltfPbrMetallicRoughness Class (Microsoft.MixedReality.Toolkit.Utilities.Gltf.Schema)](https://learn.microsoft.com/en-us/dotnet/api/microsoft.mixedreality.toolkit.utilities.gltf.schema.gltfpbrmetallicroughness?view=mixed-reality-toolkit-unity-2020-dotnet-2.8.0) — news
68. [Materiales basados en la física en Unreal Engine | Unreal Engine 5.6 Desarrllador | Epic Developer Community](https://dev.epicgames.com/documentation/es-mx/unreal-engine/physically-based-materials-in-unreal-engine) — other
69. [GltfPbrMetallicRoughness Class (Microsoft.MixedReality.Toolkit ...](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.mixedreality.toolkit.utilities.gltf.schema.gltfpbrmetallicroughness?view=mixed-reality-toolkit-unity-2019-dotnet-2.8.0) — news
70. [Base PBR Materials | KhronosGroup/glTF-Sample-Renderer | DeepWiki](https://deepwiki.com/KhronosGroup/glTF-Sample-Renderer/10.1-base-pbr-materials) — other
71. [www.khronos.org/gltf](https://www.khronos.org/files/gltf20-reference-guide.pdf) — other
72. [GltfPbrMetallicRoughness Class](https://learn.microsoft.com/ka-ge/dotnet/api/microsoft.mixedreality.toolkit.utilities.gltf.schema.gltfpbrmetallicroughness?view=mixed-reality-toolkit-unity-2018-dotnet-2.5.0) — news
73. [Materiales basados en la física en Unreal Engine | Unreal Engine 5.5 Documentación | Epic Developer Community](https://dev.epicgames.com/documentation/es-es/unreal-engine/physically-based-materials-in-unreal-engine) — other
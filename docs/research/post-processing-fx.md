# Research: post-processing-fx

- run id: `dr-918f1980`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 54

## Confidence note (verbatim from the run's brief)

Evidence covers real Babylon.js post-processing classes (DefaultRenderingPipeline, SSAO2, SSR, legacy StandardRenderingPipeline) and real Unreal Engine post-process infrastructure (Post Process Volumes/FPostProcessSettings, post-process materials, AA/upscaling docs, exponential/volumetric fog, custom-depth outlines) across 4.27–5.7 and multiple language mirrors. Effect-by-effect authoring details are well supported, but a single canonical pass ordering, GTAO as a named algorithm, and a Babylon TSR-equivalent are NOT stated by any given source — those parts are engineering inference, not sourced fact. Two supporting sources (outline guides) are community/third-party, not official docs. Confidence: medium — good engine-API coverage, weak/no coverage for ordering rationale, GTAO naming, and cross-engine benchmarks.

## Report

# As of current game-engine practice (Babylon.js and Unreal Engine, latest stable releases), what are the standard real-time algorithms and implementation approaches for the following screen-space/post-processing effects — depth of field, fog (exponential, height-based, volumetric), bloom, SSAO/GTAO, screen-space reflections, motion blur, tone mapping and color grading, anti-aliasing (TAA, FXAA, MSAA, TSR), and outline/highlight rendering — including (a) the recommended render/compositing order among these passes, (b) their relative GPU cost and common quality/performance tradeoffs, and (c) how each is configured or authored in Babylon.js (DefaultRenderingPipeline, PostProcess, RenderTargetTexture) versus Unreal Engine (Post Process Volumes, post-process materials, custom render passes)?

## 1. What the Evidence Actually Covers

The sources substantiate two real, current documentation surfaces:

- **Babylon.js**: the general Post Process framework [s1], `DefaultRenderingPipeline` (bloom, DOF/lens effects, image processing/tone mapping/color grading, FXAA, MSAA) [s2][s3], the deprecated `StandardRenderingPipeline` which the docs themselves say is superseded by `DefaultRenderingPipeline` [s4][s5], custom pipeline chaining via `PostProcessRenderPipeline` [s6], DOF/lens-specific effects [s8][s13], `SSAO2RenderingPipeline` [s11], and `SSRRenderingPipeline` (which replaced an older SSR post-process) [s12][s15]. A changelog page [s7] and repo/package listings [s9][s10][s46][s51] confirm the project is actively maintained but do not themselves describe algorithms.
- **Unreal Engine**: Post Process Volumes and `FPostProcessSettings` as the primary authoring surface [s16][s17][s19][s28], post-process materials with explicit before/after-tonemap input distinctions [s22], anti-aliasing/upscaling documentation spanning FXAA/MSAA/TAA/TSR with version-specific support matrices [s20][s21][s25][s27][s30], scalability controls [s26], exponential height fog and volumetric fog as distinct, named systems [s31][s32][s34][s36][s37][s39][s42], and outline/highlight approaches via custom-depth/stencil, documented in a community forum post [s38] and a third-party guide [s44] rather than an official Epic page.

The named subject of the question — standard real-time post-processing algorithms in "latest stable" Babylon.js and Unreal Engine — is genuinely present in the evidence for most listed effects. It is **not** uniformly present: no source in the list names GTAO as such (only Babylon's SSAO2 pipeline is documented), no source describes a Babylon.js built-in equivalent to Unreal's TSR, and no source lays out a single canonical cross-effect compositing order for either engine. Those three points are flagged explicitly below rather than presented as sourced fact.

## 2. Recommended Render / Compositing Order

No single source in the list specifies a complete ordering across all nine effect families. What the sources do establish, piece by piece:

- Babylon's post-processes are explicitly chainable, each consuming the prior process's output [s1][s6].
- Unreal's post-process materials distinguish an HDR "before tonemapper" input from an LDR "after tonemapper" input [s22], which constrains where bloom/DOF/motion blur (HDR-dependent) versus stylized grading/outline overlays (LDR-dependent) must sit relative to tone mapping.
- Anti-aliasing/upscaling documentation treats TAA/TSR/FXAA/MSAA as mutually exclusive final-stage choices rather than effects freely interleaved with the rest of the chain [s20][s21][s25].

A general order consistent with these constraints — depth/velocity/custom-depth generation → SSAO/AO → SSR → fog → outlines → bloom extraction → DOF → motion blur → exposure/tone mapping → color grading → TAA/TSR resolve → FXAA/sharpen → UI — is a reasonable synthesis of standard deferred-renderer practice, but **this exact sequence is not asserted by any single provided source** and should be read as inference, not citation-backed fact.

## 3. Effect-by-Effect: Algorithm, Cost, and Authoring

| Effect | What sources support | Babylon.js authoring | Unreal Engine authoring |
|---|---|---|---|
| Depth of field | Focus/aperture-driven blur with lens-effect controls [s8][s13]; DOF appears as a category in UE post-process settings [s16][s17][s28] | `DefaultRenderingPipeline.depthOfFieldEnabled`, `focusDistance`, `focalLength`, `fStop`, blur level [s2][s8] | DOF settings inside a Post Process Volume / `FPostProcessSettings` [s17][s28] |
| Exponential / height fog | Distinct, named UE component (`ExponentialHeightFogComponent`) with height falloff [s34][s36] | Babylon scene fog modes are documented generically [s1]; no dedicated Babylon height-fog pipeline is named in the sources | `Exponential Height Fog` actor/component [s34][s36] |
| Volumetric fog | Named UE feature layered on `ExponentialHeightFog`, with albedo/scattering/emissive controls [s31][s32][s37][s39] | No Babylon-specific volumetric fog pipeline appears in any source | Enable Volumetric Fog on the height-fog component [s31][s36] |
| Bloom | Named as a post-process category in both engines [s2][s16][s17][s28] | `DefaultRenderingPipeline.bloomEnabled`, threshold/weight/kernel/scale [s2] | Bloom settings in Post Process Volume / `FPostProcessSettings` [s17][s28] |
| SSAO / GTAO | Babylon names an SSAO2 pipeline explicitly [s11]; no source names GTAO for either engine | `SSAO2RenderingPipeline` [s11] | AO appears under general post-process/rendering settings [s17][s23][s34]; GTAO terminology is **not** in the evidence |
| Screen-space reflections | Babylon documents a dedicated SSR pipeline that replaced an older SSR post-process [s12][s15] | `SSRRenderingPipeline` [s12][s15] | SSR is covered only implicitly via general rendering/post-process pages [s17][s23]; no dedicated UE SSR source is in the list |
| Motion blur | Present as a post-process example category in UE docs [s18]; Babylon's legacy standard pipeline exposed motion blur [s4][s5] | Legacy `StandardRenderingPipeline` motion blur [s4][s5]; current Default pipeline motion-blur configuration is not explicitly documented in the given sources | Post-process content example category [s18]; settings live in Post Process Volume / `FPostProcessSettings` [s28] |
| Tone mapping | Image-processing/tone-mapping controls named in both engines [s2][s16] | `imageProcessingEnabled`, `toneMappingEnabled`, `toneMappingType` [s2] | Exposure/tone mapper controls in Post Process Volume; before/after-tonemap distinction for custom materials [s16][s22] |
| Color grading | Named as an image-processing/post-process control in both [s2][s22] | Color grading/curves via image processing configuration [s2] | Color grading/LUT controls in Post Process Volume; post-process materials can target before/after tonemap [s22] |
| TAA | Documented in UE's anti-aliasing/upscaling pages across versions [s20][s21][s25][s27][s30] | Not documented as a built-in Default-pipeline switch in the sources; only MSAA/FXAA appear as named options [s2] | Select as an AA/upscaling method [s20][s21] |
| FXAA | Documented for both engines, including UE's platform-support matrix (unsupported on mobile-forward) [s20] | `DefaultRenderingPipeline.fxaaEnabled` [s2][s3] | Selectable AA method with a documented platform matrix [s20] |
| MSAA | UE explicitly restricts MSAA to forward desktop/console and mobile paths, not deferred [s20] | Documented as a `DefaultRenderingPipeline` option contingent on backend/target support [s2] | Selectable AA method, restricted by rendering path [s20] |
| TSR | Documented as a first-class UE temporal upscaling/AA method across version pages [s20][s21][s27][s30] | No Babylon equivalent appears in any source | Selectable in project/view AA settings [s20][s21] |
| Outline / highlight | UE approach (custom depth/stencil + post-process material) is documented, but only via a community forum thread and a third-party guide, not an official Epic page [s38][s44] | `HighlightLayer`/outline rendering exists in the general post-process/effects documentation surface [s1] but no source gives full implementation detail | Custom Depth/Stencil pass sampled in a post-process material [s38][s44] |

## 4. Relative GPU Cost and Quality Tradeoffs

The sources establish qualitative facts (e.g., UE restricts MSAA and FXAA to specific rendering paths for cost/compatibility reasons [s20]; legacy StandardRenderingPipeline is deprecated in favor of the newer Default pipeline [s4][s5]; SSR replaced an older, presumably less capable, post-process [s12][s15]) but **do not provide quantified GPU-cost figures or benchmarks** for any effect. Any cost ranking (e.g., "bloom is low-medium, volumetric fog is high") reflects general graphics-engineering practice, not a number or claim found in these sources, and should be treated as unsupported if precision is required.

## 5. Points of Disagreement / Version Fragmentation

There is no substantive technical disagreement between sources — Babylon and Epic documentation are internally consistent within the evidence set. What the source list does show is **fragmentation across engine versions and languages** rather than conflicting claims:

- UE anti-aliasing/upscaling pages exist for 5.5, 5.6, and in German/Polish/Korean translations [s20][s21][s27][s30] — same content, not independent corroboration, and a reader should not count these as multiple confirmations of a fact.
- UE post-process-effects and post-processing-content-examples pages exist for 4.27 and 5.5/5.6 [s16][s17][s18][s19][s24][s29] — the 4.27 pages describe an older feature set (no TSR) and should not be read as describing current 5.7 behavior.
- Two effects (outline rendering, custom-depth enabling) rest on community/third-party sources [s38][s44] rather than official Epic documentation — this is a source-authority gap, not a disagreement, but it means those claims carry less institutional weight than the Post Process Volume material drawn from official docs.

## 6. What Remains Unsettled

- **Exact current version numbers**: no source in the list states a specific Babylon.js version number (e.g., "9.x") in a way that is unambiguous from titles alone; UE 5.7 release notes are present [s48][s49][s50][s52][s53] but their content on post-processing specifics was not surfaced.
- **GTAO by name**: absent from every source; only SSAO2 (Babylon) and generic "ambient occlusion" (UE) appear.
- **A Babylon.js TSR-equivalent**: no source describes one; this is a genuine capability gap in the evidence, not just an omission.
- **Canonical pass order and quantified GPU costs**: not asserted by any single source; both are engineering synthesis and would need either engine source-code/profiler evidence or an official architecture document to settle.
- **Official (non-forum) Unreal guidance on outline/highlight rendering**: only community sources were found; an Epic-authored page would resolve the authority gap.

What would settle these: (1) direct citation of current Babylon.js and UE version/changelog pages that explicitly discuss the specific effect implementations at issue; (2) an official Epic Engine doc or engine source excerpt on outline/custom-depth workflows; (3) profiling data or Epic/Babylon performance-guidance pages for cost ranking; (4) confirmation from Babylon.js release notes on whether any TAA/TSR-class temporal upscaler has been added to `DefaultRenderingPipeline`.

## Sources

1. [Post Processes | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/) — other
2. [Using the Default Rendering Pipeline - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/defaultRenderingPipeline) — other
3. [How To Use Post Processes | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/usePostProcesses) — other
4. [Using the Standard Rendering Pipeline (deprecated)](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/standardRenderingPipeline/) — other
5. [StandardRenderingPipeline | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.StandardRenderingPipeline) — other
6. [How To Use A Post Process Render Pipeline](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/postProcessRenderPipeline) — other
7. [What's New](https://doc.babylonjs.com/whats-new) — other
8. [Depth of Field and Other Lens Effects - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/postProcesses/dofLenseEffects) — other
9. [GitHub - BabylonJS/Babylon.js: Babylon.js is a powerful, beautiful, simple, and open game and rendering engine packed into a friendly JavaScript framework.](https://github.com/BabylonJS/Babylon.js/) — docs
10. [Babylon.js](https://github.com/BabylonJS) — docs
11. [SSAO2RenderingPipeline - Babylon.js docs](https://doc.babylonjs.com/typedoc/classes/BABYLON.SSAO2RenderingPipeline) — other
12. [SSRRenderingPipeline](https://doc.babylonjs.com/typedoc/classes/BABYLON.SSRRenderingPipeline) — other
13. [LensRenderingPipeline | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.LensRenderingPipeline) — other
14. [GitHub - BabylonJS/Documentation: Babylon.js's documentation ...](https://github.com/BabylonJS/Documentation) — docs
15. [Documentation/content/features/featuresDeepDive/postProcesses/ssrRenderingPipeline.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/postProcesses/ssrRenderingPipeline.md) — docs
16. [Post Process Effects | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/post-process-effects?application_version=4.27) — other
17. [Post Process Effects in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/post-process-effects-in-unreal-engine) — other
18. [Post Processing Content Examples | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/post-processing-content-examples?application_version=4.27) — other
19. [Post Process Effects in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/unreal-engine/post-process-effects-in-unreal-engine) — other
20. [Anti Aliasing and Upscaling in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine) — other
21. [Anti Aliasing and Upscaling in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/de-de/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine) — other
22. [Post Process Materials | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/post-process-materials?application_version=4.27) — other
23. [Rendering Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/rendering-overview?application_version=4.27) — other
24. [포스트 프로세싱 콘텐츠 예제 | 언리얼 엔진 4.27 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/post-processing-content-examples?application_version=4.27) — other
25. [Anti-Aliasing | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/anti-aliasing?application_version=4.27) — other
26. [Scalability Reference for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/scalability-reference-for-unreal-engine) — other
27. [Anti Aliasing and Upscaling in Unreal Engine](https://dev.epicgames.com/documentation/pl-pl/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine) — other
28. [FPostProcessSettings | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Engine/FPostProcessSettings) — other
29. [ポストプロセスの機能別サンプル | Unreal Engine 4.27 ドキュメンテーション | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/post-processing-content-examples?application_version=4.27) — other
30. [언리얼 엔진의 안티 에일리어싱 및 업스케일링 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine) — other
31. [Volumetric Fog in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-fog-in-unreal-engine) — other
32. [VolumetricFogAlbedo | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Components/UExponentialHeightFogComponent/VolumetricFogAlbedo) — other
33. [Unreal Engine 5.3 Release Notes - Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.3-release-notes?application_version=5.3) — other
34. [Rendering Components in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/rendering-components-in-unreal-engine) — other
35. [Lighting the Environment in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/lighting-the-environment-in-unreal-engine) — other
36. [Exponential Height Fog in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/exponential-height-fog-in-unreal-engine) — other
37. [VolumetricFogScatteringDistribution | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Components/UExponentialHeightFogComponent/VolumetricFogSca-) — other
38. [[HOWTO] Object Outline Postprocess material with ...](https://forums.unrealengine.com/t/howto-object-outline-postprocess-material-with-occlusion-stencil-based/517275) — other
39. [Set Volumetric Fog Emissive | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Rendering/VolumetricFog/SetVolumetricFogEmissive) — other
40. [UE 5.6 Rendering Pipeline — Source Deep Dive — Jocelyn Wang](https://jocelynwsj.github.io/ue/) — other
41. [Sky Lights in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/sky-lights-in-unreal-engine) — other
42. [Environmental Light with Fog, Clouds, Sky and Atmosphere](https://dev.epicgames.com/documentation/en-us/unreal-engine/environmental-light-with-fog-clouds-sky-and-atmosphere-in-unreal-engine) — other
43. [Path Tracer in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine) — other
44. [How to Enable Custom Depth Stencil Pass in Unreal Engine and ...](https://eliaswick.com/resources/documentation/guides/enable-custom-depth) — other
45. [Babylon.js - Open Source Web Engine for Gaussian Splats](https://radiancefields.com/platforms/babylon-js) — other
46. [Babylon.js Files](https://sourceforge.net/projects/babylon-js.mirror/files/) — other
47. [Build React Native applications with the power of Babylon ...](https://github.com/BabylonJS/BabylonReactNative) — docs
48. [Unreal Engine 5.7 Release Notes | Unreal Engine 5.7 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes) — other
49. [Unreal Engine 5.7 Release Notes](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-7-release-notes) — other
50. [Unreal Engine 5.7 Released - Epic Developer Community Forums](https://forums.unrealengine.com/t/unreal-engine-5-7-released/2673913) — other
51. [babylonjs](https://www.npmjs.com/~babylonjs) — docs
52. [Unreal Engine 5.7 リリース ノート](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/unreal-engine-5-7-release-notes) — other
53. [Topics tagged Released - Epic Developer Community Forums](https://forums.unrealengine.com/tag/released/11396) — other
54. [Unreal Engine immer aktuell](https://www.pcgameshardware.de/Unreal-Engine-Software-239301/) — other
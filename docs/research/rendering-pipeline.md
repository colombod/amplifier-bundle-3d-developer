# Research: rendering-pipeline

- run id: `dr-f05d0e1b`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 65

## Confidence note (verbatim from the run's brief)

The evidence does establish the subject: Babylon.js (WebGL2/WebGPU) and UE5 (Lumen/Nanite/TSR) official docs cover draw-call/batching, pass ordering, transparency sorting, culling, shading-model choice, and profiling tools (drawCallsCounter/EngineInstrumentation; stat unit/stat gpu/ProfileGPU/Unreal Insights). One concrete, documented budget figure exists: Lumen's ~4ms GPU target at 1080p/60fps. No source, however, gives a single canonical table allocating the full 16.6ms across all six stages for either engine — that breakdown is necessarily a practitioner synthesis, not a sourced fact. Several UE sources are locale duplicates of the same document (not independent corroboration), and one Babylon citation (WGSL cache) is an open GitHub issue, not settled guidance. Confidence: medium — the practices and tools are well documented, but the precise per-stage millisecond allocation the question asks for is not itself a documented artifact.

## Report

# As of the current generation of real-time engines (targeting late 2023–2025 releases), what are the documented best practices and frame-time budgets for real-time 3D rendering pipelines at a 60fps target (16.6ms budget), specifically: (1) draw call reduction and batching strategy, (2) render pass ordering, (3) transparency and depth sorting, (4) frustum and occlusion culling, (5) forward vs. deferred vs. clustered shading choice, and (6) CPU/GPU frame-time budget allocation across these stages — covering both Babylon.js (WebGL2 and WebGPU backends) and Unreal Engine 5 (with Lumen, Nanite, and TSR specifically), and what profiling tools/workflows are used to measure each of these stages in each engine?

## 1. Does the evidence cover what was asked?

Yes, substantially, but with one important gap. The source list contains official Babylon.js documentation for WebGL2 and WebGPU (optimization, snapshot rendering, instancing, thin instances, clustered lighting, occlusion queries, transparent rendering, engine instrumentation) and official Epic Games documentation for UE5 (Lumen Performance/Technical Guides, Nanite, Temporal Super Resolution, rendering-path support, profiling introduction, Unreal Insights). All six requested topics — batching, pass ordering, transparency sorting, culling, shading-model choice, and profiling — are addressed for both engines somewhere in the list.

What the evidence does **not** contain is a single document, for either engine, that lays out a full 16.6ms-frame allocation across all these stages. The only concrete, engine-published millisecond budget in the set is Lumen's documented ~4ms GPU target at 1080p for a 60fps console target [s22][s25]. Everything else below that is either a qualitative best practice or a range that this report constructs by combining several sources — it should not be read as a single sourced table.

## 2. The 60fps budget itself

60fps implies a 16.67ms frame interval; this is arithmetic, not a documented engine claim, so no citation is attached to it. Both engines document that CPU (game logic) and GPU (rendering) work overlap across frames rather than summing serially, and that profiling should separate these streams rather than reason about one FPS number. UE's profiling introduction documentation is built around exactly this separation — Game thread, Draw/render thread, and GPU time reported independently [s17][s21]. Babylon's instrumentation API is built the same way, exposing separate counters for scene evaluation, render submission, and GPU frame time [s6][s15].

Neither engine's documentation in this set states a universal per-stage ms budget (e.g., "spend X ms on culling, Y ms on shading"). The only anchor figure is Lumen's documented ~4ms GPU target at 1080p/60fps [s22][s25] — and that covers Lumen's GI/reflection work specifically, not the whole frame.

## 3. Draw call reduction and batching

**Babylon.js.** The core documented tools are hardware instancing (`InstancedMesh`, single draw call per instanced set) [s46][s52] and thin instances for very large populations where per-object JavaScript overhead matters [s48]. The `Optimizing Your Scene` guide frames `drawCallsCounter` as the primary diagnostic and recommends keeping it as low as practical while grouping by shared material/texture/state [s6]. A documented trade-off: thin instances are not independently culled — if the source mesh is visible, all instances in the buffer are drawn [s48].

**UE5.** Nanite is the documented mechanism for reducing draw-call and geometry-management overhead on suitable static meshes: cluster selection and rendering happen in a GPU-driven path rather than through per-object CPU draw submission [s57][s60]. Traditional (non-Nanite) meshes still benefit from the same instancing/merging logic UE has long documented [s37].

**Trade-off, both engines.** Aggressive merging/batching reduces draw-call count but can weaken culling granularity — a documented tension rather than a free win, since one visible fragment can force an entire merged object to render.

## 4. Render pass ordering

**Babylon.js.** Depth/opaque passes precede alpha-tested and alpha-blended geometry; blended meshes are explicitly sorted (see §5). Custom ordering per rendering group is exposed via `Scene.setRenderingOrder` [s38], and transparent-material behavior (including sort keys) is documented in the transparent-rendering guide [s32].

**UE5.** The documented conceptual flow — visibility/culling, depth and shadow work, opaque base pass (GBuffer in deferred, forward path in forward mode), lighting, then translucency and post-processing — is described across the rendering overview and the forward/deferred feature-support matrix [s37][s29][s30]. Lumen inserts its own stages into this flow, documented separately as Lumen Scene Lighting and Screen Probe Gather [s28][s55].

Neither engine's documentation in this set claims one universal ordering is correct for all scenes; both frame ordering as something to profile per render-graph configuration.

## 5. Transparency and depth sorting

**Babylon.js.** Alpha-blended meshes are sorted, documented as ordering by `alphaIndex` then camera distance [s32]. `Scene.setRenderingOrder` allows custom opaque/alpha-test/transparent sort functions [s38]. The transparent-rendering guide frames blending as inherently more expensive than alpha-test/opaque paths and recommends minimizing blended surface area [s32].

**UE5.** The rendering-path documentation states that translucency in the deferred renderer is handled through a forward-style pass rather than the opaque GBuffer path [s37][s29] — i.e., translucent materials do not get the same deferred-lighting treatment as opaque ones regardless of whether Nanite is used for the opaque geometry.

## 6. Frustum and occlusion culling

**Babylon.js.** `AbstractMesh.cullingStrategy` documents multiple strategies, with bounding-sphere-only exclusion as the fast default and bounding-box/optimistic-inclusion variants for accuracy [s43]. A dedicated GPU occlusion-query feature exists (`occlusionType` on `Mesh`) [s42], with a separate guide describing occlusion queries as asynchronous and typically consuming the prior frame's result [s45]. `Optimizing Your Scene` frames these as tools to reduce active-mesh count before submission [s6].

**UE5.** Nanite performs cluster-level visibility selection on the GPU rather than relying solely on CPU-side per-object culling [s57][s60]. Release notes document ongoing changes to GPU-culling statistics/reporting, e.g., UE5.4 added statistics reflecting indirect-argument results after GPU culling [s23]. General profiling guidance treats occlusion as the most expensive of the culling stages, applied after cheaper distance/frustum rejection [s30].

## 7. Forward vs. deferred vs. clustered shading

**Babylon.js.** The default material path is forward-style (lighting evaluated during material shading). Clustered lighting is documented as an explicit feature available on both WebGL2 and WebGPU, contingent on floating-point color-buffer support, dividing view space into 3D clusters so each pixel only evaluates lights bound to its cluster [s44]. WebGL2 feature support (UBOs, instancing, MSAA) underpins what's practical for either path [s7].

**UE5.** The supported-features-by-rendering-path documentation is the authoritative source for what forward vs. deferred each support on desktop [s29]; mobile documentation separately covers forward/mobile shading modes [s40]. Translucency remains forward-style even under the deferred path [s37][s29], and Lumen/Nanite do not change this basic choice — they add work on top of whichever path is selected [s57][s28].

## 8. CPU/GPU frame-time allocation

This is the weakest-sourced part of the question. The one concrete, documented figure is Lumen's approximate 4ms GPU budget at 1080p for a 60fps console target [s22][s25]; UE's Lumen guide's existence in three additional locale copies [s31][s36] does not add independent corroboration — they are translations of the same content. No source in this set provides:
- a documented total-frame ms split across culling/geometry/shading/transparency/post-process for UE5, or
- any documented ms allocation table for Babylon.js at all.

Any such table (including ranges like "2–4ms for shadows" or "1–3ms for TSR") is therefore a practical, unsourced synthesis, not an engine-documented fact, and should be labeled as such if used downstream.

## 9. Profiling tools and workflows

**Babylon.js.** `EngineInstrumentation`/scene counters expose `drawCallsCounter`, active-mesh evaluation time, render time, GPU frame time, and render-target time [s6][s15]. WebGPU-specific profiling concerns include snapshot-rendering's effect on CPU submission cost (not GPU cost) [s2], non-compatibility-mode implications [s4], and documented backend limitations to account for when comparing WebGL2 vs. WebGPU numbers [s9]. A WGSL shader-cache proposal exists to reduce load-time compilation cost, but as of this evidence it is an **open GitHub issue**, not settled documented practice [s8].

**UE5.** The standard workflow starts with `stat unit`/`stat gpu` and the general profiling-introduction documentation [s17][s21], moves to Unreal Insights for CPU/GPU timeline capture including thread stalls [s18][s20], and uses feature-specific guides for deep dives: Lumen's guide recommends `Stat GPU`/`ProfileGPU` for pass-level Lumen timing [s22][s25], and `stat tsr` is documented for TSR cost inspection [s27]. Ray-tracing-specific costs have a separate performance guide [s33]. VR has its own performance-testing documentation reflecting a different frame budget (not 60fps) [s24] — relevant only as a boundary case, not as evidence for the 60fps question.

## 10. Cross-source cautions (in place of a "disagreement" section)

There is no direct factual conflict between sources in this set — they are complementary vendor documentation, not competing studies. Two evidentiary weaknesses should be flagged instead of a disagreement:

- **Duplicate, not independent, sources.** Several UE citations are the same document in different locales (Lumen Performance Guide: [s22][s25][s31][s36]; profiling introduction: [s17][s39][s41]; profiling overview 4.27: [s21][s34]; 5.6 release notes: [s19][s26]). Treat these as one source each, not four.
- **Version drift.** Some UE sources describe 4.27-era profiling terminology [s21][s34][s37] rather than the 5.x pipeline the question targets; the concepts (stat unit, stat gpu) carry forward, but exact tool names/UI have evolved through 5.0–5.6 [s16][s61][s59][s63][s23][s56][s19].

## 11. What remains unsettled and what would settle it

- **No sourced per-stage ms table exists for either engine.** This would only be settled by an official Epic/Babylon document that publishes such a breakdown (not found here), or by first-party captured profiling data from a specific target scene/hardware — a documented practice, not a documented number.
- **The Lumen 4ms figure's precise scope** (whether it includes screen-probe gather, hardware RT, and surface-cache update, or only a subset) is not disambiguated by title alone in this evidence set; the Lumen Technical Details document [s28][s55] would need to be read alongside the Performance Guide to confirm.
- **The WGSL shader-cache** [s8] is unresolved by design — its status should be re-checked against the current Babylon.js issue tracker before being cited as shipped best practice.
- **Babylon.js has no documented equivalent Lumen-style numeric GPU budget** for any of its features (clustered lighting, transparency, WebGPU snapshot rendering) in this evidence set; obtaining one would require either new first-party Babylon documentation or direct benchmarking.

## Sources

1. [WebGPU Miscellaneous Optimizations](https://doc.babylonjs.com/setup/support/webGPU/webGPUOptimization/webGPUMiscellaneous) — other
2. [WebGPU Snapshot Rendering - Babylon.js documentation](https://doc.babylonjs.com/setup/support/webGPU/webGPUOptimization/webGPUSnapshotRendering) — other
3. [Documentation/content/setup/support/webGPU.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU.md) — docs
4. [Documentation/content/setup/support/webGPU/webGPUOptimization/webGPUNonCompatibilityMode.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU/webGPUOptimization/webGPUNonCompatibilityMode.md) — docs
5. [Documentation/content/setup/support/webGPU/webGPUInternals.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU/webGPUInternals.md) — docs
6. [Optimizing Your Scene | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/scene/optimize_your_scene) — other
7. [WebGL2 Support | Babylon.js Documentation](https://doc.babylonjs.com/setup/support/webGL2) — other
8. [WebGPU: speed up loading with a WGSL cache · Issue #14909 · BabylonJS/Babylon.js](https://github.com/BabylonJS/Babylon.js/issues/14909) — docs
9. [Documentation/content/setup/support/webGPU/webGPULimitations.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU/webGPULimitations.md) — docs
10. [Documentation/content/setup/support/webGPU/webGPUInternals/webGPUOverview.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU/webGPUInternals/webGPUOverview.md) — docs
11. [Documentation/content/features/featuresDeepDive/particles/particle_system/gpu_particles.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/particles/particle_system/gpu_particles.md) — docs
12. [Documentation/content/setup/support/webGPU/webGPUInternals/webGPUMiscellaneous.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/setup/support/webGPU/webGPUInternals/webGPUMiscellaneous.md) — docs
13. [Performance](https://doc.babylonjs.com/guidedLearning/createAGame/performance) — other
14. [WebGPUEngine | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.WebGPUEngine) — other
15. [EngineInstrumentation | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.EngineInstrumentation) — other
16. [Unreal Engine 5.0 Release Notes | Unreal Engine 5.0 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.0-release-notes?application_version=5.0) — other
17. [Introduction to Performance Profiling and Configuration in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine) — other
18. [Unreal Insights Reference in Unreal Engine 5 | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-insights-reference-in-unreal-engine-5) — other
19. [Unreal Engine 5.6 Release Notes | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-6-release-notes) — other
20. [Timing Insights in Unreal Engine 5. - Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/timing-insights-in-unreal-engine-5) — other
21. [Performance and Profiling Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-and-profiling-overview?application_version=4.27) — other
22. [Lumen Performance Guide for Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/de-de/unreal-engine/lumen-performance-guide-for-unreal-engine) — other
23. [Unreal Engine 5.4 Release Notes | Unreal Engine 5.4 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.4-release-notes?application_version=5.4) — other
24. [VR Performance Testing in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/vr-performance-testing-in-unreal-engine) — other
25. [Lumen Performance Guide for Unreal Engine - Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-performance-guide-for-unreal-engine) — other
26. [虚幻引擎5.6版本说明 | 虚幻引擎 5.6 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/unreal-engine-5-6-release-notes) — other
27. [Temporal Super Resolution in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/temporal-super-resolution-in-unreal-engine) — other
28. [Lumen Technical Details in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/unreal-engine/lumen-technical-details-in-unreal-engine) — other
29. [Supported Features by Rendering Path for Desktop with Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine) — other
30. [Introduction to Rendering in Unreal Engine for Unity Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-rendering-in-unreal-engine-for-unity-developers) — other
31. [언리얼 엔진의 루멘 퍼포먼스 가이드 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/lumen-performance-guide-for-unreal-engine) — other
32. [Transparent Rendering - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/materials/advanced/transparent_rendering) — other
33. [Ray Tracing Performance Guide in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/ray-tracing-performance-guide-in-unreal-engine) — other
34. [Performance and Profiling Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/ar-ar/unreal-engine/performance-and-profiling-overview?application_version=4.27) — other
35. [ConsoleVariable | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Engine/URendererSettings/ConsoleVariable) — other
36. [虚幻引擎Lumen性能指南 | 虚幻引擎 5.6 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/lumen-performance-guide-for-unreal-engine) — other
37. [Rendering Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/rendering-overview?application_version=4.27) — other
38. [Scene - Babylon.js documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.Scene) — other
39. [Introduction to Performance Profiling and Configuration in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/es-mx/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine) — other
40. [Mobile Rendering and Shading Modes for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/mobile-rendering-and-shading-modes-for-unreal-engine) — other
41. [Einführung in Performance-Profiling und -Konfiguration in Unreal Engine | Unreal Engine 5.6 Dokumentation | Epic Developer Community](https://dev.epicgames.com/documentation/de-de/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine) — other
42. [occlusionType](https://doc.babylonjs.com/typedoc/classes/BABYLON.Mesh) — other
43. [AbstractMesh | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.AbstractMesh) — other
44. [Clustered Lighting](https://doc.babylonjs.com/features/featuresDeepDive/lights/clusteredLighting/) — other
45. [Occlusion Queries - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/occlusionQueries) — other
46. [InstancedMesh](https://doc.babylonjs.com/typedoc/classes/BABYLON.InstancedMesh) — other
47. [WebGPU Support](https://doc.babylonjs.com/setup/support/webGPU) — other
48. [Thin Instances](https://doc.babylonjs.com/features/featuresDeepDive/mesh/copies/thinInstances) — other
49. [WebGPU Internals - Babylon.js Documentation](https://doc.babylonjs.com/setup/support/webGPU/webGPUInternals/) — other
50. [Babylon.js docs](https://doc.babylonjs.com/whats-new/) — other
51. [GroundMesh | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.GroundMesh) — other
52. [Instances - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/mesh/copies/instances) — other
53. [Babylon.js docs](https://doc.babylonjs.com/) — other
54. [WebGPUEngineOptions](https://doc.babylonjs.com/typedoc/interfaces/BABYLON.WebGPUEngineOptions) — other
55. [Lumen Technical Details in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-technical-details-in-unreal-engine) — other
56. [Unreal Engine 5.5 Release Notes | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-5-release-notes) — other
57. [Nanite Virtualized Geometry](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-virtualized-geometry-in-unreal-engine/?application_version=5.0) — other
58. [Hardware Ray Tracing in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/hardware-ray-tracing-in-unreal-engine) — other
59. [Unreal Engine 5.2 Release Notes | Unreal Engine 5.2 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.2-release-notes?application_version=5.2) — other
60. [Nanite in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-in-unreal-engine) — other
61. [Unreal Engine 5.1 Release Notes | Unreal Engine 5.1 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.1-release-notes?application_version=5.1) — other
62. [Lumen Global Illumination and Reflections](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine/?application_version=5.0) — other
63. [Unreal Engine 5.3 Release Notes | Unreal Engine 5.3 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.3-release-notes?application_version=5.3) — other
64. [Valley of the Ancient Sample Game for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/valley-of-the-ancient-sample-game-for-unreal-engine) — other
65. [Anti Aliasing and Upscaling in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine) — other
# Research: interaction-picking

- run id: `dr-a00e7b41`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 74

## Confidence note (verbatim from the run's brief)

Confidence: medium. The named features (ActionManager, PointerObservable/PointerInfo, GPUPicker, GizmoManager, WebXR input controllers; Enhanced Input, line traces, Interactive Tools Framework gizmos, WidgetInteractionComponent, UMG/screen-reader accessibility) are all real and present in the evidence, so this is not a case of adjacent-only sources. But coverage is uneven: Enhanced Input, the Interactive Tools Framework, WebXR input, and UMG accessibility rest on solid first-party API docs, while Babylon's GPU picking and Unreal's outline/custom-depth technique rest mainly on forum threads, a wiki mirror, and third-party guides rather than canonical doc pages. Several practical topics the question asks about — constrained-manipulation math, Unreal multi-object selection architecture, and touch-specific input handling in Unreal — are not directly addressed by any source and are flagged in the report as unsupported/inferred rather than cited. Read the full report before treating any single recommendation as authoritative.

## Report

# What are the current (as of the most recent stable releases) best practices for implementing user interaction in real-time 3D applications, specifically comparing and evaluating: (1) ray picking versus GPU color/ID picking for object selection, (2) single- and multi-object selection patterns, (3) hover/highlight rendering techniques, (4) transform gizmos and manipulators, (5) drag and constrained manipulation, (6) hit-test performance at scale (large numbers of pickable objects/meshes), (7) touch and XR controller input handling, and (8) accessibility considerations for 3D interaction — with concrete treatment of how each is implemented in Babylon.js (ActionManager, PointerObservable/PointerInfo, GPUPicker, GizmoManager, WebXR input controllers) and in Unreal Engine (Enhanced Input System, hit-testing/line traces, widget/gizmo interaction)?

## 1. Scope check: does the evidence actually cover this question?

The question names specific artifacts: Babylon.js `ActionManager`, `PointerObservable`/`PointerInfo`, `GPUPicker`, `GizmoManager`, WebXR input controllers; and Unreal's Enhanced Input System, hit-testing/line traces, and widget/gizmo interaction. All of these are genuinely present in the source list, not merely adjacent topics: `ActionManager` [s16][s18][s20], `PointerInfo`/`PointerInfoPre` [s2][s9][s10], `GPUPicker` [s61][s63][s64][s67], gizmo documentation [s7], and WebXR input classes [s1][s4][s5][s6] on the Babylon side; Enhanced Input official docs [s31][s32][s39][s40], `LineTraceByChannel`/`BreakHitResult`/`MultiLineTraceByChannel` [s37][s43][s65], and the Interactive Tools Framework gizmo classes plus `WidgetInteractionComponent` [s33][s34][s36][s41][s44][s45][s46][s52] on the Unreal side. This is a case where the evidence genuinely supports treating the question as answerable.

However, coverage depth is uneven across the eight sub-topics. Enhanced Input, the Interactive Tools Framework, WebXR input, and UMG/screen-reader accessibility rest on first-party API reference pages. Babylon's GPU picking and Unreal's selection-outline/custom-depth technique rest mainly on forum threads, a third-party wiki mirror, and community guides rather than a single canonical doc page — this matters for confidence and is discussed in Section 10. Some parts of the question (constrained-manipulation math, Unreal's multi-object selection architecture, touch-specific handling in Unreal) are not addressed by any source at all; these are marked explicitly below rather than silently filled in.

## 2. Ray picking versus GPU color/ID picking

**Babylon.js.** Scene-level picking (`scene.pick`/`pickWithRay`) returns a picking result surfaced through the pointer event system; `PointerInfo` and `PointerInfoPre` are the documented classes for pointer event/pick data [s2][s9][s10]. A separate, purpose-built `GPUPicker` exists for ID-based picking, described in a DeepWiki mirror of the Babylon-Lite picking subsystem [s61] and introduced/discussed across a sequence of forum threads: an announcement [s63], a request to expose it as a scene-level picking option [s64], and a later follow-up thread [s67]. This sequence itself signals that GPU picking's integration level (announcement → "can we get this at scene level" → continued discussion) was still being worked out in the community rather than settled as a single, stable, first-class API path — treat performance/maturity claims about `GPUPicker` as moderately, not strongly, supported.

Babylon's WebGPU engine performs operations asynchronously in general [s69], and the WebGPU breaking-changes documentation reflects an engine still evolving around WebGPU support [s62]; this is consistent with — but does not explicitly document — the common practitioner recommendation to throttle GPU-picking readback rather than run it every pointer-move. That specific recommendation should be treated as a **reasonable inference from adjacent evidence**, not a directly sourced claim.

The Gaussian Splatting documentation [s14][s22] is the one place in the evidence where a content type without ordinary triangle geometry is discussed, which is consistent with a need for picking approaches other than standard ray-triangle intersection for that content. The stronger claim — that Babylon's docs describe using "proxy meshes in a GPU picking list" specifically for splat parts — is plausible given the source titles but cannot be confirmed beyond the title-level evidence available here; treat it as directionally supported rather than verified in detail.

**Unreal Engine.** Hit-testing is done through line traces: `LineTraceByChannel` (Blueprint API) documents the standard single-hit trace and its `Trace Complex` option [s37]; `MultiLineTraceByChannel` covers multi-hit variants [s65]; `BreakHitResult` documents the fields available from a hit (actor, component, location, normal, etc.) [s43]. A separate, differently-scoped `LineTrace` utility exists for GeoReferencing use cases [s38] — this is a distinct, narrower tool and should not be conflated with general gameplay hit-testing. AR-specific hit testing is documented separately [s35], relevant to touch/mobile AR interaction specifically rather than general 3D picking.

No source in the evidence directly states "use ray picking for X object counts and GPU picking for Y" as an explicit engine recommendation in either engine; the crossover guidance here is synthesis, not a sourced claim, and is presented as such.

## 3. Single- and multi-object selection

**Babylon.js.** `ActionManager` with an `OnPickTrigger` supports simple, per-object pick reactions [s16][s18][s20]. For a centralized selection model, Babylon documents `SelectionOutlineLayer`, which has explicit `addSelection`/`clearSelection` operations and is presented in the context of a click-to-select, shift-click-to-toggle workflow [s23][s25]. `EffectLayer` is the base class both `HighlightLayer` and outline layers build on [s27], and there is a frame-graph task variant for the selection outline layer [s28], indicating this feature has been carried into Babylon's newer node/frame-graph render pipeline — a sign it is current, not legacy.

**Unreal Engine.** The evidence does **not** include a source describing an official multi-object selection API or subsystem for gameplay use. `Transforming Actors in Unreal Engine` [s42] documents actor transform operations but not a selection-set API. Any recommendation about how to architect multi-select in Unreal (a `TSet` of weak actor pointers, a dedicated selection subsystem, etc.) is standard engineering practice, **not something the provided sources establish** — this should be read as unsupported-by-source and flagged as such rather than attributed to any Unreal documentation.

## 4. Hover/highlight rendering

**Babylon.js.** `HighlightLayer` is documented as the standard glow/highlight mechanism [s17][s21]. `SelectionOutlineLayer` is the documented mechanism for crisp, editor-style outlines with explicit selection-list management [s23][s25]. Both descend from the common `EffectLayer` base [s27], and the outline layer has a frame-graph-integrated task variant [s28] — again indicating current-generation support.

**Unreal Engine.** This is the weakest-evidenced sub-topic on the Unreal side. Every source touching on custom-depth/stencil outline rendering is either a third-party guide (not an Epic doc) [s68], a community forum thread [s70][s71], a years-old forum post about outline materials [s66], or an engine bug-tracker issue [s72]. There is **no official Epic documentation page in this evidence set** describing the built-in post-process/custom-depth selection-outline workflow as a first-class, current feature. This is worth stating plainly rather than smoothing over: the technique is real and widely used in the Unreal community, but the evidence supporting it here is community/third-party, not authoritative Epic documentation, which materially lowers confidence in any specific implementation detail (e.g., exact stencil-value conventions) drawn from it.

## 5. Transform gizmos and manipulators

**Babylon.js.** The gizmo system is documented in a dedicated feature page describing attaching gizmos to meshes, bones, and other transform-bearing objects [s7]. This is the primary, and only, gizmo-specific Babylon source in the evidence; recommendations about utility-layer placement, disabling camera input during drag, and per-drag undo transactions go beyond what this single source explicitly states and should be read as standard implementation practice layered on top of the documented feature, not sourced claims in themselves.

**Unreal Engine.** This sub-topic has the richest official documentation in the whole evidence set: `UInteractiveGizmo` as the base gizmo abstraction [s34] with a documented `Setup` method [s44], `UTransformGizmo` for standard transform interaction against a `UTransformProxy` [s33], `UCombinedTransformGizmo` for the combined transform experience [s36], `UInteractiveGizmoManager` for gizmo lifecycle [s41], and the `InteractiveToolsFramework` overview tying them together [s45]. Note that these are Editor/EditorInteractiveToolsFramework and Runtime/InteractiveToolsFramework classes — the evidence does not state whether or how this framework is intended for runtime gameplay manipulators versus editor/tool-building contexts specifically; the recommendation that gameplay code should typically use a lighter custom manipulator instead is **not stated in any source** and is offered here as engineering judgment, explicitly unsupported by citation.

## 6. Drag and constrained manipulation

No source in the evidence describes the mechanics of constrained dragging (ray-plane intersection, axis projection, angle accumulation for rotation) for either engine. The one directly relevant, sourced fact is a Babylon changelog note that `ActionManager`'s `OnPickTrigger` no longer fires for a drag/swipe gesture, with `OnPickDownTrigger` available for down-state semantics instead [s19] — a genuine, citable behavioral detail about how Babylon distinguishes click from drag at the trigger level. Beyond that single fact, all guidance about plane/axis constraint math, drag-threshold detection, and transaction/undo structuring in this area is general 3D-interaction practice, **not established by the provided sources**, and should not be read as sourced.

## 7. Hit-test performance at scale

**Babylon.js.** The evidence supports that `GPUPicker` exists and can be configured with an explicit picking list, based on the forum/wiki sources already discussed [s61][s63][s64][s67]. No source in the evidence documents `isPickable`-based filtering, bounding-volume pre-tests, or spatial partitioning as an explicit performance recommendation for large scenes; the general `Meshes` feature page [s29] covers mesh properties broadly but its title does not indicate specific performance-at-scale picking guidance.

**Unreal Engine.** `LineTraceByChannel`'s documented `Trace Complex` parameter [s37] is the one concrete, sourced fact supporting the general principle that complex (per-triangle) collision is more expensive than simple collision and is an explicit opt-in — this does support the recommendation to avoid complex traces for routine hover/picking. `MultiLineTraceByChannel` [s65] supports the narrower recommendation to use multi-hit traces only when multiple depth candidates are genuinely needed. Beyond these two facts, broader claims about collision channels, spatial partitioning, or streaming interactive actors at scale are general engineering practice not established by these sources.

## 8. Touch and XR controller input

**Babylon.js.** This is well evidenced. `PointerObservable`/`PointerInfo` is presented as a device-agnostic pointer path covering mouse and touch alike [s2][s9], with forum discussion of tracking WebXR pointer state during move events as a practical extension of that same model [s15]. WebXR input is documented through `WebXRInputSource` (motion controller, world-space pointer ray) [s6], the controller lifecycle managed by the WebXR input system [s1], the introductory WebXR feature overview [s5], and `WebXRNearInteraction` for near-hand/controller interaction, including identifying the mesh under a controller pointer [s4].

**Unreal Engine.** Enhanced Input is documented as a device-abstraction input system across multiple engine-version pages [s31][s32][s39][s40], though none of the source titles specifically calls out touch-input handling as a named capability — this is a real gap: the evidence establishes Enhanced Input as the general input abstraction layer but does not specifically document touch-gesture handling within it. World-space UI interaction is documented via `WidgetInteractionComponent`, which raycasts against a `WidgetComponent` and is documented at the API level [s46][s52], with additional locale-mirrored documentation [s59] and a Python API binding [s60]. AR-specific hit testing is a separate, explicitly documented feature relevant to touch/mobile AR contexts [s35].

## 9. Accessibility

**Babylon.js.** Evidence here consists of a single feature page describing an accessibility layer that can generate HTML "twin" elements exposing scene interactions defined through `ActionManager`, specifically supporting pick, left-pick, and right-pick triggers for keyboard/assistive-tech users [s30]. This is real and directly on-topic, but it is thin — one page, focused on `ActionManager`-driven interactions rather than a general accessible-object-model for arbitrary 3D content.

**Unreal Engine.** This is comparatively well evidenced for 2D/UMG accessibility: official screen-reader support documentation exists across several locales/versions [s49][s53][s54][s55], a broader "Blind Accessibility Features Overview" [s56], an `FScreenReaderUser` API reference [s58], `CommonUI` design guidance and API [s47][s48], `UWidget` API references [s50][s51], and general UMG/Slate UI-authoring documentation [s57]. Notably, **none of these sources describe accessibility for the 3D viewport or gizmo/manipulator interaction itself** — they document screen-reader support for UMG widgets. The recommendation that every 3D transform operation should have a keyboard/screen-reader-accessible UMG equivalent is a reasonable extension of what these sources establish, but the sources do not themselves claim that 3D gizmo interaction is, or should be, exposed this way — that specific bridge is unsupported by citation and is presented here as a design recommendation, not a documented fact.

## 10. Where the evidence disagrees or is internally inconsistent

No two sources make directly contradictory factual claims about the same mechanism. But two real inconsistencies are worth surfacing rather than smoothing over:

- **Babylon GPU picking's maturity is itself unsettled in the evidence.** The sequence of forum threads — an announcement [s63], a subsequent request asking that it be made available "as a picking option at scene level" [s64], and a later follow-up thread [s67] — reads as a feature whose integration point in the standard picking API was still being negotiated by the community, not a finished, canonical scene-level option from the outset. Anyone relying on GPU picking as a drop-in scene-level default should verify current API shape against the release notes rather than assume it matches ordinary `scene.pick` ergonomics.
- **Unreal's outline/custom-depth technique has no single canonical source.** The evidence contains a third-party guide [s68], two community forum threads describing different specifics (selective custom depth per object [s70]; replicating the editor's outline look [s71]), an older forum thread about outline materials [s66], and an old bug-tracker issue [s72]. These are not contradictory claims so much as a scattered, multi-year, community-patched set of approaches to the same problem, with no first-party Epic doc consolidating them in this evidence set — meaning "the" recommended way to do selection outlining in Unreal is not actually established by the sources, only a range of workable community approaches.

## 11. What remains unsettled, and what would settle it

- **Whether `GPUPicker` is now a fully integrated, scene-level-default picking path in current stable Babylon.js**, or still an opt-in feature layered on top of ray picking. Settled by: the current Babylon.js `Scene` API reference/changelog entry for the picking-mode option, not forum discussion.
- **The canonical, currently-recommended Unreal technique for selection/hover outlining.** Settled by: an official Epic documentation page (not forum posts or third-party guides) describing the supported custom-depth/stencil or equivalent workflow for the current engine version.
- **Whether Unreal's Interactive Tools Framework gizmos are intended/supported for runtime gameplay use, or are editor/tool-building constructs only.** Settled by: an Epic doc or source explicitly scoping `UInteractiveGizmo`/`UTransformGizmo` usage outside editor tooling.
- **Whether Enhanced Input has first-class, documented touch-gesture handling** (as opposed to only mouse/gamepad/motion-controller mapping). Settled by: an Enhanced Input doc page or API reference explicitly covering touch input triggers/modifiers.
- **Constrained-manipulation math (plane/axis dragging, rotation-angle accumulation) for either engine** is not covered by any source here and remains entirely a matter of general practice rather than documented engine guidance. Settled by: an official tutorial or sample from either engine demonstrating the gizmo drag math.
- **A documented bridge between 3D interaction (picking, gizmos) and accessibility APIs** in either engine. Settled by: a source that explicitly connects object selection/manipulation state to a screen-reader-exposed model, beyond Babylon's single `ActionManager`-twin page [s30] and Unreal's UMG-only screen-reader docs.

## Sources

1. [WebXR Controllers Support | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/webXR/webXRInputControllerSupport) — other
2. [PointerInfo | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.PointerInfo) — other
3. [Babylon.js](https://github.com/BabylonJS/) — docs
4. [WebXRNearInteraction | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.WebXRNearInteraction) — other
5. [ES6 support with Tree Shaking](https://doc.babylonjs.com/features/featuresDeepDive/webXR/introToWebXR) — other
6. [WebXRInputSource - Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.WebXRInputSource) — other
7. [Documentation/content/features/featuresDeepDive/mesh/gizmo.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/mesh/gizmo.md) — docs
8. [GitHub - BabylonJS/Babylon.js: Babylon.js is a powerful, beautiful, simple, and open game and rendering engine packed into a friendly JavaScript framework.](https://github.com/BabylonJS/Babylon.js/) — docs
9. [Interacting With Scenes | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/scene/interactWithScenes) — other
10. [PointerInfoPre](https://doc.babylonjs.com/typedoc/classes/BABYLON.PointerInfoPre) — other
11. [GitHub - BabylonJS/Documentation: Babylon.js's documentation ...](https://github.com/BabylonJS/Documentation) — docs
12. [Listening to click events on mesh - Babylon.js Forum](https://forum.babylonjs.com/t/listening-to-click-events-on-mesh/42068) — other
13. [GUI | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/modules/BABYLON.GUI) — other
14. [Documentation/content/features/featuresDeepDive/mesh ...](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/mesh/gaussianSplatting.md) — docs
15. [Tracking webxr pointer during move events - Questions](https://forum.babylonjs.com/t/tracking-webxr-pointer-during-move-events/41330) — other
16. [Actions | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/events/actions) — other
17. [HighlightLayer](https://doc.babylonjs.com/typedoc/classes/BABYLON.HighlightLayer) — other
18. [ActionManager | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.ActionManager) — other
19. [What's New](https://doc.babylonjs.com/whats-new) — other
20. [ActionManager](https://doc.babylonjs.com/typedoc/classes/BABYLON.ActionManager) — other
21. [Highlighting Meshes | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/mesh/highlightLayer) — other
22. [Gaussian Splatting | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/mesh/gaussianSplatting) — other
23. [Selection Outline](https://doc.babylonjs.com/features/featuresDeepDive/mesh/selectionOutlineLayer/) — other
24. [Scene - Babylon.js documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.Scene) — other
25. [SelectionOutlineLayer - Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/babylon.selectionoutlinelayer) — other
26. [Babylon.js docs](https://doc.babylonjs.com/) — other
27. [EffectLayer | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.EffectLayer) — other
28. [FrameGraphSelectionOutlineLay...](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.FrameGraphSelectionOutlineLayerTask) — other
29. [Meshes | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/mesh/) — other
30. [Interaction](https://doc.babylonjs.com/toolsAndResources/accessibility/screenReaders) — other
31. [Enhanced Input in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input-in-unreal-engine) — other
32. [Enhanced Input in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/unreal-engine/enhanced-input-in-unreal-engine) — other
33. [UTransformGizmo | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Editor/EditorInteractiveToolsFramework/EditorGizmos/UTransformGizmo) — other
34. [UInteractiveGizmo | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/InteractiveToolsFramework/UInteractiveGizmo) — other
35. [How To Perform AR Hit Testing in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/how-to-perform-ar-hit-testing-in-unreal-engine) — other
36. [UCombinedTransformGizmo | Unreal Engine 5.6 Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/InteractiveToolsFramework/UCombinedTransformGizmo) — other
37. [Line Trace By Channel | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Collision/LineTraceByChannel) — other
38. [Line Trace | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/GeoReferencing/Utilities/LineTrace) — other
39. [Enhanced Input | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/enhanced-input?application_version=4.27) — other
40. [EnhancedInput | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/EnhancedInput) — other
41. [UInteractiveGizmoManager | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/InteractiveToolsFramework/UInteractiveGizmoManager) — other
42. [Transforming Actors in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/transforming-actors-in-unreal-engine) — other
43. [Break Hit Result | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Collision/BreakHitResult) — other
44. [UInteractiveGizmo::Setup | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/InteractiveToolsFramework/UInteractiveGizmo/Setup) — other
45. [InteractiveToolsFramework | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/InteractiveToolsFramework) — other
46. [UMG Widget Interaction Components in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/umg-widget-interaction-components-in-unreal-engine) — other
47. [Design Guidelines for Using CommonUI in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/design-guidelines-for-using-commonui-in-unreal-engine) — other
48. [CommonUI | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/CommonUI) — other
49. [Supporting Screen Readers in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/supporting-screen-readers-in-unreal-engine) — other
50. [UWidget | Unreal Engine 5.6 Documentation - Epic Games Developers](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/UWidget) — other
51. [UWidget | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/UWidget) — other
52. [UWidgetInteractionComponent::UWidgetInteractionComponent | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/UWidgetInteractionComponent/__ctor) — other
53. [언리얼 엔진의 스크린 리더 지원 | 언리얼 엔진 5.5 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/supporting-screen-readers-in-unreal-engine) — other
54. [虚幻引擎中的屏幕朗读器支持 | 虚幻引擎 5.5 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/supporting-screen-readers-in-unreal-engine) — other
55. [Supporting Screen Readers | 언리얼 엔진 4.27 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/supporting-screen-readers?application_version=4.27) — other
56. [Blind Accessibility Features Overview in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/blind-accessibility-features-overview-in-unreal-engine) — other
57. [Creating User Interfaces With UMG and Slate in Unreal ...](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-user-interfaces-with-umg-and-slate-in-unreal-engine) — other
58. [FScreenReaderUser | Unreal Engine 5.7 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/ScreenReader/FScreenReaderUser) — other
59. [언리얼 엔진의 UMG 위젯 인터랙션 컴포넌트 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/umg-widget-interaction-components-in-unreal-engine) — other
60. [unreal.WidgetInteractionComponent - Epic Games Developers](http://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/WidgetInteractionComponent?application_version=5.3) — other
61. [GPU Picking | BabylonJS/Babylon-Lite | DeepWiki](https://deepwiki.com/BabylonJS/Babylon-Lite/4.3-gpu-picking) — other
62. [WebGPU Breaking Changes](https://doc.babylonjs.com/setup/support/webGPU/webGPUBreakingChanges/) — other
63. [New feature: GPU Picking - Announcements](https://forum.babylonjs.com/t/new-feature-gpu-picking/51106) — other
64. [Add gpu picking as picking option at scene level](https://forum.babylonjs.com/t/add-gpu-picking-as-picking-option-at-scene-level/57666) — other
65. [Multi Line Trace By Channel | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Collision/MultiLineTraceByChannel) — other
66. [Material for Orange Outlines on Selected Actor - Rendering](https://forums.unrealengine.com/t/material-for-orange-outlines-on-selected-actor/8020) — other
67. [New feature: GPU Picking - Page 3 - Announcements - Babylon.js](https://forum.babylonjs.com/t/new-feature-gpu-picking/51106?page=3) — other
68. [How to Enable Custom Depth Stencil Pass in Unreal Engine and ...](https://eliaswick.com/resources/documentation/guides/enable-custom-depth) — other
69. [WebGPUEngine | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.WebGPUEngine) — other
70. [How to use custom depth selectively for certain objects? - Rendering](https://forums.unrealengine.com/t/how-to-use-custom-depth-selectively-for-certain-objects/842232) — other
71. [How can I create an outline highlight similar to the one used in the editor?](https://forums.unrealengine.com/t/how-can-i-create-an-outline-highlight-similar-to-the-one-used-in-the-editor/281631) — other
72. [The Unreal Engine Issues and Bug Tracker](https://issues.unrealengine.com/issue/UE-13738) — other
73. [GitHub - StudioGobo/UECollisionQueryTools: The collision query tools mentioned in the UnrealFest 23 talk Making Sense Of Collision Data](https://github.com/studiogobo/uecollisionquerytools) — docs
74. [Line Trace - Unreal Examples](https://unrealexamples.com/docs/line-trace-single-by-channel) — other
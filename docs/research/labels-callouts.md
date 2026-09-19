# Research: labels-callouts

- run id: `dr-fef5da29`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 59

## Confidence note (verbatim from the run's brief)

Confidence: medium. The two named engine features (Babylon.js AdvancedDynamicTexture/linkWithMesh/HtmlMesh; Unreal WidgetComponent/screen-space UMG) are real, well-documented, and the sources give concrete, citable API facts and known limitations for both. But the broader 'current best practices' framing of the question — decluttering algorithms, LOD-under-density fading policy, and cross-distance legibility thresholds — is only thinly supported: a handful of academic view-management papers touch on it, and much of any detailed taxonomy or algorithm beyond what is cited below is not attributable to a source in this list and must be treated as general domain knowledge, not evidence. Report separates sourced facts from unsupported general practice accordingly.

## Report

# For real-time 3D applications (games, engineering/data visualization, simulation), what are the current best practices for rendering and managing on-screen labels, annotations, and callouts — covering billboarding orientation modes, text rendering technique choice, leader-line/anchor behavior, depth-based occlusion handling, decluttering/collision-avoidance placement, level-of-detail fading under label density, and legibility across viewing distances — with concrete guidance and known limitations for two specific engine implementations: Babylon.js (GUI AdvancedDynamicTexture, linkWithMesh, HtmlMesh) and Unreal Engine (WidgetComponent, screen-space UMG)?

## 1. Coverage check: what the evidence actually establishes

The question names two concrete implementations — Babylon.js `AdvancedDynamicTexture` / `linkWithMesh` / `HtmlMesh`, and Unreal's `WidgetComponent` / screen-space UMG — and these ARE directly documented in the source list: Babylon's GUI system and 3D GUI controls [s1][s2][s3][s4][s6][s9][s13][s51], `HtmlMesh` [s7], and Unreal's Widget Components across several doc mirrors [s16][s17][s19][s21][s22][s23][s24][s25][s26][s27][s29][s30]. So the core subject of the question is real and evidenced, not adjacent-and-mistaken-for-real.

However, the question also asks for "current best practices" across seven cross-cutting UX/algorithmic topics (billboarding taxonomy, decluttering, LOD-under-density, leader-line routing, legibility across distances). For these, the source list contains only a small number of academic/technical papers on label placement and view management [s31][s32][s37][s42], plus scattered forum/blog threads showing practitioner problems rather than settled practice [s5][s8][s12][s15][s34][s35][s36][s38][s40][s48][s49][s54][s57]. There is no source here that is itself a "current best practices" guide for real-time 3D label systems. Where a claim below has no bracketed citation, it should be read as unverified general engineering practice, not something the provided evidence demonstrates — this report flags those explicitly rather than attaching a false citation to them.

## 2. Billboarding and orientation modes

**Sourced:** Unreal's `WidgetComponent` exposes a documented Geometry Mode choice between a flat **Plane** and a curved **Cylinder** projection for world-space widgets [s16][s17][s19][s21][s22][s30] — this is the concrete, engine-native equivalent of "cylindrical vs. planar billboard" the question asks about, and it is an actual, citable feature rather than a general concept. Babylon's forum and 3D-GUI documentation separately show that (a) 2D GUI controls attached to a mesh can be rotated to track a mesh's orientation via manual math [s5], and (b) users have run into billboard/occlusion interaction problems specifically [s15], indicating billboard orientation and depth visibility are handled as separate, not automatically reconciled, concerns in Babylon.

**Not directly sourced here:** a general taxonomy of spherical/cylindrical/screen-aligned/axial/fixed/camera-scaled billboarding, and rules for when to use each, is standard real-time-graphics practice but is not established by any single source in this list. Treat any such taxonomy as background domain knowledge, not evidence-backed guidance.

**Adjacent-but-different technology:** a VTK-specific issue about `vtkBillboardTextActor3D` rendering behind a raycast volume [s34] documents a real billboard/depth ordering bug, but VTK is not Babylon.js or Unreal — it is only weak, adjacent evidence that billboard-vs-depth ordering problems recur across 3D toolkits generally.

## 3. Text rendering technique choice

**Sourced:** Both engines offer more than one text path. Unreal has an official signed-distance-field (SDF) text feature, documented as giving resolution-independent scaling and efficient outline/distance effects, with an explicit caveat that it is an approximation that can lose quality at very small sizes or with thin/delicate glyphs and lacks normal hinting [s46][s47]. Babylon.js has a dedicated MSDF text add-on aimed at sharp, scale-independent 2D/3D text including billboarded and instanced paragraphs [s50], discussed further in Babylon's 3D GUI deep-dive documentation [s51]. Community threads on both sides corroborate real-world friction: Unreal forum reports of font/text artifacts [s48] and font-shadow behavior questions [s54], and community experiments rendering UE5 distance-field fonts [s49].

**Not directly sourced here:** general claims about when to prefer bitmap vs. SDF vs. DOM/HTML text for label counts, update rates, or accessibility are standard practice but not demonstrated by a specific source in this list beyond the SDF/MSDF caveats above.

## 4. Leader-line and anchor behavior

**Sourced, Babylon-specific:** Babylon's GUI `Line` control supports `connectedControl`, letting one endpoint track another GUI control — a documented, concrete building block for leader-line construction from a fixed anchor to a moving label [s1]. This is real and citable; it is not, however, a full leader-routing solver.

**Sourced, general literature:** Papers on "hedgehog" view management [s31] and dynamic annotation of interactive environments [s42] address anchor/label separation and the layout of connecting geometry in view-management contexts, and an AGILE 2015 paper addresses dynamic label routing for map-like point features [s37]. These support the general existence of academic leader-line/anchor research, but the sources do not give engine-specific leader-line APIs for Unreal (no Unreal leader-line control is documented in this list) — in Unreal, a leader line would need a custom UMG paint pass or line-rendering widget, which is not itself demonstrated by a specific source here.

## 5. Depth-based occlusion handling

This is one of the better-evidenced topics, and it is also where the clearest practical tension in the evidence shows up (see Section 11).

**Babylon:** Meshes expose an `occlusionType` property [s14], and Babylon's occlusion-queries feature documents that a queried mesh's visibility test is only meaningful if it is rendered after potential occluders, with depth-buffer state preserved across render groups [s44] — i.e., occlusion queries are a coarse, order-dependent tool, not a drop-in label-occlusion system. `HtmlMesh` content rendered as a scene mesh can be occluded by, and occlude, other meshes, per Babylon's own documentation [s6][s7]. Separately, forum threads show users needing manual workarounds to hide a `linkWithMesh`-style GUI placeholder when the tracked mesh is hidden by other 3D geometry [s8], and a distinct forum thread specifically about "billboard occlusion" confirms this is an open, recurring question for Babylon users [s15]. A GitHub feature request about reverse-depth-buffer support [s43] is tangential evidence that Babylon's depth-precision handling has known edge cases, relevant to occlusion accuracy at long range but not label-specific.

**Unreal:** The Widget Component documentation states plainly that world-space widgets participate in the 3D depth buffer (and so can be occluded) while screen-space widgets are rendered outside the 3D world and are therefore never depth-occluded [s16][s17][s19][s21][s22][s30]. Community threads confirm this is a genuine practical limitation people run into: one forum thread is explicitly titled around screen-space widget-component occlusion [s36], a Reddit thread separately asks about combining screen-space widgets with depth occlusion [s38], and another forum thread asks about z-ordering multiple screen-space text widgets [s40] — none of the provided sources show Epic providing an official, built-in solution for depth-occluding screen-space UMG; the pattern in the evidence is community-built workarounds (line traces, custom depth checks), not a documented first-party feature.

## 6. Decluttering and collision-avoidance placement

**Sourced:** Babylon documents a `moveToNonOverlappedPosition()` mechanism for GUI controls, using an `overlapGroup` and requiring manual invocation (e.g., from a render observer) — a real, but partial, built-in overlap-avoidance tool, not a priority-aware layout solver [s1]. Unreal's `WidgetComponent` exposes `ModifyProjectedLocalPosition`, an API hook that lets code adjust a widget's projected screen-space position after projection [s29] — this is the concrete mechanism a custom Unreal decluttering/collision-avoidance system would hook into, though the source does not itself describe a complete algorithm. On the research side, the Hedgehog Labeling paper [s31], a University of Würzburg/AGILE 2015 paper on dynamic label placement [s37], and a thesis on dynamic annotation of interactive environments [s42] are academic sources specifically about label placement and view management, supporting the general existence of force-directed/constrained placement and temporal-stability concerns in this literature, though they are not games-engine-specific and their exact algorithmic content is not detailed in the evidence provided.

**Not directly sourced here:** the specific "placement rings" scheme (ring 0/1/2/3, clustering into "12 sensors" style badges) is a plausible, common industry pattern but is not demonstrated by any source in this list — present it as general practice, not evidence-backed fact.

## 7. Level-of-detail fading under label density

This is the weakest-evidenced part of the question. No source in this list gives a specific fade-threshold, pixel-height cutoff, or density-based LOD policy for labels. The closest adjacent material is the same academic view-management papers [s31][s37][s42], which by their nature address handling many simultaneous annotations, and Unreal's UMG optimization guidance [s27] and invalidation/retainer mechanisms [s18][s24][s25][s26][s28], which address *rendering-cost* management under many widgets rather than *visual* LOD/fading policy per se. Any specific LOD table (full label → icon → cluster badge → hidden, with hysteresis) should be treated as general practice, not something these sources establish.

## 8. Legibility across viewing distances

Only indirectly covered. The SDF/MSDF sources [s46][s47][s50][s51] speak to legibility trade-offs at small sizes and across scale changes, and Microsoft's MRTK-for-Unreal text feature page [s53] is at least on-topic for VR/mixed-reality text legibility, though its specific content is not detailed in the evidence given. General rules about minimum character height, outlines/backing plates, and avoiding color-only encoding are standard UI practice but are not established by a specific source here.

## 9. Babylon.js: implementation guidance and known limitations

- `AdvancedDynamicTexture.linkWithMesh` tracks a mesh's projected screen position for a linked GUI control, with `linkOffsetX`/`linkOffsetY` for screen-space offsets, documented in Babylon's GUI docs [s1][s6]; the typedoc pages confirm the class surface [s2][s3][s4].
- Leader-line style connections can be built with a GUI `Line`'s `connectedControl` [s1].
- Overlap avoidance has a partial built-in tool (`moveToNonOverlappedPosition` + `overlapGroup`), requiring manual per-frame invocation [s1].
- Occlusion is not automatic for fullscreen/linked GUI: users have had to ask how to hide a linked placeholder when its mesh is hidden by other geometry [s8], and a separate thread specifically raises billboard-occlusion behavior as a problem [s15]. Babylon's occlusion-query feature [s44] and mesh `occlusionType` property [s14] are the available primitives, but they require correct render-order/depth-state handling and are not a turnkey label-occlusion system.
- `HtmlMesh` can be used as a genuine scene mesh (participating in occlusion) or as an overlay [s6][s7]; it is documented as suited to rich HTML content rather than large numbers of simple labels.
- MSDF text is available as an add-on for scale-independent text, including billboarded and instanced use [s50][s51].
- 3D GUI controls (distinct from fullscreen `AdvancedDynamicTexture`) are documented separately [s9][s13], and are the relevant place to look for billboard-mode behavior on 3D GUI containers, though the exact property semantics are not detailed in the evidence retrieved.

## 10. Unreal Engine: implementation guidance and known limitations

- `WidgetComponent` has two modes — World Space (depth-tested, can be occluded, supports Plane/Cylinder geometry) and Screen Space (rendered outside the 3D world, never depth-occluded) — documented consistently across multiple doc mirrors [s16][s17][s19][s21][s22][s23][s30]; the Python API confirms the same component surface for scripting [s20].
- `ModifyProjectedLocalPosition` gives a code hook to adjust a widget's projected position, the natural integration point for a custom decluttering/collision-avoidance pass [s29].
- Performance for many widgets is addressed through Slate/UMG invalidation [s18], the `bUseInvalidationInWorldSpace` flag for world-space widgets specifically [s24], Invalidation Boxes [s25], Retainer Boxes/Panels that render children to a texture and can update at a reduced rate [s26][s28], and official UMG optimization guidelines [s27].
- Screen-space occlusion is a documented, unresolved practitioner pain point: separate forum and Reddit threads ask specifically about occluding screen-space widget components with world depth [s36][s38], and another asks about z-ordering among screen-space text widgets [s40] — the sources show community workaround-seeking, not an official first-party feature for this.
- Text rendering can use SDF fonts for scale-independence, with documented small-size/thin-glyph/hinting caveats [s46][s47]; community threads separately report font artifacts [s48], font-shadow questions [s54], and custom-font rendering difficulties [s57].
- A custom-nameplate forum thread [s35] and a general Unreal UI blog post [s59] indicate practitioners commonly build bespoke world-space nameplate/label systems rather than relying solely on the stock `WidgetComponent` defaults, though neither source gives a validated best-practice recipe.

## 11. Where the evidence shows real tension (not averaged away)

The clearest, evidence-backed tension is between engine design intent and practitioner need on **screen-space depth occlusion**: both Babylon (fullscreen `AdvancedDynamicTexture`/`linkWithMesh`) and Unreal (screen-space `WidgetComponent`/UMG) are documented as rendering their screen-space layer without native depth occlusion [s1][s6][s16][s17]. In both ecosystems, the sources show unresolved community requests for depth-aware screen-space labels — Babylon users asking how to hide a linked placeholder behind geometry [s8] and separately raising billboard/occlusion interaction problems [s15]; Unreal users asking directly about screen-space widget occlusion [s36][s38] and z-ordering [s40]. None of the provided sources show either engine vendor shipping a complete, official solution to this; the available primitives (Babylon occlusion queries [s44], mesh `occlusionType` [s14]; Unreal's world-space mode and `ModifyProjectedLocalPosition` [s29]) are building blocks, not finished features. This is a genuine, source-supported gap between documentation and community need — it should not be smoothed into a claim that either engine "handles occlusion well" for screen-space labels.

## 12. What remains unsettled, and what would settle it

- **Unsettled:** whether there is an engine-recommended (rather than community-improvised) pattern for depth-occluding screen-space labels in either engine. This would be settled by an official Epic or Babylon.js engineering document or changelog entry specifically addressing screen-space depth occlusion, which is not present in this source set.
- **Unsettled:** concrete LOD/fade thresholds and decluttering algorithms for dense label sets in real-time 3D applications. This would be settled by a dedicated, engine- or domain-specific best-practices article/benchmark (e.g., from Epic, Babylon.js core team, or a peer-reviewed HCI/visualization venue) rather than the general view-management papers currently available [s31][s37][s42].
- **Unsettled:** whether Babylon's 3D GUI (`Container3D`, mesh-based controls) natively exposes the same billboard-mode granularity Unreal's `WidgetComponent` documents (Plane/Cylinder) — the relevant Babylon pages are in evidence [s9][s13][s51] but their exact billboard-mode property semantics are not detailed here; direct inspection of those pages' content would settle it.
- **Unsettled:** legibility thresholds (minimum pixel/angular size, contrast rules) for labels across viewing distances in either engine — no source here provides calibrated numbers.

## Sources

1. [The Babylon GUI | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/gui/gui) — other
2. [AdvancedDynamicTexture | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.GUI.AdvancedDynamicTexture) — other
3. [AdvancedDynamicTexture | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_gui.AdvancedDynamicTexture) — other
4. [GUI | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/modules/BABYLON.GUI) — other
5. [How can you rotate elements in 2D with respect to a mesh or model ...](https://forum.babylonjs.com/t/how-can-you-rotate-elements-in-2d-with-respect-to-a-mesh-or-model-in-3d-in-the-scene/19337) — other
6. [GUI | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/gui) — other
7. [Documentation/content/addons/htmlMesh.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/addons/htmlMesh.md) — docs
8. [Hide GUI placeholder when it's hidden by 3d model - Babylon.js](https://forum.babylonjs.com/t/hide-gui-placeholder-when-its-hidden-by-3d-model/6113) — other
9. [3D GUI Controls | BabylonJS/Babylon.js | DeepWiki](https://deepwiki.com/BabylonJS/Babylon.js/12.2-3d-gui-controls) — other
10. [2D GUI - Reactylon](https://www.reactylon.com/docs/gui/2d) — other
11. [Babylon.js docs](https://doc.babylonjs.com/addons) — other
12. [GUI Rectangle remains stable](https://forum.babylonjs.com/t/gui-rectangle-remains-stable/44609) — other
13. [Babylon 3D GUI](https://doc.babylonjs.com/features/featuresDeepDive/gui/gui3D) — other
14. [occlusionType](https://doc.babylonjs.com/typedoc/classes/BABYLON.Mesh) — other
15. [Billboard occlusion - Questions - Babylon.js](https://forum.babylonjs.com/t/billboard-occlusion/30216) — other
16. [Widget Components in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/widget-components-in-unreal-engine) — other
17. [Widget Components in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/widget-components-in-unreal-engine) — other
18. [Invalidation in Slate and UMG for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/invalidation-in-slate-and-umg-for-unreal-engine) — other
19. [언리얼 엔진의 위젯 컴포넌트 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/widget-components-in-unreal-engine) — other
20. [unreal.WidgetComponent¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/WidgetComponent?application_version=5.1) — other
21. [Widget Components in Unreal Engine](https://dev.epicgames.com/documentation/ru-ru/unreal-engine/widget-components-in-unreal-engine?application_version=5.6) — other
22. [Widget コンポーネント | Unreal Engine 4.27 ドキュメンテーション | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/widget-components?application_version=4.27) — other
23. [UWidgetComponent | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/UWidgetComponent) — other
24. [bUseInvalidationInWorldSpace | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/UWidgetComponent/bUseInvalidationInWorldSpace) — other
25. [Using the Invalidation Box for UMG in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-invalidation-box-for-umg-in-unreal-engine) — other
26. [URetainerBox | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/URetainerBox) — other
27. [Optimization Guidelines for UMG in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/optimization-guidelines-for-umg-in-unreal-engine) — other
28. [unreal.RetainerBox¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/RetainerBox?application_version=5.1) — other
29. [UWidgetComponent::ModifyProjectedLocalPosition | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/Components/UWidgetComponent/ModifyProjectedLocalPosition) — other
30. [Widget Components in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/pl-pl/unreal-engine/widget-components-in-unreal-engine%3Fapplication_version=5.3%3Fapplication_version=5.3%3Fapplication_version=5.3%3Fapplication_version=5.3%3Fapplication_version=5.3) — other
31. [Hedgehog Labeling: View Management Techniques](https://www.markustatzgern.com/papers/2014_vr_hedgehog.pdf) — other
32. [Institut für Informatik](https://mediatum.ub.tum.de/doc/1195222/1195222.pdf) — other
33. [Advances in Real-Time Rendering in Games](https://www.advances.realtimerendering.com/s2015/DynamicOcclusionWithSignedDistanceFields.pdf) — other
34. [vtkBillboardTextActor3D text always renders behind raycast volume](https://discourse.vtk.org/t/vtkbillboardtextactor3d-text-always-renders-behind-raycast-volume/983) — other
35. [Custom nameplates in 3d worldspace - Unreal Engine Forums](https://forums.unrealengine.com/t/custom-nameplates-in-3d-worldspace/141145) — other
36. [Widget component 'screen space' occlusion. - Unreal Engine Forum](https://forums.unrealengine.com/t/widget-component-screen-space-occlusion/23449) — other
37. [F. Bac¸˜ao et al. (eds.): AGILE 2015, Geographic Information Science as an Enabler](https://www1.pub.informatik.uni-wuerzburg.de/pub/schwartges/dynaroutelab/smhw-lsrim-agile15.pdf) — other
38. [Screen space widgets with depth occlusion](https://www.reddit.com/r/unrealengine/comments/1qshvz2/screen_space_widgets_with_depth_occlusion/) — other
39. [[PDF] Game Graphics & Real-time Rendering - UCSC Creative Coding](https://creativecoding.soe.ucsc.edu/courses/cmpm163/slides/W6_Thurs.pdf) — academic
40. [Can I ordering a screen space text widget? - Unreal Engine Forums](https://forums.unrealengine.com/t/can-i-ordering-a-screen-space-text-widget/437492) — other
41. [World-Space RenderTargets - Rive](https://rive.app/docs/game-runtimes/unreal/in-world-textures) — other
42. [Dynamic Annotation of Interactive Environments](https://otik.uk.zcu.cz/bitstream/11025/6862/1/Maass.pdf) — other
43. [Feature: Reverse Depth Buffer (z-buffer) · Issue #7133 · BabylonJS/Babylon.js](https://github.com/BabylonJS/Babylon.js/issues/7133) — docs
44. [Occlusion Queries - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/occlusionQueries) — other
45. [[Unreal Engine] Billboard - velog](https://velog.io/@imeamangryang/Unreal-Engine-Billboard) — other
46. [Using Signed Distance Field Text Rendering in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-signed-distance-field-text-rendering-in-unreal-engine) — other
47. [언리얼 엔진에서 부호화된 디스턴스 필드 텍스트 렌더링 사용하기 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/using-signed-distance-field-text-rendering-in-unreal-engine) — other
48. [What's the deal with weird text artifacts? - Rendering](https://forums.unrealengine.com/t/whats-the-deal-with-weird-text-artifacts/45165) — other
49. [【UE5】Font の Distance Field を試してみる #UI](https://qiita.com/MumeiKibou/items/52871665e7131be2d16d) — other
50. [MSDF Text | Babylon.js Documentation](https://doc.babylonjs.com/addons/msdfText/) — other
51. [Documentation/content/features/featuresDeepDive/gui/gui3D.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/gui/gui3D.md) — docs
52. [Text rendering using multi channel signed distance fields](https://news.ycombinator.com/item?id=20020664) — other
53. [Text](https://learn.microsoft.com/en-us/previous-versions/mixed-reality/mrtk-unreal/ux-tools/features/text) — news
54. [Ue5.5 font shadows - UI - Epic Developer Community Forums](https://forums.unrealengine.com/t/ue5-5-font-shadows/2186579) — other
55. [Babylon.GUI官方文档翻译- ljzc002](https://www.cnblogs.com/ljzc002/p/7699162.html) — other
56. [babylon-gui文档笔记](https://blog.csdn.net/weixin_45727272/article/details/108098654) — other
57. [How to make clean big text renderers with custom fonts?](https://forums.unrealengine.com/t/how-to-make-clean-big-text-renderers-with-custom-fonts/328859) — other
58. [JSDoc: Source: ui/widget/text-area.js](https://www.vrspace.org/babylon/jsdoc/ui_widget_text-area.js.html) — other
59. [Working with User Interface (UI) in Unreal Engine](https://vrealmatic.com/unreal-engine/ui) — other
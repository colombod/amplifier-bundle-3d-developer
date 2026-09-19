# Research: camera-navigation

- run id: `dr-4d67b519`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 70

## Confidence note (verbatim from the run's brief)

Confidence: medium. The named Babylon.js classes (ArcRotateCamera, UniversalCamera/FreeCamera, FollowCamera) and Unreal tools (Spring Arm, Camera/CineCameraActor, Sequencer) are all directly documented in the sources, and clipping-plane/z-fighting/log-depth guidance is well supported by multiple graphics references. But most cross-engine UX rules requested — fit-to-view margins, transition-duration numbers, minimap patterns, motion-sickness rules — have no dedicated source in this set and are general engineering convention, not sourced fact. No source documents a distinct Babylon 'FlyCamera' or a standalone Unreal 'CameraActor' page (only its underlying component). Several Spring Arm/Orthographic sources are the same document in different languages/versions, not independent corroboration.

## Report

# For 3D visualization/interactive applications (general-purpose, not restricted to a single industry), what are the current best practices for: (1) choosing among orbit/arcball, fly, first-person, and turntable camera controls; (2) focus/frame-selection ('fit to view') behavior; (3) smooth camera transitions and easing; (4) near/far clipping plane settings and depth-buffer precision (e.g., z-fighting, logarithmic depth buffers); (5) perspective vs orthographic projection choice; (6) navigation techniques for large-scale scenes; (7) minimap and overview+detail UI patterns; and (8) avoiding user disorientation or motion sickness — and specifically, how are these practices realized in Babylon.js's camera classes (ArcRotateCamera, UniversalCamera/FreeCamera, FollowCamera) and in Unreal Engine's Spring Arm component, Camera/CineCameraActor, and Sequencer cinematic camera tools?

## 1. Does the Evidence Cover What Was Asked?

Yes, for the engine-specific part of the question — with some gaps that matter. Every named class has direct documentation in the source list: ArcRotateCamera [s4][s11], FollowCamera [s1][s2], UniversalCamera/FreeCamera [s1][s3][s7], Spring Arm [s16][s17][s18][s20][s24][s28][s60], CineCameraActor/UCineCameraComponent [s29][s65][s69], and Sequencer's cinematic camera tooling [s19][s21][s23][s26][s27]. So this is not a case of adjacent-source drift on the named engine classes.

However, two omissions matter and are stated plainly rather than papered over:

- **No source documents a Babylon `FlyCamera` class.** Free-flight navigation in this report is therefore covered only through UniversalCamera/FreeCamera [s1][s3][s7]; any specific "FlyCamera" behavior is unsupported by this evidence set.
- **No source documents a plain Unreal `CameraActor` page.** The evidence covers `UCameraComponent` [s59][s62][s63] and `FMinimalViewInfo`/`MakeMinimalViewInfo` [s61][s68], which a Camera Actor wraps, but not the actor class itself.

Separately, the broad cross-industry "best practices" the question asks for (fit-to-view margins, transition-duration ranges, minimap UI conventions, motion-sickness rules) are **not supported by any general UX/HCI/game-design source in this list**. The list contains official engine API docs plus a strong cluster of computer-graphics references on depth-buffer precision [s31][s33][s34][s35][s36][s37][s38][s39][s40][s41][s42][s43][s44][s45], but nothing resembling a camera-navigation usability study. Those parts of the answer below are marked as general convention, not sourced claims.

## 2. Choosing Among Camera Control Modes

| Mode | Typical use | Engine realization |
|---|---|---|
| Orbit/turntable | Inspecting an object or bounded scene around a pivot | Babylon `ArcRotateCamera`: target, azimuth (`alpha`), elevation (`beta`), radius, always aimed at target [s4][s11] |
| Fly / free navigation | Environments with no single target | Babylon's current documentation presents `UniversalCamera` as the general free/first-person camera with keyboard, mouse, touch and gamepad input; older guide material still lists Universal and Free Camera separately [s1][s3][s7] |
| First-person | Walkthroughs, immersive environments | Same Universal/Free camera family in Babylon [s1][s3][s7] |
| Follow / third-person | Character or vehicle tracking | Babylon `FollowCamera`, positioned relative to a target mesh via `radius`, `heightOffset`, `rotationOffset` [s1][s2]; Unreal Spring Arm + attached Camera Component, which maintains a target arm length and reacts to obstruction [s18][s20][s25][s60] |
| Authored/cinematic | Fixed or planned camera moves for presentation | Unreal Cine Camera Actor / `UCineCameraComponent` [s29][s65][s69] driven through Sequencer [s19][s21][s23][s26][s27] |

A caveat on terminology: Babylon's `ArcRotateCamera` is a **target-centered orbit camera constrained by elevation limits**, not a mathematically unconstrained trackball/arcball. The class documentation itself frames it in terms of alpha/beta/radius/target rather than free virtual-sphere rotation [s4][s11]. Whether "arcball" behavior in the stricter sense is available depends on how a developer configures beta limits and roll — this is not separately documented in the sources provided.

For Unreal, the standard third-person pattern is Spring Arm plus attached Camera Component, with the arm retracting on obstruction and returning when clear [s18][s20][s25][s60]. Distinct plain `CameraActor` documentation was not found in the sources (see §1); what is documented is the underlying `UCameraComponent` [s59][s62][s63].

## 3. Focus / "Fit to View" Behavior

General practice (not tied to a specific source in this evidence set): compute a bounding volume for the selection, aim at its center or a semantic pivot, choose a distance from projected size and field of view, add a margin so the object doesn't touch the viewport edge, preserve the current azimuth/elevation where possible, and animate the move briefly and interruptibly. These are standard conventions, but no source here documents them as a formal specification — they are presented as general engineering practice, not as sourced fact.

What the sources do support directly:

- Babylon's `ArcRotateCamera` exposes a `zoomOn`-style operation to move to the minimum distance at which supplied meshes are fully visible, documented on the class itself [s4][s11].
- Babylon's behavior system, including `FramingBehavior`, provides an automated fit operation around a target mesh or bounds, with configurable options and a documented default framing duration; it can be set to stop when the user manually intervenes [s46][s51].
- Unreal's `CameraCutTrack` in Sequencer binds a camera to a section of the timeline and supports cuts between cameras [s19][s23]; this is the authored-cinematic analogue of "fit to view," not a runtime focus operation. Ordinary runtime "fit selection" logic for Unreal is not documented in this source set — it is application-level, built from actor bounds and a Sequencer- or code-driven blend.

## 4. Smooth Camera Transitions and Easing

General duration/curve guidance (ease-in/ease-out, quaternion interpolation, interruptibility) is standard convention and is **not sourced** to any item in this list — treat it as general practice, not an evidenced claim.

What is source-backed:

- Babylon's `FramingBehavior` supplies an animated framing transition with a configurable duration and a stop-on-user-zoom option [s46][s51].
- Babylon's `FollowCamera` approaches its goal position using acceleration and a maximum speed rather than teleporting [s1][s2].
- Unreal's Spring Arm exposes camera lag and camera rotation lag, with lag substepping intended to keep damping stable under variable frame rates. This is documented consistently across the Spring Arm guide, which appears in the source list under several language/version variants [s18][s20][s24][s28][s60] — these are the same underlying documentation, not independent confirmations.
- Unreal's Sequencer provides authored keyframe interpolation and camera cuts [s19][s21][s23][s26][s27], and its jib/dolly/crane rig tooling is the mechanism for tracking or crane-style camera moves [s70].

## 5. Clipping Planes and Depth-Buffer Precision

This is the most thoroughly sourced part of the evidence. The core rule supported across multiple technical references: perspective depth precision is non-linear and concentrated near the camera, so the **far/near ratio**, not the absolute far distance, governs precision and z-fighting risk [s36][s44]. A near plane of zero or an unnecessarily small value is a primary cause of instability [s38][s43].

Z-fighting itself — caused by coplanar geometry, excessive scene scale, or an overly large clip range — is treated consistently across course material, forum discussion, and vendor documentation [s33][s34][s35][s37][s38][s42]. For very large or planetary-scale scenes, sources describe multi-frustum or logarithmic depth strategies as a practical mitigation, with Cesium's hybrid multi-frustum log-depth approach and an academic comparison of depth-buffer techniques for large/detailed scenes offered as concrete implementations [s39][s40]. However, logarithmic depth is not a free fix: community reports describe depth-flickering artifacts even with logarithmic buffers [s41], and general graphics references note the near-plane/far-plane tradeoffs inherent to any single-frustum approach [s31][s32][s45].

Engine specifics:

- Babylon's `ArcRotateCamera` documentation gives concrete default clip values (`minZ`/`maxZ`) and explicitly warns that the depth buffer is finite, meaning a distant far plane can cause depth fighting [s4][s11]. These are documented defaults, not universal recommendations, and should be tuned to the scene's unit scale.
- Babylon supports an opt-in logarithmic depth buffer for Standard Materials, enabled via a material flag and falling back to linear depth when the browser lacks the needed extension [s49].
- Unreal exposes near/far clipping through camera view information (`FMinimalViewInfo`) and the Camera Component [s59][s62][s68], with additional custom clipping controls on the Cine Camera Component [s65][s69]. Orthographic cameras in Unreal expose explicit near/far clip settings and can auto-calculate orthographic planes from orthographic width [s30][s58][s61][s62].
- The Spring Arm's collision probe prevents the camera from entering geometry but is not a depth-precision mechanism and does not resolve z-fighting [s16][s17].

## 6. Perspective vs. Orthographic Projection

| Projection | Sourced support |
|---|---|
| Perspective | Default/general camera mode across Babylon's camera classes [s3][s9] and Unreal's Camera Component [s59][s62][s63] |
| Orthographic | Unreal has a dedicated Orthographic Camera feature with near/far clip and auto-calculated ortho planes from width [s30][s58][s61][s62]; the projection mode itself is switched through a documented Blueprint API [s66] |

No source in this set documents a formal recommendation on *when* to switch (e.g., "use orthographic for measurement, perspective for immersion") — that guidance is general convention, not sourced fact. What is sourced is that both engines expose the mode as an explicit, queryable/settable property rather than an implicit side effect of other camera parameters [s30][s58][s61][s62][s66].

## 7. Navigation in Large-Scale Scenes

Babylon documents a dedicated Geospatial camera path intended for map-like navigation around a spherical planet, distinct from the standard ArcRotate/Universal/Follow classes [s57]. Beyond this, general large-scene navigation heuristics (semantic level-of-detail, speed scaling by altitude/selection size, teleport bookmarks, origin rebasing) are **not documented by any source in this set** — no Unreal World Partition, level-streaming, or origin-rebasing documentation was provided, so claims about "use Unreal's large-world streaming architecture" should be read as general engineering convention, not an evidenced claim from these sources.

## 8. Minimap and Overview+Detail

The general pattern (persistent overview pane, frustum footprint drawn on the overview, linked selection, simplified render target) is standard practice but **not documented by any source here** as a formal UI specification.

The one concrete building block the evidence does support is Unreal's `USceneCaptureComponent2D`, which is the mechanism for rendering a secondary view (such as an overview camera) to a texture that a UI widget can display [s67]. No equivalent dedicated minimap mechanism is documented for Babylon in this source set — a second camera/viewport approach is plausible given Babylon's general camera architecture [s3][s9], but no source specifically describes it as a minimap pattern.

## 9. Avoiding Disorientation and Motion Sickness

Most of the specific mitigation rules requested (stable world-up, avoiding sudden teleportation, minimizing head bob/shake, comfort settings) are general VR/game-design convention and are **not supported by any source in this list** — there is no motion-sickness or comfort-design research source provided.

What the sources do support as concrete, relevant mechanisms:

- Babylon's `ArcRotateCamera` supports elevation limits that can prevent unwanted flips through the pole [s4][s11].
- Babylon's `FramingBehavior` animation is documented as interruptible/stoppable on user zoom, which avoids fighting the user for control during a transition [s46][s51].
- Unreal's Spring Arm collision probe prevents the camera from clipping through geometry, and its channel/probe-size properties are documented on the component API [s16][s17]. Camera lag and rotation lag (with substepping) are documented in the Spring Arm guide family, present under several locale/version duplicates [s18][s20][s24][s28][s60].
- Babylon separately documents a `WebXRCamera` class for immersive VR/AR [s53], which is outside the three classes the question asked about but is where VR-specific comfort settings would live in Babylon; this source does not itself detail motion-sickness mitigation.

## 10. Disagreement, Ambiguity, and Source-Quality Notes

- **Universal vs. Free Camera terminology is not fully consistent across the Babylon sources provided.** Current feature documentation appears to treat Universal Camera as the unified general-purpose camera, while other/legacy material in the set still lists Universal and Free Camera as separate entries [s1][s3][s7]. The sources do not resolve which framing is current; both are cited here rather than silently picking one.
- **The Spring Arm guide and the Orthographic Camera guide each appear multiple times as language/version variants** [s16][s17][s18][s20][s24][s28][s60] and [s30][s58] respectively. These are the same underlying documentation, not independent corroborating sources, and should not be read as five or two separate confirmations of the same fact.
- **Several sources in the list are third-party or tutorial-level restatements** of official Babylon concepts rather than primary documentation: a React wrapper's camera guide [s5], a Medium essay [s6], a Tutorialspoint tutorial [s12], a how.dev walkthrough [s14], and a Dart port's class reference [s15]. These are usable for general corroboration of camera taxonomy but carry less authority than the official Babylon docs and typed API references, and are not relied on for specific numeric claims in this report.
- No source in the list is a general UX, HCI, or game-design research source. All the "best practice" numbers and rules the question asks for beyond the two engines' own documented API behavior (margins, transition-duration ranges, minimap conventions, motion-sickness rules) are general engineering convention here, not something these sources establish.

## 11. What Remains Unsettled and What Would Settle It

- **Whether Babylon currently recommends Universal Camera exclusively or still treats Free Camera as a distinct, supported class** is unresolved from these sources; it would be settled by checking Babylon's current class-hierarchy listing directly rather than mixed-vintage guide pages.
- **Whether a Babylon `FlyCamera` exists and how it differs from Universal/Free Camera** is unsupported by any source here; it would require a source specifically documenting that class.
- **Exact numeric defaults** (fit-to-view margin percentages, transition durations, `FramingBehavior`'s exact default framing time) are asserted in the underlying documentation pages cited [s46][s51] but were not independently re-verified against the live class reference here; confirming them would require reading the literal default property values on the current `FramingBehavior` API page.
- **Unreal's runtime "fit selection" behavior** is not documented by any official source in this set; it is presented as an application-level responsibility. Settling this would require either an Editor-focus API reference or a first-party runtime-framing sample from Epic.
- **Large-world navigation and minimap implementation** in Unreal (streaming, World Partition, LOD) and in Babylon (dedicated minimap pattern) are not documented at all in this source set; settling these would require adding Unreal's World Partition/level-streaming documentation and a Babylon multi-viewport/minimap example to the evidence.
- **Motion-sickness mitigation** is entirely general convention here; settling it would require a VR/game-design comfort-guideline source, which is absent from this list.

## Sources

1. [Constructing a Follow Camera](https://doc.babylonjs.com/features/featuresDeepDive/cameras/camera_introduction) — other
2. [Additional Cameras - BabylonJS Guide](https://babylonjsguide.github.io/intermediate/Cameras) — other
3. [Cameras | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/cameras) — other
4. [ArcRotateCamera - Babylon.js docs](https://doc.babylonjs.com/typedoc/classes/BABYLON.ArcRotateCamera) — other
5. [Cameras | Reactylon](https://www.reactylon.com/docs/cameras) — other
6. [Cameras: Is what you see, what you get? | by Babylon.js](https://babylonjs.medium.com/cameras-is-what-you-see-what-you-get-c2c4d6a28207) — other
7. [Customizing Camera Inputs - Babylon.js documentation](https://doc.babylonjs.com/features/featuresDeepDive/cameras/customizingCameraInputs) — other
8. [ORTHOGRAPHIC_CAMERA](https://doc.babylonjs.com/typedoc/classes/BABYLON.FollowCamera) — other
9. [Camera | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.Camera) — other
10. [Camera Input Controllers | BabylonJS/Babylon.js | DeepWiki](https://deepwiki.com/BabylonJS/Babylon.js/4.2-camera-input-controllers) — other
11. [ArcRotateCamera | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.ArcRotateCamera) — other
12. [BabylonJS - Cameras - Tutorialspoint](https://www.tutorialspoint.com/babylonjs/babylonjs_cameras.htm) — other
13. [ArcFollowCamera](https://doc.babylonjs.com/typedoc/classes/BABYLON.ArcFollowCamera) — other
14. [BabylonJS cameras - Educative.io](https://how.dev/answers/babylonjs-cameras) — other
15. [ArcRotateCamera class](https://pub.dev/documentation/babylon_dart/latest/babylon/ArcRotateCamera-class.html) — other
16. [USpringArmComponent | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/GameFramework/USpringArmComponent) — other
17. [unreal.SpringArmComponent¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/SpringArmComponent?application_version=5.1) — other
18. [Using Spring Arm Components | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-spring-arm-components?application_version=4.27) — other
19. [Cinematic Camera Cut Track in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/cinematic-camera-cut-track-in-unreal-engine) — other
20. [Using Spring Arm Components in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/de-de/unreal-engine/using-spring-arm-components-in-unreal-engine) — other
21. [Cinematics and Movie Making in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/cinematics-and-movie-making-in-unreal-engine) — other
22. [Quick Start Guide to Components and Collision in Unreal Engine CPP | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/quick-start-guide-to-components-and-collision-in-unreal-engine-cpp) — other
23. [Creating Camera Cuts Using Sequencer in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-camera-cuts-using-sequencer-in-unreal-engine) — other
24. [使用虚幻引擎弹簧臂组件 | 虚幻引擎 5.5 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/using-spring-arm-components-in-unreal-engine) — other
25. [Camera Components | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/camera-components?application_version=4.27) — other
26. [How to Animate Cinematic Cameras in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/how-to-animate-cinematic-cameras-in-unreal-engine) — other
27. [Cameras in Sequencer | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/cameras-in-sequencer?application_version=4.27) — other
28. [언리얼 엔진의 스프링 암 컴포넌트 사용법 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/using-spring-arm-components-in-unreal-engine) — other
29. [Cine Camera Actor | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/cine-camera-actor?application_version=4.27) — other
30. [Orthographic Camera in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/orthographic-camera-in-unreal-engine) — other
31. [[PDF] Computer Graphics CSE 167 Lecture 4](https://cseweb.ucsd.edu/classes/wi20/cse167-a/lec4.pdf) — academic
32. [[PDF] Cascaded Deferred Rendering - Diva-Portal.org](https://www.diva-portal.org/smash/get/diva2:832419/FULLTEXT01.pdf) — other
33. [z-fighting – Computer Graphics 2023](https://dis.dankook.ac.kr/lectures/cg23/2023/11/10/z-fighting/) — other
34. [z-fighting – Computer Graphics 2025](https://dis.dankook.ac.kr/lectures/cg25/2025/11/05/z-fighting/) — other
35. [Practical Analysis on the Z-fighting and the Logarithmic Depth Tests for Computer Graphics](https://medium.com/@e92rodbearings/practical-analysis-on-the-z-fighting-and-the-logarithmic-depth-tests-for-computer-graphics-43509504e065) — other
36. [Depth Buffer Precision](https://www.khronos.org/opengl/wiki/Depth_Buffer_Precision) — other
37. [Avoiding Z-Fighting](https://gamedev.net/forums/topic/420434-avoiding-z-fighting/) — other
38. [12 The Depth Buffer](https://www.opengl.org/archives/resources/faq/technical/depthbuffer.htm) — other
39. [Hybrid Multi-Frustum Logarithmic Depth Buffer](https://cesium.com/blog/2018/05/24/logarithmic-depth/) — other
40. [Comparison of Depth Bu er Techniques for Large and ...](https://elib.dlr.de/187280/1/Comparison%20of%20Depth%20Buffer%20Techniques%20for%20Large%20and%20Detailed%203D%20Scenes.pdf) — other
41. [Depth flickering artifacts with logarithmic depth buffer](https://community.khronos.org/t/depth-flickering-artifacts-with-logarithmic-depth-buffer/108569) — other
42. [Large View Distances and Z-Fighting](https://www.gamedev.net/forums/topic/611591-large-view-distances-and-z-fighting/) — other
43. [back/front cliping plane ratio](https://community.khronos.org/t/back-front-cliping-plane-ratio/42948) — other
44. [Depth Precision Visualized – Nathan Reed's coding blog](https://www.reedbeta.com/blog/depth-precision-visualized/) — other
45. [Logarithmic Depth Buffer](https://www.gamedeveloper.com/programming/logarithmic-depth-buffer) — other
46. [FramingBehavior](https://doc.babylonjs.com/typedoc/classes/BABYLON.FramingBehavior) — other
47. [AnaglyphArcRotateCamera | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.AnaglyphArcRotateCamera) — other
48. [StereoscopicArcRotateCamera](https://doc.babylonjs.com/typedoc/classes/BABYLON.StereoscopicArcRotateCamera) — other
49. [Logarithmic Depth Buffer](https://doc.babylonjs.com/features/featuresDeepDive/materials/advanced/logarithmicDepthBuffer) — other
50. [Babylon.js docs](https://doc.babylonjs.com/) — other
51. [Camera Behaviors](https://doc.babylonjs.com/features/featuresDeepDive/behaviors/cameraBehaviors) — other
52. [Mathematics of the depth metric when generating shadow maps and ...](https://doc.babylonjs.com/features/featuresDeepDive/lights/mathShadows/) — other
53. [WebXRCamera](https://doc.babylonjs.com/typedoc/classes/BABYLON.WebXRCamera) — other
54. [SSAO2RenderingPipeline - Babylon.js docs](https://doc.babylonjs.com/typedoc/classes/BABYLON.SSAO2RenderingPipeline) — other
55. [TouchCamera | Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/_babylonjs_core.TouchCamera) — other
56. [Events](https://doc.babylonjs.com/features/featuresDeepDive/events) — other
57. [Geospatial | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/geospatial/) — other
58. [Orthographic Camera in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/orthographic-camera-in-unreal-engine?application_version=5.6) — other
59. [UCameraComponent | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Camera/UCameraComponent) — other
60. [Using Spring Arm Components in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-spring-arm-components-in-unreal-engine) — other
61. [Make Minimal View Info | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Utilities/Struct/MakeMinimalViewInfo) — other
62. [UCameraComponent | Unreal Engine 5.7 Documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/UCameraComponent) — other
63. [unreal.CameraComponent¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/CameraComponent?application_version=4.27) — other
64. [Unreal Engine Console Variables Reference | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-console-variables-reference?application_version=5.6) — other
65. [UCineCameraComponent | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/CinematicCamera/UCineCameraComponent) — other
66. [Set Projection Mode | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Camera/SetProjectionMode) — other
67. [USceneCaptureComponent2D | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Components/USceneCaptureComponent2D) — other
68. [FMinimalViewInfo | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/Camera/FMinimalViewInfo) — other
69. [unreal.CineCameraComponent¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/CineCameraComponent?application_version=5.2) — other
70. [Camera Jibs and Dollies in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/camera-jibs-and-dollies-in-unreal-engine) — other
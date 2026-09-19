# Research: review-and-quality

- run id: `dr-61e20e13`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 75

## Confidence note (verbatim from the run's brief)

Evidence solidly documents GPU/CPU profiling tools (RenderDoc, PIX, Unreal Insights/stat commands, Unity Profiler/GPU Usage Profiler, Android GPU Inspector, Xcode Metal tools, WebGPU/Chrome tracing) and Unity's Graphics Test Framework for image-comparison regression testing, plus Khronos specs on sRGB conversion and blend/premultiplied-alpha behavior. The sources do NOT establish formalized industry 'frame-budget checklists,' 'graphics code-review checklists,' or documented normal/tangent-space conventions — those are not artifacts described in any provided source, so claims about them are unsupported, not evidenced. Spector.js, named in the question, and vendor tools like Nsight/RGP/Snapdragon Profiler/PresentMon are also absent from the source list. Confidence: medium — roughly half the seven-part question (profiling, visual QA, color/alpha correctness) is well grounded; the rest (budgets, checklists, cross-hardware normal/tangent/depth pitfalls) rests on general engineering plausibility rather than the given evidence.

## Report

# In professional real-time 3D graphics development (game engines, game studios, and real-time rendering teams working with engines like Unreal Engine, Unity, or custom engines, targeting PC, console, and/or mobile platforms), what methodologies, tools, and checklists do teams currently use to: (1) profile GPU and CPU performance (e.g., RenderDoc, PIX, Unreal Insights/stat commands, Spector.js, browser GPU profiling tools), (2) analyze and enforce frame budgets, (3) identify and avoid common rendering performance anti-patterns, (4) perform visual quality QA and regression testing including golden-image or perceptual-diff testing, (5) validate rendering correctness and consistency across devices and mobile hardware, (6) detect and prevent numerical/correctness pitfalls such as color space and gamma handling errors, depth precision issues, normal/tangent space mistakes, and premultiplied alpha bugs, and (7) structure graphics-specific code review checklists?

## 1. What the evidence actually covers

The sources substantially cover professional-grade **profiling tools** across engines and platforms (RenderDoc, PIX, Unreal Insights/stat commands, Unity's Profiler/GPU Usage Profiler/Frame Timing Manager, Android GPU Inspector, Xcode's Metal capture/debugger, and several web-GPU profiling utilities), and they cover **Unity's Graphics Test Framework**, a genuine image-comparison/regression-testing system with documented settings (`ImageComparisonSettings`). They also include Khronos specifications describing sRGB texture/framebuffer conversion and blend-function behavior relevant to color space and premultiplied alpha, and Epic/Android documentation on mobile rendering guidance (overdraw from masked/translucent materials, material/texture practices).

However, several parts of the question are **not** established by the evidence as named, documented industry artifacts:

- No source describes a formal "frame budget" methodology, checklist, or numeric convention (e.g., 16.67 ms targets) — frame-timing *tools* are documented, but not a budget-setting or budget-enforcement checklist.
- No source documents a "graphics-specific code review checklist" as a named practice or artifact.
- No source discusses tangent/normal-space conventions (object vs. tangent space, handedness, TBN orthonormalization) at all.
- Depth-precision guidance (reverse-Z, near/far plane choice) is not documented by any source in usable form — the one candidate source, a raw HLSL shader file from Microsoft's DirectX-Graphics-Samples [s60], is code, not prose documentation, and its content on this topic cannot be confirmed from the source list.
- **Spector.js**, explicitly named in the question, is absent from the source list. The web-tooling sources instead cover WebGPU Inspector [s61], Chrome DevTools tracing/dropped-frame analysis [s64][s66][s67][s70][s73], and general WebGPU documentation [s70].
- Vendor tools frequently used in industry — Nsight Graphics, Radeon GPU Profiler, Snapdragon Profiler, PresentMon — are not in the source list and must not be treated as evidenced.
- One source, on GPU profiling for Apple Silicon in the context of LLM inference [s74], is about machine-learning workload profiling, not real-time 3D rendering, and is not used below.

Where the report below states something as supported, a citation is attached. Where the original question's specificity (checklists, thresholds, budgets, tangent/normal conventions) exceeds what the sources document, that is stated plainly rather than filled in.

## 2. GPU and CPU profiling tools and methodology (well supported)

**RenderDoc**: A frame-capture/replay debugger with a Performance Counter Viewer that samples available GPU counters per draw event [s1], a Python API for querying and fetching counter data programmatically [s7][s10], and general documentation of its capture/replay model [s13]. Meta/Oculus-specific guidance describes using RenderDoc (including a Meta fork) for profiling Quest/Oculus content and for Unreal-targeted mobile VR workflows [s5][s14].

**PIX (Windows/Xbox)**: Microsoft's PIX documentation covers GPU captures [s24], timing captures that combine CPU sampling with GPU timing data [s20], a tutorial for diagnosing CPU frame-time spikes using PIX [s26], and newer timing-capture features added in PIX 2201.24 [s27].

**Unreal Engine**: Epic's own documentation describes an "Introduction to Performance Profiling and Configuration" workflow [s16][s21], a legacy 4.27 "Performance and Profiling Overview" [s25], and Unreal Insights as a dedicated profiling system [s19][s23]. Third-party writeups (Intel [s17], GPUOpen [s18], and several blogs [s22][s29][s30]) describe practical use of `stat` commands and Insights for narrowing down CPU/GPU/render-thread bottlenecks; note that [s22][s29][s30] are secondary blog sources of unverified editorial rigor, not Epic documentation, and [s29] is a translated repost of the same content as [s17].

**Unity**: Unity's manual pages on "Graphics performance and profiling" (multiple engine versions) [s3][s4][s8][s9], the GPU Usage Profiler module (current and legacy versions) [s6][s11][s12][s15], and the Frame Timing Manager [s28] together document Unity's built-in CPU/GPU timing and profiling tooling.

**Mobile and console-adjacent tools**: Android GPU Inspector is documented for frame-trace capture and GPU analysis on Android [s63][s69]. Apple's Xcode Metal tooling is documented for capturing and debugging Metal workloads, including programmatic capture and GPU performance optimization guidance [s62][s65][s71][s72][s75].

**Web**: Chrome/WebGPU-specific profiling is documented through the WebGPU Inspector browser extension [s61], Chrome's WebGPU developer documentation [s70], a dedicated "Profiling WebGPU — Chrome CPU" guide [s64], guidance on interpreting dropped frames in Chrome DevTools [s66], a general roundup of web-game profiling tools [s67], and the older `about:tracing`-based WebGL profiling technique [s73]. A community-maintained "100 tips" document for three.js also touches on performance practice, though it is not official documentation [s68].

**Other engines**: vvvv (a visual live-programming environment, not a mainstream game engine) documents its own GPU debugging workflow [s2] — relevant only as a minor data point outside the Unreal/Unity/console mainstream the question centers on.

## 3. Frame budgets (weakly supported)

The evidence documents the *tools* that would be used to measure frame time (Unreal's stat/Insights tooling [s16][s17][s18][s19], Unity's Frame Timing Manager [s28], PIX timing captures [s20][s26][s27]) but does **not** document a named methodology, checklist, or numeric standard for defining, allocating, or enforcing a frame budget across CPU/GPU/thermal/bandwidth categories. Any statement of specific millisecond targets (e.g., 16.67 ms for 60 Hz) is arithmetic, not something attributed to a source here, and should not be presented as an industry-standard checklist. What the evidence supports is only that frame-timing measurement tools exist and are actively used and documented by engine vendors; how teams turn those numbers into a formal "budget" process is unsupported by these sources.

## 4. Rendering anti-patterns (partially supported)

Epic's mobile performance documentation explicitly discusses overdraw risk from masked and translucent materials and points to diagnostic views for finding costly shading work on mobile [s35]. A companion Epic mobile debugging/optimization document covers debugging and optimization workflows specific to mobile targets [s37], and Android developer documentation on Unreal memory optimization addresses memory-related mobile concerns [s36]. Android's general "Materials and Shaders" optimization guide recommends preferring opaque materials and efficient texture practices for mobile [s43]. Google's Filament documentation, as a real-time PBR rendering engine's own reference material, may describe rendering-architecture practices relevant to performance, though the source list gives only its top-level documentation/about pages rather than a specific anti-pattern checklist [s31][s42].

Beyond these mobile-specific and Filament-adjacent points, the evidence does **not** provide a general, cross-platform "anti-pattern list" (e.g., PSO/permutation explosion, synchronous readbacks, redundant intermediate render targets) as a documented industry checklist. Statements to that effect would be extrapolation, not sourced fact.

## 5. Visual quality QA and regression testing (well supported, Unity-specific)

Unity's Graphics Test Framework is directly documented across several package versions: an overview of the framework's purpose [s38][s39][s40][s41][s44][s45], and the `ImageComparisonSettings` API class, which defines configurable comparison behavior (documented across versions 7.3, 7.8, and 8.1 of the package) [s32][s33][s34]. This establishes that Unity teams have an official, versioned, documented mechanism for rendering a test image and comparing it against a reference image with configurable thresholds — the closest thing in this evidence set to a "golden image" testing system.

No equivalent documented framework for Unreal Engine, or for engine-agnostic perceptual-diff tooling (e.g., SSIM- or ΔE-based comparison outside Unity's own settings), appears in the source list. Any claim of a broader industry-standard perceptual-diff practice beyond Unity's own tooling is not supported here.

## 6. Cross-device and mobile correctness (partially supported)

Android GPU Inspector's frame-profiling capability [s63][s69] and Xcode's Metal capture/debugger tooling [s62][s65][s71][s72][s75] give concrete, vendor-documented mechanisms for inspecting GPU behavior on Android and Apple hardware respectively. Epic's mobile performance and debugging documentation [s35][s37] and Android's Unreal memory-optimization guidance [s36] describe Unreal-specific mobile concerns. This supports the general claim that mobile/console teams have platform-specific profiling and debugging tools available.

The evidence does **not** document a cross-platform device/API matrix methodology, minimum/recommended/high-end tiering practice, or thermal/battery-state test protocol as a named industry checklist. These would need to be flagged as inferred practice, not sourced fact, if included.

## 7. Numerical and rendering-correctness pitfalls (mixed support)

**Color space and gamma**: Khronos's OpenGL wiki page on required image formats [s51] and the OpenGL ES 3.0 and 3.2 specifications [s52][s49] document sRGB texture and framebuffer conversion behavior at the API level, and a Khronos community forum thread specifically discusses OpenGL color-space handling in practice [s48]. The OpenGL 2.0 and 3.3 specifications [s54][s50] and the framebuffer wiki page [s46] provide underlying API context. This is reasonably solid, specification-level support for color-space/gamma-handling pitfalls being a real, documented API concern.

**Premultiplied alpha and blending**: `glBlendFunc` reference pages for OpenGL ES 3.x [s53][s55] and the Khronos wiki's "Blending" article [s58] document the blend-factor mechanics relevant to straight vs. premultiplied alpha. The WebGL 1.0 specification explicitly defines conversions between premultiplied and non-premultiplied alpha at canvas and texture-upload boundaries [s56], which directly supports treating premultiplied-alpha handling as a real, spec-documented correctness concern. Microsoft's DirectXTK `CommonStates` wiki page [s59] documents predefined blend-state helpers (a plausible but unconfirmed link to premultiplied-alpha blend state conventions in a DirectX context).

**Depth precision**: Not documented by any source in usable form. The only depth/motion-vector-adjacent source is a raw HLSL compute shader file from Microsoft's DirectX-Graphics-Samples repository [s60], which is source code rather than prose guidance; no claim about reverse-Z or precision reasoning can be reliably attributed to it from the source list alone.

**Normal/tangent space**: Not addressed by any provided source. Any statement about tangent-space conventions, handedness, or TBN reconstruction would be entirely unsupported by this evidence set.

## 8. Graphics-specific code review checklists (not supported)

No source in the provided list documents a graphics- or rendering-specific code review checklist, pull-request template, or review methodology. This part of the question has no evidentiary support at all in the given sources; any detailed checklist here would be fabricated rather than sourced.

## 9. Disagreement

No direct disagreement between sources was found on the topics they do cover — the Unity manual pages across versions [s3][s4][s8][s9], the Graphics Test Framework pages across package versions [s38][s39][s40][s41][s44][s45], and the RenderDoc/PIX/Epic documentation are describing the same or evolving tools rather than competing claims. The secondary blog sources on Unreal profiling [s22][s29][s30] are consistent in substance with Epic's own documentation [s16][s17][s18][s21][s25] rather than contradicting it, though as non-official sources they carry less authority. No genuine methodological dispute (e.g., competing schools of thought on how to define a frame budget, or on perceptual-diff thresholds) is visible anywhere in this evidence set — but that absence reflects the narrow, mostly tool-documentation nature of the sources, not necessarily an absence of real disagreement in the field.

## 10. What remains unsettled and what would settle it

- **Frame budget methodology**: unsettled. Would require sourced material from engine vendors, platform holders (Sony/Microsoft/Nintendo cert requirements), or published GDC/technical-conference talks specifically defining budget allocation practice.
- **Graphics code review checklists**: unsettled. Would require studio engineering-practice documents, published style guides, or GDC-style talks on graphics code review.
- **Normal/tangent-space and depth-precision conventions**: unsettled. Would require shader-authoring guides, engine shader documentation (Unreal/Unity shader docs), or graphics-programming textbooks/specifications not present in this source list.
- **Cross-engine perceptual-diff standards**: partially unsettled. Unity's own framework is documented [s32][s33][s34][s38-s41][s44][s45]; whether other engines/studios use comparable, named, documented systems is not established here.
- **Spector.js and named vendor tools (Nsight, RGP, Snapdragon Profiler, PresentMon)**: unsettled in this evidence set entirely — none appear in the sources, so their actual current usage cannot be confirmed or denied from this evidence alone, only from sources not provided.

## Sources

1. [Performance Counter Viewer¶](https://renderdoc.org/docs/window/performance_counter_viewer.html) — other
2. [GPU Debugging in vvvv | vvvv gamma documentation](https://thegraybook.vvvv.org/reference/libraries/3d/gpu-debugging.html) — other
3. [Graphics performance and profiling - Unity - Manual](https://docs.unity3d.com/6000.0/Documentation/Manual/graphics-performance-profiling.html) — docs
4. [Unity - Manual: Graphics performance and profiling](https://docs.unity3d.com/6000.3/Documentation/Manual/graphics-performance-profiling.html) — docs
5. [How to Level Up Your Profiling With RenderDoc for Oculus](https://developers.meta.com/horizon/blog/how-to-level-up-your-profiling-with-renderdoc-for-oculus/) — other
6. [GPU Profiler - Unity - Manual](https://docs.unity3d.com/550/Documentation/Manual/ProfilerGPU.html) — docs
7. [API Reference: Performance Counters¶](https://renderdoc.org/docs/python_api/renderdoc/counters.html) — other
8. [Collect rendering performance data • Unity Editor • Unity Docs](https://docs.unity.com/en-us/engine/6000.0/manual/analysis/graphics-performance-profiling/profile-rendering) — docs
9. [Graphics performance and profiling](https://docs.unity3d.com/6000.2/Documentation/Manual/graphics-performance-profiling.html) — docs
10. [Fetch GPU Counter Data¶](https://renderdoc.org/docs/python_api/examples/renderdoc/fetch_counters.html) — other
11. [GPU Usage Profiler module - Unity - Manual](https://docs.unity3d.com/6000.5/Documentation/Manual/ProfilerGPU.html) — docs
12. [GPU Usage Profiler module](https://docs.unity3d.com/6000.0/Documentation/Manual/ProfilerGPU.html) — docs
13. [RenderDoc](https://renderdoc.org/docs/index.html) — other
14. [Use RenderDoc Meta Fork for GPU Profiling - Meta for Developers](https://developers.meta.com/horizon/documentation/unreal/ts-renderdoc-for-oculus/) — other
15. [GPU Usage Profiler module - Unity - Manual](https://docs.unity3d.com/Manual/ProfilerGPU.html) — docs
16. [Introduction to Performance Profiling and Configuration in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine) — other
17. [Unreal Engine Optimization Guide: Profiling Fundamentals](https://www.intel.com/content/www/us/en/developer/articles/technical/unreal-engine-optimization-profiling-fundamentals.html) — other
18. [Unreal Engine Performance Guide](https://gpuopen.com/learn/unreal-engine-performance-guide/) — other
19. [Profiling with Unreal Insights](https://unrealcommunity.wiki/6100e8169c9d1a89e0c34528) — other
20. [Profile the CPU and GPU with timing captures - Win32 apps](https://learn.microsoft.com/en-us/windows/win32/direct3dtools/pix/articles/timing-captures/pix-timing-captures) — news
21. [Introduction to Performance Profiling and Configuration in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/es-mx/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine) — other
22. [Unreal Engine 5 Optimisation: Profiling and Fixes](https://respawn.outlookindia.com/amp/story/gaming/gaming-guides/unreal-engine-optimization-profiling-and-fixing-common-bottlenecks) — other
23. [Unreal Insights Performance Profiling Guide | Bugnet Blog](https://bugnet.io/blog/unreal-insights-performance-profiling-guide) — other
24. [GPU Captures - PIX on Windows](https://devblogs.microsoft.com/pix/gpu-captures/) — news
25. [Performance and Profiling Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-and-profiling-overview?application_version=4.27) — other
26. [Tutorial: Using PIX to diagnose spikes in CPU frame time](https://developer.microsoft.com/en-us/games/articles/2024/02/tutorial-using-pix-to-diagnose-spikes-in-cpu-frame-time/) — docs
27. [PIX 2201.24: New Timing Capture Features - PIX on Windows](https://devblogs.microsoft.com/pix/pix-2201-24/) — news
28. [Unity - Manual: Frame timing manager introduction](https://docs.unity3d.com/6000.0/Documentation/Manual/frame-timing-manager.html) — docs
29. [[UE][번역] 언리얼 엔진 최적화 가이드: 프로파일링 기초](https://winterseaotter31.tistory.com/24) — other
30. [UE5 Performance Profiling 101: Finding and Fixing Bottlenecks](https://www.strayspark.studio/blog/ue5-performance-profiling-101) — other
31. [filament/docs/Filament.md.html at main · google/filament](https://github.com/google/filament/blob/main/docs/Filament.md.html) — docs
32. [Class ImageComparisonSettings | Graphics Tests Framework](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@7.3/api/UnityEngine.TestTools.Graphics.ImageComparisonSettings.html) — docs
33. [Class ImageComparisonSettings | Graphics Tests Framework](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@7.8/api/UnityEngine.TestTools.Graphics.ImageComparisonSettings.html) — docs
34. [Class ImageComparisonSettings | Graphics Tests Framework | 8.1.0 ...](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.1/api/UnityEngine.TestTools.Graphics.ImageComparisonSettings.html) — docs
35. [Performance Guidelines for Mobile Devices in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-guidelines-for-mobile-devices-in-unreal-engine) — other
36. [Ottimizzazione della memoria di gioco di Unreal su Android](https://developer.android.com/games/engines/unreal/unreal-reduce-memory?hl=it) — docs
37. [Debugging and Optimization for Mobile in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/debugging-and-optimization-for-mobile-in-unreal-engine) — other
38. [About the Graphics Test Framework | Graphics Test Framework | 8.6.3-exp.1](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.6/manual/index.html) — docs
39. [About the Graphics Test Framework | Graphics Tests Framework | 8.2.2-exp.1](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.2/manual/index.html) — docs
40. [About the Graphics Test Framework | Graphics Test Framework | 8.9.1-exp.1](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.9/manual/index.html) — docs
41. [Using the Graphics Test...](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.13/manual/index.html) — docs
42. [About](https://google.github.io/filament/Filament.html) — other
43. [Materials and Shaders](https://developer.android.com/games/optimize/materials) — docs
44. [Using the Graphics Test Framework](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@7.2/manual/index.html) — docs
45. [About the Graphics Test Framework | Graphics Test Framework | 8.8.0-exp.1](https://docs.unity3d.com/Packages/com.unity.testframework.graphics@8.8/manual/index.html) — docs
46. [Framebuffer - OpenGL Wiki](https://wikis.khronos.org/opengl/Framebuffer_) — other
47. [eglCreatePlatformWindowSurface](https://registry.khronos.org/EGL/sdk/docs/man/html/eglCreatePlatformWindowSurface.xhtml) — other
48. [OpenGL color space - Khronos Forums](https://community.khronos.org/t/opengl-color-space/77386) — other
49. [OpenGL® ES](https://registry.khronos.org/OpenGL/specs/es/3.2/es_spec_3.2.pdf) — other
50. [The OpenGL](https://registry.khronos.org/OpenGL/specs/gl/glspec33.compatibility.pdf) — other
51. [Required formats](https://wikis.khronos.org/opengl/Image_Format) — other
52. [OpenGL](https://registry.khronos.org/OpenGL/specs/es/3.0/es_spec_3.0.pdf) — other
53. [glBlendFunc](https://registry.khronos.org/OpenGL-Refpages/es3/html/glBlendFunc.xhtml) — other
54. [The OpenGL](https://registry.khronos.org/OpenGL/specs/gl/glspec20.pdf) — other
55. [glBlendFunc](https://registry.khronos.org/OpenGL-Refpages/es3.1/html/glBlendFunc.xhtml) — other
56. [WebGL Specification](https://registry.khronos.org/webgl/specs/latest/1.0/) — other
57. [Index of /OpenGL/specs/gl](https://registry.khronos.org/OpenGL/specs/gl/) — other
58. [Blending](https://wikis.khronos.org/opengl/Blending) — other
59. [CommonStates](https://github.com/microsoft/DirectXTK/wiki/CommonStates) — docs
60. [DirectX-Graphics-Samples/MiniEngine/Core/Shaders/CameraMotionBlurPrePassCS.hlsl at master · microsoft/DirectX-Graphics-Samples](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/MiniEngine/Core/Shaders/CameraMotionBlurPrePassCS.hlsl) — docs
61. [WebGPU Inspector - Chrome Web Store](https://chromewebstore.google.com/detail/webgpu-inspector/holcbbnljhkpkjkhgkagjkhhpeochfal) — other
62. [Capturing a Metal workload in Xcode | Apple Developer ...](https://developer.apple.com/documentation/xcode/capturing-a-metal-workload-in-xcode) — docs
63. [Frame profiling overview - Android Developers](https://developer.android.com/agi/frame-trace/frame-profiler) — docs
64. [Profiling WebGPU - Chrome CPU - Toji.dev](https://toji.dev/webgpu-profiling/chrome-devtools.html) — other
65. [Metal Tools - Apple Developer](https://developer.apple.com/library/archive/documentation/Miscellaneous/Conceptual/MetalProgrammingGuide/Dev-Technique/Dev-Technique.html) — docs
66. [How to Interpret Dropped Frames in Chrome DevTools: A Guide for We…](https://www.tutorialpedia.org/blog/interpreting-dropped-frame-in-chrome-dev-tools/) — other
67. [Profiling Tools for Web Games](https://abratabia.com/web-game-performance/profiling-tools.php) — other
68. [glance/docs/notes/threejs-best-practices-100-tips.md at main ...](https://github.com/Xqd9912/glance/blob/main/docs/notes/threejs-best-practices-100-tips.md) — docs
69. [Android GPU Inspector (AGI) - Android Developers](https://developer.android.com/agi) — docs
70. [WebGPU - Chrome for Developers](https://developer.chrome.com/docs/web-platform/webgpu) — docs
71. [Metal debugger | Apple Developer Documentation](https://developer.apple.com/documentation/xcode/metal-debugger) — docs
72. [Optimizing GPU performance](https://developer.apple.com/documentation/xcode/optimizing-gpu-performance) — docs
73. [Profiling your WebGL Game with the about:tracing flag | Articles](https://web.dev/articles/abouttracing) — other
74. [GPU Profiling on Apple Silicon - vllm-metal](https://docs.vllm.ai/projects/vllm-metal/en/latest/profiling/) — docs
75. [Capturing Metal Commands Programmatically | Apple Developer ...](https://developer.apple.com/documentation/metal/capturing-metal-commands-programmatically) — docs
# Research: particle-systems

- run id: `dr-62b8b232`  (tool: `deep-research` 0.9.0, depth medium, backend perplexity)
- source count: 60

## Confidence note (verbatim from the run's brief)

Babylon.js's ParticleSystem/GPUParticleSystem/SolidParticleSystem and Unreal Niagara are both directly documented in the evidence, so the subjects are real and the broad architecture comparison (CPU vs GPU simulation, module design, sorting APIs, flipbooks, Niagara Fluids vs Babylon's screen-space Fluid Renderer, scalability controls) is well supported. However, several specific techniques the question asks about — soft particles by name, a canonical explosion-FX recipe (flash/fireball/smoke/debris/shockwave), curl-noise/vector-field/SDF flow-field methods, and SPH/FLIP terminology — are NOT named in any provided source for either engine and are flagged below as unsupported industry inference rather than sourced fact. There is also a genuine conflict in the Niagara evidence: official renderer/API docs describe working sort controls, while independent community reports describe recurring translucency-sorting problems in practice. Confidence: medium.

## Report

# As of current stable releases of Babylon.js (ParticleSystem, GPUParticleSystem, SolidParticleSystem) and Unreal Engine Niagara, what techniques are used in production real-time visual effects for: (1) CPU vs GPU particle simulation architecture, (2) emitter/module design, (3) soft particles, (4) sorting and blending of transparent particles, (5) flipbook/sprite-sheet smoke, (6) explosion FX composition (core flash, fireball, smoke, debris, shockwave), (7) energy/flow-field-driven effects, (8) fluid simulation approximations (SPH, FLIP, screen-space fluid rendering), (9) volumetric smoke rendering, and (10) performance cost control -- and how do the two engines' specific implementations of these techniques compare?

## 1. Scope check: do the named systems actually appear in the evidence?

Yes, directly. Babylon.js `ParticleSystem` [s1][s5][s13], `GPUParticleSystem` [s2][s54], and `SolidParticleSystem` [s3][s6][s12][s50][s51] are each covered by dedicated official documentation pages, as is `Unreal Niagara` [s16][s20][s22][s24]. So this is not a case of the evidence covering only an adjacent topic — the named subjects exist and are documented.

That said, three caveats affect how far the evidence reaches:

- Several 'Niagara' sources are legacy 4.27 documentation [s21][s22][s24][s31] rather than current-release pages, and several others are the same page translated into multiple languages [s26][s27][s29][s30][s32][s38][s39] — these add no independent confirmation, only redundant copies of one English source.
- A number of Babylon and Niagara sources are community forum posts or blogs rather than official docs [s15][s36][s37][s45][s55][s57][s58][s60]. These are treated below as weaker, second-tier evidence, not as confirmation of official behavior.
- Several specific technique names the question asks about (soft particles, SPH, FLIP, curl noise, vector fields, a named explosion-FX recipe) do not appear in the titles/descriptions of any source provided. Those are called out explicitly as unsupported rather than answered with invented citations.

## 2. CPU vs. GPU particle simulation architecture

Babylon.js draws an explicit line between `ParticleSystem` (CPU-simulated, GPU-rendered) and `GPUParticleSystem` (simulation kept on the GPU) [s1][s2][s8][s54]. The GPU system has a narrower feature set — e.g., no sub-emitters and restrictions on manual emission/gradients [s2] — and Babylon can fall back to the CPU system when GPU particles aren't available [s8]. `SolidParticleSystem` is a third, distinct category: one mesh holding many particle instances, with per-particle update logic supplied by the developer rather than a built-in solver [s3][s6][s12][s50][s51].

Niagara lets each emitter choose a CPU or GPU simulation target, and Epic's own guidance stresses that this choice must be consistent with the emitter's spawn/update/simulation scripts [s17]. GPU Simulation Stages extend this to iterative computation over particle arrays, grids, and render targets [s20][s23]. Niagara's CPU/GPU split is therefore a per-emitter, per-script decision inside one authoring framework, while Babylon's is a choice between three separate class-level systems.

## 3. Emitter / module design

Babylon's standard `ParticleSystem` exposes emission, lifetime, color, size, gravity, direction, and sprite-sheet properties as flat settings on one object [s1][s13][s49][s56], with custom behavior added via update callbacks. A graph-based authoring tool, the Node Particle Editor, is documented as a newer addition [s53]. SPS provides no built-in emitter/recycler/physics layer at all — the developer implements `updateParticle` and calls `setParticles()` [s3][s12][s50][s51].

Niagara's native model is an ordered module stack per emitter (Spawn and Update stages for System, Emitter, and Particle, extended by Events and Simulation Stages), and module order is documented as functionally significant [s16][s20][s22][s24][s28]. This is a materially more modular, data-driven authoring model than anything documented for Babylon's built-in systems.

## 4. Soft particles

No source in the evidence set names 'soft particles' or 'depth fade' as a documented feature of either engine. What the evidence does support: Babylon has a forum thread describing developers sampling a depth texture (`depthSampler`) from a custom `ShaderMaterial` and post-process [s58], and a separate forum thread about particles being hidden behind other geometry [s60] — both consistent with depth-aware custom-shader techniques, but neither is an official, named 'soft particle' feature. For Niagara, the closest documented material is a general transparency/materials page [s43] and the renderer reference pages [s31][s32][s38][s39], none of which name a soft-particle technique in their titles/descriptions.

**This is an area where the requested technique is standard industry practice but is not established by the provided evidence for either engine — treat any specific soft-particle claim beyond 'depth textures exist and are used in custom materials' as unsupported.**

## 5. Sorting and blending of transparent particles

Babylon: particle blend-mode options are part of the standard particle property set [s1][s56]. SPS depth-sorting of mesh particles is discussed in a community forum thread [s55] rather than confirmed in the primary SPS documentation pages provided [s3][s12][s50][s51] — so this should be treated as community-reported, not officially documented in the sources given.

Niagara: renderer-level sorting is documented through the API — `sort_mode` and related properties on sprite and mesh renderers [s33][s34][s41], and system/component-level `TranslucencySortPriority` [s35], with general transparency behavior discussed in the materials docs [s43]. This is a stronger, more explicit documentation trail than anything found for Babylon.

However — see Section 12 — independent community sources describe Niagara's sorting behaving unreliably in practice [s36][s37][s45], which sits in tension with the API documentation's implication that sorting is a solved, controllable setting.

## 6. Flipbook / sprite-sheet smoke

Babylon's `ParticleSystem` documents sprite-sheet/animated-billboard properties as part of its standard property set [s1][s13][s49][s56].

Niagara has an explicit, purpose-built tool for this: the Niagara Flipbook Baker, documented as converting a simulation (including volumetric smoke/gas) into a tiled flipbook texture for cheap runtime playback via a sprite emitter [s25] (translated copies at [s26][s29][s30]). Sprite renderer flipbook/sub-UV controls are also referenced in the renderer API [s33]. This gives Niagara a documented simulation-to-flipbook production pipeline that has no equivalent described in the Babylon sources provided.

## 7. Explosion FX composition (core flash, fireball, smoke, debris, shockwave)

**No source in the evidence set documents an explosion effect, by name, for either engine.** The specific compositional recipe in the question (flash/fireball/smoke/debris/shockwave as separate layered emitters) is standard real-time VFX practice, but it is not confirmed by any provided title or description and must be treated as unsupported here.

What the evidence does support, at the level of general capability: Babylon can combine multiple `ParticleSystem`/`GPUParticleSystem` instances, SPS meshes, and custom materials, since these are independent objects that can be composed by application code [s1][s2][s3][s54]; Niagara's system/emitter/event architecture is documented as supporting multiple emitters per system with event-driven triggering between them [s16][s20][s22][s24]. Beyond that general composability, any specific claim about how an explosion is authored in either engine is not something these sources establish.

## 8. Energy / flow-field-driven effects

No source names curl noise, vector fields, attractors, or signed-distance fields for either engine. What is documented: Babylon CPU particles support arbitrary per-particle update logic [s1][s13], and GPU particles have a narrower set of built-in behaviors and shader hooks [s2]. Niagara's GPU Simulation Stages are documented as supporting iterative computation over grids and render targets, which is the kind of infrastructure a flow-field system would need [s20][s23], and the 5.0 release notes are cited in connection with this capability [s23]. But none of the sources confirm which named field-driving techniques are actually built in.

**Treat any specific mention of curl noise, vector fields, or SDFs in either engine as unsupported by this evidence** — only the general 'custom code / GPU simulation stage' infrastructure is documented.

## 9. Fluid simulation approximations (SPH, FLIP, screen-space rendering)

Babylon's Fluid Renderer is explicitly documented as a **screen-space rendering** technique: it takes particle positions (from a `ParticleSystem`, `GPUParticleSystem`, or custom buffer) and renders depth/thickness/diffuse textures that are processed into a fluid-like surface [s14][s46][s47][s48][s52]. This is a rendering technique, not a physical solver — the documentation does not claim it performs SPH or FLIP simulation, and none of the Babylon sources use those terms.

Niagara Fluids is documented as providing 2D and 3D templates for real-time fire, smoke, and gas [s18][s19][s27]. The provided sources describe this as a grid-based workflow but **do not use the terms SPH or FLIP** in their titles/descriptions, so this evidence cannot confirm which specific solver class Niagara Fluids implements — that claim would need to come from the underlying technical documentation, which is not in this source list.

## 10. Volumetric smoke rendering

Niagara Fluids is the only source-documented path to volumetric-style smoke/gas simulation in either engine [s18][s19][s27], with the Flipbook Baker providing a documented route to bake such simulations into cheap runtime sprites [s25]. Babylon's sources describe only sprite/flipbook-based smoke via the standard particle property set [s1][s13][s49][s56]; the Fluid Renderer is explicitly a liquid-surface rendering tool, not a smoke/volume renderer, per its own documentation [s14][s46][s47][s48]. So on this specific point the evidence supports a real asymmetry: Niagara has a documented volumetric smoke pipeline, Babylon's documented tools do not.

## 11. Performance cost control

Niagara has a dedicated, current-release scalability document covering quality levels, culling, significance, and platform overrides [s17], plus release notes that may bear on performance changes [s23][s42]. Epic's own guidance in that document makes an important point: every Niagara simulation carries CPU overhead for system/emitter management even when the particle simulation itself runs on the GPU — choosing GPU only moves part of the cost [s17].

Babylon's cost-control evidence is more indirect: the GPU particle documentation discusses capacity, emission, and disposal behavior (stopping emission does not automatically remove already-rendered particles; disposal is required) [s2][s54]. One third-party Japanese blog benchmarks Babylon SPS against Unity WebGL FPS behavior when converting meshes to particles [s15] — this is useful as an independent performance data point, but it compares Babylon to Unity, not to Niagara/Unreal, so it cannot be used to support any Babylon-vs-Niagara performance comparison.

## 12. Where the evidence disagrees

**Niagara sorting: official API vs. community reports.** The API documentation presents sorting as a working, controllable feature — `sort_mode` on renderers [s33][s34][s41] and `TranslucencySortPriority` at system/component level [s35] — alongside a general transparency materials guide [s43]. Independent, non-official sources describe the opposite experience in practice: a community sorting tutorial written specifically to work around sorting problems [s36], a blog post titled around 'Translucency Sorting Wrong' with a fix [s37], and a Japanese blog reporting that Niagara sorting 'doesn't go well' [s45]. These are not necessarily contradictory in the strict sense — a documented feature can still be reported as unreliable or hard to configure correctly by users — but the evidence does not let us resolve whether this reflects version-specific bugs (the sources span different, sometimes ambiguous version numbers, e.g., 'Niagara 2.3', 'Niagara 5.3'), edge cases (GPU-simulated or heavily overlapping particles), or a persistent limitation. **This should be reported as an open disagreement, not resolved by picking one side.**

**Level of abstraction.** Across nearly every topic, the sources support a genuine architectural asymmetry rather than a like-for-like feature comparison: Babylon's documented systems are lower-level primitives that the developer composes and extends in code, while Niagara is documented as a considerably more complete authoring framework (modules, events, simulation stages, scalability settings, dedicated fluids/flipbook tooling). This is a real, source-supported difference, not an averaging artifact.

## 13. What remains unsettled, and what would settle it

- **Soft particles, explosion-FX composition, and flow-field techniques (curl noise, vector fields, SDFs)** are not documented by name in any provided source for either engine. Settling this would require primary technical/source-code documentation (e.g., Niagara module reference pages for force/turbulence modules, or Babylon shader/material source) rather than the overview and API-reference pages available here.
- **Whether Niagara Fluids is SPH-, FLIP-, or grid-based**, and what algorithm Babylon's Fluid Renderer uses internally beyond 'screen-space depth/thickness reconstruction,' is not confirmed by titles/descriptions alone. The Babylon 'Technical Implementation Details' page [s46] and Niagara Fluids pages [s18][s19][s27] would need to be read in full technical depth to confirm solver classes.
- **The Niagara sorting disagreement** (Section 12) is unresolved: it's unclear from this evidence whether the community-reported sorting problems are current, version-specific, or particular to certain renderer/simulation configurations. This would be settled by checking whether the bug reports [s36][s37][s45] are tied to specific, now-superseded engine versions, and whether current release notes [s42] mention sorting fixes.
- **Cross-engine performance comparison** is not directly supported: the one performance benchmark in evidence compares Babylon to Unity, not Niagara [s15]. A genuine Babylon-vs-Niagara performance comparison would require a benchmark that runs comparable effects in both engines, which is not present in this source set.

## Sources

1. [Particle System | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/particle_system/) — other
2. [GPU Particles](https://doc.babylonjs.com/features/featuresDeepDive/particles/particle_system/gpu_particles/) — other
3. [Physics and Solid Particles | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/solid_particle_system/sps_physics/) — other
4. [Particles | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles) — other
5. [ParticleSystem - Babylon.js documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.ParticleSystem) — other
6. [SolidParticle - Babylon.js Documentation](https://doc.babylonjs.com/typedoc/classes/BABYLON.SolidParticle) — other
7. [Particle Systems - Babylon.js documentation](https://doc.babylonjs.com/guidedLearning/createAGame/particleSystems/) — other
8. [CPU and GPU Particle Systems | BabylonJS/Babylon.js | DeepWiki](https://deepwiki.com/BabylonJS/Babylon.js/11.1-cpu-and-gpu-particle-systems) — other
9. [Particle Systems | BabylonJS/Babylon.js | DeepWiki](https://deepwiki.com/BabylonJS/Babylon.js/11-particle-systems) — other
10. [Babylon.js Specifications](https://www.babylonjs.com/specifications/) — other
11. [Babylon.js/packages/dev/core/src/Particles/particleHelper.ts at master · BabylonJS/Babylon.js](https://github.com/BabylonJS/Babylon.js/blob/master/packages/dev/core/src/Particles/particleHelper.ts) — docs
12. [Managing Solid Particles - Babylon.js documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/solid_particle_system/manage_sps_particles/) — other
13. [Particle System Intro | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/particle_system/particle_system_intro) — other
14. [Fluid Renderer - Babylon.js docs](https://doc.babylonjs.com/features/featuresDeepDive/particles/fluid_renderer) — other
15. [meshをParticleにしたときのFPS変化をBabylon.jsとUnity WebGLで比較しました - CrossRoad](https://www.crossroad-tech.com/entry/babylonjs-solidparticlesystem-unity) — other
16. [Overview of Niagara Effects for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-niagara-effects-for-unreal-engine) — other
17. [Scalability and Best Practices for Niagara | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/scalability-and-best-practices-for-niagara) — other
18. [Niagara Fluids in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-fluids-in-unreal-engine) — other
19. [Niagara Fluids in Unreal Engine | Unreal Engine 5.5 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/unreal-engine/niagara-fluids-in-unreal-engine) — other
20. [Key Concepts in Niagara Effects for Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/key-concepts-in-niagara-effects-for-unreal-engine) — other
21. [Niagara Content Examples | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-content-examples?application_version=4.27) — other
22. [Niagara Key Concepts | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-key-concepts?application_version=4.27) — other
23. [Unreal Engine 5.0 Release Notes | Unreal Engine 5.0 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.0-release-notes?application_version=5.0) — other
24. [Niagara Overview | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-overview?application_version=4.27) — other
25. [Niagara Flipbook Baker Quick Start Guide in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-flipbook-baker-quick-start-guide-in-unreal-engine) — other
26. [Guide de démarrage rapide du générateur Niagara flipbook dans l'Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/fr-fr/unreal-engine/niagara-flipbook-baker-quick-start-guide-in-unreal-engine) — other
27. [Unreal Engine の Niagara Fluids | Unreal Engine 5.6 ドキュメンテーション | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/niagara-fluids-in-unreal-engine) — other
28. [Editor UI Reference for Niagara Effects in Unreal Engine | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/editor-ui-reference-for-niagara-effects-in-unreal-engine) — other
29. [Guía de inicio rápido de la herramienta de integración Niagara Flipbook en Unreal Engine | Unreal Engine 5.6 Desarrllador | Epic Developer Community](https://dev.epicgames.com/documentation/es-mx/unreal-engine/niagara-flipbook-baker-quick-start-guide-in-unreal-engine) — other
30. [Kurzanleitung für Niagara Flipbook Baker für die Unreal Engine | Unreal Engine 5.5 Dokumentation | Epic Developer Community](https://dev.epicgames.com/documentation/de-de/unreal-engine/niagara-flipbook-baker-quick-start-guide-in-unreal-engine) — other
31. [Niagara Renderers | Unreal Engine 4.27 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/niagara-renderers?application_version=4.27) — other
32. [虚幻引擎Niagara渲染模块参考 | 虚幻引擎 5.5 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/render-module-reference-for-niagara-effects-in-unreal-engine) — other
33. [unreal.NiagaraSpriteRendererProperties¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/NiagaraSpriteRendererProperties?application_version=4.27) — other
34. [unreal.NiagaraMeshRendererProperties¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/NiagaraMeshRendererProperties?application_version=4.27) — other
35. [TranslucencySortPriority | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/Niagara/UNiagaraSystem/TranslucencySortPriority) — other
36. [[Niagara 5.3] Sorting Mini Tutorial](https://realtimevfx.com/t/niagara-5-3-sorting-mini-tutorial/24380) — other
37. [Fix: Unreal Niagara 2.3 Translucency Sorting Wrong | Bugnet Blog](https://bugnet.io/blog/fix-unreal-niagara-2-3-translucency-sorting-wrong) — other
38. [Unreal Engine における Niagara エフェクトの Render モジュール リファレンス | Unreal Engine 5.6 ドキュメンテーション | Epic Developer Community](https://dev.epicgames.com/documentation/ja-jp/unreal-engine/render-module-reference-for-niagara-effects-in-unreal-engine) — other
39. [언리얼 엔진의 나이아가라 이펙트용 렌더 모듈 레퍼런스 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/render-module-reference-for-niagara-effects-in-unreal-engine) — other
40. [unreal.NiagaraLightRendererProperties¶](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/NiagaraLightRendererProperties?application_version=5.4) — other
41. [UNiagaraMeshRendererProperties | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/Niagara/UNiagaraMeshRendererProperties) — other
42. [Unreal Engine 5.4 Release Notes | Unreal Engine 5.4 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.4-release-notes?application_version=5.4) — other
43. [Using Transparency in Unreal Engine Materials | Unreal Engine 5.6 Documentation | Epic Developer Community](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-transparency-in-unreal-engine-materials) — other
44. [UE5.5 Niagara 渲染器原创](https://blog.csdn.net/qq_30100043/article/details/146240028) — other
45. [UE4 Niagara ソートがうまくいかない](https://gldhuehelcvfx.hatenablog.com/entry/2020/12/21/211656) — other
46. [Technical Implementation Details | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/fluid_renderer/fluid_implementation) — other
47. [Using the Fluid Renderer | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/fluid_renderer/fluid_using/) — other
48. [Fluid Rendering Demos - Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/fluid_renderer/fluid_demos) — other
49. [Basic Particle Properties | Babylon.js Documentation](https://doc.babylonjs.com/features/featuresDeepDive/particles/particle_system/particles_tuning/) — other
50. [An Introduction To The Solid Particle System](https://doc.babylonjs.com/features/featuresDeepDive/particles/solid_particle_system/sps_intro) — other
51. [Solid Particle System](https://doc.babylonjs.com/features/featuresDeepDive/particles/solid_particle_system) — other
52. [Fluid Rendering GUI](https://doc.babylonjs.com/features/featuresDeepDive/particles/fluid_renderer/fluid_gui) — other
53. [Node Particle Editor Blocks | Babylon.js Documentation](https://doc.babylonjs.com/toolsAndResources/npe/npeBlocks) — other
54. [Documentation/content/features/featuresDeepDive/particles/particle_system/gpu_particles.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/particles/particle_system/gpu_particles.md) — docs
55. [SPS Particle Depth Sort](https://www.html5gamedevs.com/topic/33473-sps-particle-depth-sort/) — other
56. [Documentation/content/features/featuresDeepDive/particles/particle_system/customizingParticles.md at master · BabylonJS/Documentation](https://github.com/BabylonJS/Documentation/blob/master/content/features/featuresDeepDive/particles/particle_system/customizingParticles.md) — docs
57. [Ordering of transparent ParticleSystem with BufferGeometry in Threejs](https://stackoverflow.com/questions/18879128/ordering-of-transparent-particlesystem-with-buffergeometry-in-threejs) — other
58. [Getting depthSampler using custom shaderMaterials and ...](https://forum.babylonjs.com/t/getting-depthsampler-using-custom-shadermaterials-and-custom-postprocess/51911) — other
59. [Getting Started - Chapter 6 - Particle Spray | Babylon.js Documentation](https://doc.babylonjs.com/features/introductionToFeatures/chap6/particlespray/) — other
60. [Particles will be hidden behind Plane - Questions - Babylon.js](https://forum.babylonjs.com/t/particles-will-be-hidden-behind-plane/48142) — other
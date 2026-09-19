# 3D visualization bench — awareness

Thirteen lenses for building 3D visualization systems, plus platform
specialists. Each owns one question; none answers another's. Delegate rather
than reason from memory — every one of them carries reference knowledge this
session does not.

| The question in front of you | Lens |
|---|---|
| Should this even be 3D? What is encoded in what? | `3d-developer:dataviz-strategist` |
| How is the scene organized, streamed, culled, LOD'd? | `3d-developer:scene-architect` |
| What does a frame cost, in what order does it draw? | `3d-developer:rendering-engineer` |
| What do surfaces look like — PBR, shaders, node graphs? | `3d-developer:shading-artist` |
| How does the user pick, select, hover, manipulate? | `3d-developer:interaction-designer` |
| How does the user move and get framed on things? | `3d-developer:camera-navigator` |
| Depth of field, fog, bloom, SSAO, outlines, AA | `3d-developer:postfx-artist` |
| Labels, annotations, callouts, billboarding, leader lines | `3d-developer:label-callout-designer` |
| Motion over time — keyframes, skeletal, transitions | `3d-developer:animation-engineer` |
| Smoke, explosions, energy flow, fluid, GPU particles | `3d-developer:particle-fx-artist` |

**Platform layer, used after the design exists** — these turn an
engine-agnostic design into real API calls, and are only present if their
behavior is composed: `3d-developer:babylonjs-specialist` (web, WebGL2/WebGPU)
and `3d-developer:unreal-specialist` (UE5, C++/Blueprint/Niagara).

**Review.** Three critics, run cold and independent, never in conversation with
each other: `perf-budget-critic` (will it hold frame time), `visual-quality-critic`
(does it read correctly to a human eye), `correctness-critic` (color space, depth
precision, alpha, normals, units — the errors that look like art direction).
Run all three at once with `load_skill(skill_name="3d-design-review")`, or gate
a pipeline on them with `3d-developer:recipes/design-review-pipeline.yaml`.

**Two rules worth carrying here rather than in an agent.** (1) Every domain
design must state its frame-cost claim explicitly — draw calls, passes, texture
memory — because a critic cannot refute a budget nobody wrote down. (2) Design
engine-agnostic, implement platform-specific; reaching for a Babylon or Unreal
API before the domain lens has spoken is how a design gets shaped by whatever
the engine made easy.

# 3D visualization bench — awareness

Thirteen lenses for building 3D visualization systems, plus opt-in platform
specialists. Each owns one question. Delegate rather than answer from memory.

| The question in front of you | Lens |
|---|---|
| Should this even be 3D? What is encoded in what? | `3d-developer:dataviz-strategist` |
| How is the scene organized, streamed, culled, LOD'd? | `3d-developer:scene-architect` |
| What does a frame cost, in what order does it draw? | `3d-developer:rendering-engineer` |
| What are surfaces made of — PBR, shaders, node graphs? | `3d-developer:shading-artist` |
| How does the user pick, select, manipulate? | `3d-developer:interaction-designer` |
| How does the user move and get framed on things? | `3d-developer:camera-navigator` |
| Depth of field, fog, bloom, SSAO, outlines, AA | `3d-developer:postfx-artist` |
| Labels, callouts, billboarding, leader lines | `3d-developer:label-callout-designer` |
| Motion over time — keyframes, skeletal, transitions | `3d-developer:animation-engineer` |
| Smoke, explosions, energy flow, fluid, GPU particles | `3d-developer:particle-fx-artist` |

**Platform layer — used after the design exists**, and only present if its
behavior is composed: `3d-developer:babylonjs-specialist` (web, WebGL2/WebGPU),
`3d-developer:unreal-specialist` (UE5).

**Review.** Three critics, cold and independent, never in conversation with each
other: `perf-budget-critic` (holds frame time), `visual-quality-critic` (reads
correctly, regression catchable), `correctness-critic` (color space, depth
precision, alpha, units — errors that look like art direction). Run all three at
once with `load_skill(skill_name="3d-design-review")`; gate a pipeline with
`@3d-developer:recipes/design-review-pipeline.yaml`. The whole arc — scope,
design, reconcile, review, revise — is `load_skill(skill_name="3d-build-loop")`.

**Two standing rules.** Every design states its frame-cost claim explicitly; a
critic cannot refute a budget nobody wrote down. And design engine-agnostic,
implement platform-specific — reaching for an engine API first is how a design
gets shaped by whatever that engine made easy.

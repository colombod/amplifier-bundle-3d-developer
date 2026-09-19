---
bundle:
  name: 3d-developer
  version: 0.1.0
  description: >-
    3D visualization development bench. Ten domain lenses (scene architecture,
    rendering, shading, interaction, camera/navigation, post-FX, labels and
    callouts, animation, particles, and when-3D-is-even-right), two platform
    specialists (Babylon.js and Unreal Engine), and three adversarial critics
    (performance budget, visual quality, correctness) that review the work
    before it ships.

includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  - bundle: 3d-developer:behaviors/3d-core
  - bundle: 3d-developer:behaviors/platform-babylonjs
  - bundle: 3d-developer:behaviors/platform-unreal
---

# The 3D visualization bench

Building a 3D visualization is not one skill. It is ten, and they disagree with
each other for good reasons — the scene architect wants fewer, bigger batches;
the label designer wants a screen-space pass per annotation; the perf critic
wants both of them to justify the frame time. This bundle makes each of those
lenses a separate agent so the disagreement happens *before* the frame budget
blows, not after.

(The routing table lives in `context/3d-developer-awareness.md`, loaded once by
the `3d-core` behavior — not re-mentioned here, because the two channels are not
deduplicated against each other.)

## How to use it

1. **Scope first.** If it is not obvious that the thing should be 3D at all,
   start with `3d-developer:dataviz-strategist`. It is the only lens allowed to
   answer "this should be a 2D chart."
2. **Design per domain.** Delegate to the domain lens that owns the question.
   Each returns a design with an explicit frame-cost estimate.
3. **Pick the platform late.** The domain lenses are engine-agnostic on purpose.
   Hand their output to `3d-developer:babylonjs-specialist` or
   `3d-developer:unreal-specialist` for the actual API-level implementation.
4. **Review before you ship.** Load the `3d-design-review` skill (or run
   `recipes/design-review-pipeline.yaml`) to fan the three critics out cold
   against the same artifact and get a BLOCK / PASS verdict with dissent
   recorded.

The critics never see each other's findings before voting. That is deliberate:
three independent verdicts are evidence, three verdicts that converged by
talking to each other are one verdict wearing a hat.

# Animation systems for real-time 3D visualization

Claims sourced to engine documentation are marked **[doc]**; engineering generalization
is marked **[synthesis]**. Never present synthesis as documented fact.

## 1. Clock layers

Most animation bugs are clock bugs. Say which clock each piece of motion rides.

| Clock | Advanced by | Used for | Breaks if |
|---|---|---|---|
| Wall clock | real seconds | UI transitions, idle motion | replay must be identical |
| Variable render delta | time since last frame | tweens, easing, presentation | unclamped after a stall: one frame teleports everything |
| Fixed simulation step | accumulator, constant dt | physics, springs, deterministic replay | mixed with the variable delta |
| Authored clip time | seek position on a timeline | clips, shots, scrubbing | assumed equal to display rate |
| Data time | dataset timestamps | temporal playback | conflated with playback speed |

**Core rule [synthesis]:** define motion per *second*, clamp large deltas, use a
fixed-step accumulator where determinism matters. Frame-count motion is a defect.

**Authored rate is not display rate [doc, Babylon]:** the frame rate given to a keyframe
`Animation` is a nominal authoring timebase, evaluated against elapsed scene time via
the render loop; `Scene` exposes delta-time and deterministic-lockstep facilities.

**Sequencer keeps four time concepts separate [doc, Unreal]:** display rate, tick
resolution, play rate, time dilation, under a rational-frame-rate tick model. A
cinematic playback rate propagates time scaling into materials and particles; a custom
time-step class fixes the engine tick for genlocked output. Whether Niagara has its own
fixed-timestep mode is not established by the research.

## 2. Keyframe vs tween vs procedural

| Approach | Use when | Cost | Watch out |
|---|---|---|---|
| Authored keyframes | repeatable, scrubbable, art-directed | per-channel evaluation each frame | endpoints fixed at author time |
| Runtime tween | endpoints computed at runtime | one interpolation per property per frame | per-frame allocation; no interrupt policy |
| Procedural per frame | motion derives from state | arbitrary CPU | unscrubbable, hard to test |
| Physics / simulation | contact, inertia, secondary motion | solver cost; non-deterministic under variable dt | cannot scrub backwards |

A usable abstraction supports scalar/vector/quaternion/color channels, per-property
interpolation, easing, loop modes, cancellation and seeking **[doc, Babylon]**. Rotation
uses quaternions, not Euler, interpolated with slerp **[doc, Babylon retargeting]**.
Easing **[synthesis]**: ease-in-out when the viewer reads one object travelling;
ease-out for arrivals after input; linear wherever rate itself encodes a value, since an
eased ramp is unreadable as a rate.

## 3. Skeletal animation and blending

**Babylon [doc]:** `Skeleton` plus bones/linked transform nodes; glTF imports surface as
`AnimationGroup` objects. Retargeting matches bone-related targets by name (including
linked transform-node names) and morph targets by mesh/morph name. Cross-fade by ramping
per-group weights (`setWeightForAllAnimatables`).

**Unreal [doc]:** the AnimGraph combines poses via blend nodes, additive ops, overrides
and per-bone blending -- state machines for discrete states, Blend Spaces for continuous
parameters, Layered Blend Per Bone for a named bone and its children (upper body over
locomotion). Blend Masks/Profiles make bone-influence definitions reusable across
characters; Virtual Bones add control points without editing the skeleton.

Cost **[synthesis]**: skinning scales with characters x bones x blend layers. Degrade
distant-skeleton update rate first, then blend layers, then bone count.

## 4. Morph targets

**Babylon [doc]:** a target must have *exactly* the base mesh's vertex count; geometry is
base plus the weighted sum of target deltas. Influence animates as a plain float and
several morph animations can run together in an `AnimationGroup`; glTF exporter options
control which morph animation data survives export.

**Unreal:** no morph/blendshape page was provided; only the machinery that *drives*
them -- Animation Blueprint curves and Control Rig. Guidance on corrective shapes,
naming layers, or capping crowd targets is **unverified**.

Cost **[synthesis]**: each *active* target adds per-vertex bandwidth and shader work.
Budget by active count, not authored count.

## 5. Procedural and physics-driven motion

**Babylon [doc]:** procedural motion runs from `scene.onBeforeRenderObservable`, ahead of
each rendered frame; `TrailMesh` and `GreasedLineBaseMesh` are helper meshes outside the
`Animation`/`AnimationGroup` system.

**Unreal [doc]:** Skeletal Control nodes (Two Bone IK, FABRIK, Look At, Transform/Modify
Bone, CCD IK, Spline IK) do procedural bone-space work in the AnimGraph, largely in
component space. The Rigid Body node simulates physics on a skeletal-mesh segment inside
the AnimGraph, and `AnimNode_RigidBodyWithControl` combines it with Control Rig. Physics
workflows use a **blend weight from 0.0 (fully keyframed) to 1.0 (fully simulated)**,
via `Set All Bodies Below Simulate Physics` / `... Physics Blend Weight`; Physics
Components simulate selected bone groups while animation continues.

## 6. Timeline and sequencing

**Babylon [doc]:** `AnimationGroup` is the timeline unit -- `start`, `pause`, `stop`,
`goToFrame`, `speedRatio`. Imported glTF animations arrive as scene groups.

**Unreal [doc]:** Sequencer handles camera shots, actor transforms, animation and Control
Rig tracks; sequence-assigned animation blends with live Animation Blueprint output via
a Slot node and track weight, and `UMovieSceneSequencePlayer` plays, pauses and scrubs
Level Sequences at runtime. Take Recorder captures live runs into Level Sequence assets
-- the documented route from a non-scrubbable run to a scrubbable asset.

**Scrub rule [synthesis]:** anything stateful (physics, accumulators, particle history)
is not scrubbable; bake to keyframes, a recorded sequence, or a VAT.

## 7. Data-driven transitions (synthesis -- not documented by any source)

**No source documents this as a named feature or best practice.** Everything below is
reasoning applied to documented primitives; say so when relying on it.

- Form: `value = a + (b - a) * easedProgress`; lerp position, slerp rotation.
- **Object constancy**: bind motion to stable data IDs, never array indices. If the same
  datum is not the same object across the update, the motion teaches nothing.
- **Stage enter / update / exit** rather than all at once, and allow only one semantic
  change per move -- repositioning *and* re-valuing in one tween is unreadable.
- **Tracking limit**: viewers follow a handful of independently moving objects; beyond
  that, motion conveys aggregate change and should be designed as such.
- **Updates faster than the transition**: retarget from current, drop intermediates, or
  queue. Offer an instant path for reduced-motion preferences.
- At scale use one shared normalized time parameter on the GPU (section 8). Niagara user
  parameters and Sequencer curve tracks are Unreal's documented mechanisms for
  population-scale value changes -- not framed as visualization tooling by any source.

## 8. Animating many objects

| Technique | Scales to | Cost | Constraint |
|---|---|---|---|
| Per-object animations/tweens | 10^2-10^3 **[synthesis, order of magnitude]** | CPU per target per frame | dominates frame time before the GPU does |
| Instances + shared time uniform | 10^4-10^6 **[synthesis]** | one draw call, vertex work | motion closed-form in (time, instance attributes) |
| Baked vertex animation texture | deforming crowds **[doc, Babylon]** | texture VRAM, vertex sample | precomputed; cannot react at runtime |
| GPU particle/mesh system | very large **[doc]** | GPU sim; motion vectors for correct blur | emitter-driven work is particle-fx-artist's |

**Babylon [doc]:** baked vertex animation precomputes motion into a texture sampled by the
vertex shader, avoiding per-instance CPU skinning; `computeMaxExtents` gives bounding
boxes for culling/LOD on instanced sets. The API shape of `BakedVertexAnimationManager`
is **not** established by the research. **Unreal:** VATs, Instanced/Hierarchical
Instanced Static Meshes and reduced update rates for distant agents are the general
techniques, but **no source here is a VAT best-practices page** -- treat as unverified.

## 9. Symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Speed varies with hardware or scene load | animating off frame count |
| Everything teleports after a stall | unclamped delta time |
| "Deterministic" replay drifts | simulation on the variable delta; mixed clocks |
| Rotation takes a weird arc or flips | Euler lerp instead of slerp |
| Frame time scales with object count, GPU idle | per-object CPU evaluation; move to a shared-time GPU path |
| Transition jumps when data updates mid-flight | no retarget policy; restart from stale origin |
| Viewer cannot say what changed | broken constancy, or two changes in one move |
| Scrub produces wrong state | stateful simulation on a scrubbable track |
| Morph geometry explodes or is ignored | vertex count differs from base mesh |

## 10. Platform pointers (handoff, not tutorial)

| Technique | Babylon.js | Unreal Engine |
|---|---|---|
| Keyframes | `BABYLON.Animation`, easing functions | animation curves, Transform tracks |
| Timeline | `AnimationGroup` (`goToFrame`, `speedRatio`) | Sequencer, `UMovieSceneSequencePlayer` |
| Skeleton, clips, blending | `Skeleton`, glTF groups, per-group weights | Skeletal Mesh, blend nodes, Blend Spaces, Layered Blend Per Bone |
| Morph targets | `MorphTarget` influence float | curve-driven AnimBP nodes, Control Rig |
| Procedural / physics | `onBeforeRenderObservable`, `TrailMesh` | Skeletal Control nodes, Control Rig, Rigid Body node |
| Crowd-scale baked motion | vertex animation texture, thin instances | VAT with ISM/HISM, Niagara mesh renderer |
| Timing controls | `Scene` delta time, deterministic lockstep | display rate, tick resolution, play rate |

Exact class names, signatures and version behaviour belong to the platform specialist.

## 11. Pitfalls beyond the symptom table

1. **No interrupt policy.** Interruption is normal in interactive visualization; decide
   retarget-vs-cancel, and never allocate a tween per frame.
2. **Baking motion that must react** -- if the data shapes the motion, a VAT is wrong.
3. **Assuming Unreal 4.27 workflows match 5.6.** Several topics have a 4.27-era page and
   a differently titled 5.6-era one (Skeletal Controls vs. Animation Blueprint Skeletal
   Controls; Control Rig Blueprints vs. Animating with Control Rig); release notes
   confirm material change. Verify the current page.
4. **Inventing an API** the research never established instead of asking the platform
   specialist.

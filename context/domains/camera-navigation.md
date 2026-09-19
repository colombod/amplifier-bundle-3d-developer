# Camera and navigation - domain reference

## Evidence status (read before quoting anything below)

- **[DOC]** - documented in engine or graphics references (Babylon.js class docs,
  Unreal docs, Khronos/OpenGL depth references, Cesium, academic).
- **[CONV]** - engineering convention, **not** backed by a source in the evidence
  set. A reasoned default, never established fact.

**[CONV]**, because no UX, HCI, or VR-comfort source existed: fit-to-view margins,
transition durations, minimap conventions, motion-sickness rules, large-world
navigation heuristics. Two gaps: **no source documents a Babylon `FlyCamera`** or
**a plain Unreal `CameraActor`** (only `UCameraComponent`). Assert neither.

## Control-scheme decision table

| Scene shape | Scheme | Why it wins | Cost |
|---|---|---|---|
| Bounded object with a pivot | Turntable (azimuth/elevation/radius, world-up locked, elevation clamped) | Up is never lost; pole flip impossible [DOC: `ArcRotateCamera` has elevation limits] | Some orientations unreachable |
| Arbitrary orientation *is* the task | Arcball / trackball | Any orientation in one gesture | User can lose up; needs a reset |
| Traversable environment, no subject | Fly (6-DOF) | Freedom in open volume | Gets lost; needs speed scaling + home |
| Architectural walkthrough | First-person walk (eye height, yaw+pitch, roll locked) | Human-scale reference frame | Cannot see the whole model; add frame-all |
| Moving subject | Follow / third-person with lag | Stays framed under motion [DOC: `FollowCamera` accelerates to goal with max speed; Unreal Spring Arm + Camera Component with lag] | Lag tuning; obstruction |
| Presentation | Authored / keyframed | Deterministic, repeatable | Not interactive |

Terminology caution [DOC]: `ArcRotateCamera` is a **target-centred orbit camera
parameterized by alpha/beta/radius/target with elevation limits** - not an
unconstrained trackball. Whether it can act as a true arcball depends on beta and
roll configuration, undocumented in the sources. "Arcball" and "ArcRotateCamera"
are not synonyms.

## Perspective vs orthographic

Both engines expose projection mode as an explicit settable property, not an
emergent side effect of other camera parameters [DOC].

| Use | Projection | Reason |
|---|---|---|
| Size comparison, axis reading, measurement, plan/elevation | Orthographic | Foreshortening makes equal sizes read as different with distance - the user's answer becomes *wrong* |
| Spatial understanding, immersion, walkthrough | Perspective | Foreshortening *is* the depth cue |
| Small multiples | Orthographic, shared camera params | Panes stay literally comparable |

The switching rule is [CONV] - no source recommends when to switch - though it
follows from projection geometry, not taste. Orthographic loses distance cues
entirely, so overlap, occlusion, and shading carry all depth information. It still
needs clip planes; Unreal's orthographic camera exposes near/far clip and
auto-calculates ortho planes from orthographic width [DOC].

## Fit-to-bounds

Documented [DOC]: `ArcRotateCamera` has a `zoomOn`-style op moving to the minimum
distance at which supplied meshes are fully visible; `FramingBehavior` performs an
animated fit around a mesh or bounds, configurable duration, stoppable when the
user intervenes. Unreal has **no documented runtime fit-selection** - app-level
code from actor bounds plus a blend.

The math (geometry, not a sourced spec):

1. A **bounding sphere** (radius `r`) is orientation-invariant - one distance
   works from any angle. An **AABB** fits tighter but its required distance
   changes as the camera orbits, making the framing "breathe".
2. `d_v = r / sin(fov_y / 2)`.
3. `tan(fov_x/2) = aspect * tan(fov_y/2)`, then `d_h = r / sin(fov_x/2)`. Use
   `d = max(d_v, d_h)` so it fits on the constrained axis.
4. Margin last: `d *= (1 + m)`; `m` ~0.1-0.2 [CONV - no sourced figure exists].
5. Preserve current azimuth/elevation; move only target and radius, so the user
   keeps their orientation [CONV].
6. Orthographic: distance does not affect framing. Half-height = `r * (1 + m)`,
   half-width = `aspect * half-height`; distance only sets clipping.

Degenerate cases: empty selection (fall back to frame-all); zero-radius bounds
(clamp to a scene-scale minimum, or `d` goes to zero and the camera lands inside
geometry); bounds larger than the far plane (recompute clip planes *before* the
move).

## Transitions and easing

Source-backed [DOC]: `FramingBehavior` animates with a configurable duration and
can stop on user zoom; `FollowCamera` approaches its goal by acceleration and max
speed rather than teleporting; Spring Arm exposes camera lag and rotation lag with
**lag substepping** to keep damping stable under variable frame rate; Sequencer
provides keyframe interpolation, camera cuts (`CameraCutTrack` binds a camera to a
timeline section), and jib/dolly/crane rigs.

Convention [CONV], no sourced numbers: ease-in-ease-out over linear; quaternion
slerp for rotation, or - for a turntable - interpolate alpha/beta/radius directly
so the path stays on the orbit manifold instead of cutting through the object;
cancel on first user input; a few hundred milliseconds.

The variable-frame-rate trap: per-frame `lerp(current, target, k)` changes its
time constant with frame rate. Spring Arm's lag substepping exists for exactly
this [DOC]; hand-rolled damping needs an exponential form using `dt`.

## Depth precision and clipping planes

The best-sourced area of the evidence [all DOC]:

- Perspective depth precision is **non-linear and concentrated near the camera**,
  so the **far/near ratio** - not the absolute far distance - governs precision
  and z-fighting risk.
- A near plane at zero, or unnecessarily small, is a primary cause of instability.
- Z-fighting comes from coplanar geometry, excessive scene scale, or an overly
  large clip range.
- `ArcRotateCamera` documents default `minZ`/`maxZ` values and warns that a distant
  far plane causes depth fighting because the buffer is finite. Those are
  **defaults**, not recommendations - tune to the scene's unit scale.

| Strategy | Buys | Costs | Evidence |
|---|---|---|---|
| Push the near plane out | Largest gain per unit of effort | Near geometry clips; unusable if the user can approach surfaces | [DOC] |
| Pull the far plane in + fog/cull | Modest - the ratio moves slowly | Visible far clipping | [DOC] |
| Logarithmic depth | Precision across huge ranges | Per-fragment depth write can defeat early-Z; **flicker still reported with it** | [DOC: Babylon opt-in Standard Material flag, linear fallback; flicker reports documented] |
| Multi-frustum (render per slice) | Near-arbitrary range | One extra pass per slice | [DOC: Cesium hybrid multi-frustum + log depth] |
| Reversed-Z with float depth | Large gain, near-free | Needs float depth, `GREATER` compare, depth cleared to zero | **[CONV]** - standard practice, **absent here** |
| Origin rebasing / camera-relative coords | Fixes float precision in world *position* - a different problem | Invalidates cached world transforms | **[CONV]** |

Common confusion [DOC]: Spring Arm's collision probe stops the camera entering
geometry. That is occlusion handling, **not** depth precision - it does nothing
for z-fighting.

## Large-scale and multi-scale navigation

Documented [DOC]: Babylon has a dedicated **Geospatial** camera path for map-like
navigation around a spherical planet, distinct from ArcRotate/Universal/Follow.

Everything else is **[CONV]** - speed scaling, semantic LOD for navigation,
teleport bookmarks, origin rebasing; no Unreal World Partition or level-streaming
documentation was in the evidence. The convention worth stating anyway: scale
translation and zoom step by distance-to-nearest-surface or selection radius, so
one control behaves identically at every scale. A constant speed across six orders
of magnitude is unusable at both ends at once.

## Overview+detail and minimaps

The pattern - persistent overview pane, detail frustum footprint drawn on it,
linked selection, simplified overview render - is standard practice but **no
source documents it as a UI specification** [CONV]. The one documented building
block is Unreal's `USceneCaptureComponent2D`, rendering a secondary view to a
texture a UI widget can display [DOC]; no Babylon minimap mechanism is documented,
though a second camera plus viewport is plausible [CONV].

Cost: an overview pane is a **second scene render** - a full extra pass plus a
render target - unless reduced (lower resolution, fewer layers, a static plan, or
a 2D footprint). State which reduction is used.

## Re-orientation, comfort, and motion sickness

**[CONV]** unless marked otherwise - no comfort-design research source was in the
evidence set.

Documented mechanisms that happen to serve comfort [DOC]: `ArcRotateCamera`
elevation limits prevent flips through the pole; `FramingBehavior` is
interruptible on user zoom; Spring Arm's probe prevents camera-through-geometry.
Babylon's `WebXRCamera` is where XR comfort settings live, though that source
details no mitigation.

Testable defaults, never thresholds [CONV]: stable world up; never move or rotate
the XR camera without user input; constant velocity over acceleration ramps in XR;
snap-turn default, smooth optional; teleport alongside continuous locomotion;
comfort vignetting; head-bob off; always a recentre action.

## Platform pointers

Enough to hand off - not an API tutorial. Verify current signatures and defaults
with `babylonjs-specialist` / `unreal-specialist`.

| Technique | Babylon.js | Unreal |
|---|---|---|
| Orbit / turntable | `ArcRotateCamera` (alpha, beta, radius, target, beta limits) | Spring Arm driven by controller rotation |
| Fly / first-person | `UniversalCamera` - current docs treat it as the unified free/first-person camera (keyboard, mouse, touch, gamepad); older material lists `FreeCamera` separately, unresolved. **No `FlyCamera`.** | `UCameraComponent` on a pawn |
| Follow / third-person | `FollowCamera` (radius, heightOffset, rotationOffset) | `USpringArmComponent` (arm length, camera lag, rotation lag, collision probe) + Camera Component |
| Fit to view | `zoomOn`-style op; `FramingBehavior` (animated, configurable duration, stop-on-user-zoom) | **Not documented** - app-level from actor bounds + blend |
| Authored moves | Animation / behaviors | `UCineCameraComponent` via Sequencer; `CameraCutTrack`; jib/dolly/crane rigs |
| Clip planes | `minZ` / `maxZ` | `FMinimalViewInfo` / Camera Component; custom clipping on Cine Camera Component |
| Orthographic | Camera projection mode | Orthographic Camera feature: near/far clip, ortho planes from ortho width; `SetProjectionMode` |
| Log depth | Opt-in Standard Material flag; linear fallback | Not documented in the evidence set |
| Minimap / overview | Not documented; second camera + viewport plausible only | `USceneCaptureComponent2D` to a render target |
| XR | `WebXRCamera` | Not covered in the evidence set |

## Pitfalls - symptom to likely cause

| Symptom | Likely cause | First move |
|---|---|---|
| Surfaces flicker where faces are close | Far/near ratio too large, usually a near plane near zero | Raise the near plane; log depth is the *second* move [DOC] |
| Flicker persists with log depth on | Log depth is not a complete fix [DOC] | Multi-frustum, or reversed-Z if supported |
| Camera "breathes" while orbiting | Fit distance from an AABB that rotates with the view | Fit a bounding sphere |
| Frame-selection lands inside the object | Zero-radius bounds, or margin applied before `max(d_v, d_h)` | Clamp radius to a scene minimum; margin last |
| Fits vertically, cut off at the sides | Only `fov_y` used; aspect ignored | Take `max(d_v, d_h)` |
| Camera flips at the top of an orbit | Unclamped elevation through the pole | Clamp beta short of the pole [DOC] |
| Follow jitters, or differs at 30 vs 120 fps | Per-frame lerp damping, no `dt` correction | Exponential damping with `dt`, or substep [DOC] |
| Transition fights the user mid-animation | Animation not cancelled on input | Stop-on-user-input [DOC] |
| Slow far out, uncontrollable up close | Constant speed in a multi-scale scene | Scale speed by distance or selection radius [CONV] |
| Sizes read wrong in a comparison view | Perspective foreshortening in a measurement task | Make that view orthographic |

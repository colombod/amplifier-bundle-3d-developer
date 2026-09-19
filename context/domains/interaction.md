# Interaction, picking and manipulation — domain reference

## Evidence status

The underlying research is medium confidence and unevenly sourced. Three tiers are
kept distinct throughout: **documented** (first-party Babylon or Epic page),
**community-sourced** (forum threads, a wiki mirror, third-party guides) and
**general practice** (standard engineering, no source in the research).

Covered by no source: constrained-manipulation math, Unreal multi-object selection,
touch specifics in Unreal, picking thresholds, and any bridge from 3D manipulation
to accessibility APIs.

## 1. Picking paths

| Path | Cost | Scales with | Best for |
|---|---|---|---|
| CPU ray / line trace | Per-candidate intersection | Surviving candidates x triangles | Precise surface data (point, normal, UV) |
| GPU colour/ID pick | Extra pass + readback + sync point | Framebuffer resolution | Dense overlap, instancing, point/splat, thin geometry |
| Bounds-only CPU pick | Ray vs AABB/sphere | Candidate count | Hover, when exact hit is unneeded |
| Hybrid | Cheap continuously, exact on commit | — | Hover-heavy UIs |

**Babylon (documented):** `scene.pick` / `pickWithRay` surface results through the
pointer event system; `PointerInfo` and `PointerInfoPre` carry pointer and pick
data. A purpose-built `GPUPicker` exists, but is **community-sourced**: a DeepWiki
mirror plus a forum arc (announcement, then a request to expose it "as a picking
option at scene level", then a follow-up). Its integration point was still being
negotiated — do not assume `scene.pick` ergonomics; verify against release notes.

**Unreal (documented):** hit-testing is line traces. `LineTraceByChannel` is the
single-hit trace and documents a **`Trace Complex`** option — per-triangle
collision, explicitly opt-in, therefore the expensive path.
`MultiLineTraceByChannel` returns multiple hits; `BreakHitResult` exposes actor,
component, location and normal. AR hit testing and the GeoReferencing `LineTrace`
utility are separate, narrower tools — conflate neither with gameplay traces.

**Crossover (general practice — no source gives a number):** the decision is about
*rejection*, not object count. Where a bounding-volume or spatial pre-test kills
most candidates cheaply, CPU picking stays flat. Where it cannot — dense overlap,
instanced fields, point clouds, Gaussian splats (no triangle geometry) — GPU ID
picking's object-count independence wins. State the query rate you assumed.

**Readback (inference, flagged as such in the research):** Babylon's WebGPU engine
is asynchronous in general, so the advice to throttle readback and consume it a
frame late follows from that, but is not documented.

## 2. Hit-test performance at scale

Ordered pipeline for a hover query:

1. **Reject by pickable set** — keep decorative geometry out of hit tests. (Babylon
   `isPickable` filtering is not documented as a performance recommendation; Unreal
   collision channels serve the role. General practice.)
2. **Broad phase** — spatial index (BVH/octree/grid) or frustum plus bounds. Keep
   the acceleration structure separate from the transform hierarchy; they update at
   different frequencies.
3. **Narrow phase** — ray vs bounds, then triangles only if surface detail is
   needed. In Unreal this is the `Trace Complex` decision — the one sourced fact
   behind "avoid complex traces for hover".
4. **Tie-break** — nearest hit, then a stated deterministic rule.
5. **Throttle** — coalesce pointer-move events; never one pick per event at 120+ Hz.

Use `MultiLineTraceByChannel` only where several depth candidates are needed.

## 3. Selection models

| Model | Gesture | Note |
|---|---|---|
| Single | click | Replaces the set |
| Additive / toggle | shift- or ctrl-click | Babylon documents `SelectionOutlineLayer` in exactly this workflow |
| Box / marquee | drag on empty space | Screen-space test, not a ray: project bounds, or read an ID-buffer region |
| Lasso | freehand drag | Point-in-polygon against projected centroids |
| Hierarchical | click node, alt-click descends | Decide once: does selection mean the node or its leaves |

**Babylon (documented):** `ActionManager` with `OnPickTrigger` gives per-object pick
reactions; `SelectionOutlineLayer` provides a centralized selection list
(`addSelection` / `clearSelection`).

**Unreal (unsupported by source):** no source describes an official multi-object
selection API for gameplay; `Transforming Actors` documents transform operations,
not a selection set. Any architecture you choose is standard engineering, not
Unreal doctrine.

**General practice:** hold one authoritative selection set in application state and
let render layers subscribe. Three independent notions of "selected" desynchronise
on the first deletion-during-hover.

## 4. Hover and highlight rendering

**Babylon (documented):** `HighlightLayer` is the standard glow mechanism;
`SelectionOutlineLayer` gives crisp editor-style outlines. Both derive from
`EffectLayer`, and a `FrameGraphSelectionOutlineLayerTask` exists — the outline
layer is carried into the newer frame-graph pipeline, so it is current, not
legacy.

**Unreal (community-sourced only — state this plainly):** the custom-depth/stencil
outline technique is real and widely used, but here it rests on a third-party guide,
forum threads with differing specifics and a bug-tracker issue. **No first-party
Epic page** consolidates it; confidence in specifics, especially stencil-value
conventions, is low.

Cost shape (general practice): each effect layer is an extra pass over the
highlighted subset plus a composite, scaling with membership — which is why
highlighting everything along the cursor path degrades badly. Distinguish hover
from selected by more than hue.

## 5. Gizmos and manipulators

**Babylon (documented):** one gizmo feature page covers attaching gizmos to meshes,
bones and other transform-bearing objects — the *only* gizmo-specific Babylon
source. Utility-layer placement, camera suppression during drag and per-drag undo
transactions are layered practice, not sourced claims.

**Unreal (documented — best-evidenced sub-topic):** `UInteractiveGizmo` is the base
abstraction (documented `Setup`); `UTransformGizmo` drives transform interaction
against a `UTransformProxy`; `UCombinedTransformGizmo` provides the combined
experience; `UInteractiveGizmoManager` owns lifecycle.

**Open question:** these span Editor and Runtime InteractiveToolsFramework modules,
and no source says whether the framework is meant for runtime gameplay manipulators
or editor tooling. The advice that gameplay should use a lighter custom manipulator
is uncited judgment.

## 6. Drag and constrained manipulation

One citable fact: Babylon's changelog notes that `ActionManager`'s `OnPickTrigger`
no longer fires for a drag/swipe gesture; `OnPickDownTrigger` carries down-state
semantics. Everything below is **general practice, not sourced**:

- **Plane translate** — intersect the pointer ray with a plane through the object,
  oriented to a chosen axis pair or to the camera.
- **Axis translate** — intersect the plane containing the axis most perpendicular
  to the view direction, then project onto the axis.
- **Rotate** — accumulate the signed angle between successive projections onto the
  rotation plane, so passing +/-180 degrees works.
- **Grab offset** — capture `hitPoint - objectOrigin` at drag start and hold it
  constant; re-deriving each frame causes the snap-to-cursor jump.
- **Drag threshold** — a few pixels before a press becomes a drag.
- **Camera suppression** — gizmo drag and orbit usually share a button.
- **Snapping** — quantize in the constraint's own space; surface the pre-snap value.
- **Undo** — one transaction per drag, not one per frame.

## 7. Touch and XR input

**Babylon (well documented):** `PointerObservable` / `PointerInfo` is a
device-agnostic pointer path covering mouse and touch alike. WebXR input is
documented via `WebXRInputSource` (motion controller, world-space pointer ray), the
WebXR input system managing controller lifecycle, and `WebXRNearInteraction` for
near hand/controller interaction.

**Unreal:** Enhanced Input is the documented device-abstraction layer. **Gap:** no
source documents touch-gesture handling within it — the abstraction is established,
touch specifics are not. World-space UI is documented via
`WidgetInteractionComponent`, which raycasts against a `WidgetComponent`.

Design implication (general practice): route mouse, pen, touch and XR far-ray
through one pick-source abstraction yielding an origin and a direction, varying only
target size and commit gesture. Near/hand interaction is proximity, not a ray, and
needs its own path.

## 8. Latency and target size

No source gives numbers. These are common HCI rules of thumb, **unverified against
this evidence set** — state them explicitly and expect challenge:

| Budget | Rule of thumb |
|---|---|
| Input to visible feedback | Under ~100 ms reads as instantaneous; hover one frame late is fine, a click commit is not |
| Pick cost per hover query | A 4 ms pick at 60 Hz eats a quarter of the frame |
| Touch target | ~9 mm / ~44 pt physical minimum |
| Pointer target | WCAG 2.5.8 suggests 24x24 CSS px |

## 9. Accessibility

**Babylon (documented, thin):** an accessibility layer can generate HTML "twin"
elements exposing `ActionManager` interactions — pick, left-pick, right-pick. One
page, `ActionManager`-scoped, not a general accessible object model.

**Unreal (documented, UMG only):** screen-reader support, a Blind Accessibility
overview, `FScreenReaderUser`, `CommonUI`, `UWidget`. **None describe accessibility
for the 3D viewport or gizmo interaction.**

**Therefore (design recommendation, explicitly uncited):** no documented bridge
from 3D picking and manipulation to an accessibility API exists in either engine.
Build it: a keyboard-reachable list of selectable entities, numeric transform entry
equivalent to every gizmo drag, focus order and announcements on selection change,
and a non-visual hover affordance.

## Platform pointers

| Technique | Babylon.js | Unreal Engine |
|---|---|---|
| CPU pick | `scene.pick`, `pickWithRay`, `PointerInfo` | `LineTraceByChannel`, `MultiLineTraceByChannel`, `BreakHitResult` |
| GPU ID pick | `GPUPicker` (explicit picking list) | no equivalent in this evidence |
| Pick reaction | `ActionManager`, `OnPickTrigger`, `OnPickDownTrigger` | Enhanced Input action, then a trace |
| Selection set | `SelectionOutlineLayer.addSelection` / `clearSelection` | no documented API — build your own |
| Highlight / outline | `HighlightLayer`, `SelectionOutlineLayer`, `EffectLayer` | custom depth + stencil + post-process (community-sourced) |
| Gizmos | gizmo feature page (mesh/bone attach) | `UInteractiveGizmo`, `UTransformGizmo`, `UTransformProxy`, `UInteractiveGizmoManager` |
| Device-agnostic input | `PointerObservable` | Enhanced Input |
| XR input | `WebXRInputSource`, `WebXRNearInteraction` | not covered here |
| Accessibility | `ActionManager` HTML twins | UMG screen readers, `FScreenReaderUser` |

Hand anything past this table — signatures, parameters, version behaviour — to the
platform specialist.

## Pitfalls: symptom to likely cause

| Symptom | Likely cause |
|---|---|
| Object jumps to the cursor on grab | Grab offset re-derived each frame, not captured at drag start |
| Click fires after a drag, or never | No drag threshold; Babylon's `OnPickTrigger` skips drag/swipe |
| Camera orbits while dragging a gizmo | Camera input not suppressed for the drag |
| Hover laggy only on some machines | Synchronous GPU readback stalling the pipeline |
| Hit test fine until the scene grows | No broad phase; or one full pick per pointer-move |
| Picking spikes on a few objects | `Trace Complex` on dense meshes |
| Thin or small objects unselectable | Pixel-exact ray, no widened target |
| Highlight persists on a deleted object | Outline membership treated as source of truth |
| Works on desktop, not on touch | Targets not resized for fingers; hover-only affordances |
| Undo replays a drag frame by frame | Transaction opened per frame, not per drag |


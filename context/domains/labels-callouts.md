# Labels, annotations, and callouts in real-time 3D

Evidence discipline: **[documented]** = backed by engine documentation or a
published paper in the source research. **[convention]** = standard practice the
research explicitly could *not* attribute to a source - a usable default, not a
fact. Present conventions as conventions.

---

## 1. The three substrates

| | Billboarded quad (bitmap) | SDF / MSDF text | DOM-HTML overlay at projected position |
|---|---|---|---|
| Crisp under zoom | No - atlas-locked | Yes, resolution-independent; **but** SDF is an approximation that degrades at very small sizes and on thin/delicate glyphs, and lacks hinting **[documented]** | Always |
| Occluded by geometry | Yes | Yes | **No** - outside the 3D pipeline |
| Gets fog / DOF / bloom / grading | Yes | Yes | No |
| Selectable, copyable, screen-reader reachable | No | No | Yes |
| Works in XR | Yes | Yes | No |
| Dominant cost | Draw calls + texture memory, one quad per label unless instanced | Glyph-atlas VRAM; batches to few draw calls | CPU projection + style write + browser layout/reflow, per label per frame |
| Scales to hundreds | Poorly without instancing | Best | Degrades on layout, not GPU |

**Babylon [documented]:** MSDF text ships as an add-on for sharp,
scale-independent 2D/3D text, including billboarded and instanced paragraphs.
`HtmlMesh` is the hybrid: usable as a genuine scene mesh (so it *does* occlude
and get occluded) or as a flat overlay, and documented as suited to rich HTML
content rather than many simple labels.

**[convention]** Dozens of rich, accessible panels → DOM / `HtmlMesh`. Hundreds
to thousands of short labels → SDF/MSDF atlas. Bitmap quads only for a small,
fixed, art-directed set.

---

## 2. Billboarding and orientation

**[documented]** Unreal's `WidgetComponent` exposes a Geometry Mode of flat
**Plane** vs curved **Cylinder** for world-space widgets. Read this precisely: it
is widget *surface curvature*, which the research treats as the engine-native
analogue of planar-vs-cylindrical - not a billboard-rotation enum.

**[documented]** In Babylon, orientation and depth visibility are separate,
un-reconciled concerns: making a 2D GUI control track a mesh's orientation needs
manual math, and billboard/occlusion interaction is a recurring user problem.

**[convention]** The mode taxonomy - no source establishes it:

| Mode | Behaviour | Use when | Failure mode |
|---|---|---|---|
| Spherical | Faces camera on all axes | Free orbit; must read from anywhere | Surface signage "swims" off the object |
| Cylindrical (yaw-only) | Rotates about world up only | Camera can pitch; real ground plane | Edge-on and unreadable from directly above |
| Screen-aligned | Lives in 2D, position from projection | Maximum legibility, HUD-like | No depth cue; occlusion must be added by hand |
| Fixed-screen-size | World-anchored, constant pixel height | Must stay readable at all distances | **Size no longer encodes distance** - re-encode via opacity, leader length, or explicit ordering |

---

## 3. Anchoring and leader lines

- **[documented]** Babylon: `AdvancedDynamicTexture.linkWithMesh` tracks a mesh's
  projected screen position, with `linkOffsetX`/`linkOffsetY` for screen-space
  offset. GUI `Line.connectedControl` lets one endpoint track another control - a
  real leader-line building block, not a routing solver.
- **[documented]** Unreal: no leader-line control appears in the research. It
  requires a custom UMG paint pass or line widget; no source gives a recipe.
- **[documented]** Academic view-management work on anchor/label separation and
  connecting geometry exists (hedgehog labeling; dynamic annotation of
  interactive environments; AGILE 2015 dynamic label routing). The research
  confirms the literature exists but does not reproduce its algorithms.
- **[convention]** Anchor to a stable feature (bounding-box face centre, a named
  attachment point), never a vertex LOD can delete. Give the leader a minimum
  length so it reads as a line. When the anchor leaves the viewport, choose
  explicitly between edge-clamping with a direction indicator and dropping the
  label - silence is not a policy.

---

## 4. Occlusion - the real, evidenced tension

Best-evidenced topic here, and the evidence shows a gap between engine design
intent and practitioner need. Do not smooth it over.

**[documented] Unreal:** world-space widgets participate in the depth buffer and
can be occluded; **screen-space widgets render outside the 3D world and are never
depth-occluded.** Community threads ask specifically for screen-space widget
occlusion, for depth-occluded screen-space widgets, and for z-ordering among
screen-space text widgets. No source shows Epic shipping an official solution -
the pattern is community workarounds (line traces, custom depth checks).

**[documented] Babylon:** the fullscreen `AdvancedDynamicTexture` / `linkWithMesh`
path is likewise not depth-occluded; users have asked how to hide a linked
placeholder when its mesh is behind other geometry, and billboard occlusion is a
separate open thread. Primitives available: mesh `occlusionType` and the
occlusion-queries feature - documented as order-dependent (the query is only
meaningful if the mesh renders after potential occluders, with depth state
preserved across render groups). Building blocks, not a label-occlusion system.
`HtmlMesh` used as a scene mesh is the exception that occludes correctly.

**[documented, adjacent]** A VTK billboard-text-behind-volume issue is weak
cross-toolkit evidence that billboard-vs-depth ordering recurs generally.

**[convention]** The three honest policies:

| Policy | Implementation | Per-frame cost |
|---|---|---|
| Draw through | Depth test off for the label layer; accept X-ray | ~free |
| Hide when occluded | One ray/line trace or depth sample per visible label | O(N) CPU scene queries - the usual budget killer |
| Dim / dash occluded | Same test, different presentation; keeps labels discoverable | Same O(N) |

Amortize by round-robin testing a subset per frame, and add hysteresis on the
visible/occluded transition or labels flicker at silhouette edges.

---

## 5. Decluttering and collision avoidance

**[documented]** Babylon has `moveToNonOverlappedPosition()` with `overlapGroup`
- real, but requires manual per-frame invocation (e.g. a render observer) and is
not priority-aware. Unreal exposes `ModifyProjectedLocalPosition`, the correct
hook for a custom declutter pass after projection; the source gives no algorithm.
Neither engine handles overlap for you.

**[convention]**

| | Greedy candidate-slot | Global optimization (force-directed / annealed) |
|---|---|---|
| Cost | ~O(N log N) with a screen-space uniform grid | Iterative, multiple passes per frame |
| Quality | Adequate at sparse-to-medium density | Packs dense sets better |
| Temporal stability | Stable if candidate order and priorities are fixed | Poor - small camera deltas cause large relayouts |
| Verdict | Default for a moving camera | Static label sets, or run on settle |

Stability matters more than packing: fixed priority order (high-priority labels
never yield), hysteresis before relocating, a per-frame movement cap, and
deterministic candidate ordering so one camera pose always yields one layout.

The "placement rings" scheme (rings 0-3 around the anchor, clustering into
badges) is plausible and common but **not** demonstrated by any source - one
candidate-slot scheme, not established practice.

---

## 6. Density LOD and fading

Weakest-evidenced part of the domain. **No source gives a fade threshold,
pixel-height cutoff, or density LOD policy for labels.** The nearest material is
the same view-management papers, plus Unreal's UMG optimization guidance and
invalidation/retainer mechanisms - which address *rendering cost* under many
widgets, not visual LOD policy.

**[convention]** A defensible ladder, to be stated and validated: full label →
abbreviated → icon/dot → cluster badge ("12 sensors") → hidden. Trigger on
projected size, distance, or local screen density, with separate enter/exit
thresholds at every rung. A cluster badge needs a defined expansion behaviour or
it is a dead end.

---

## 7. Legibility across distance

**[documented]** SDF/MSDF sources cover legibility trade-offs at small sizes and
across scale (see the caveat in section 1). MRTK-for-Unreal has an on-topic text
feature page for VR/MR legibility.

**[documented, negative]** No source provides calibrated minimum pixel or angular
sizes or contrast rules. Any number here is convention.

**[convention]** Contrast must be background-independent - outline/halo or an
opaque backing plate; text colour alone fails against a moving scene. Never
encode meaning in colour alone. Set a minimum rendered cap-height and cull below
it rather than shipping sub-pixel text.

---

## 8. Cost model

Per visible label per frame: one world→screen projection, one transform or style
write, optionally one occlusion trace, plus the shared declutter pass. This is
main-thread CPU and is independent of GPU headroom - a label system can tank
frame time on an idle GPU. SDF atlas text is the path that collapses many labels
into few draw calls **[documented]**; DOM overlay converts GPU cost into browser
layout/reflow **[convention]**.

---

## 9. Platform pointers (hand off, do not tutorial)

**Babylon.js:** `AdvancedDynamicTexture` · `linkWithMesh` + `linkOffsetX/Y` · GUI
`Line.connectedControl` · `moveToNonOverlappedPosition` + `overlapGroup` · MSDF
text add-on · 3D GUI controls / `Container3D` · `HtmlMesh` · mesh `occlusionType`
and occlusion queries.

**Unreal:** `WidgetComponent` World Space vs Screen Space · Geometry Mode
Plane/Cylinder · `ModifyProjectedLocalPosition` · `bUseInvalidationInWorldSpace`,
Invalidation Box, Retainer Box, UMG optimization guidelines · SDF text rendering.

Exact property semantics and signatures belong to the platform specialists.
Whether Babylon's 3D GUI natively exposes billboard-mode granularity comparable
to Unreal's Plane/Cylinder is **unsettled** in the research - do not assert it.

---

## 10. Symptom → likely cause

| Symptom | Likely cause |
|---|---|
| Labels visible through walls, unintentionally | Screen-space/fullscreen GUI path - never depth-occluded in either engine; no policy was chosen |
| Labels jitter, swap, or flicker under small camera motion | Declutter pass with no hysteresis, no fixed priority order, or non-deterministic candidate order |
| Frame time collapses with label count while GPU sits idle | Per-label projection + unamortized per-label occlusion traces on the main thread |
| Near and far labels indistinguishable | Fixed-screen-size billboarding with distance not re-encoded |
| Text fuzzy or broken only at small sizes / on thin glyphs | SDF approximation limits and absent hinting **[documented]** |
| Label detaches visually from the scene | DOM overlay in a post-processed pipeline - no fog, DOF, or grading applied |
| Text illegible over bright or busy background | No halo/outline or backing plate; contrast carried by text colour alone |
| Callout points at nothing after a camera move | Anchored to geometry that LOD or streaming swapped out |
| VRAM spike after localization | Glyph atlas sized for Latin, then asked to carry a CJK character set |
| "Engine will handle overlap" assumed in the design | It does not; both engines expose hooks only |

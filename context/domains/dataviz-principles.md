# 3D Data Visualization Principles — reference knowledge

Evidence base: graphical-perception literature (Cleveland & McGill and later
replications), the RAND gratuitous-3rd-dimension experiment, extraneous-information
and 3D-pie-chart studies, St. John et al. shape-vs-relative-position studies,
Moreland's colormap advice, volume-rendering and transfer-function surveys, and
point-cloud LOD/streaming papers. Confidence is **medium** — separate studies, not
one replicated authority. Claims marked **[unsupported]** are not backed by it.

## 1. The 3D-or-not decision

**The removal test.** Delete the third axis. If a meaningful data relationship
disappears, 3D is justified; if only decoration disappears, use 2D. This is the
criterion the critique literature proposes.

**The task split** — the most consistent finding across task-comparison studies:

| Reader task | Better | Why |
|---|---|---|
| Understand a shape / identify an object | 3D | Surface form is the data |
| Containment, connectivity, enclosure | 3D | Topology is spatial |
| Trace a trajectory or flow path | 3D | Path is spatial |
| Approximate layout / navigation | 3D | Layout is spatial |
| Precise relative position or distance | 2D | Foreshortening and occlusion corrupt it |
| Compare magnitudes, rank values | 2D | Position/length are the accurate channels |
| Read an exact value | 2D or table | 3D offers no path to precision |

Data that earns 3D: medical imaging, CFD, geology, terrain, molecular geometry,
simulation, navigable environments — *when* the task is in the top four rows.

**When both task families are real**, the established answer is not a compromise
view but **coordinated multiple views**: linked 2D slices, map plus profile,
scatterplot matrix, or a combined 2D+3D interface. Measurement lives in the 2D
panes, shape understanding in the 3D pane.

## 2. Encoding channels

Accuracy ordering from the graphical-perception program, best first:
position on a common scale; position on non-aligned scales / length; angle and
slope; area; volume, density, saturation; hue and shading. 3D corrections:

- Depth-axis position is **not** common-scale position — it inherits
  foreshortening and occlusion on top of the base ordering.
- Volume sits near the bottom. If you scale a glyph, prefer one linear dimension.
- Color competes with shading (§4), weakening an already-weak channel.

**Allocation rule:** most precision-critical variable → screen-aligned X or Y;
second → the other screen axis or length; a "more-or-less" variable → depth;
ordinal/categorical → hue; a fourth quantitative variable means the view is
overloaded.

**Motion/animation as an encoding channel is [unsupported]** here — no dedicated
source covers it. Claims about animated transitions, change blindness, or
motion-encoded time are judgment, not finding.

## 3. Depth cues, occlusion, projection

Depth is reconstructed from multiple, often conflicting cues.

| Cue | Strength | Cost / risk |
|---|---|---|
| Occlusion (interposition) | Strongest ordinal cue | Hides data; ordering, not magnitude |
| Motion parallax / rotation | Strong, reliable | Needs interactivity; absent in static images |
| Stereo | Strong | Hardware-dependent; XR only |
| Perspective size change | Moderate | Introduces foreshortening error |
| Cast shadows / shading | Moderate | Consumes the luminance channel (§4) |
| Drop lines to a reference plane | Moderate, and *quantitative* | Adds geometry and clutter |

**Occlusion is data loss** — studied both as information loss and as a source of
depth-ordering error in clutter. Whatever is frontmost wins; everything behind it
goes unreported. Mitigations:

| Technique | Buys | Costs |
|---|---|---|
| Viewpoint change / enforced camera | No geometry change | Needs one good viewpoint to exist |
| Clipping plane / cross-section | Exact, interpretable | Reader manages the plane |
| Cutaway | Context around a focus | Per-case authoring |
| Selective transparency | Reveals interiors | Order-dependent; weakens depth ordering |
| Depth peeling / OIT | Correct transparency | Extra passes, real frame cost |
| Focus+context | Keeps the surround | Distortion must stay legible |
| Isolate selection | Unambiguous | Loses spatial relations while active |

**Projection.** Perspective foreshortening means equal data distances do not map
to equal screen distances — a documented mechanism behind degraded
position/magnitude accuracy. Countermeasures: orthographic or weak perspective,
axis-aligned reference planes, gridded base plane, drop lines. Orthographic makes
near/far ambiguous; accept that when quantitative reading matters.

**Unsettled:** the evidence does not quantify where occlusion or distortion erases
3D's shape advantage. Claim no threshold there.

## 4. Color on shaded surfaces

The central confound, explicit in Moreland's advice: **shading communicates
normals and curvature through luminance, and data color usually varies in
luminance too.** Viewers cannot cleanly separate them.

- Prefer colormaps whose lightness varies little enough that data color is not
  read as shading; perceptually uniform maps exist precisely for this.
- **Rainbow/jet is a documented hazard** on 2D or 3D alike: uneven perceptual
  lightness creates false boundaries and hides real gradients. Every color source
  agrees.
- Lightness-constancy research shows viewers *attempt* this separation — real,
  but not reliable.
- Always check under color-vision deficiency; always ship a calibrated legend.
- **Escape hatch:** if data must be read precisely off a surface, remove the
  competition — unlit material, shape carried by silhouette or contours.

Map type: sequential and perceptually uniform for unipolar magnitude; diverging
with a clearly neutral midpoint only when a reference value is meaningful;
categorical and CVD-safe (≤ ~8 classes) for categories; restricted-lightness
sequential, or an unlit surface, whenever the value must be read precisely.

## 5. Volume rendering vs isosurfaces

The surveys treat these as **complementary, not either/or**.

| | Direct volume rendering | Isosurface extraction |
|---|---|---|
| Method | Ray casting, splatting, shear-warp, texture slicing; composites transfer-function-mapped samples along rays | Marching Cubes and relatives; polygonize one threshold |
| Shows | Multiple materials, fuzzy boundaries, no prior geometry | One crisp surface per level |
| Hard part | **Transfer-function specification** — the documented central difficulty (a dedicated "bake-off" plus a design-method survey) | **Threshold choice**, plus noise sensitivity |
| Cost | Higher compute; compositing-order and clutter issues | Cheap once extracted; easy to light |
| Failure mode | Opaque mush, or hidden structure, from a bad transfer function | Silently discards what the threshold excluded |

**Decision rule:** if the reader must trust a boundary, extract an isosurface and
disclose the threshold. If structure is genuinely fuzzy or multi-material, use DVR
and budget real effort for the transfer function.

## 6. Point clouds and meshes at scale

Documented toolkit for large point clouds: **octree indexing, level of detail,
culling, progressive/streaming rendering**. For very large N, simultaneous LOD
generation and rendering (hierarchy built while the user interacts) and
virtualized point-cloud rendering are documented approaches.

The tradeoff is **fidelity vs interactive latency**: aggressive decimation loses
narrow features and rare points — for anomaly detection, exactly the data. State
what your LOD may drop.

**Mesh-specific practice at scale (decimation, indexed buffers, out-of-core
streaming) is [unsupported]** beyond what generalizes from point clouds.

## 7. What the critique literature established

- RAND controlled study (bars, pies, tables, with and without a gratuitous third
  dimension): **no accuracy benefit for bar charts**, a **small but significant
  negative effect for pie charts**.
- The extraneous-information study reports reduced accuracy more broadly across 3D
  bar and pie presentation, including foreground-slice prominence obscuring data.
- Tufte-style critique treats gratuitous 3D as the paradigm case of non-data ink.
- **Genuine tension:** "useful junk" research finds embellishment can aid
  memorability even while not aiding — or modestly hurting — immediate accuracy.
  The sources do not adjudicate accuracy vs memorability.
- **Unresolved:** whether 3D harm to bar charts is reliably null (RAND) or real
  (extraneous-information study). Claim neither.

## 8. Symptom → likely cause

| Symptom | Likely cause |
|---|---|
| Reviewers disagree which bar is taller | Depth-axis magnitude read under perspective |
| Back half of the data invisible | Occlusion with no management plan |
| A ridge appears that isn't in the data | Rainbow colormap's uneven lightness |
| Bright value or bright face? | Colormap lightness competing with shading |
| Volume render is opaque mush | Transfer function over-assigns opacity |
| Feature vanished when zooming out | LOD decimation dropped narrow/rare features |
| Values look bigger than they are | Quantity mapped to volume, or a foreground slice |

## 9. Platform pointers

Enough to hand off. The API decision belongs to the platform specialist.

**Babylon.js** — orthographic: `Camera.ORTHOGRAPHIC_CAMERA` plus
`orthoLeft/Right/Top/Bottom`. Cross-sections: `scene.clipPlane` (`clipPlane2..6`)
or material-level clipping. Large point sets: `PointsCloudSystem`, thin instances,
`SolidParticleSystem`. DVR has **no built-in path** — a custom ray-march
`ShaderMaterial` over `RawTexture3D`. Transparency order: `alphaIndex` / depth
pre-pass; OIT is a custom pass. Unlit data surface: `unlit` on `PBRMaterial`.

**Unreal Engine** — orthographic: `CameraComponent.ProjectionMode`. Volumetric
data: `SparseVolumeTexture` and volumetric material domains. Large point sets:
LiDAR Point Cloud plugin or Niagara. Dense meshes: Nanite (handles LOD/streaming
structurally). Isolation and outlines: Custom Depth / stencil. Unlit data surface:
the `Unlit` shading model. Cross-sections: clip-plane material functions.

## 10. Pitfalls

- **Extruding a 2D chart.** Depth encodes nothing; the pie result is negative.
- **Encoding decided after the scene exists.** Occlusion management and projection
  change what geometry must be built — deciding them late is a rebuild.
- **Static 3D image.** Rotation and parallax are the cues that redeem 3D; a static
  render discards them and keeps every distortion.
- **Claiming a precise read that isn't available.** Every 3D view has reads it
  cannot support; failing to state them is the failure.
- **Free camera plus a quantitative claim.** If the reader can orbit,
  foreshortening shifts under them and no comparison stays stable.
- **"Explore the data" accepted as a task.** Specifies nothing, justifies
  nothing.

# amplifier-bundle-3d-developer

An [Amplifier](https://github.com/microsoft/amplifier) bundle for building 3D
visualization systems: ten engine-agnostic domain lenses, two platform
specialists (Babylon.js, Unreal Engine 5), three adversarial critics, and two
review skills that run them.

## Why it is shaped this way

Building a 3D visualization is not one skill. It is ten, and they disagree with
each other for good reasons. The scene architect wants fewer, bigger batches.
The label designer wants a screen-space pass per annotation. The post-FX artist
wants depth of field that blurs the labels the label designer just made crisp.
The perf critic wants all three to justify their frame time.

A single generalist agent resolves those tensions silently — usually in favour
of whichever concern it was thinking about last, and always without telling you
a tradeoff was made. So each lens is a separate agent that owns exactly one
question, refuses the others by name, and is required to state an explicit
frame-cost claim. The disagreement then happens in front of you, *before* the
frame budget blows, rather than in a profiler afterwards.

The critics are separate from the designers for the same reason, one step
further: a critic that helped produce the design is not reviewing it.

## The bench

Fifteen agents. Each one's frontmatter carries an explicit "DO NOT USE WHEN"
routing list, so they hand off to each other rather than guessing.

### Domain lenses — engine-agnostic design

| Agent | The one question it owns | Reach for it when |
|---|---|---|
| `3d-developer:dataviz-strategist` | Should this be 3D at all, and what is encoded in what? | Before anything is architected. The only lens permitted to answer "this should be a 2D chart" — and required to say so when true |
| `3d-developer:scene-architect` | How is content structured, partitioned, tiered, instanced, held resident? | Object counts or dataset size grow; streaming, LOD, octrees, memory climbing during navigation |
| `3d-developer:rendering-engineer` | What does a frame cost, and in what order does it draw? | "It's slow"; a new pass or shadow cascade needs justifying; forward vs deferred vs clustered; sizing a 16.6ms or 11.1ms budget |
| `3d-developer:shading-artist` | What is this surface made of, and what does it cost per pixel? | PBR setup, node/shader graphs, channel packing, transparency sorting, shader permutation counts |
| `3d-developer:interaction-designer` | How does a person point at, select, or manipulate scene content? | Picking the wrong object, hover killing the frame rate, gizmos, box-select over a million points, XR input, 3D accessibility |
| `3d-developer:camera-navigator` | How does the viewer move through and frame the scene? | Orbit vs fly vs turntable, "frame selection", ortho-vs-perspective honesty, near/far planes, z-fighting, motion sickness |
| `3d-developer:postfx-artist` | Which whole-frame passes run, in what order, at what fill-rate cost? | DOF, fog, bloom, SSAO, SSR, motion blur, tone mapping, outlines, AA choice (MSAA/FXAA/TAA/TSR) |
| `3d-developer:label-callout-designer` | How does text bind to a thing in 3D and stay readable as the camera moves? | Labels overlapping, callouts pointing at hidden geometry, SDF vs DOM overlay, leader lines, decluttering, draw-through policy |
| `3d-developer:animation-engineer` | What moves, driven by what clock, and what does the motion mean? | Keyframes, skeletal blending, morph targets, timeline scrubbing, data-driven transitions, animating tens of thousands of objects |
| `3d-developer:particle-fx-artist` | What is emitted, simulated, and blended — and what does the overdraw cost? | Smoke, explosions, sparks, flow, fluid; "the explosion reads as one flat puff"; transparent overdraw suspected in the frame |

### Platform specialists — used after the design exists

| Agent | The one question it owns | Reach for it when |
|---|---|---|
| `3d-developer:babylonjs-specialist` | Which named Babylon.js construct implements this, and what does it cost in a browser? | The design is settled and you need `DefaultRenderingPipeline`, thin instances vs `InstancedMesh`, `GPUPicker`, WebGL2-vs-WebGPU feature availability, GLB decode cost, device-pixel-ratio fill rate |
| `3d-developer:unreal-specialist` | Which UE5 construct implements this, Blueprint or C++, at what cost? | The design is settled and you need Nanite eligibility, Lumen, World Partition/HLOD, Niagara modules, Sequencer, post-process volumes, UMG, Unreal Insights |

### Critics — they judge, they do not design

| Agent | The one question it owns | Reach for it when |
|---|---|---|
| `3d-developer:perf-budget-critic` | Will this hold frame time, and what measurement would settle it? | A design or diff is finished; a per-frame cost is unstated; a claimed optimisation needs its evidence checked by someone told nothing about what others concluded |
| `3d-developer:visual-quality-critic` | Can a human eye make the intended reading, and would a regression be caught? | A screenshot, render or design needs an independent read-check; judging whether a golden-image test at a given resolution and tolerance would catch the defect |
| `3d-developer:correctness-critic` | Which of these is a numerical error wearing the costume of an art-direction choice? | Colours washed out after a blend, dark halos on cut-outs, lighting inverting on mirrored meshes, normal maps reading inverted, bar heights no longer matching their numbers |

## Install

Three entry points. Pick one.

**Everything** — foundation + the core bench + both platform layers, as a
primary bundle:

```bash
amplifier bundle add git+https://github.com/colombod/amplifier-bundle-3d-developer@main
amplifier bundle use 3d-developer
```

**Web only** — foundation + the core bench + the Babylon.js specialist. No
Unreal knowledge is loaded:

```bash
amplifier bundle add "git+https://github.com/colombod/amplifier-bundle-3d-developer@main#subdirectory=bundles/with-babylonjs.yaml"
amplifier bundle use 3d-developer-babylonjs
```

**Unreal only** — the mirror image; no Babylon.js knowledge is loaded:

```bash
amplifier bundle add "git+https://github.com/colombod/amplifier-bundle-3d-developer@main#subdirectory=bundles/with-unreal.yaml"
amplifier bundle use 3d-developer-unreal
```

### Layering onto a bundle you already run

The three root bundles above *replace* your primary bundle. More often you want
the capability added to whatever you already use — that is the `3d-core`
behavior, with `--app`:

```bash
amplifier bundle add "git+https://github.com/colombod/amplifier-bundle-3d-developer@main#subdirectory=behaviors/3d-core.yaml" --app
```

That gives you the ten domain lenses, the three critics, the two skills, and a
~250-token awareness pointer. It deliberately does **not** include either
platform specialist. Add the ones you actually target, the same way:

```bash
# web
amplifier bundle add "git+https://github.com/colombod/amplifier-bundle-3d-developer@main#subdirectory=behaviors/platform-babylonjs.yaml" --app

# Unreal
amplifier bundle add "git+https://github.com/colombod/amplifier-bundle-3d-developer@main#subdirectory=behaviors/platform-unreal.yaml" --app
```

The split is the point: a web-only consumer never carries Unreal knowledge in
its agent catalog, and vice versa.

## Using it

The heavy reference knowledge lives inside the agents — each domain lens
`@mention`s its own domain doc, so it costs tokens only in that agent's
sub-session, never in yours. Delegate rather than reason from memory.

The intended order: **scope** (`dataviz-strategist` first, and let it say no) →
**design per domain**, engine-agnostic → **reconcile** the summed frame costs →
**review cold** → **then** pick a platform. Designing against an engine's
conveniences before that last step is how a design ends up shaped by whatever
the engine made easy.

Two skills drive that loop:

| Skill | What it does |
|---|---|
| `3d-build-loop` | Runs the whole multi-domain job **inline, in your session**, so it can see the conversation the request was made in: scope it, fan the relevant lenses out in parallel, add up their cost claims and name the collisions, run the critics, revise, hand off to a platform. Capped at two revision rounds |
| `3d-design-review` | Runs **just the review**, in a forked session: three critics, one artifact, one verdict |

Both are user-invocable (`/3d-build-loop`, `/3d-design-review`) and
model-invocable via `load_skill`.

## The review loop

`3d-design-review` fans all three critics out in a single response, each with
`context_depth="none"` and the same artifact — not a summary of it. No critic is
told what the others were asked. Three verdicts reached independently are three
pieces of evidence; three verdicts reached after the critics read each other are
one verdict wearing a hat.

The aggregation rules are fixed, not judgement calls:

- **Worst wins.** Any `FAIL` → **BLOCK**. Any `CONCERN` → **PASS-WITH-NOTES**.
  All `PASS`/`N/A` → **PASS**.
- **A FAIL is never softened** because the other two critics were happy. No
  averaging.
- **`N/A` is an abstention, not a blocker** ("no performance surface here"), and
  is never aggregated into one.
- **Silent errors go first** — anything the correctness critic flags as
  rendering plausibly with wrong values outranks everything, because it is the
  class nobody else catches.
- **Fail loud.** A critic that errors or returns no structured verdict is
  reported as such and the overall verdict becomes `INDETERMINATE` — never
  `PASS`, never a stand-in.
- **Every claim is attributed and at least one line per critic is quoted
  verbatim.** No anonymous synthesis.
- A second round happens **only** on a direct conflict — two critics holding
  opposing positions on the *same* finding — and relays each position verbatim
  and uncurated. One round only. A standing disagreement is a real tradeoff for
  you to resolve, not a failure to converge.

**Recipes — the gated, resumable variants.** Two live in `recipes/`, both
`schema_version: 2` with a closed dependency set:

| Recipe | What it adds over the skill |
|---|---|
| `recipes/design-review-pipeline.yaml` | The three critics over one existing `artifact`, then a human approval gate (`default: deny`) before any revision runs. Checkpointed and resumable. |
| `recipes/build-3d-feature.yaml` | The full arc: `dataviz-strategist` scoping with a gate that lets you stop when the verdict is "this should be 2D", then a fixed design set (scene → shading → camera → rendering, rendering last because it owns the total budget), then the cold critics, a gate, and a conditional platform handoff. |

Two honest notes on the recipes. First, a recipe step's `agent:` is a static
string, so the critics run **sequentially** rather than in parallel — their
independence is preserved (each is a fresh sub-session and no critic's prompt
interpolates another's output) but wall-clock is 3×. The skill is the parallel
path. Second, `build-3d-feature.yaml`'s design set is **fixed** for the same
reason; the file carries a commented extension recipe, including the step people
forget — adding a lens means adding its output to the reconcile prompt, or the
design gains a cost the budget never sees.

Both recipes pin the bundle at `@main`. Pin them to a tag or SHA before anything
depends on them.

## How the knowledge was built

Every knowledge-base doc under `context/` was distilled from a `deep-research`
run. The full report and complete source list for each is committed under
`docs/research/`, with the run id recorded in the file: **eleven runs, 736
sources total** (`deep-research` 0.9.0, depth medium, backend perplexity).

The runs, with their source counts: `3d-dataviz-principles` (75),
`animation-systems` (59), `camera-navigation` (70), `interaction-picking` (74),
`labels-callouts` (59), `materials-shading` (73), `particle-systems` (60),
`post-processing-fx` (54), `rendering-pipeline` (65), `review-and-quality` (75),
`scene-architecture` (72).

Each report opens with a verbatim confidence note and a scope check that
separates **sourced engine facts** from **analytical generalization**, and says
plainly where the evidence ran out. Examples actually in the files: the
scene-architecture run documents Babylon's `TransformNode`/`AssetContainer`/thin
instances and Unreal's World Partition/HLOD/Nanite from real doc pages, then
states that the cross-engine "architectural principles" layer is generalization
no source asserts. The rendering-pipeline run found exactly one published
millisecond budget (Lumen's ~4ms GPU target at 1080p/60) and says the per-stage
16.6ms breakdown is practitioner synthesis, not a documented artifact. The
review-and-quality run reports that no source documents a graphics code-review
checklist, tangent-space conventions, or Spector.js at all.

The knowledge bases carry those evidence tags through — `[doc]` /
`[documented]` versus generalization, per claim. That is also why the agents are
instructed to say **"verify against current docs"** rather than produce an API
signature from memory: the distilled docs are a map of what was documented at
distillation time, not a substitute for the engine's reference.

## Limitations

Read these before trusting anything the bench tells you.

- **It is version-point-in-time.** Babylon.js moves fast and the distilled docs
  will drift. Several Unreal pages the research found were 4.27-era, with
  continuity into UE5 unconfirmed — the `animation-systems` and
  `materials-shading` runs both flag pages that appear to describe the same
  feature under different names across eras, and do not merge them silently.
- **Some areas had thin or no documentation** and are marked as convention
  rather than fact: fit-to-view margins, transition-duration ranges, minimap
  patterns and motion-sickness rules (camera-navigation); data-driven transition
  patterns (animation); frame-budget checklists, graphics code-review
  checklists, and normal/tangent-space conventions (review-and-quality); the
  entire cross-engine principles layer (scene-architecture).
- **No number in this repo is a measurement taken from this repo.** Every frame
  cost, threshold and budget is either cited to a source or labelled as
  generalization. The agents produce *falsifiable cost claims with the
  measurement that would settle them* — they do not produce measurements.
- **There are no automated tests of agent output quality.** Nothing here
  verifies that a lens gives good advice; the critics are the only quality
  mechanism, and they are themselves LLM agents.
- **Two platforms only.** three.js, Unity, Godot, WebGPU-native and Vulkan/DX12
  work are not covered. The domain lenses are engine-agnostic and will still be
  useful; the implementation handoff will not.

## Contributing: adding a third platform layer

The platform split is designed so a new engine costs **zero changes to existing
files**. Add three:

1. `behaviors/platform-<name>.yaml` — a behavior whose only job is
   `agents: include: [3d-developer:<name>-specialist]`. Copy
   `behaviors/platform-babylonjs.yaml`; it is twelve lines.
2. `agents/<name>-specialist.md` — one specialist. Its `meta.description` must
   carry the same routing contract as the existing two: what it is authoritative
   on, and an explicit **DO NOT USE WHEN** list naming the domain lens that owns
   each engine-agnostic question, plus the sibling platform specialists.
3. `context/platforms/<name>.md` — the domain-to-construct map, `@mention`ed
   from the agent body so it loads only in that agent's sub-session. Carry
   evidence tags through as the existing platform docs do.

Optionally add `bundles/with-<name>.yaml` (three includes: foundation,
`3d-core`, your behavior) for a pre-composed root bundle.

Do **not** add the specialist to `behaviors/3d-core.yaml`. Core stays
engine-agnostic so consumers only pay for the engines they target.

## License

MIT — see [LICENSE](LICENSE).

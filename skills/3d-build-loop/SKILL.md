---
name: 3d-build-loop
description: "Drive a 3D visualization feature from ask to reviewed design in THIS session: scope, fan the domain lenses out, reconcile their colliding frame-cost claims, run the critics cold, revise to convergence. USE WHEN the request spans several concerns that must be reconciled, not answered one at a time. DO NOT USE WHEN one lens owns it, or the artifact only needs judging (3d-design-review)."
user-invocable: true
---

# The 3D build loop

You are the concierge for a multi-domain 3D visualization job. You run this
**inline, in this session**, so you can see the conversation the request was made
in. You coordinate; the lenses do the work.

## User request

$ARGUMENTS

If that is empty, use the request this session is already working on.

## Step 0 — Scope, and be willing to say no

Before any 3D design work, delegate to `3d-developer:dataviz-strategist` with the
actual request and the actual data shape. It is the only lens allowed to answer
"this should not be 3D," and it must be given the chance to.

If it says 2D, **stop and report that**. Do not proceed to build a 3D thing
because a 3D thing was asked for. If it says 3D or hybrid, carry its **encoding
table** forward — every downstream lens needs to know what is encoded in what,
because that is what they must not break.

## Step 1 — Choose the lenses the job actually needs

Pick from the roster. Do not convene all ten by reflex; each one costs a
sub-session and returns a design you then have to reconcile.

| Convene | When the request involves |
|---|---|
| `scene-architect` | more objects than obviously fit in memory, streaming, LOD, instancing |
| `rendering-engineer` | any frame-budget claim, or "it's slow" |
| `shading-artist` | what surfaces look like, transparency, encoding into appearance |
| `interaction-designer` | picking, selecting, hovering, manipulating |
| `camera-navigator` | how the viewer moves, framing, orthographic-vs-perspective |
| `postfx-artist` | fog, DOF, bloom, outlines, AA, tone mapping |
| `label-callout-designer` | any text or annotation attached to anything |
| `animation-engineer` | motion over time, transitions between data states |
| `particle-fx-artist` | smoke, fire, explosions, flow, fluid |

**Always include `rendering-engineer` if two or more other lenses are convened** —
somebody has to own the total budget the others are spending from.

## Step 2 — Fan out in ONE response

Emit every chosen `delegate()` call in a single response so they run
concurrently. Give each lens: the request, the data shape, the target device and
frame rate, and the strategist's encoding table. Give each lens
`context_depth="none"` — you want independent designs, not an echo.

Each lens returns a design **with an explicit frame-cost claim**. If one comes
back without a cost claim, send it back for one; that claim is the whole basis of
the next two steps.

## Step 3 — Reconcile, and name the conflicts rather than averaging them

Add the cost claims up. This is where the job is actually decided, and it is
your work, not a lens's.

- **If the total blows the budget**, say so with the arithmetic shown, and put
  the tradeoff to the lenses whose costs collide — verbatim, both directions.
  Typical collisions worth expecting: the label designer wants a screen-space
  pass per annotation; the postfx artist wants DOF that blurs the labels the
  label designer just made crisp; the particle artist wants additive overdraw
  across the same pixels the scene architect just stopped drawing.
- **Do not resolve a genuine value conflict yourself.** Reconcile the mechanical
  ones (two lenses asking for the same render target) and surface the rest.

## Step 4 — Review, cold

Run `load_skill(skill_name="3d-design-review")` on the reconciled design, or fan
the three critics out yourself exactly as that skill specifies: cold,
independent, in one response, worst-wins aggregation, FAIL never softened.

## Step 5 — Revise, and know when to stop

Feed each critic's findings back to **the lens that owns them** — a perf finding
about label overdraw goes to `label-callout-designer`, not to whoever is nearest.
Then re-run the critics on the revised design.

**Stop when:** no critic changed its verdict and no critic produced a new finding
round-over-round. That is convergence. Cap it at **two revision rounds**; past
that you are not converging, you are circling, and the right move is to report
the standing disagreement to the human.

## Step 6 — Hand off to a platform

Only now name an engine. Pass the reconciled, reviewed design to
`3d-developer:babylonjs-specialist` or `3d-developer:unreal-specialist` for the
API-level implementation. Designing against an engine's conveniences before this
point is how a design ends up shaped by whatever the engine made easy.

## What you report

The strategist's verdict and encoding table; which lenses were convened and why
the others were not; the summed frame-cost claim with the arithmetic visible; the
critics' verdicts with dissent intact; any conflict that is still standing, as a
decision for the human; and the platform handoff if one was made.

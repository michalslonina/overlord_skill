# Overlord

A Claude Code skill that turns the foreground session into an orchestrator: work is
delegated to background agents, and routing is cost-aware — trivial tasks to cheap models,
deep tasks to strong ones.

**Status: spec only.** Nothing is implemented, nothing is installed, no hook is wired. This
repo currently holds the agreed design.

## Layout

| Path | What |
|---|---|
| `SKILL.md` | The skill spec — modes, tiers, pool lifecycle, caveats |
| `docs/design.md` | The design record: decisions, their reasons, corrections, open questions |
| `docs/prior-art.md` | Prior-art survey that started this |

## The shape of it

Two orthogonal settings:

- **`mode`** — `off` / `normal` / `eager` / `auto`; *who* does the work. The ladder is also
  the enforcement spectrum: `eager` is the only mode that arms a blocking hook.
- **`model`** — `auto` / alias / full id; *which model* delegated work lands on. `auto` is
  the cost-aware classifier.

Most of the machinery is harness, not code: a standing pool is **agent teams**, anti-stall
is the **`TeammateIdle` hook**, and goal conformance is the **shared task list**. The one
thing that is genuinely unbuilt is inter-agent visibility — there is no hook or event for
message bodies, so seeing the traffic means polling mailbox files.

## Before building

Two things gate this, both in `docs/design.md`:

1. **Which concrete models back each alias.** Unmapped, `model: haiku` sends a Claude model
   id to a non-Claude endpoint and fails at request time. And if the session model is
   itself the cheap tier, delegation buys context and parallelism but no money.
2. **Agent teams are experimental and untested on a non-Claude session model.** Suggested
   first step is a two-teammate smoke test rather than building on the assumption.

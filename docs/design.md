# Overlord — design record

Date: 2026-10-07
Status: spec agreed; nothing implemented, nothing installed.

This records what was decided and why, so the reasoning survives. `SKILL.md` is the spec;
this is the argument behind it.

## The problem

Delegate foreground work to background models, cost-aware: trivial tasks to cheap models,
deep ones to strong ones. The foreground stays on overwatch — planning, dispatching,
verifying, integrating.

## Two mechanisms that are orthogonal

The starting point was an existing `cost-aware-model-router` skill, and it is **not** the
same thing as what was being asked for:

- `query_model.py` POSTs a prompt to an external API and prints text. No tools, no file
  access, no memory between calls. It cannot edit a file or run a test, so it can never
  delegate a *task* — only answer a *prompt*.
- Subagents and teammates have tools and their own context. These delegate tasks.

The reusable part of the router is its **decision policy**, not its transport. Notably it
was configured but never used — zero invocations in `history.jsonl` since it was written.

The two split cleanly rather than competing: because `ANTHROPIC_BASE_URL` is pinned to one
provider, delegated work can only reach models that provider serves. The router's free
OpenRouter tier is therefore **unreachable from subagents** and stays where it belongs —
text-only work with no tools to lose. The sensitivity gate survives as its concern.

## Settings

| Setting | Values | Governs |
|---|---|---|
| `mode` | `off` `normal` `eager` `auto` | who does the work |
| `model` | `auto` \| alias \| full id | which model delegated work lands on |

Resolved from an earlier open question: the four modes are a **delegation-aggressiveness**
ladder, which makes the ladder itself the enforcement spectrum. The `normal` vs `auto`
collision is resolved by making `auto` the per-task choice between the two fixed postures.

## Decisions and their reasons

**`auto` flips normal ↔ eager, never off.** Auto reaching `off` means the foreground
deciding how constrained it is, and it will drift there under load. Backwards on large
tasks, and nearly free of benefit since `normal` already permits trivial edits.

**`auto` cannot be independently enforced.** The flip decision is the foreground's own; a
hook is a separate process per tool call and cannot observe it. Making it enforceable needs
a state file the agent writes and the hook reads — which is a *commitment device*, not a
boundary: the agent owns the file. Since the file must exist anyway for static `eager` to
be readable, `auto` writing to it is nearly free — but it should be understood as "the
agent locks itself in," not "the agent is prevented."

**Tamper-proofing was dropped as too much.** Full analysis: the box has no passwordless
`sudo`, so a root-owned state file set via `sudo` is a real boundary — but the user is in
the `docker` group, which is root without a password, and the agent runs as the same UID.
So no permission scheme separates agent from user. The useful substitute is **authority as
a band**: a hardened file holds `mode`, a soft file holds the within-task choice consulted
only when `mode=auto`. Tampering then only picks between postures already sanctioned. In
`eager`, `Edit`/`Write` are blocked so the hook and settings cannot be edited through
normal tools — it is `Bash` that leaks.

**Keep while hot, reap when cold.** Corrected mid-design: an idle agent burns no tokens, so
the cost objection to keeping agents alive was wrong. The real speedup is prompt-cache
warmth, which resumption *can* retain within the TTL. Warmth decays, so aliveness pays only
inside the window — hold for near-term reuse, reap on TTL or goal completion, never eagerly.
The foreground owns the lifecycle via `TaskStop`.

**The pool is teammates, not running-idle subagents.** No warm-pool feature is documented;
the persistence model for subagents is resume-by-name. Teammates are the supported
long-lived, addressable, revivable mechanism.

## Corrections made during design

Recorded because they were load-bearing and stated wrongly at the time:

1. Claimed subagent sends route through the parent, so oversight would be structural and
   free. **Wrong** — the quoted wording is stale. Named subagents message each other
   directly; teammates message by name with no relay. The topology is a mesh, so oversight
   is work, not a consequence of the shape.
2. Claimed the parent sees all communication. **Wrong** — no hook or event exists for
   inter-agent message bodies. Only final results, plus mailbox/transcript files on disk.
3. Claimed resumption preserves content but not warmth. **Too strong** — a resumed run can
   keep reading the cache the original run warmed, bounded by TTL.
4. Claimed `model: haiku` might silently fall back to the session model. **Wrong** — the
   alias resolves through the provider mapping; unmapped, it fails loudly at request time.

## Open questions

- **Which concrete models back each alias.** Unresolved, and it decides whether the tier
  axis is real. If the session model is itself the cheap tier, delegation buys context and
  parallelism but no money.
- **Whether to adopt experimental agent teams at all**, given they are untested on a
  non-Claude session model.
- **Whether the hook is ever installed.** It is not written yet; the lockout risk (a
  misparse fails *closed*, a timeout fails *open*) means it should be landed in `normal`
  first and verified before `eager` is wired.

## Suggested first step

A throwaway two-teammate smoke test: confirm two named agents actually exchange a message
on this setup before any of the above is load-bearing.

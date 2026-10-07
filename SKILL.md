---
name: overlord
description: Use when the main session should act as an orchestrator rather than a worker — delegating research, implementation and verification to background agents while the foreground stays on overwatch. Covers delegation posture (off/normal/eager/auto), cost-aware model tiering for delegated work, and the lifecycle of a standing agent pool. Not for tasks touching secrets, credentials, or personal/financial/health data.
---

# Overlord

The foreground session plans, decomposes, dispatches, verifies and integrates. It does not
do the work itself. Work goes to background agents, and the routing of that work is
cost-aware: trivial tasks to cheap models, deep tasks to strong ones.

## Two settings, orthogonal

| Setting | Values | Governs |
|---|---|---|
| `mode` | `off` `normal` `eager` `auto` | *who* does the work |
| `model` | `auto` \| `haiku`/`sonnet`/`opus` \| full model id | *which model* delegated work lands on |

Both accept `auto`, and in both it means the same thing on its own axis: let the system
decide. Modes and tiers do not interact — every delegated task gets a tier in `normal`,
`eager` and `auto` alike.

## The mode ladder

| Mode | Foreground may do | Hook |
|---|---|---|
| `off` | everything — no delegation | stands down |
| `normal` | trivial reads, one-line edits | advisory, never blocks |
| `eager` | nothing — delegate it all | blocks main-thread `Edit`/`Write` |
| `auto` | per-task `normal` or `eager` | armed only when it chose `eager` |

`auto` flips **normal ↔ eager only**. `off` stays manual: auto reaching for `off` on a
large task is the exact case this skill exists to catch, and since `normal` already
permits trivial edits, the practical difference is nearly nil.

`auto` is a **judgement mode**; `eager` is a **guarantee mode**. The flip decision belongs
to the foreground agent, so `auto` cannot be independently enforced — a hook is a separate
process per tool call and cannot see which posture the agent chose. Only a statically-set
`eager` is a value the hook can read and trust.

## The sensitivity gate

Sits above both axes and survives every mode except `off`.

Sensitive content — credentials, personal/financial/health data, private network layout —
never goes to a third-party relay. Because `ANTHROPIC_BASE_URL` is fixed to one provider,
delegated work never leaves the direct provider, so this gate is mostly about text-only
routes through OpenRouter (see `cost-aware-model-router`), which is where it was always
really needed.

## Tiers

Aliases resolve through the provider's mapping, **not** to the session model and not to an
error at spawn. On a non-Claude provider the aliases must be remapped or requests are
rejected at the API layer:

```
ANTHROPIC_DEFAULT_HAIKU_MODEL   -> the cheap tier
ANTHROPIC_DEFAULT_SONNET_MODEL  -> the strong tier
```

A full model id in the `model` setting bypasses the aliases entirely — the only way to
pin something outside the three-alias map. **Open question: which concrete models back
each alias.** Left unset, `model: haiku` sends a Claude model id to a non-Claude endpoint
and fails loudly at request time.

## Pool lifecycle: keep while hot, reap when cold

Subagents do not stay running; the persistence model is resume-by-name, and a resumed run
retains its full history and can keep reading the prompt cache the original run warmed.
Teammates are the mechanism for genuinely long-lived agents.

Reaping is **not** eager. Warmth is what makes the next call fast, and it decays with the
prompt-cache TTL (default ~5 min; `subagentPromptCacheTtl` extends it, `1h` is the max).
An agent held across a long gap is no faster than a resumed one — both pay a cold context.
So: hold while reuse is near-term, reap on TTL or goal completion. Same reasoning as
`reap-orphans.sh` one level down — a reaped-too-early agent costs a cold restart, a
kept-too-long agent costs a slot.

The foreground owns the lifecycle (`TaskStop`), never a timer acting alone.

## Anti-stall and goal conformance

Both are harness features, not things to build:

- **`TeammateIdle` hook** — fires as a teammate goes idle; exit 2 feeds it a message and
  keeps it working. This is the anti-stall mechanism.
- **Shared task list** — `TaskCreate`/`TaskUpdate` with dependencies and file-locked
  claiming. The lead owns it. This is "work proceeds according to the goal".

## Visibility: the gap

There is **no hook, event or tap for inter-agent message bodies**. The parent sees that an
agent ran and its final result — not the traffic between agents. Named subagents can
message each other directly, and teammates message by name with no relay, so the topology
is a mesh, not a star.

Reading the traffic means polling files, not subscribing: team mailboxes at
`~/.claude/teams/<team>/inboxes/<agent>.json`, subagent transcripts at
`~/.claude/projects/<project>/<sessionId>/subagents/agent-<id>.jsonl`.

This is the one requirement that is real work.

## Caveats

- Agent teams are **experimental and off by default** (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`),
  and are documented as having known limits around session resumption, task coordination
  and shutdown.
- Teams require an **interactive** session; in `-p`/SDK no teammates are spawned.
- One team per session, no nested teams, lead fixed. 3–5 teammates is the suggested start.
- `Explore` and `Plan` return no agent id, so they **cannot be resumed** — a prewarmed pool
  must use `general-purpose` or custom agents.
- Concurrency is capped at 20 subagents by default, and **resuming bypasses the cap check**,
  so resumes can push the running count past it.
- Anthropic does not support routing Claude Code to non-Claude models through a gateway;
  none of the team or messaging behaviour is verified on such a session.

# Overlord skill: prior art search

Date: 2026-10-07

Goal: a skill that forces all research and implementation into subagent sessions, keeps the foreground agent on overwatch, routes tasks to models by complexity (simple tasks to haiku) and offers a manual mode override.

## Verdict

No existing skill does all of it. Partial matches exist.

## Closest matches

- [oh-my-claude-code](https://sharedcontext.ai/plugins/external/zephyrpersonal/oh-my-claude-code): multi-agent orchestration with a FREE -> CHEAP -> EXPENSIVE cost-aware delegation framework. Nearest to "aggressive optimization".
- [wshobson/agents](https://www.sourcepulse.org/projects/11025561): large set of subagents, each pinned to haiku, sonnet or opus. Routing is static (per agent file). No overwatch role, no per-task classifier.
- [christophermark gist](https://gist.github.com/christophermark/50e0487fce92d7de176bdc8bb5c8ea5e): orchestrator delegating to cheaper subagents.
- [MindStudio write-up](https://www.mindstudio.ai/blog/smart-orchestrator-cheaper-sub-agent-models-claude-code): describes the pattern (expensive orchestrator plans and reviews, cheap workers execute). Article, not installable.
- [orchestrate skill](https://claudepluginhub.com/skills/dkritarth-orchestrate-skills-orchestrate/orchestrate) and [launch-sub-agent](https://claudemarketplaces.com/skills/neolabhq/context-engineering-kit/launch-sub-agent): partial overlap, not read.

## Gaps nobody seems to fill

- **Hard enforcement.** Skills only advise; the foreground agent can still do the work itself. Forcing delegation needs a `PreToolUse` hook that blocks `Edit`, `Write` and heavy `Bash`/`Read` in the foreground session.
- **Manual override**, for example `/overlord mode haiku|auto|off`, or per-task pinning.
- **Tier rules based on risk as well as size**, with an escalation path when a haiku worker fails.

## Caveats

Only two searches were run, and only result summaries were read, not the repos. Read oh-my-claude-code in full before building, to see how much is reusable.

## Next step

Scaffold with the repo's `create-plugin` skill, starting with brainstorming.

## Sources

- https://gist.github.com/christophermark/50e0487fce92d7de176bdc8bb5c8ea5e
- https://www.mindstudio.ai/blog/smart-orchestrator-cheaper-sub-agent-models-claude-code
- https://www.sourcepulse.org/projects/11025561
- https://sharedcontext.ai/plugins/external/zephyrpersonal/oh-my-claude-code
- https://claudepluginhub.com/skills/dkritarth-orchestrate-skills-orchestrate/orchestrate
- https://claudemarketplaces.com/skills/neolabhq/context-engineering-kit/launch-sub-agent

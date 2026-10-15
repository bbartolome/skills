# Skills

Personal agent skills for Claude Code.

## Reference

These split on one axis: who can invoke them. **User-invoked** skills are reachable only when you type them (e.g. `/to-opencode`); their job is to orchestrate. **Model-invoked** skills can be invoked by you _or_ reached for automatically by the agent when the task fits.

**Model-invoked**

- **[to-opencode](./to-opencode/SKILL.md)**: Ultra-lean architect/orchestrator for large coding tasks. Explores the codebase via subagents, designs the solution, then generates a single queued batch bash script of stateless `claude -p` and `opencode run` calls, launched in the background, instead of writing specs/tests/implementation code inline.

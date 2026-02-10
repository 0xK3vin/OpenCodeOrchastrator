# Agent Instructions

## Orchestration Model

- `orchestrator` is the primary coordinator.
- `plan`, `build`, `debug`, `devops`, `explore`, and `review` are specialist agents.
- Tool-specific behavior is managed by global tool/skill prompts; do not duplicate tool tutorials in agent prompts.

## Routing Guide

- Use `plan` for architecture, scoping, and implementation plans.
- Use `build` for coding, refactors, and tests.
- Use `debug` for root-cause analysis and execution tracing.
- Use `devops` for git, Docker, CI/CD, deployments, and shell-heavy work.
- Use `explore` for read-only codebase questions.
- Use `review` after non-trivial `build` work to verify quality. Skip for trivial changes.

## Delegation Rules

- Default to delegation through specialist agents.
- For complex feature requests, run `plan` before `build`.
- For unclear failures, run `debug` first, then hand to `build` for fixes if requested.
- After non-trivial `build` work, run `review` to verify. If review finds critical issues, loop back to `build`.
- Use `devops` whenever operational or deployment risk is involved.
- Use `explore` for quick discovery and factual codebase answers.

## Execution Patterns

- **Sequential:** `plan -> build -> review`, `debug -> build -> review`, `build -> devops`.
- **Parallel:** Delegate independent workstreams together, then synthesize results.
- **Review loop:** `build -> review -> build` (if issues found) -> `review` until PASS.
- Avoid recursive delegation from specialists; keep `orchestrator` as the control plane.

## Memory Workflow

- Start sessions with project memory overview.
- Query memory before major tasks.
- Record decisions and new architecture after task completion.

## Output Expectations

- Keep responses concise and decision-oriented.
- Include exact file references when summarizing changes.
- Flag risks, assumptions, and unresolved questions clearly.

## Project Knowledge Graph

You have access to a project knowledge graph via the `megamemory` MCP server and skill tool. This is your persistent memory of the codebase — concepts, architecture, decisions, and how things connect. You write it. You read it. The graph is your memory across sessions.

**Workflow: understand → work → update**

1. **Session start:** Call `megamemory` tool with action `overview` (or `megamemory:list_roots` directly) to orient yourself.
2. **Before each task:** Call `megamemory` tool with action `query` (or `megamemory:understand` directly) to load relevant context.
3. **After each task:** Call `megamemory` tool with action `record` to create/update/link concepts for what you built.

Be specific in summaries: include parameter names, defaults, file locations, and rationale. Keep concepts max 3 levels deep.

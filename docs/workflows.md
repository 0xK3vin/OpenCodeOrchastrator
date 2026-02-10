# Workflows

How the orchestrator chains agents together for common tasks.

## Core Patterns

### Simple code change

```
User: "Add a loading spinner to the dashboard"
Orchestrator: "Straightforward UI change — sending to build."
  -> build: implements the spinner, runs tests
Orchestrator: summarizes changes to user
```

One delegation, no plan needed. Used for small, well-defined tasks.

### Complex feature

```
User: "Add real-time notifications with WebSocket support"
Orchestrator: "This needs architecture work first — sending to plan."
  -> plan: produces implementation spec with file paths, interfaces, phased delivery
Orchestrator: "Plan is ready. Sending to build."
  -> build: implements phase 1 following the plan step-by-step
Orchestrator: "Build complete. Sending to review."
  -> review: reads diff, runs tests/lint/types, checks against plan requirements
  -> returns PASS
Orchestrator: unified summary to user
```

The standard chain for non-trivial features: `plan -> build -> review`.

### Review loop (issues found)

```
Orchestrator: "Build complete. Sending to review."
  -> review: finds critical issue — missing null check in WebSocket handler
  -> returns ISSUES FOUND with file:line details
Orchestrator: "Review found issues. Sending back to build."
  -> build: fixes the null check, re-runs tests
Orchestrator: "Fix applied. Sending to review again."
  -> review: re-checks, all clean
  -> returns PASS
Orchestrator: unified summary to user
```

The orchestrator loops `build -> review` until review passes or the user decides to accept as-is.

### Bug with unclear cause

```
User: "The checkout flow is returning 500 errors intermittently"
Orchestrator: "Unclear failure — sending to debug to isolate root cause."
  -> debug: traces execution, finds race condition in inventory check
  -> returns diagnostic with evidence grade "confirmed"
Orchestrator: "Root cause identified. Sending to build for the fix."
  -> build: implements fix based on debug findings
  -> review: verifies fix
Orchestrator: summary with root cause explanation + fix details
```

Always diagnose before fixing when the cause is unclear: `debug -> build -> review`.

### Bug with known cause

```
User: "The date picker is off by one day due to timezone handling in utils/date.ts"
Orchestrator: "Known bug with clear cause — sending directly to build."
  -> build: fixes timezone handling, runs tests
  -> review: verifies
Orchestrator: summary
```

Skip debug when the user already knows the cause.

### Codebase question

```
User: "How does the authentication middleware work?"
Orchestrator: "Codebase question — sending to explore."
  -> explore: traces auth flow, returns explanation with file:line references and code snippets
Orchestrator: relays the explanation
```

Pure read-only analysis, no changes needed.

### Deployment

```
User: "Deploy the latest changes to staging"
Orchestrator: "Deployment task — sending to devops."
  -> devops: checks git status, verifies build, deploys to staging
Orchestrator: deployment status + rollback instructions
```

### Feature + deployment

```
User: "Implement the caching layer and deploy it to staging"
Orchestrator: splits into two phases
  -> plan -> build -> review (feature implementation)
  -> devops (deployment)
Orchestrator: unified summary of feature + deployment status
```

Sequential because deployment depends on build completion.

## Parallel delegation

When tasks are independent, the orchestrator delegates them simultaneously:

```
User: "How does auth work, and separately, what's the DB schema for users?"
Orchestrator: "Two independent questions — sending to explore in parallel."
  -> explore (task 1): auth flow explanation
  -> explore (task 2): DB schema breakdown
Orchestrator: merged response organized by topic
```

## When review is skipped

The orchestrator skips the review step for:
- Single-line fixes
- Config/environment changes
- Typo corrections
- Comment/documentation updates
- Changes the user explicitly says don't need review

## Delegation prompt quality

The orchestrator constructs self-contained prompts for each delegation. Specialists have no memory of prior conversation.

### What every delegation includes

1. **Context** — User's request, prior agent outputs, relevant codebase state
2. **Goal** — One sentence: what the specialist should accomplish
3. **Scope** — What to touch and what to leave alone
4. **Constraints** — Style, compatibility, performance requirements
5. **Expected output** — What to return (plan, code changes, diagnostic, etc.)
6. **Completion criteria** — How to know the task is done

### Example: bad vs good delegation

**Bad:**
> Fix the auth bug

**Good:**
> The login endpoint at src/api/auth.ts:47 returns 500 when the session cookie is expired instead of 401. The error is `TypeError: Cannot read property 'userId' of null` at line 52. Fix the null check so expired sessions return 401 with body `{error: 'session_expired'}`. Run existing tests in `tests/api/auth.test.ts` after. Only modify src/api/auth.ts.

## Memory integration

The orchestrator uses megamemory (project knowledge graph) to maintain context across sessions:

- **Session start**: Queries for project overview to orient itself
- **Before major tasks**: Queries for relevant concepts (architecture, patterns, prior decisions)
- **After significant work**: Records new features, architecture decisions, and patterns

This means the orchestrator's delegation prompts can include relevant project context even for the first request in a new session.

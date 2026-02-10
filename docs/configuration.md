# Configuration

How the agent system is configured across OpenCode's config files.

## File Overview

| File | Purpose |
|------|---------|
| `opencode.json` | Agent task permissions, MCP servers, plugins |
| `AGENTS.md` | Global instructions injected into ALL agents |
| `agents/*.md` | Individual agent prompts with frontmatter config |

## Agent Markdown Frontmatter

Each agent file uses YAML frontmatter to declare its configuration:

```yaml
---
description: One-line description shown in agent picker
mode: primary | all
model: provider/model-name
permission:
  read: allow | deny
  grep: allow | deny
  glob: allow | deny
  edit: allow | deny
  write: allow | deny
  bash:
    "*": allow | ask | deny
    "pattern*": allow | deny
  task:
    "*": deny
    "agent-name": allow
---
```

### Mode

- `primary` — The default agent. Cannot be delegated to. Only one agent can be primary.
- `all` — Available both via Tab-switching (direct use) AND as a delegation target for the orchestrator's Task tool.

### Model

Format is `provider/model-name`. Examples:
- `anthropic/claude-opus-4-6`
- `anthropic/claude-sonnet-4-20250514`
- `openai/gpt-5.3-codex`

Adjust models to your preference and budget.

### Permissions

Permissions control what tools an agent can use.

| Permission | Controls |
|-----------|----------|
| `read` | Reading files |
| `grep` | Content search |
| `glob` | File pattern matching |
| `edit` | Editing existing files |
| `write` | Creating new files |
| `bash` | Shell command execution |
| `task` | Delegating to other agents |

Values: `allow`, `deny`, `ask` (prompts user for confirmation).

### Bash Permission Patterns

Bash permissions use glob patterns with **last match wins** semantics:

```yaml
bash:
  "*": allow              # Default: allow everything
  "rm *": deny            # Then deny specific patterns
  "rm -*": deny
  "rmdir*": deny
```

The command `rm -rf /tmp/foo` matches both `"*"` (allow) and `"rm -*"` (deny). Since last match wins, it's denied.

### Task Permission Patterns

Same last-match-wins system:

```yaml
task:
  "*": deny               # Default: can't delegate
  "plan": allow           # Except to these specific agents
  "build": allow
```

## opencode.json

The JSON config provides a second layer of permissions that supplements (not replaces) the frontmatter. Used primarily for task delegation permissions.

```json
{
  "agent": {
    "orchestrator": {
      "permission": {
        "task": {
          "*": "deny",
          "plan": "allow",
          "build": "allow",
          "debug": "allow",
          "devops": "allow",
          "explore": "allow",
          "review": "allow"
        }
      }
    },
    "plan": {
      "permission": { "task": { "*": "deny" } }
    },
    "build": {
      "permission": { "task": { "*": "deny" } }
    },
    "debug": {
      "permission": { "task": { "*": "deny" } }
    },
    "devops": {
      "permission": { "task": { "*": "deny" } }
    },
    "explore": {
      "permission": { "task": { "*": "deny" } }
    },
    "review": {
      "permission": { "task": { "*": "deny" } }
    }
  }
}
```

This ensures only the orchestrator can delegate. All specialists have `task: { "*": "deny" }`.

## AGENTS.md

`AGENTS.md` sits at the config root and its contents are injected into **every** agent as system instructions. Use it for cross-cutting concerns:

- Routing guide (which agent handles what)
- Delegation rules and execution patterns
- Memory workflow (megamemory usage)
- Output expectations

Do NOT put agent-specific instructions here — those go in the individual agent files.

Do NOT duplicate tool usage instructions here — those are handled by global tool/skill prompts (e.g., `tool/*.ts` files for megamemory and osgrep).

## MCP Servers

MCP servers are configured in `opencode.json` under the `mcp` key. These provide external tool access to all agents:

```json
{
  "mcp": {
    "server-name": {
      "type": "local" | "remote",
      "command": ["command", "args"],    // for local
      "url": "https://...",              // for remote
      "environment": { "KEY": "value" }, // optional env vars
      "enabled": true
    }
  }
}
```

The agents in this setup are designed to work with these MCP servers but don't require them. Tool-specific behavior instructions live in `tool/*.ts` skill files, not in agent prompts.

## Bash Deny List

All agents with bash access share the same deny list for destructive commands:

```yaml
"rm *": deny
"rm -*": deny
"rmdir*": deny
"mkfs*": deny
"dd *": deny
"shutdown*": deny
"reboot*": deny
"halt*": deny
"poweroff*": deny
```

This is a safety net, not a security boundary. The agents aren't adversarial — the deny list prevents accidental destructive operations.

Note: Commands prefixed with `sudo` (e.g., `sudo rm -rf /`) would bypass these patterns. This is an accepted trade-off — the agents have no reason to use sudo unless explicitly asked.

## Customization

### Changing models

Edit the `model` field in each agent's frontmatter. Common substitutions:
- Replace `opus` with `sonnet` to reduce cost (trades reasoning quality)
- Replace `gpt-5.3-codex` with any coding-capable model
- Use the same model everywhere for simplicity

### Adjusting permissions

- To make an agent more restricted: change `allow` to `ask` or `deny`
- To make an agent less restricted: change `deny` to `allow`
- To add bash deny patterns: add more glob entries after `"*": allow`

### Adding agents

1. Create `agents/new-agent.md` with frontmatter
2. Add `"new-agent": { "permission": { "task": { "*": "deny" } } }` to `opencode.json`
3. Add `"new-agent": "allow"` to the orchestrator's task permissions (in both frontmatter and opencode.json)
4. Update `AGENTS.md` routing guide
5. Update the orchestrator's prompt to include the new agent in its routing table

### Removing agents

1. Delete the agent file from `agents/`
2. Remove its entry from `opencode.json`
3. Remove it from the orchestrator's task permissions
4. Update `AGENTS.md` and the orchestrator's prompt

# Claude Code Quick Reference

## CLI Commands

| Command | Description |
|---------|-------------|
| `claude` | Start interactive REPL |
| `claude "task"` | Start REPL with initial prompt |
| `claude -p "task"` | Headless mode (non-interactive) |
| `claude -c` | Continue most recent conversation |
| `claude -c "task"` | Continue conversation with new prompt |
| `claude -r <id>` | Resume session by ID |
| `claude -r <name>` | Resume session by name |

## Essential CLI Flags

| Flag | Purpose |
|------|---------|
| `-p "prompt"` | Headless/non-interactive mode |
| `-c` / `--continue` | Continue last session |
| `-r` / `--resume <id>` | Resume specific session |
| `--model <model>` | Override model |
| `--output-format text\|json\|stream-json` | Output format (headless) |
| `--max-turns <n>` | Limit agentic loop iterations |
| `--budget-tokens <n>` | Set token spending limit |
| `--system-prompt-file <path>` | Replace system prompt from file |
| `--append-system-prompt-file <path>` | Add to system prompt from file |
| `--allowedTools <tools>` | Comma-separated allowed tools |
| `--disallowedTools <tools>` | Comma-separated denied tools |
| `--dangerously-skip-permissions` | Skip all permission prompts (sandboxed envs only) |
| `--permission-prompt-tool <mcp>` | Delegate permissions to MCP tool |
| `--permission-mode <mode>` | Set permission mode |
| `--fork-session` | Fork the resumed session |
| `--session-name <name>` | Name the session |
| `--verbose` | Enable verbose logging |

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/help` | Show commands and shortcuts |
| `/clear` | Clear conversation, start fresh |
| `/compact` | Manually compact context |
| `/context` | Show context window usage |
| `/config` | Open/manage configuration |
| `/init` | Initialize CLAUDE.md |
| `/model` | Switch model mid-session |
| `/mcp` | MCP server status and token costs |
| `/permissions` | View/manage permission rules |
| `/sandbox` | View sandbox status |
| `/agents` | List available subagents |
| `/doctor` | Run diagnostics |
| `/rename` | Rename current session |
| `/resume` | List and resume sessions |
| `/teleport` | Transfer session to another surface |
| `/desktop` | Open session in desktop app |
| `/ide` | Open session in IDE extension |

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Shift+Tab` | Cycle permission modes |
| `Esc Esc` (double) | Rewind to previous checkpoint |
| `Ctrl+O` | Multi-line input mode |
| `Ctrl+C` | Interrupt current operation |
| `Ctrl+D` | End session |

## Permission Modes

| Mode | Behavior |
|------|----------|
| **Plan** | Read-only. No file edits, no write commands. |
| **Default** | Asks permission for edits and commands. |
| **Auto-accept edits** | File edits proceed; shell commands still prompt. |
| **Full auto-accept** | Everything proceeds without prompting. |
| **Bypass** | CLI flag only. Requires sandbox. |

## Permission Hierarchy (highest to lowest)

1. **Managed** -- IT-deployed, cannot be overridden
2. **CLI arguments** -- flags at invocation
3. **Local** -- `.claude/settings.local.json` (gitignored)
4. **Project** -- `.claude/settings.json` (version-controlled)
5. **User** -- `~/.claude/settings.json`

Rule evaluation order: **Deny > Ask > Allow**, first match wins.

## CLAUDE.md Best Practices

- Keep under 500 lines -- every line costs context on every request
- Put only what Claude cannot infer from code (build commands, style deviations, architectural decisions, environment quirks)
- Do NOT include standard language conventions, framework docs, or obvious project structure
- Use `@path` imports to pull in external reference files
- Use the "Compact Instructions" section for rules that must survive auto-compaction
- Update CLAUDE.md at the end of every productive session with lessons learned
- Add targeted fixes for repeated errors (specific wrong behavior AND correct behavior)
- Commit project-level CLAUDE.md to version control for team benefit
- Keep personal preferences in user-level or local-scope files, not shared CLAUDE.md

## Context Cost Model

**Costs every request:**
- CLAUDE.md content (all imported files too)
- MCP tool definitions (names, descriptions, JSON schemas)
- Skill descriptions (short summaries)

**Isolated cost (subagents):**
- Subagent work runs in its own context window
- Only the returned summary costs main context

**Zero cost (hooks):**
- Hooks execute outside the context window entirely
- PreToolUse, PostToolUse, SessionStart -- all free

## Model Selection

| Model | Use For | Cost |
|-------|---------|------|
| **Sonnet** | Default for most coding work. Fast, reliable, cost-effective. | Low |
| **Opus** | Architecture decisions, complex refactors, ambiguous/high-stakes work. | High |
| **Haiku** | Explore subagent, file search, quick reconnaissance. | Lowest |

**Pattern:** Opus for orchestration/planning, Sonnet subagents for implementation, Haiku subagents for exploration.

## Top Prompt Patterns

### Verification Criteria (highest leverage)
Tell Claude how to verify its own work. Tests, expected output, visual targets.
```
Add auth to the API. Run `pytest tests/auth/` after. /api/protected should
return 401 without token, 200 with valid token.
```

### Backpressure Through Tooling
Your linter, type checker, and test suite ARE part of your prompt. Strict tooling = better output. Pre-commit hooks create the tightest feedback loop.

### Spec-First
Ask Claude to write a spec/design doc before any code. Review the spec. Then implement. The spec survives compaction and session restarts.

### Plan-Before-Execute
```
Before making changes, read relevant files and produce a plan. List every
file you will modify, every new file, every behavioral change. Wait for
my approval.
```

### Exhaustive Questioning
```
Before you start, ask me questions exhaustively about anything you are
uncertain about. Do not proceed until you have asked every question.
```

### Constraint Explicitness
State what cannot change, trade-off preferences, and flexibility boundaries.
```
Do not modify the database schema. Performance > readability. You can add
files in src/utils/ but not src/core/. No new dependencies.
```

### Self-Critique
```
Rate this implementation 1-10. What would you improve? What edge cases
might break? What would a senior engineer object to?
```

### Strawman Proposal
```
Here is my rough approach: [description]. This might be wrong. Improve
on it or suggest alternatives.
```

### Role Assignment
```
You are the orchestrator. Assign each subagent a clear, bounded task.
Review output before integrating.
```

### Simplicity Directive
```
Write the simplest implementation that satisfies requirements.
No unnecessary abstractions.
```

## Four-Phase Workflow

1. **Explore** -- Plan mode. Read codebase. Understand patterns.
2. **Plan** -- Still plan mode. Outline approach. Review before implementing.
3. **Implement** -- Switch to normal/auto-accept. Execute the reviewed plan.
4. **Commit** -- Review diff. Commit frequently. Small commits = cheap insurance.

## MCP Configuration

**.mcp.json** (project root, commit to VCS):
```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["path/to/server.js"],
      "env": { "API_KEY": "${ENV_VAR}" }
    }
  }
}
```

- Use `/mcp` to check status and per-server token costs
- MCP keeps credentials out of context (server-side, not in conversation)
- Background subagents CANNOT use MCP tools
- Hook pattern for MCP tools: `mcp__<server>__<tool>`

## Subagent Types

| Type | Model | Tools | Use Case |
|------|-------|-------|----------|
| **Explore** | Haiku | Read-only | Navigate unfamiliar code |
| **Plan** | Same as parent | Read-only | Complex planning tasks |
| **General** | Configurable | All tools | Implementation work |
| **Bash** | Configurable | Shell only | Command sequences |
| **Custom** | Configurable | Configurable | Defined in `.claude/agents/` |

Key constraint: subagents cannot spawn other subagents (two-level hierarchy only).

## Common Failure Modes and Quick Fixes

| Problem | Fix |
|---------|-----|
| Claude forgets instructions mid-session | Context exhaustion. Start fresh session with good CLAUDE.md. Use `/compact`. |
| Bash env vars not persisting | Chain commands: `NODE_ENV=prod npm test`. Each bash call is a fresh shell. |
| MCP tools suddenly stop working | Run `/mcp`. Server disconnected silently. Reconnect or restart session. |
| Checkpoint restore misses changes | Checkpoints only track Write/Edit tools, not bash. Use `git checkout .` instead. |
| Claude repeats same error across sessions | Add targeted fix to CLAUDE.md with wrong AND correct behavior. |
| Over-complex solutions | Add simplicity directive to prompt and CLAUDE.md. Provide reference code. |
| Subagents flood main context | Instruct subagents to return concise summaries or write results to files. |
| Session lost after crash | `claude -c` to continue, or `claude -r <id>` to resume by ID. |
| Claude goes down wrong path | Commit first. Then either correct (if approach is right, details wrong) or `git checkout .` and restart fresh. |
| Stop hook infinite loop | Add iteration limits to stop hooks. Test with intentionally failing conditions. |

## Slot Machine Recovery

1. Commit current state
2. Let Claude run for 20-30 minutes (time-boxed)
3. Binary decision: accept result or `git revert` and retry with better prompt
4. No correction spiral -- clean restart beats patching a bad approach

## Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | API key |
| `ANTHROPIC_MODEL` | Override default model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Model for Haiku-level tasks |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Compaction threshold (default 95) |
| `CLAUDE_CODE_USE_BEDROCK` | Enable Bedrock (`1`) |
| `CLAUDE_CODE_USE_VERTEX` | Enable Vertex AI (`1`) |
| `ENABLE_TOOL_SEARCH` | MCP tool search/deferred loading |
| `CLAUDE_CODE_ENABLE_AGENT_TEAMS` | Enable experimental teams |
| `CLAUDE_CODE_THINKING_MODE` | `enabled`, `disabled`, `budget` |
| `CLAUDE_CODE_MAX_TOKENS` | Max tokens per response |
| `CLAUDE_CODE_BUDGET_TOKENS` | Token budget (headless) |
| `CLAUDE_CODE_MAX_TURNS` | Max turns (headless) |

## Headless / CI Pattern

```bash
# Basic headless
claude -p "task description" --output-format json

# Pipe input
git diff HEAD~1 | claude -p "Review for security issues" --output-format json

# CI with devcontainer + network isolation
claude -p "fix failing tests" --dangerously-skip-permissions

# Reproducible prompts
claude -p "$(cat task.txt)" --system-prompt-file ci-prompt.md --output-format json
```

## Hook Configuration (settings.json)

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash(rm *)",
      "handler": { "type": "command", "command": "exit 1" }
    }],
    "SessionStart": [{
      "matcher": "*",
      "handler": { "type": "command", "command": "echo 'Session started'" }
    }]
  }
}
```

Hook decisions: `allow` (proceed), `deny` (block), `undefined` (fall through).
Hooks run with full user permissions. Zero context cost.

## Permission Rule Syntax

```json
{
  "permissions": {
    "deny": ["Bash(rm *)", "Write(*.env)"],
    "ask": ["Bash(git push *)"],
    "allow": ["Bash(npm test)", "Bash(git status)"]
  }
}
```

Patterns: `Tool`, `Tool(specifier)`, `mcp__server__tool`. Glob wildcards supported.

## Git Worktrees for Parallel Work

```bash
git worktree add ../feature-auth feature/auth
git worktree add ../feature-payments feature/payments
# Run independent Claude Code sessions in each directory
```

## One-Third Reality

~33% of tasks succeed on first attempt. This is normal. The correction cycle is fast. Invest in verification infrastructure (tests, types, linting) to improve the rate.

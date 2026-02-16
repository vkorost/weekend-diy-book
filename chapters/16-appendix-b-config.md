# Appendix B: Configuration Reference

## Settings.json Keys by Scope

### JSON Schema for Settings

Add the `$schema` line to any `settings.json` for autocomplete and inline validation in editors that support JSON schema:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": { ... },
  "env": { ... }
}
```

### Permission Rules

| Key | Type | Description |
|-----|------|-------------|
| `permissions.allow` | `string[]` | Tools auto-approved without prompting |
| `permissions.ask` | `string[]` | Tools requiring user confirmation |
| `permissions.deny` | `string[]` | Tools always blocked |

Rule syntax: `Tool` or `Tool(specifier)` with glob pattern support. Evaluation order: **Deny > Ask > Allow**, first match wins across all scopes.

Examples:
- `Bash(npm test)` -- specific bash command
- `Write(*.env)` -- file pattern
- `mcp__server__tool` -- MCP tool
- `Bash(git *)` -- wildcard command

### Environment Variables (Settings)

| Key | Type | Description |
|-----|------|-------------|
| `env` | `object` | Key-value pairs injected into Claude's environment |

### Hook Configuration

| Key | Type | Description |
|-----|------|-------------|
| `hooks` | `object` | Event-to-handler mappings (see Hook Schema below) |

### Model Settings

| Key | Type | Description |
|-----|------|-------------|
| `model` | `string` | Default model identifier |
| `availableModels` | `string[]` | Models available for selection |

### Authentication and Credentials

| Key | Type | Description |
|-----|------|-------------|
| `apiKeyHelper` | `string` | Custom script (executed via `/bin/sh`) to generate an auth value. Value is sent as `X-Api-Key` and `Authorization: Bearer` headers for model requests |
| `awsAuthRefresh` | `string` | Custom script that modifies the `.aws` directory for credential refresh |
| `awsCredentialExport` | `string` | Custom script that outputs JSON with AWS credentials |
| `forceLoginMethod` | `string` | Restrict login to `claudeai` (Claude.ai accounts) or `console` (API billing accounts) |
| `forceLoginOrgUUID` | `string` | UUID of an organization to auto-select during login (requires `forceLoginMethod`) |

### Session and Behavior

| Key | Type | Description |
|-----|------|-------------|
| `cleanupPeriodDays` | `number` | Sessions inactive longer than this are deleted at startup. Default: 30. Set to `0` for immediate cleanup |
| `language` | `string` | Preferred response language (e.g., `"japanese"`, `"spanish"`, `"french"`) |
| `outputStyle` | `string` | Configure output style to adjust system prompt |
| `plansDirectory` | `string` | Customize where plan files are stored. Path relative to project root. Default: `~/.claude/plans` |
| `showTurnDuration` | `boolean` | Show turn duration messages (e.g., "Cooked for 1m 6s"). Default: true |
| `teammateMode` | `string` | How agent team teammates display: `auto` (picks split panes in tmux/iTerm2), `in-process`, or `tmux` |

### UI and Display

| Key | Type | Description |
|-----|------|-------------|
| `autoUpdatesChannel` | `string` | Release channel: `"stable"` (about one week old, skips regressions) or `"latest"` (most recent). Default: `"latest"` |
| `spinnerTipsEnabled` | `boolean` | Show tips in spinner while working. Default: true |
| `spinnerVerbs` | `object` | Customize spinner verbs. `mode`: `"replace"` (use only your verbs) or `"append"` (mix with defaults). `verbs`: `string[]` |
| `terminalProgressBarEnabled` | `boolean` | Enable terminal progress bar in Windows Terminal and iTerm2. Default: true |
| `prefersReducedMotion` | `boolean` | Reduce or disable UI animations (spinners, shimmer, flash effects) for accessibility |

### File Suggestion

| Key | Type | Description |
|-----|------|-------------|
| `fileSuggestion` | `object` | Custom script for `@` file path autocomplete in large monorepos |
| `respectGitignore` | `boolean` | Whether the `@` file picker respects `.gitignore` patterns. Default: true |

The `fileSuggestion` script receives JSON on stdin with a `query` field and outputs newline-separated file paths on stdout.

### Sandbox Settings

| Key | Type | Description |
|-----|------|-------------|
| `sandbox.enabled` | `boolean` | Enable bash sandboxing (macOS, Linux, WSL2). Default: false |
| `sandbox.autoAllowBashIfSandboxed` | `boolean` | Auto-approve bash commands when sandboxed. Default: true |
| `sandbox.excludedCommands` | `string[]` | Commands that run outside the sandbox (e.g., `["git", "docker"]`) |
| `sandbox.allowUnsandboxedCommands` | `boolean` | Allow commands to bypass sandbox via `dangerouslyDisableSandbox`. Set to `false` to enforce strict sandboxing. Default: true |
| `sandbox.network.allowUnixSockets` | `string[]` | Unix socket paths accessible in sandbox (e.g., SSH agent sockets) |
| `sandbox.network.allowAllUnixSockets` | `boolean` | Allow all Unix socket connections. Default: false |
| `sandbox.network.allowLocalBinding` | `boolean` | Allow binding to localhost ports (macOS only). Default: false |
| `sandbox.network.allowedDomains` | `string[]` | Domains accessible from sandbox. Default: common package registries |
| `sandbox.network.httpProxyPort` | `number` | Custom HTTP proxy port for sandbox network filtering |
| `sandbox.network.socksProxyPort` | `number` | Custom SOCKS proxy port for sandbox network filtering |
| `sandbox.enableWeakerNestedSandbox` | `boolean` | Enable weaker sandbox for unprivileged Docker environments (Linux/WSL2 only). Reduces security. Default: false |

**Complete sandbox example:**

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker"],
    "network": {
      "allowedDomains": ["registry.npmjs.org", "api.github.com"],
      "allowLocalBinding": true
    }
  },
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(secrets/**)",
      "Write(.env)",
      "Write(.env.*)",
      "Write(secrets/**)"
    ]
  }
}
```

### Attribution Settings

| Key | Type | Description |
|-----|------|-------------|
| `attribution.commit` | `string` | Custom git commit attribution text. Supports `\n` for multi-line trailers |
| `attribution.pr` | `string` | Custom PR description attribution text. Set to `""` to disable PR attribution |

**Default commit attribution:**

```
Generated with Claude Code

Co-Authored-By: Claude <noreply@anthropic.com>
```

**Customization example:**

```json
{
  "attribution": {
    "commit": "Generated with AI\n\nCo-Authored-By: AI <ai@example.com>",
    "pr": ""
  }
}
```

### Plugin Settings

| Key | Type | Description |
|-----|------|-------------|
| `enabledPlugins` | `string[]` | Active plugin identifiers in `plugin-name@marketplace-name` format |
| `extraKnownMarketplaces` | `object[]` | Additional plugin marketplace sources with explicit trust boundaries |

Plugin hooks are defined in `hooks/hooks.json` within the plugin root directory. Hooks include an optional top-level `description` field. Plugin scripts reference their own directory via the `${CLAUDE_PLUGIN_ROOT}` variable.

### MCP Server Approval

| Key | Type | Description |
|-----|------|-------------|
| `enableAllProjectMcpServers` | `boolean` | Automatically approve all MCP servers defined in project `.mcp.json` files |
| `enabledMcpjsonServers` | `string[]` | Approve specific MCP servers from `.mcp.json` (e.g., `["memory", "github"]`) |
| `disabledMcpjsonServers` | `string[]` | Reject specific MCP servers from `.mcp.json` |

### Managed-Only Settings

| Key | Type | Description |
|-----|------|-------------|
| `strictKnownMarketplaces` | `boolean` | Restrict plugins to approved marketplaces only |
| `allowManagedHooksOnly` | `boolean` | Block user, project, and plugin hooks; only managed and SDK hooks run |
| `allowManagedPermissionRulesOnly` | `boolean` | Prevent user/project settings from defining `allow`, `ask`, or `deny` permission rules |
| `disableAllHooks` | `boolean` | Disable all hooks and any custom status line |
| `allowedMcpServers` | `object[]` | Allowlist of MCP servers users can configure. Undefined = no restrictions. Empty array = lockdown. Denylist takes precedence |
| `deniedMcpServers` | `object[]` | Denylist of MCP servers that are explicitly blocked across all scopes |

---

## Settings Scope Hierarchy

| Priority | Scope | Location | Shared | Overridable |
|----------|-------|----------|--------|-------------|
| 1 (highest) | Managed | System directories or server-managed (IT-deployed) | Org-wide | No |
| 2 | CLI | Command-line flags | No | Session only |
| 3 | Local | `.claude/settings.local.json` | No (gitignored) | By managed/CLI |
| 4 | Project | `.claude/settings.json` | Yes (VCS) | By managed/CLI/local |
| 5 (lowest) | User | `~/.claude/settings.json` | No | By all above |

### When to Use Each Scope

| Scope | Use For |
|-------|---------|
| **Managed** | Security policies enforced org-wide, API key helpers, forced login methods, MCP allowlists/denylists, sandbox requirements |
| **User** | Personal preferences (model, language, spinner verbs, reduced motion), personal API keys, user-level permissions |
| **Project** | Shared team standards (permissions, hooks, MCP servers), coding conventions, attribution settings |
| **Local** | Personal overrides for a specific project, experimental settings, credentials you do not want committed |

### How Scopes Interact

Settings merge across scopes. For most keys, higher-priority scopes override lower ones. Permission rules follow a specific evaluation order: **deny rules first, then ask, then allow** — the first matching rule wins, evaluated across all scopes.

### Server-Managed Settings

Organizations without device management infrastructure can use server-managed settings, which deliver configurations from Anthropic's servers for Claude for Enterprise customers. These function identically to file-based managed settings and cannot be overridden by user or project settings.

---

## Environment Variables

Claude Code supports over 70 environment variables. The following are the most commonly used, grouped by function.

### API and Authentication

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | API key for direct Anthropic access |
| `CLAUDE_CODE_USE_BEDROCK` | Enable cloud provider Bedrock integration (`1`) |
| `CLAUDE_CODE_USE_VERTEX` | Enable cloud provider Vertex AI integration (`1`) |
| `ANTHROPIC_BASE_URL` | Custom API base URL |
| `CLAUDE_CODE_AUTH_TOKEN` | Override authentication token |

### Model Configuration

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_MODEL` | Override default model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Model for lightweight operations (Haiku tasks) |
| `CLAUDE_CODE_MAX_TOKENS` | Maximum tokens per response |
| `CLAUDE_CODE_MAX_THINKING_TOKENS` | Maximum tokens for thinking/reasoning |

### Proxy and Network

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY` / `HTTPS_PROXY` | HTTP(S) proxy URL |
| `NO_PROXY` | Comma-separated bypass domains |
| `ANTHROPIC_PROXY_URL` | Anthropic-specific proxy URL |

### Context and Compaction

| Variable | Description |
|----------|-------------|
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Auto-compaction threshold percentage (default: 95) |
| `CLAUDE_CODE_CONTEXT_LIMIT` | Maximum context window size |

### Feature Flags

| Variable | Description |
|----------|-------------|
| `ENABLE_TOOL_SEARCH` | Enable/disable MCP tool search and deferred loading |
| `CLAUDE_CODE_ENABLE_AGENT_TEAMS` | Enable experimental agent teams |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable telemetry and non-essential network calls |

### Headless/CI

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_ENTRYPOINT` | Set to `cli` for headless environments |
| `CLAUDE_CODE_BUDGET_TOKENS` | Token budget for headless sessions |
| `CLAUDE_CODE_MAX_TURNS` | Maximum conversation turns in headless mode |

### Output and Formatting

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_OUTPUT_FORMAT` | Output format for headless mode (`text`, `json`, `stream-json`) |
| `CLAUDE_CODE_HIDE_TOOL_OUTPUT` | Suppress tool output display in terminal |

### Debugging and Diagnostics

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_DEBUG` | Enable debug logging (`1`) |
| `CLAUDE_CODE_LOG_LEVEL` | Set logging verbosity level |
| `CLAUDE_CODE_VERBOSE` | Enable verbose output |

### Session and Behavior

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_SKIP_CLAUDE_MD` | Skip loading CLAUDE.md files (`1`) |
| `CLAUDE_CODE_THINKING_MODE` | Thinking mode (`enabled`, `disabled`, `budget`) |
| `CLAUDE_CODE_THINKING_BUDGET` | Token budget for extended thinking |

### Telemetry

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_DISABLE_TELEMETRY` | Disable usage telemetry (`1`) |
| `CLAUDE_CODE_TELEMETRY_ENDPOINT` | Custom telemetry collection endpoint |

---

## Tools Inventory

| Tool | Description |
|------|-------------|
| `Bash` | Execute shell commands. Working directory persists; environment does not. |
| `Read` | Read file contents. Supports images, PDFs, notebook files. |
| `Write` | Create or overwrite files. |
| `Edit` | Exact string replacement in files. Requires unique match. |
| `Glob` | File pattern matching. Returns paths sorted by modification time. |
| `Grep` | Content search with regex support. File type filtering, context lines. |
| `WebFetch` | Fetch and process URL content. Auto-upgrades HTTP to HTTPS. |
| `WebSearch` | Web search with domain filtering. Returns formatted results. |
| `Task` | Spawn subagents with isolated context. Supports foreground/background. |
| `Skill` | Invoke skill-based workflows within conversation. |
| `NotebookEdit` | Edit notebook cells (.ipynb). Replace, insert, or delete. |
| `LSP` | Language Server Protocol operations. Diagnostics, go-to-definition, references. |
| `TeamCreate` | Create agent team (experimental). |
| `TaskCreate` | Create task in agent team task list. |
| `TaskUpdate` | Update task status in agent team. |
| `TaskList` | List tasks for an agent team. |
| `SendMessage` | Send message between agent team members. |
| `TeamDelete` | Delete an agent team. |
| `TodoRead` | Read current task/todo list. |
| `TodoWrite` | Write/update task/todo items. |

---

## Hook Configuration Schema

### Common Input Fields

Every hook event receives these fields on stdin as JSON:

| Field | Description |
|-------|-------------|
| `session_id` | Current session identifier |
| `transcript_path` | Path to conversation JSON file |
| `cwd` | Current working directory when the hook is invoked |
| `permission_mode` | Active permission mode: `"default"`, `"plan"`, `"acceptEdits"`, `"dontAsk"`, or `"bypassPermissions"` |
| `hook_event_name` | Name of the event that fired |

Each event adds its own fields (e.g., `tool_name`, `tool_input` for PreToolUse).

### Event Types

| Event | Fires | Can Block? | Event-Specific Fields |
|-------|-------|------------|----------------------|
| `PreToolUse` | Before any tool executes | Yes | `tool_name`, `tool_input` |
| `PostToolUse` | After any tool executes | Yes (via decision) | `tool_name`, `tool_input`, `tool_output` |
| `PostToolUseFailure` | When a tool execution fails | Yes (via decision) | `tool_name`, `error`, `is_interrupt` |
| `UserPromptSubmit` | Before processing a user prompt | Yes | `prompt` |
| `Notification` | On status notifications (`permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`) | No | `message`, `type` |
| `Stop` | When Claude stops generating | Yes (via decision) | `stop_hook_active` |
| `SessionStart` | At session initialization | No | `source` (`startup`, `resume`, `clear`, `compact`), `model`, optionally `agent_type` |
| `SessionEnd` | At session termination | No | `reason` (`clear`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other`) |
| `PermissionRequest` | Before showing permission dialog | Yes | `tool_name`, permission details; supports `updatedInput`, `updatedPermissions` |
| `PreCompact` | Before context compaction | No | `trigger` (`manual` or `auto`), `custom_instructions` |
| `PostCompact` | After context compaction | No | Compacted context |
| `SubagentStart` | When a subagent spawns | No | `agent_id`, `agent_type` |
| `SubagentStop` | When a subagent completes | Yes (via decision) | `agent_id`, `agent_type`, `agent_transcript_path`, `stop_hook_active` |
| `TeammateIdle` | When agent team teammate is about to go idle | Yes (exit code 2) | `teammate_name` |
| `TaskCompleted` | When a task is marked completed | Yes (exit code 2) | `task_id`, `task_subject` |

### Handler Types (3)

**Command Handler**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "handler": {
          "type": "command",
          "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate.sh"
        }
      }
    ]
  }
}
```
Executes a shell command. Receives event JSON on stdin. Communicates results via exit codes and stdout JSON.

**Prompt Handler**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write(*.sql)",
        "handler": {
          "type": "prompt",
          "prompt": "Review this SQL file write for safety. Allow or deny."
        }
      }
    ]
  }
}
```
Sends context plus prompt to a Claude model (Haiku by default) for single-turn evaluation. Returns structured `{"ok": true/false, "reason": "..."}` decision. Uses `$ARGUMENTS` placeholder for event data.

**Agent Handler**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "handler": {
          "type": "agent",
          "agent": "security-reviewer",
          "tools": ["Read", "Grep"]
        }
      }
    ]
  }
}
```
Spawns a subagent with multi-turn tool access for complex verification. Up to 50 turns.

### Handler Common Fields

| Field | Required | Description |
|-------|----------|-------------|
| `timeout` | No | Timeout in seconds. Default: 10 (command), 30 (prompt), 60 (agent) |
| `statusMessage` | No | Custom spinner message displayed while hook runs |
| `once` | No | If `true`, runs only once per session then is removed. Skills only |
| `async` | No | If `true`, runs in background without blocking. Output delivered on next turn via `systemMessage` or `additionalContext`. Cannot block actions |

### Exit Code Behavior

| Exit Code | Meaning | Effect |
|-----------|---------|--------|
| **0** | Success | Action proceeds. Stdout is parsed for JSON output fields. Stdout shown in verbose mode (`Ctrl+O`) except for `UserPromptSubmit` and `SessionStart` where it becomes context |
| **2** | Blocking error | Action is blocked. Stdout/JSON ignored. Stderr text fed back to Claude as error message |
| **Other** | Non-blocking error | Stderr shown in verbose mode. Execution continues |

### JSON Output Fields (on exit 0)

| Field | Type | Effect |
|-------|------|--------|
| `continue` | `boolean` | If `false`, stops further hook processing for this event |
| `stopReason` | `string` | Message shown when stopping |
| `suppressOutput` | `boolean` | If `true`, suppresses default hook output |
| `systemMessage` | `string` | Message injected into conversation as system context |
| `additionalContext` | `string` | Extra context added to the event |
| `updatedInput` | `object` | Modified tool input before execution (PreToolUse, PermissionRequest) |
| `updatedPermissions` | `object` | Modified permission settings (PermissionRequest; equivalent to "always allow") |
| `permissionDecision` | `string` | `"allow"`, `"deny"`, or `"ask"` for PreToolUse hooks (inside `hookSpecificOutput`) |
| `decision` | `string` | `"block"` to stop the action (UserPromptSubmit, PostToolUse, PostToolUseFailure, Stop, SubagentStop) |

### Environment Variables in Hooks

| Variable | Description |
|----------|-------------|
| `$CLAUDE_PROJECT_DIR` | Project root directory. Wrap in quotes for paths with spaces |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin root directory, for scripts bundled with a plugin |
| `$CLAUDE_ENV_FILE` | (SessionStart only) File path for persisting environment variables. Write `export` statements to this file to make variables available in all subsequent Bash commands |

**CLAUDE_ENV_FILE example:**
```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
  echo 'export DEBUG_LOG=true' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

### Matcher Syntax

- Exact tool name: `Bash`, `Write`, `Edit`
- Tool with specifier: `Bash(npm *)`, `Write(*.env)`
- MCP tools: `mcp__servername__toolname`
- Regex patterns supported in specifiers
- `*` matches any tool

### Hook Decision Control (PreToolUse)

PreToolUse uses `hookSpecificOutput.permissionDecision`:

| Decision | Effect |
|----------|--------|
| `"allow"` | Proceed without user prompt |
| `"deny"` | Block execution; `reason` shown to Claude |
| `"ask"` | Show permission prompt (can combine with `updatedInput` to show modified input) |
| Omitted / no output | Fall through to normal permission evaluation |

### Hook Decision Control (PermissionRequest)

| Decision | Effect |
|----------|--------|
| `"approve"` | Auto-approve without user prompt |
| `"deny"` | Auto-deny without user prompt |
| Omitted / no output | Show normal permission prompt |

Supports `updatedInput` and `updatedPermissions` (equivalent to "always allow" options).

### Hook Decision Control (TeammateIdle / TaskCompleted)

These events use exit codes only, not JSON decision control:

| Exit Code | Effect |
|-----------|--------|
| `0` | Allow the action (teammate goes idle, task marked complete) |
| `2` | Block the action; stderr fed back as feedback to continue working |

### Hook Security Model

Hooks are snapshotted at session startup. If hooks are modified externally during a session, Claude Code warns and requires review in the `/hooks` menu before changes apply. This prevents malicious mid-session changes.

---

## MCP Configuration Format

### .mcp.json (Project Root)

```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["path/to/server.js"],
      "env": {
        "API_KEY": "${ENV_VAR_NAME}"
      },
      "cwd": "./optional-working-directory"
    },
    "remote-server": {
      "url": "https://mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    }
  }
}
```

### Server Configuration Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | `string` | Yes (local) | Executable to launch |
| `args` | `string[]` | No | Command arguments |
| `env` | `object` | No | Environment variables (supports `${VAR}` interpolation) |
| `cwd` | `string` | No | Working directory for server process |
| `url` | `string` | Yes (remote) | SSE endpoint for remote servers |
| `headers` | `object` | No | HTTP headers for remote connections |

### Managed MCP Settings

| Key | Type | Description |
|-----|------|-------------|
| `mcpServers.allowlist` | `string[]` | Only these servers may be configured |
| `mcpServers.denylist` | `string[]` | These servers are blocked |

### Plugin Marketplace Sources

| Source Type | Format |
|-------------|--------|
| Git hosting platform | `githost:org/repo` |
| Git URL | `git:https://example.com/repo.git` |
| npm | `npm:package-name` |
| URL | `url:https://example.com/plugin.tar.gz` |
| File | `file:/path/to/plugin` |
| Directory | `directory:/path/to/plugin-dir` |
| Host pattern | `host:*.internal.company.com` |

---

## Full Settings.json Example

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git *)",
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Write(.env)",
      "Write(.env.*)",
      "Read(secrets/**)"
    ]
  },
  "env": {
    "NODE_ENV": "development",
    "LOG_LEVEL": "info"
  },
  "model": "claude-opus-4-6",
  "language": "english",
  "autoUpdatesChannel": "stable",
  "showTurnDuration": true,
  "spinnerVerbs": {
    "mode": "append",
    "verbs": ["Pondering", "Crafting"]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "excludedCommands": ["docker", "git"],
    "network": {
      "allowedDomains": ["registry.npmjs.org", "api.github.com"]
    }
  },
  "attribution": {
    "commit": "Generated with Claude Code\n\nCo-Authored-By: Claude <noreply@anthropic.com>",
    "pr": "Generated with Claude Code"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "handler": {
          "type": "command",
          "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate-bash.sh"
        }
      }
    ]
  }
}
```

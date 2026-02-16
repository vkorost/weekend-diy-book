# Appendix A: Command Reference

## CLI Commands

| Command | Description |
|---------|-------------|
| `claude` | Start interactive REPL in current directory |
| `claude "task"` | Start interactive REPL with initial prompt |
| `claude -p "task"` | Non-interactive (headless) mode; print response and exit |
| `claude -c` | Continue most recent conversation in current directory |
| `claude -c "task"` | Continue most recent conversation with new prompt |
| `claude -r <id>` | Resume a specific session by ID |
| `claude -r <name>` | Resume a specific session by name |

## CLI Flags

### Session and Input

| Flag | Description |
|------|-------------|
| `--continue`, `-c` | Continue most recent conversation |
| `--resume`, `-r <id>` | Resume session by ID or name |
| `--fork-session` | Create new session ID preserving conversation history (use with `--resume` or `--continue`) |
| `--from-pr <number>` | Resume sessions linked to a specific pull request; accepts PR number or URL |
| `--session-name <name>` | Name the current session |
| `--model <model>` | Override the default model |
| `--agent <name>` | Run a custom agent as the main thread (overrides `agent` setting) |
| `--agents <json>` | Define custom subagents dynamically via JSON |
| `--permission-mode <mode>` | Set permission mode (default, plan, auto-edit, full-auto, bypassPermissions) |
| `--teleport` | Resume a web session in your local terminal |
| `--teammate-mode <mode>` | Set agent team display: `auto` (default), `in-process`, or `tmux` |

### System Prompt

| Flag | Description |
|------|-------------|
| `--system-prompt <text>` | Replace the system prompt entirely with provided text (headless mode only) |
| `--system-prompt-file <path>` | Replace the system prompt entirely with file contents (headless mode only) |
| `--append-system-prompt <text>` | Add custom instructions to the end of the default system prompt (interactive and headless) |
| `--append-system-prompt-file <path>` | Append file contents to the default system prompt (interactive and headless) |

### Output Control (Headless Mode)

| Flag | Description |
|------|-------------|
| `--output-format text` | Plain text output (default for `-p`) |
| `--output-format json` | JSON object with result, cost, duration, and session ID |
| `--output-format stream-json` | Newline-delimited JSON stream for real-time processing |
| `--max-turns <n>` | Limit the number of agentic turns |
| `--budget-tokens <n>` | Set token budget for the session |
| `--fallback-model <model>` | Model to use if primary model is unavailable |

### Permission Flags

| Flag | Description |
|------|-------------|
| `--allowedTools <tools>` | Comma-separated list of tools to allow without prompting |
| `--disallowedTools <tools>` | Comma-separated list of tools to deny |
| `--dangerously-skip-permissions` | Skip all permission prompts (use only in sandboxed environments) |
| `--permission-prompt-tool <mcp_tool>` | Delegate permission decisions to an MCP tool |

### Miscellaneous

| Flag | Description |
|------|-------------|
| `--verbose` | Enable verbose logging |
| `--no-cache` | Disable prompt caching |
| `--version` | Print version and exit |
| `--help` | Print help and exit |

## Slash Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands and shortcuts |
| `/clear` | Clear conversation history and start fresh |
| `/compact` | Manually compact conversation to reclaim context space |
| `/config` | Open or manage configuration |
| `/context` | Visualize current context usage as a colored grid |
| `/copy` | Copy last assistant response to clipboard |
| `/cost` | Show token usage statistics |
| `/desktop` | Hand off CLI session to the desktop application (macOS/Windows) |
| `/doctor` | Run diagnostics to check for common issues |
| `/agents` | List available subagents |
| `/export [filename]` | Export conversation to file or clipboard |
| `/hooks` | Interactive hook management menu (view, add, delete hooks) |
| `/ide` | Open current session in the IDE extension |
| `/init` | Initialize CLAUDE.md in current project (analyzes codebase for build systems, test frameworks, patterns) |
| `/mcp` | Show MCP server status and per-server token costs |
| `/model` | Switch model mid-session; with Opus, use left/right arrows to adjust effort level |
| `/permissions` | View and manage permission rules |
| `/plan` | Enter plan mode directly from the prompt |
| `/rename` | Rename the current session |
| `/resume` | List and resume previous sessions (opens session picker) |
| `/rewind` | Rewind conversation and/or code, or summarize from a selected message |
| `/sandbox` | View sandbox status and settings |
| `/stats` | Visualize daily usage, session history, streaks, and model preferences |
| `/statusline` | Set up status line UI in terminal |
| `/tasks` | List and manage background tasks |
| `/teleport` | Resume a remote session from the web (subscribers only) |
| `/terminal-setup` | Install terminal key bindings for multiline input and shortcuts |
| `/vim` | Enable vim-style editing mode |

## Keyboard Shortcuts

### Navigation and Control

| Shortcut | Action |
|----------|--------|
| `Enter` | Send prompt / confirm |
| `Ctrl+C` | Cancel current input or generation |
| `Ctrl+D` | Exit Claude Code session |
| `Up` / `Down` | Navigate prompt history |
| `Left` / `Right` | Cycle through dialog tabs in permission dialogs and menus |
| `Esc` `Esc` (double) | Open rewind menu: restore code and/or conversation, or summarize from selected message |
| `Shift+Tab` | Cycle permission modes (plan → default → auto-edit → full-auto; includes delegate mode when agent team active) |

### Session and Model

| Shortcut | Action |
|----------|--------|
| `Alt+P` / `Option+P` | Switch models without clearing current prompt |
| `Alt+T` / `Option+T` | Toggle extended thinking mode (run `/terminal-setup` first) |

### Tools and Output

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Reverse search through command history (type query, `Ctrl+R` to cycle matches, `Tab`/`Esc` to accept, `Enter` to accept and execute, `Ctrl+C` to cancel) |
| `Ctrl+G` | Open prompt in default text editor for editing |
| `Ctrl+O` | Toggle verbose output showing detailed tool usage and execution |
| `Ctrl+B` | Background a running task (tmux users must press twice due to tmux prefix key) |
| `Ctrl+T` | Toggle task list display (shows up to 10 tasks in terminal status area) |
| `Ctrl+L` | Clear terminal screen (keeps conversation history) |
| `Ctrl+V` / `Cmd+V` (iTerm2) / `Alt+V` (Windows) | Paste image from clipboard |

## Permission Modes

| Mode | Behavior |
|------|----------|
| **Plan** | Read-only; Claude cannot modify files or run write commands |
| **Default** | Asks permission for file edits and shell commands |
| **Auto-accept edits** | File edits proceed without asking; shell commands still prompt |
| **Full auto-accept** | All operations proceed without asking |
| **Bypass permissions** | Skip all checks (CLI flag only; requires sandboxed environment) |

## Output Formats

### `--output-format text`
Plain text response. Default for headless mode. Suitable for piping to other CLI tools.

### `--output-format json`
Single JSON object on completion:
- `result` — response text
- `cost` — token costs
- `duration` — execution time
- `session_id` — session identifier
- `is_error` — boolean

### `--output-format stream-json`
Newline-delimited JSON messages during execution. Each line is a JSON object with a `type` field indicating the event kind (assistant message, tool use, tool result, system message). Suitable for real-time monitoring and integration.

## Tool Names for Permission Rules

| Tool | Description |
|------|-------------|
| `Bash` | Shell command execution |
| `Read` | File reading |
| `Write` | File creation/overwrite |
| `Edit` | File editing (string replacement) |
| `Glob` | File pattern matching |
| `Grep` | Content search |
| `WebFetch` | URL fetching |
| `WebSearch` | Web search |
| `Task` | Subagent spawning |
| `Skill` | Skill invocation |
| `NotebookEdit` | Notebook cell editing (.ipynb) |
| `TodoRead` | Read task lists |
| `TodoWrite` | Write/update task lists |

Permission rules use `Tool` or `Tool(specifier)` syntax with glob patterns. Example: `Bash(git *)` matches any git command.

## MCP Tool Pattern for Permission Rules

MCP tools follow the pattern `mcp__<server>__<tool>`. Glob patterns are supported.

Examples:
- `mcp__myserver__query` — specific tool on specific server
- `mcp__myserver__*` — all tools on a specific server
- `mcp__*__*` — all MCP tools

---

## Multiline Input Methods

| Method | Shortcut | Context |
|--------|----------|---------|
| Quick escape | `\` + `Enter` | Works in all terminals |
| macOS default | `Option+Enter` | Default on macOS |
| Shift+Enter | `Shift+Enter` | Works out of the box in iTerm2, WezTerm, Ghostty, Kitty |
| Control sequence | `Ctrl+J` | Line feed character for multiline |
| Paste mode | Paste directly | Auto-detected for code blocks, logs |

For terminals not listed above (VS Code terminal, Alacritty, Zed, Warp), run `/terminal-setup` to install the `Shift+Enter` binding.

---

## Bash Mode (`!` Prefix)

Run bash commands directly without Claude interpreting them by prefixing input with `!`:

```
! npm test
! git status
! ls -la
```

- Adds command output to conversation context
- Shows real-time progress and output
- Does not require Claude to interpret or approve the command
- Supports `Ctrl+B` backgrounding for long-running commands
- History-based autocomplete: type partial command and press `Tab` to complete from previous `!` commands in the current project

---

## Background Bash Commands

Claude Code runs bash commands in the background, returning a unique task ID immediately while the command executes asynchronously.

**Triggering:**
- Prompt Claude to run a command in the background
- Press `Ctrl+B` to move a running Bash invocation to the background (tmux users press `Ctrl+B` twice)

**Behavior:**
- Output is buffered; Claude retrieves it using the `TaskOutput` tool
- Each background task has a unique ID for tracking and retrieval
- Tasks are automatically cleaned up when Claude Code exits

**Common backgrounded commands:** build tools (webpack, vite, make), package managers (npm, yarn), test runners (jest, pytest), development servers, long-running processes (docker, terraform)

**Disable:** Set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`

---

## Prompt Suggestions

After Claude responds, grayed-out suggestions appear based on conversation history and git history.

- Press `Tab` to accept the suggestion, or `Enter` to accept and submit
- Start typing to dismiss
- Runs as a background request reusing the prompt cache (minimal additional cost)
- Skipped when the cache is cold, after the first turn, in non-interactive mode, and in plan mode

**Disable:** Set `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false` or toggle in `/config`

---

## Task List

Claude creates task lists for complex multi-step work, visible in the terminal status area.

- `Ctrl+T` toggles the task list view (displays up to 10 tasks)
- Ask Claude directly to show all tasks or clear them
- Tasks persist across context compactions
- Share a task list across sessions: `CLAUDE_CODE_TASK_LIST_ID=my-project claude`

---

## Session Picker Shortcuts

The `/resume` command (or `claude --resume` without arguments) opens an interactive session picker.

| Shortcut | Action |
|----------|--------|
| `Up` / `Down` | Navigate between sessions |
| `Left` / `Right` | Expand or collapse grouped sessions |
| `Enter` | Select and resume the highlighted session |
| `P` | Preview session content |
| `R` | Rename the highlighted session |
| `/` | Search to filter sessions |
| `A` | Toggle between current directory and all projects |
| `B` | Filter to sessions from current git branch |
| `Esc` | Exit the picker or search mode |

---

## Vim Editor Mode

Enable with `/vim` command or configure permanently via `/config`.

### Mode Switching

| Command | Action | From Mode |
|---------|--------|-----------|
| `Esc` | Enter NORMAL mode | INSERT |
| `i` | Insert before cursor | NORMAL |
| `I` | Insert at beginning of line | NORMAL |
| `a` | Insert after cursor | NORMAL |
| `A` | Insert at end of line | NORMAL |
| `o` | Open line below | NORMAL |
| `O` | Open line above | NORMAL |

### Navigation (NORMAL Mode)

| Command | Action |
|---------|--------|
| `h`/`j`/`k`/`l` | Move left/down/up/right |
| `w` | Next word |
| `e` | End of word |
| `b` | Previous word |
| `0` | Beginning of line |
| `$` | End of line |
| `^` | First non-blank character |
| `gg` | Beginning of input |
| `G` | End of input |
| `f{char}` | Jump to next occurrence of character |
| `F{char}` | Jump to previous occurrence of character |
| `t{char}` | Jump to just before next occurrence |
| `T{char}` | Jump to just after previous occurrence |
| `;` | Repeat last f/F/t/T motion |
| `,` | Repeat last f/F/t/T motion in reverse |

In NORMAL mode, if the cursor is at the beginning or end of input and cannot move further, the arrow keys navigate command history instead.

### Editing (NORMAL Mode)

| Command | Action |
|---------|--------|
| `x` | Delete character |
| `dd` | Delete line |
| `D` | Delete to end of line |
| `dw`/`de`/`db` | Delete word/to end/back |
| `cc` | Change line |
| `C` | Change to end of line |
| `cw`/`ce`/`cb` | Change word/to end/back |
| `yy`/`Y` | Yank (copy) line |
| `yw`/`ye`/`yb` | Yank word/to end/back |
| `p` | Paste after cursor |
| `P` | Paste before cursor |
| `>>` | Indent line |
| `<<` | Dedent line |
| `J` | Join lines |
| `.` | Repeat last change |

### Text Objects (NORMAL Mode)

Text objects work with operators like `d`, `c`, and `y`:

| Command | Action |
|---------|--------|
| `iw`/`aw` | Inner/around word |
| `iW`/`aW` | Inner/around WORD (whitespace-delimited) |
| `i"`/`a"` | Inner/around double quotes |
| `i'`/`a'` | Inner/around single quotes |
| `i(`/`a(` | Inner/around parentheses |
| `i[`/`a[` | Inner/around brackets |
| `i{`/`a{` | Inner/around braces |

# Chapter 2: The Permission and Trust Architecture

## What You'll Learn

Every capable tool creates a tension: the more it can do, the more damage it can cause. Claude Code runs bash commands, edits files, makes network requests, and orchestrates subagents across your entire codebase. The permission system that governs all of this is not a simple on/off switch. It is a layered architecture with five scopes, three evaluation phases, OS-level sandboxing, hook-based extensibility, and a checkpoint system that silently does not cover everything you think it covers.

Most developers encounter this system as a series of confirmation prompts they click through. That is like encountering a car's safety systems by hitting things. What follows breaks down how the trust architecture actually works -- from the organizational policies that override everything you configure locally, to the glob patterns that decide whether a specific tool invocation gets denied, asked, or auto-approved. You will understand why sandbox isolation makes auto-approve safe in some contexts and dangerous in others, how the complete hook event system gives you programmable control over every phase of the agentic loop, why three distinct hook types (command, prompt, and agent) exist and when each is appropriate, and why the devcontainer model has a credential exfiltration problem that no configuration can fully solve.

You will also learn how agentic systems are reshaping security itself -- democratizing expertise so that any engineer can perform reviews that once required specialists, while simultaneously equipping adversaries with the same capabilities. The organizations that win this race are the ones that embed security into their agentic workflows from the start.

After reading this, you will be able to design a permission configuration that matches your actual risk tolerance -- not the default one Claude Code shipped with, and not the wide-open one you switched to because the prompts annoyed you.

---

## The Five-Scope Hierarchy

Claude Code's settings follow a strict precedence order. Understanding this order is the difference between configuring something that works and configuring something that silently gets overridden.

The hierarchy, from highest to lowest priority:

1. **Managed** -- IT-deployed policies in system directories. Cannot be overridden by anything.
2. **CLI arguments** -- Flags passed at invocation time. Override everything below.
3. **Local** -- `.claude/settings.local.json`. Per-machine, gitignored. Your personal overrides.
4. **Project** -- `.claude/settings.json`. Checked into version control. Shared with the team.
5. **User** -- `~/.claude/settings.json`. Your global defaults.

The critical insight is at the top. Managed settings exist so that an organization can enforce policies that individual developers cannot circumvent. If your IT team deploys a managed-settings.json that denies access to a tool, no amount of local or project configuration will re-enable it. The hierarchy is not a suggestion. It is enforcement.

This creates two distinct experiences of Claude Code. Individual developers working on personal projects interact mostly with User and Project scopes. Engineers inside an enterprise may discover that certain capabilities are locked down before they ever see them. Both experiences are intentional.

### Where Settings Live

User settings reside in `~/.claude/settings.json` -- your global defaults that follow you across every project. Project settings live in `.claude/settings.json` within a repository and get committed to version control, meaning the whole team shares them. Local settings go in `.claude/settings.local.json` in the same `.claude/` directory but are gitignored by convention, giving each developer machine-specific overrides without polluting the shared configuration.

Managed settings are the outlier. They sit in system directories controlled by IT -- locations where regular users do not have write access. They are the organizational trump card, and they are designed to be exactly that.

## Permission Rule Evaluation

Within each scope, permission rules follow a three-phase evaluation with first-match-wins semantics. This sounds simple. It has sharp edges.

The phases evaluate in this order:

1. **Deny** -- If a tool invocation matches any deny rule, it is blocked. Period. No further evaluation.
2. **Ask** -- If it matches an ask rule, the user is prompted for approval.
3. **Allow** -- If it matches an allow rule, it proceeds without prompting.

If nothing matches, the default behavior for the tool applies.

Rules use a `Tool` or `Tool(specifier)` syntax with glob pattern support. You can write `Bash(npm test)` to match a specific bash command, `Bash(rm *)` to match destructive deletions, or `Write(*.env)` to match writes to environment files. The glob patterns make this system expressive and also fragile -- an overly broad pattern in a deny rule can silently block operations you intended to allow.

### First Match Wins

This is where developers get tripped up. If you have a deny rule for `Bash(rm *)` and an allow rule for `Bash(rm -rf node_modules)`, the deny rule fires first. Your allow rule never executes. The ordering within each phase matters, and the phase ordering (deny before ask before allow) is immutable.

The practical implication: write your deny rules narrow and your allow rules broad. A deny rule is a hard wall. Make sure it blocks only what you intend.

### Sensitive File Exclusion

The `permissions.deny` configuration replaces an older mechanism called `ignorePatterns`. It provides explicit file-level access control, and you should use it for anything you would not want an AI agent reading or modifying.

The standard targets: `.env` files, credentials directories, private keys, secrets managers, API key configuration. Any file that would be dangerous if its contents appeared in an API request to a cloud service. Claude Code processes file contents through external APIs. If a file contains production database credentials, reading that file sends those credentials to an API endpoint. The `permissions.deny` list is your firewall for that.

## Sandbox Isolation

Claude Code provides OS-level sandboxing for bash commands. This is not a conceptual boundary. It is an operating system enforcement mechanism that restricts what processes spawned by Claude Code can actually do.

The sandbox restricts network access to a configurable domain allowlist. By default, Claude Code can reach the domains it needs for its own operation, but arbitrary network access is blocked. You can add domains to the allowlist for your specific workflow -- your package registry, your internal APIs, your cloud provider endpoints.

The sandbox also supports Unix socket path allowlists, local port binding controls, and command-level exclusions. It is granular enough to express "allow npm install but block curl to arbitrary URLs."

### Auto-Approve Under Sandbox

Here is where the sandbox changes the trust calculus. When sandboxing is active, Claude Code can auto-approve bash commands because the blast radius is contained. A sandboxed `rm -rf /` is still destructive to local files but cannot exfiltrate data to an external server. A sandboxed `curl` cannot reach domains outside the allowlist.

This is the design intent behind the sandbox: make autonomous operation safe by constraining the environment rather than constraining the agent. It trades permission prompts for isolation boundaries. For many workflows, this is the right trade.

But the sandbox has an escape hatch. By default, if Claude Code needs to run an unsandboxed command, it can prompt the user. Managed settings can disable this escape hatch entirely, which is the enterprise-correct configuration when you want hard guarantees.

## Checkpoints: The Safety Net with Gaps

Before every file edit, Claude Code snapshots the affected files. If something goes wrong, pressing Escape twice rewinds both the code and the conversation to the checkpoint state. It is an undo system, and a good one, for what it covers -- direct file edits made through Claude Code's Write and Edit tools.

Checkpoints have significant gaps, most notably that file changes made through bash commands are invisible to the system. The full catalog of checkpoint limitations and their implications is covered in Chapter 10. The key implication for your security posture: if you are doing anything that modifies the filesystem through bash, your safety net is git, not checkpoints. Commit before you start. Commit frequently. Revert when needed.

## Devcontainer Security

Development containers provide network-level isolation with a default-deny firewall. The devcontainer configuration restricts which hosts Claude Code can reach, creating an environment where `--dangerously-skip-permissions` becomes less dangerous because the container itself limits the blast radius.

This is the enterprise answer to "how do I let Claude Code run autonomously in CI without a human approving every command." You do not remove the permission system. You wrap the execution environment in network isolation so that even unrestricted operation cannot reach things it should not reach.

Anthropic provides a reference devcontainer implementation -- a container definition with a custom firewall restricting network access, session persistence, and editor integration preconfigured. It is designed as a starting point for organizations that need to run Claude Code in isolated environments, whether for CI pipelines, security-sensitive projects, or headless autonomous workflows.

### The Exfiltration Caveat

Here is the hard truth the documentation acknowledges directly: a devcontainer cannot prevent credential exfiltration from a malicious project. If a project's code contains instructions that cause Claude Code to read credentials and encode them in output that reaches an allowed endpoint, the container's firewall does not help. The data leaves through an approved channel.

This is not a theoretical concern. Supply chain attacks that embed instructions in code comments or README files are a known vector. A devcontainer protects against accidental network access. It does not protect against intentional abuse by project code that the agent executes.

The mitigation is layered defense: sensitive file exclusion via `permissions.deny` to prevent reading credentials in the first place, combined with container isolation to limit where data can go. Neither layer alone is sufficient. Together they raise the bar substantially.

## The Complete Hook Event System

Hooks are the extensibility layer of the permission system. They are not limited to the two events most developers discover first. The full hook event catalog covers every significant moment in the agentic lifecycle, and understanding all of them unlocks control that permission rules alone cannot achieve.

### Hook Event Catalog

Here is every hook event, what triggers it, and whether it can block the action:

**PreToolUse** -- Fires before any tool executes. Can block the tool call, allow it without prompting, or escalate to the user. Can also modify the tool's input before execution. This is the workhorse of the hook system.

**PostToolUse** -- Fires after a tool succeeds. Cannot block (the tool already ran), but can provide context to Claude about the result. Use it for logging, linting, or injecting additional information.

**PostToolUseFailure** -- Fires after a tool execution fails. Receives the error information and whether the failure was caused by user interruption. Useful for custom error reporting or triggering fallback actions.

**PermissionRequest** -- Fires when Claude Code is about to show a permission dialog to the user. Distinct from PreToolUse -- this fires specifically when the user would be prompted. Your hook can auto-approve, auto-deny, modify the tool input, or apply permission rules equivalent to the "always allow" options the user would see in the dialog. This effectively replaces the interactive prompt with programmatic evaluation.

**Notification** -- Fires when Claude Code sends notifications, matching on notification type: permission prompts, idle prompts, authentication events, and elicitation dialogs. Cannot block notifications but can inject context into the conversation.

**SubagentStart** -- Fires when a subagent is spawned. Cannot block subagent creation but can inject additional context into the subagent's starting state. Matches on agent type: built-in agents or custom agent names from your `.claude/agents/` directory.

**SubagentStop** -- Fires when a subagent finishes. Can block the subagent from stopping, forcing it to continue working. Useful for quality gates that verify a subagent's output before accepting it.

**TeammateIdle** -- Fires when an agent team teammate is about to go idle. Exit code 2 prevents the teammate from going idle and feeds your stderr message back as feedback. Does not support matchers -- fires on every occurrence. Use this to enforce quality gates, like checking that build artifacts exist before allowing a teammate to stop working.

**TaskCompleted** -- Fires when a task is marked as completed, either through an explicit update or when a teammate finishes with in-progress tasks. Exit code 2 blocks completion and feeds stderr back as feedback. Use this to enforce completion criteria -- run the test suite, verify lint passes, check that the task's deliverables exist.

**PreCompact** -- Fires before context compaction. Matches on trigger type: `manual` (user ran `/compact`) or `auto` (context window filled). Receives any custom instructions the user passed to `/compact`. Cannot block compaction but can perform preparation -- logging, saving state, or injecting context that should survive the compaction.

**SessionEnd** -- Fires when a session terminates. Receives a reason code: `clear`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, or `other`. Cannot block termination but is valuable for cleanup tasks, logging session summaries, or triggering post-session workflows.

### Three Hook Types

Not all hooks are shell scripts. Claude Code supports three distinct hook handler types, each suited to different verification needs:

**Command hooks** (`type: "command"`) are the default. They run a shell command, receive the event's JSON input on stdin, and communicate results through exit codes and stdout. Use these for deterministic checks: pattern matching, file existence, command validation. Fast and predictable.

**Prompt hooks** (`type: "prompt"`) use an LLM to evaluate the action. Instead of executing a script, the hook sends the event input along with your prompt to a fast model (Haiku by default) and receives a structured yes/no decision. This is for checks that require judgment rather than pattern matching -- evaluating whether a code change follows architectural conventions, whether a bash command is appropriate for the current task, or whether Claude should be allowed to stop working.

```json
{
  "type": "prompt",
  "prompt": "Evaluate if Claude should stop: $ARGUMENTS. Check if all tasks are complete, no errors remain, and no follow-up is needed."
}
```

The `$ARGUMENTS` placeholder gets replaced with the hook's JSON input. The model returns `{"ok": true}` to allow or `{"ok": false, "reason": "..."}` to block, with the reason fed back to Claude as its next instruction.

**Agent hooks** (`type: "agent"`) are like prompt hooks but with multi-turn tool access. Instead of a single LLM call, an agent hook spawns a subagent that can read files, search code, and inspect the codebase to verify conditions. The subagent runs for up to 50 turns before returning its decision. Use these when verification requires inspecting actual files or test output, not just evaluating the hook input data alone.

```json
{
  "type": "agent",
  "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS",
  "timeout": 120
}
```

The default timeout for agent hooks is 60 seconds (vs. 30 for prompt hooks and 600 for command hooks). The response schema is the same as prompt hooks: `{"ok": true}` to allow, `{"ok": false, "reason": "..."}` to block.

Prompt and agent hooks support these events: PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, UserPromptSubmit, Stop, SubagentStop, and TaskCompleted. TeammateIdle does not support prompt or agent hooks.

### Async Hooks

By default, hooks block Claude's execution until they complete. For long-running tasks -- test suites, deployments, external API calls -- this creates an unacceptable delay. Async hooks solve this.

Set `"async": true` on a command hook to run it in the background. Claude continues working immediately. When the background process finishes, any `systemMessage` or `additionalContext` in the hook's JSON output gets delivered to Claude on the next conversation turn. Async hooks cannot block or control behavior -- the action they would have controlled has already completed by the time they finish.

The classic use case: a PostToolUse hook that runs your test suite after every file write. The tests run in the background while Claude continues working. If tests fail, the failure message appears in Claude's context on the next turn, prompting it to address the issue.

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "command",
      "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/run-tests-async.sh",
      "async": true,
      "timeout": 300
    }]
  }]
}
```

Each async execution creates a separate background process. There is no deduplication -- if Claude writes five files in rapid succession, the hook fires five times.

### Hook Input and Output

Every hook receives a JSON payload on stdin with common fields: `session_id`, `transcript_path`, `cwd`, `permission_mode`, and `hook_event_name`. Event-specific fields are added depending on the hook type -- tool hooks get `tool_name` and `tool_input`, subagent hooks get `agent_id` and `agent_type`, and so on.

The exit code from your hook tells Claude Code what to do:

**Exit 0** means success. Claude Code parses stdout for JSON output fields. For most events, stdout is only shown in verbose mode.

**Exit 2** means block. The exact behavior depends on the event: PreToolUse blocks the tool call, PermissionRequest denies the permission, Stop prevents Claude from stopping, TeammateIdle keeps the teammate working, TaskCompleted prevents the task from closing. The stderr message is fed to Claude as feedback.

**Other exit codes** are treated as hook errors and ignored. Claude Code continues as if the hook did not fire.

JSON output supports several fields across all events: `continue` (boolean, override the exit code decision), `stopReason` (for Stop hooks), `suppressOutput` (hide stdout from verbose mode), `systemMessage` (added to Claude's context on the next turn), and `additionalContext` (added immediately).

### PreToolUse: Modifying Input Before Execution

PreToolUse hooks have a unique capability: they can modify the tool's input before it executes. The `updatedInput` field in the hook's JSON output replaces specific fields in the tool's input parameters.

Combine `updatedInput` with `"permissionDecision": "allow"` to silently rewrite and auto-approve a command. Combine it with `"permissionDecision": "ask"` to show the modified input to the user for approval.

This is flexible and exactly as dangerous as it sounds. A hook that rewrites bash commands can add safety flags, prepend logging, or wrap commands in monitoring harnesses. It can also introduce vulnerabilities if the rewriting logic is flawed. Treat `updatedInput` hooks with the same security scrutiny you would apply to any code that rewrites executable commands.

### Hook Handler: The "once" Field

For hooks defined in skills (not in settings files), the `once` field causes the hook to run only once per session and then remove itself. This is useful for skill initialization -- a hook that runs setup logic on the first tool invocation and then gets out of the way.

### The /hooks Interactive Menu

The `/hooks` command opens an interactive menu showing all configured hooks with labels indicating their source: `[User]`, `[Project]`, `[Local]`, `[Plugin]`. From this menu you can view hook details, add new hooks, delete existing ones, and toggle the `disableAllHooks` setting. It is the quickest way to audit what hooks are active and where they came from.

### Hook Security Model

Hooks are snapshotted at startup. Claude Code captures the state of all configured hooks when the session begins and uses that snapshot for the entire session. If someone -- or something -- modifies hook files on disk mid-session, Claude Code detects the change and displays a warning. The modified hooks do not take effect until you review them in the `/hooks` menu. This prevents malicious or accidental hook modifications from silently changing agent behavior during an active session.

Hooks run with full user permissions. A hook is an arbitrary command that executes on your machine with your credentials, your filesystem access, your network access. There is no sandbox around hooks.

This means hook code requires the same security scrutiny as any other code that runs with your permissions. Input validation matters. Shell quoting matters. Path traversal prevention matters. A hook that naively interpolates tool input into a shell command is a code injection vulnerability.

For enterprises, the `allowManagedHooksOnly` setting in managed configuration restricts hooks to those defined in the managed scope. Individual developers and project-level settings cannot add hooks. This prevents a compromised project from installing hooks that exfiltrate data or modify tool behavior.

## Hook Patterns for Security

The most common security-oriented hook patterns:

### Block Destructive Commands

A PreToolUse hook on Bash that reads the command from the JSON input, checks for dangerous patterns, and returns a deny decision:

```bash
#!/bin/bash
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive rm -rf blocked by hook"
    }
  }'
  exit 0
fi
exit 0  # allow the command
```

When Claude tries to run a destructive command, the hook intercepts it, returns the deny decision, and Claude sees the reason in its context. It will either find an alternative approach or explain why it needs the command.

### Lint on Write

A PostToolUse hook that triggers your linter whenever Claude edits a file:

```json
{
  "PostToolUse": [{
    "matcher": "Edit|Write",
    "hooks": [{
      "type": "command",
      "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/check-style.sh"
    }]
  }]
}
```

The linting script runs after each edit, and any failures appear in Claude's context, prompting it to fix the style issues before moving on. This turns your linter into a real-time quality gate for the agentic loop.

### Sound Effects as Session Feedback

Not every hook is about security. One creative pattern uses hooks to play audio cues on lifecycle events -- a sound when the session starts, when you submit a prompt, when Claude finishes, and when context compacts. The implementation is a one-line command hook on each event that runs the system audio player in the background (with `&` to avoid blocking). On macOS, that is `afplay`; on Linux, `aplay` or `paplay`.

This sounds frivolous, but developers who use it report that the audio feedback helps them maintain awareness of Claude's state while working on something else during autonomous runs. You hear when it finishes. You hear when context compacts (a cue to check whether important instructions survived). It turns background awareness from a tab-switching chore into a passive sensory channel.

### Security Validation Skill Hook

Skills can embed hooks in their frontmatter. A security-focused skill might include a PreToolUse hook that runs a validation script before every Bash command executed while the skill is active:

```yaml
---
description: Security review workflow
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

The hook is scoped to the skill's lifecycle -- it only runs when the skill is active, and it removes itself when the skill completes.

### Multi-Criteria Stop Prompt Hook

A prompt hook on the Stop event that evaluates three conditions before allowing Claude to finish:

```json
{
  "Stop": [{
    "hooks": [{
      "type": "prompt",
      "prompt": "You are evaluating whether Claude should stop working. Context: $ARGUMENTS\n\nAnalyze the conversation and determine if:\n1. All user-requested tasks are complete\n2. Any errors need to be addressed\n3. Any follow-up actions were mentioned but not completed\n\nIf ALL conditions are satisfied, return {\"ok\": true}. If ANY are not met, return {\"ok\": false, \"reason\": \"specific explanation\"}."
    }]
  }]
}
```

This uses a fast model to evaluate whether Claude's work is actually done before letting it stop. If the evaluation finds incomplete work, Claude receives the reason and continues.

### Audit Logging

A PreToolUse hook that logs every tool invocation with timestamps, inputs, and the decision to a file or monitoring service. Zero runtime impact on the developer, full visibility for security teams.

### File Access Control

A PreToolUse hook on Write and Edit that restricts modifications to specific directory trees, enforcing that Claude Code only touches code in the current project. Combined with `permissions.deny` for read restrictions, this creates a comprehensive file-level access boundary.

### MCP Tool Gating

PreToolUse hooks matching `mcp__server__tool` patterns that log or restrict access to external service integrations. MCP tools follow the naming pattern `mcp__<server>__<tool>`, and you can use regex in matchers -- `mcp__memory__.*` matches all tools from a memory server, `mcp__.*__write.*` matches write operations across all MCP servers.

## Settings Scope Characteristics

Each settings scope has distinct properties that matter for team workflows:

**User scope** (`~/.claude/settings.json`): Personal. Follows you everywhere. Good for your editor preferences, your personal permission rules, your default model selection. Never shared.

**Project scope** (`.claude/settings.json`): Shared via version control. The team's agreed-upon configuration. Hooks, permissions, plugin enablements, environment variables that everyone needs. Changes go through code review like any other committed file.

**Local scope** (`.claude/settings.local.json`): Per-machine overrides. Gitignored. Your personal tweaks on top of the project configuration. Machine-specific paths, personal API keys for development services, tools you want that the team configuration does not include.

**Managed scope**: IT-controlled. Invisible to most developers. Enforces organizational policy. Cannot be inspected or overridden from the CLI. The existence of this scope is why "it works on my machine but not on my work machine" is a real conversation in enterprise Claude Code deployments.

The interplay between these scopes is where configuration gets interesting. A project might allow a tool. A managed policy might deny it. The managed policy wins, silently. No error message, no explanation -- the tool simply is not available. If you are debugging why something works on personal projects but not at work, check whether managed settings exist.

## MCP as Security-Controlled Access

When Claude Code needs to interact with external services, there are two paths: run a CLI command via bash, or use an MCP server.

The MCP path is better for security. An MCP server provides structured tool interfaces with defined inputs and outputs. Every invocation goes through the permission system, can be intercepted by hooks, and produces structured logs. A bash command running `curl` with credentials provides none of that visibility -- credentials appear in the command string, in the conversation context, and potentially in API calls to the model provider.

The security-conscious approach to Claude Code integration is: build MCP servers for your sensitive data sources. Do not give Claude bash access to your production databases. Give it an MCP tool that wraps the database access with proper authentication, logging, and query restrictions. Chapter 5 covers the full architecture of MCP and the detailed security case for MCP over CLI access.

One legal team at Anthropic raised an important concern from the product counsel perspective: as MCP integrations reach deeper into sensitive systems, conservative security postures will create barriers. The more capable the tooling becomes, the more access it needs, and the more friction security policies generate. Organizations should expect this tension and plan for it -- building compliance tooling proactively rather than reactively, establishing MCP access governance frameworks before the integrations proliferate.

## Security Dual-Use Risk

Agentic coding tools are dual-use technology. The same capabilities that help a security team audit code and generate patches also help an attacker find vulnerabilities and generate exploits. Claude Code's ability to read an entire codebase, understand its architecture, and make targeted changes is exactly as useful for defense as it is for offense.

This is not a hypothetical. Security researchers use Claude Code to find bugs. Attackers can use it the same way. The democratization of expertise that makes Claude Code valuable for non-security-engineers to write secure code also makes it valuable for non-security-engineers to find insecure code.

### Agentic Cyber Defense

The trends are moving in two directions simultaneously. On the defensive side, agentic systems are making it possible for any engineer to perform security reviews, hardening, and monitoring that previously required specialized expertise. Security knowledge is becoming democratized -- an engineer without a security background can now use Claude Code to audit code paths, identify common vulnerability patterns, and generate hardened implementations.

On the offensive side, threat actors will scale attacks using the same tools. The prediction from industry analysis is clear: automated agentic systems will enable security responses at machine speed, automating detection and response to match the pace of autonomous threats. The organizations that prepare for this -- baking security into their agentic workflows from the start rather than bolting it on later -- will be better positioned to defend against adversaries using the same technology.

### Security Review as a Daily Practice

The security engineering team at Anthropic demonstrated a practical pattern: copying infrastructure-as-code plans into Claude Code to ask "what is this going to do, and am I going to regret it?" Infrastructure changes that require security approval -- deployment configurations, network policy changes, access control modifications -- get reviewed by Claude in seconds rather than sitting in a queue for the security team. This creates tighter feedback loops and eliminates developer blocks while waiting for security review.

This is not replacing security review. It is making security review continuous. Instead of a bottleneck at the end of the pipeline, security analysis happens at the moment of creation. The security team still reviews the final configuration, but Claude catches the obvious issues before they get that far.

The organizational response is not to restrict Claude Code -- that forfeits the defensive advantage while doing nothing about offensive use. The response is to ensure your security architecture does not depend on attackers being unsophisticated. Assume your adversaries have access to tools at least as capable as yours. Then build your defenses accordingly.

## Paper Trading: Safe Experimentation Boundaries

The concept of paper trading -- simulating real operations without real consequences -- applies directly to Claude Code adoption. Before giving an agent access to production systems, production databases, or production credentials, establish a boundary where it can operate freely without risk.

This means: sandbox environments with synthetic data. Staging systems that mirror production topology but contain nothing sensitive. Local development environments with mocked external services. The agent can run autonomously, make mistakes, discover edge cases, and crash without consequences.

The pattern from practitioners who have run agents in autonomous loops for extended periods is consistent: start with paper trading. Validate behavior in safe environments. Expand the boundary gradually. The trust architecture supports this progression -- you can start with maximum restriction and relax constraints as you build confidence.

The permission system, sandbox isolation, hooks, and managed settings together create a gradient from "Claude Code cannot do anything without asking" to "Claude Code can do everything within this isolated environment." The right position on that gradient depends on your context, your risk tolerance, and how much you have validated in safe environments first.

---

## Key Takeaways

- The scope hierarchy is strict -- Managed settings override everything, and there is no escape from organizational policy.
- Permission rules evaluate Deny before Ask before Allow, with first-match-wins semantics, so write deny rules narrow and allow rules broad.
- Checkpoints cover direct file edits but not bash command side effects -- git is your real safety net.
- Devcontainers provide network isolation but cannot prevent credential exfiltration from malicious project code; layer defenses, and use Anthropic's reference implementation as a starting point.
- The hook system has thirteen events covering the entire agentic lifecycle: PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, Notification, SubagentStart, SubagentStop, Stop, TeammateIdle, TaskCompleted, PreCompact, SessionEnd, and more.
- Three hook types serve different verification needs: command hooks for deterministic checks, prompt hooks for LLM-evaluated judgments, and agent hooks for multi-turn investigation with file access.
- Async hooks run in the background without blocking Claude; use them for test suites and long-running validation.
- Hooks are snapshotted at startup -- mid-session modifications trigger warnings and require review in the `/hooks` menu before taking effect.
- PreToolUse hooks can modify tool input before execution via `updatedInput`, enabling command rewriting, safety flag injection, and monitoring harnesses.
- MCP servers provide better security visibility than raw bash commands for sensitive data access; build MCP servers for your sensitive data sources.
- Security knowledge is being democratized by agentic tools; any engineer can now perform reviews that once required specialists, but adversaries gain the same capabilities.
- Start with maximum restriction and paper-trading environments, then relax constraints as you validate agent behavior.

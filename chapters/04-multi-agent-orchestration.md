# Chapter 4: Multi-Agent Orchestration

## What You'll Learn

The moment your task outgrows a single context window, you have two choices: fight the constraint or distribute the work. Multi-agent orchestration is how you distribute the work.

Claude Code ships with a subagent system that lets you spin up isolated AI workers, each with its own context window, its own tool permissions, and its own system prompt. For more ambitious coordination, an experimental agent teams system lets multiple independent sessions communicate through seven coordination primitives, shared task lists, and direct messaging. These are fundamentally different architectures with different cost profiles, different failure modes, and different sweet spots.

This chapter covers both in depth. You will learn the six built-in subagent types, how to define custom subagents with persistent memory, hooks, turn limits, and preloaded skills. You will understand the seven agent team primitives, four message types, three display modes, task dependencies, file-locking for concurrent claims, and plan approval workflows. You will see why a practitioner's four-agent trading hierarchy degraded to a single agent because simplicity won, and why a QA swarm of five parallel agents audited an entire blog in three minutes. You will also learn the task system that coordinates work across sessions and subagents, and why the cheapest orchestration strategy is almost always the correct one.

---

## The Architecture of Subagents

A subagent is a Claude Code session within a session. It gets its own context window, its own system prompt, and a restricted set of tools. When it finishes, it returns a summary to the parent session. Then its context window is discarded.

The key constraint: subagents cannot spawn other subagents. This creates a strict two-level hierarchy. You have an orchestrator (your main session) and workers (subagents). There is no middle management. No delegation chains. No recursive agent trees. This is a deliberate design decision, not a limitation. Recursive agent hierarchies sound elegant and produce chaos.

Six built-in subagent types ship with Claude Code:

**Explore subagents** run on a smaller, cheaper model optimized for speed. They are read-only -- they can search files, read code, and navigate the codebase, but they cannot modify anything. Use them when you need to understand something before deciding what to do. They are fast and cheap enough to spin up casually.

**Plan subagents** inherit the main session's model. They are also read-only, but they bring the full reasoning capability of the orchestrator model to bear on research and planning tasks. Use them when the planning itself is complex enough to warrant dedicated context.

**General-purpose subagents** have access to all tools. They can read, write, execute commands, and perform multi-step tasks. These are your workhorses for implementation.

**Bash subagents** are specialized for command execution. They are useful when you need to run a sequence of shell commands without polluting the main context with verbose output.

**Claude Code Guide subagents** are specialized for answering questions about Claude Code itself -- its features, configuration, and best practices. They consult the product's own documentation.

**Statusline-setup subagents** handle the narrow task of configuring terminal status line integrations. They exist because the setup process is fiddly enough to warrant isolated context.

You can also define custom subagents as Markdown files with YAML frontmatter, specifying system prompts, tool restrictions, model preferences, and turn limits. These definitions live in `.claude/agents/` for project scope or `~/.claude/agents/` for user scope. Plugins can supply them too, at the lowest priority.

### Custom Subagent Definition Fields

Beyond the basics of system prompt and tool restrictions, custom subagent definitions support several fields that shape behavior in important ways:

**`maxTurns`** limits the number of agentic turns before a subagent stops. This is a safety valve. A subagent exploring a codebase might spiral into increasingly tangential searches. Setting `maxTurns: 20` caps the exploration and forces a result. Without it, a subagent on a dead-end path burns tokens until context runs out.

**`mcpServers`** configures which MCP servers are available to a specific subagent. You can reference servers defined in `.mcp.json` by name or inline a full server configuration. This means a database-reader subagent can have access to your database MCP server while a code-reviewer subagent has none. Tool access scoped to the subagent's role, not the session's full capability set.

**`skills`** preloads skill content into the subagent's context at launch. In the main session, skills load on demand -- only the description is present until Claude invokes the skill. Subagents work differently. Skills passed to a subagent are fully injected into its context at startup because the subagent needs to act immediately without the interactive skill-loading flow. This means every skill you preload costs context from turn one.

**`memory`** enables persistent memory, covered in detail later in this chapter.

### The `--agents` CLI Flag

For quick testing or CI automation, you can pass subagent definitions as JSON directly when launching Claude Code instead of writing Markdown files:

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Reviews code for style and correctness",
    "prompt": "You are a code reviewer. Check for bugs, style violations, and security issues.",
    "tools": ["Read", "Glob", "Grep"],
    "model": "sonnet"
  },
  "debugger": {
    "description": "Debugs failing tests",
    "prompt": "You are a debugging specialist. Analyze test failures and suggest fixes.",
    "tools": ["Read", "Glob", "Grep", "Bash"],
    "model": "sonnet",
    "maxTurns": 30
  }
}'
```

The `--agents` flag accepts the same fields as file-based definitions: `description`, `prompt`, `tools`, `disallowedTools`, `model`, `permissionMode`, `mcpServers`, `hooks`, `maxTurns`, `skills`, and `memory`. This is particularly useful in headless pipelines where you want to define specialized agents without committing Markdown files to the repository.

### The `--agent` Flag

A related but distinct flag: `--agent` runs a custom agent definition as the main thread rather than as a subagent. If you have a `code-reviewer` agent defined in `.claude/agents/code-reviewer.md`, running `claude --agent code-reviewer` starts the session using that agent's system prompt, tools, and configuration. The session runs as a top-level agent, not as a worker within another session. You can restrict which subagents this top-level agent can spawn using `Task(agent_type)` syntax, creating controlled hierarchies where a review agent can only delegate to explorer-type subagents.

## Context Isolation Is the Point

The most important thing subagents give you is not parallelism. It is context isolation.

When you ask the main session to read a large file, parse verbose test output, or explore a sprawling directory tree, all of that content accumulates in the main context window. Do this enough times and you hit the ceiling described in Chapter 1 -- instructions get forgotten, work gets dropped, quality degrades.

Subagents solve this by containing the mess. A subagent can read fifty files, parse a thousand lines of test output, explore every directory in the repo, and the main session only sees the summary. The verbose intermediate work stays in the subagent's context and is discarded when the subagent finishes.

This is why experienced users route expensive context operations through subagents even when they do not need parallelism. Reading a large codebase module? Subagent. Running a comprehensive test suite and analyzing failures? Subagent. Searching for usage patterns across hundreds of files? Subagent. The goal is keeping the orchestrator's context lean so it can coordinate effectively across dozens of tasks without losing coherence.

The tradeoff is that summaries are lossy. A subagent that returns a detailed result consumes main context proportional to the length of that result. If you spin up ten subagents and each returns a page of findings, you have ten pages of results in your main window. The solution is to instruct subagents to be concise in their returns -- specific answers, not comprehensive reports.

## Foreground vs. Background Execution

Subagents can run in the foreground or the background, and the distinction matters more than it appears.

**Foreground subagents** block the main conversation. The orchestrator waits for them to finish before proceeding. They have full access to MCP tools, can ask clarifying questions if they get stuck, and integrate seamlessly with the permission system. Use foreground execution when the subagent's result determines the next step.

**Background subagents** run concurrently while the main session continues. This is where parallelism actually happens -- you can have multiple background subagents exploring different parts of the codebase or implementing different components simultaneously. But background execution comes with restrictions: no MCP tools, no clarifying questions, and permissions must be pre-negotiated before launch. If a background subagent encounters something that needs human approval and was not pre-approved, it auto-denies and moves on.

Background subagents can be resumed in the foreground if they complete with unresolved questions or partial results. This is useful for tasks that start as background exploration but surface something that needs interactive discussion. Press **Ctrl+B** to background a running foreground task -- it continues working while you regain the main session. If a background subagent fails due to missing permissions, you can resume it in the foreground to retry with interactive prompts.

The practical pattern is to use foreground for tasks where you need the result immediately and background for tasks that can run while you focus on something else. A common orchestration workflow: launch three background subagents to explore three modules, continue working on something else in the main session, then review their results when they complete.

### Resuming Subagents

Subagent transcripts persist independently of the main conversation. They are stored as separate files at `~/.claude/projects/{project}/{sessionId}/subagents/`. When the main conversation compacts, subagent transcripts are unaffected. You can resume a subagent after restarting Claude Code by resuming the same session -- ask Claude to "continue that code review" and it picks up from the persisted transcript.

This persistence means subagent work is not lost when the main session resets. A subagent that spent twenty minutes analyzing an authorization module has its full transcript on disk. Resuming gives it that context back without re-doing the analysis.

### Subagent Auto-Compaction

Subagents support automatic compaction using the same logic as the main conversation. By default, auto-compaction triggers at approximately 95% capacity. For subagents that handle large volumes of data -- parsing thousands of lines of test output, reading dozens of files -- this means they can work through content that exceeds their context window by periodically summarizing and continuing.

Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` to a lower percentage (for example, `50`) to trigger compaction earlier. This applies to both main conversations and subagents. Earlier compaction reduces peak context consumption at the cost of potentially losing detail from earlier turns. For subagents doing broad exploration, earlier compaction is usually fine. For subagents doing precise analysis where every detail matters, let them run closer to capacity before compacting.

## Subagent Hooks

Custom subagents support lifecycle hooks that run at specific points during execution. These are defined in the agent's YAML frontmatter and work with the same event model as session-level hooks (Chapter 2), but scoped to the subagent.

Three hook events fire within the subagent's execution: **PreToolUse** runs before the subagent uses a tool and can block or modify the invocation. **PostToolUse** runs after a tool completes and can transform output or trigger side effects. **Stop** runs when the subagent finishes -- but at runtime, it is automatically converted to a **SubagentStop** event.

Two additional hook events fire at the project level, outside the subagent's own context: **SubagentStart** fires when any subagent spawns, and **SubagentStop** fires when any subagent finishes. These are defined in your project or user settings, not in the subagent definition itself. A SubagentStart hook can inject `additionalContext` into the subagent -- instructions or data that supplement its system prompt without modifying the agent definition. A SubagentStop hook can block the subagent from stopping (exit code 2), useful for enforcing that a subagent must produce certain artifacts before it is allowed to finish.

A practical example: a database-reader subagent with a PreToolUse hook on the Bash tool that checks whether commands contain SQL write operations (INSERT, UPDATE, DELETE, DROP). If the hook detects a write statement, it returns a deny decision, preventing the subagent from modifying production data regardless of what its instructions say. This is defense in depth -- the subagent's system prompt says "read only," and the hook enforces it mechanically.

## Subagent Patterns

Three patterns recur in production subagent usage. They are worth naming because they solve specific orchestration problems.

**Chain subagents** run in sequence, where each subagent completes its task and returns results to the orchestrator, which then passes relevant information to the next subagent. Subagent A analyzes the authentication module and returns a list of vulnerabilities. The orchestrator passes that list to Subagent B, which generates fixes. Subagent B's output goes to Subagent C, which writes tests. Each subagent has clean context with only the information it needs. The chain structure prevents context accumulation while maintaining information flow.

**Isolate high-volume operations** by routing anything that produces large output to a subagent. Running a comprehensive test suite might generate thousands of lines of output. Fetching documentation from an API might return hundreds of pages. Processing log files might involve scanning megabytes of text. If this happens in the main session, it crowds out everything else. A subagent absorbs the volume and returns a concise summary: "14 tests failed, all in the auth module, all related to session expiry."

**Parallel research** spawns multiple subagents simultaneously to investigate different aspects of a problem. Three subagents explore the authentication module, the database layer, and the API endpoints in parallel. Each returns findings. The orchestrator synthesizes them into a unified understanding. This is faster than sequential exploration and keeps the orchestrator's context free of the intermediate exploration steps.

## Agent Teams: The Experimental Layer

Agent teams are a different architecture entirely. Where subagents are workers within a session, agent teams are independent Claude Code sessions that coordinate through shared infrastructure.

### Enabling Agent Teams

Agent teams are disabled by default. Enable them by setting the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable to `1`, either in your shell or through settings:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Then describe what you want in natural language. Claude creates the team, spawns teammates, and coordinates work based on your prompt.

### The Seven Team Primitives

The system provides seven tools that handle the full lifecycle of team coordination.

**TeamCreate** initializes a team. It creates a team directory and configuration file on disk. The `team_name` parameter is the namespace that links everything -- tasks, messages, and config all live under it.

```
TeamCreate({ "team_name": "auth-refactor", "description": "Refactor authentication module" })
```

**TaskCreate** defines a unit of work. Each task becomes a JSON file on disk. The lead creates tasks before spawning teammates, providing enough detail in the description to serve as a prompt for the agent that picks it up.

```
TaskCreate({
  "subject": "Extract JWT handling into jwt.ts",
  "description": "Move all token signing and verification logic from auth.ts to a new jwt.ts module...",
  "team_name": "auth-refactor"
})
```

**TaskUpdate** moves work through the pipeline. Teammates use it to claim tasks (setting status to `in_progress` with an owner) and to mark tasks complete. The status field prevents two agents from working on the same task simultaneously.

```
TaskUpdate({ "taskId": "1", "status": "in_progress", "owner": "jwt-agent" })
TaskUpdate({ "taskId": "1", "status": "completed" })
```

**TaskList** returns all tasks with their current status. Teammates call this after completing a task to find what is next. There is no centralized scheduler -- each teammate polls TaskList, finds unowned pending tasks, and claims one. This is the shared coordination mechanism.

**Task (with team_name)** spawns a teammate. The critical detail: each teammate is a full Claude Code session with its own context window. Teammates load the same project context (CLAUDE.md, MCP servers, skills) but do not inherit the lead's conversation history. You can specify the model for each teammate independently.

```
Task({
  "subagent_type": "general-purpose",
  "name": "jwt-agent",
  "team_name": "auth-refactor",
  "model": "sonnet"
})
```

**SendMessage** is what makes teams different from subagents. Any teammate can message any other teammate directly. It supports four message types:

- **`message`**: direct communication between two agents. A teammate reports findings to the lead, or one teammate asks another for information.
- **`broadcast`**: reaches all teammates at once. Use sparingly -- costs scale with team size because every teammate processes the message.
- **`shutdown_request` / `shutdown_response`**: graceful teardown protocol. The lead sends a shutdown request; the teammate acknowledges with a response. A teammate can reject the shutdown with an explanation if it has unfinished work.
- **`plan_approval_response`**: quality gate. The lead approves or rejects a teammate's implementation plan.

```
// Teammate reports findings to lead
SendMessage({
  "type": "message",
  "recipient": "lead",
  "content": "JWT extraction complete. Created jwt.ts with signToken and verifyToken exports."
})

// Lead requests graceful shutdown
SendMessage({
  "type": "shutdown_request",
  "recipient": "jwt-agent",
  "content": "All tasks complete, shutting down team."
})
```

**TeamDelete** removes the team config and all task files from disk. Called after all teammates have shut down.

### The Team Lead Abstraction

What makes agent teams more than parallel subagents is the team lead's coordination role. The lead is the session that creates the team. It is responsible for creating tasks and defining work breakdown, spawning teammates with appropriate roles, monitoring progress through the task list, synthesizing results from multiple agents, and handling graceful shutdown.

The task files on disk and SendMessage are the only coordination channels -- there is no shared memory. Teammates each have their own conversation history and context window, independent from the lead and from each other. This independence is the source of both the architecture's strength (true parallel work with full context isolation) and its cost (each teammate is a full Claude Code session burning tokens independently).

A lead can be put into **delegate mode** (Shift+Tab after starting a team). In delegate mode, the lead cannot implement tasks itself -- it can only coordinate: spawning, messaging, shutting down teammates, and managing tasks. This prevents the lead from doing work that should be distributed to teammates, which is a surprisingly common failure mode.

### Display Modes

Agent teams support three display modes, configured via the `teammateMode` setting or the `--teammate-mode` flag:

**`auto`** (default) uses split panes if you are already running inside a tmux session, and in-process otherwise.

**`in-process`** runs all teammates inside your main terminal. Use Shift+Up/Down to select a teammate and type to message them directly. Press Enter to view a teammate's session, Escape to interrupt their current turn, Ctrl+T to toggle the task list. Works in any terminal with no extra setup.

**`tmux`** gives each teammate its own terminal pane. You can see everyone's output simultaneously and click into a pane to interact directly. Split-pane mode requires tmux or iTerm2 -- it does not work in a code editor's integrated terminal.

```bash
# Force in-process for a single session
claude --teammate-mode in-process
```

### Task Dependencies and File Locking

Tasks can depend on other tasks. A pending task with unresolved dependencies cannot be claimed until those dependencies complete. When a teammate finishes a task that others depend on, blocked tasks unblock automatically. This creates a natural wave pattern: independent tasks run in parallel first, dependent tasks follow.

Task claiming uses file locking to prevent race conditions when multiple teammates try to claim the same task simultaneously. Without file locking, two agents polling TaskList at the same moment could both attempt to claim the same task, leading to duplicated work. The lock ensures exactly one agent owns each task.

### Plan Approval

For complex or risky tasks, you can require teammates to plan before implementing. The teammate works in read-only plan mode until the lead approves their approach:

```
Spawn an architect teammate to refactor the authentication module.
Require plan approval before they make any changes.
```

When a teammate finishes planning, it sends a plan approval request to the lead. The lead reviews the plan and either approves it (teammate exits plan mode and begins implementation) or rejects it with feedback (teammate stays in plan mode, revises, and resubmits). You can influence the lead's judgment by giving it criteria in your prompt: "only approve plans that include test coverage" or "reject plans that modify the database schema."

This is the difference between "trust the agents to do the right thing" and "verify the approach before spending tokens on implementation." For a team of five agents running in parallel, each burning thousands of tokens per minute, the plan approval step is cheap insurance.

### Team Hook Events

Two hook events are specific to agent teams:

**TeammateIdle** fires when an agent team teammate is about to go idle -- it has finished its current work and is ready to stop. Exit code 2 from this hook forces the teammate to continue working. A practical use: a hook that checks whether build artifacts exist in the expected output directory before allowing the teammate to stop. If the artifacts are missing, the hook rejects the idle state and the teammate keeps working.

**TaskCompleted** fires when a task is marked as completed via TaskUpdate, or when a teammate finishes its turn with in-progress tasks still assigned. Exit code 2 blocks the completion. This enables quality gates: a hook that runs the test suite when a task is marked complete, blocking the completion if tests fail. The teammate sees the failure output and can self-correct.

### Agent Teams vs. Subagents: Detailed Comparison

| | Subagents | Agent Teams |
|---|---|---|
| **Context** | Own window; results return to the caller | Own window; fully independent |
| **Communication** | Report results back to the main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-claim |
| **Token cost** | Lower: results summarized back to main | Higher: each teammate is a full session |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |

The distinction matters most when agents need to talk to each other. An API teammate that finishes type definitions and directly messages the UI teammate to start integration work -- that is an agent team pattern. If each worker just reports back to the orchestrator, subagents are cheaper and simpler.

### Limitations

Agent teams are experimental and come with real constraints:

- **No session resumption** for in-process teammates. If a teammate crashes, its work and context are lost.
- **Task status can lag** -- teammates sometimes forget to mark tasks complete, which can block dependent tasks.
- **One team per session.** You cannot run nested teams or multiple teams from one lead session.
- **Split panes require tmux or iTerm2**, not a code editor's integrated terminal.
- **All teammates start with the lead's permission settings.** You cannot give one teammate broader permissions than the lead has.

## Subagents vs. Teams: When to Use What

The decision between subagents and teams maps to scope, cost, and coordination complexity.

**Subagents** are right when: the work decomposes into independent tasks that can be completed without coordination, each task fits in a single subagent session, and you need results quickly and cheaply. A typical subagent workflow takes minutes and costs a fraction of a main session.

**Teams** are right when: the work requires sustained coordination between specialists, tasks have dependencies that need to be tracked, or the problem requires different agents maintaining long-running context about different aspects of the system. A typical team workflow takes longer and costs several times what subagents would.

**Neither** is right when a single well-prompted session can handle the task. This is more common than you think. Before reaching for multi-agent orchestration, ask whether the task genuinely exceeds what one context window can handle. If the answer is no, a single agent with clear instructions will outperform a multi-agent setup every time. The coordination overhead of multi-agent systems is not free, and it introduces failure modes that single agents do not have.

The hierarchy of preference: single agent first, subagents when context isolation is needed, teams when sustained coordination is needed. Work your way up, not down.

## Agent Team Case Studies

### The QA Swarm

A practitioner pointed agent teams at their blog with a single prompt: use a team of agents to QA the blog before production deploy. The lead created five tasks and spawned five parallel agents, each running on a cheaper model:

| # | Task | Agent | What It Checked |
|---|------|-------|-----------------|
| 1 | Core page responses | qa-pages | 16 URLs for correct HTTP status codes |
| 2 | Blog post rendering | qa-posts | 83 posts for headings, meta tags, working images |
| 3 | Navigation and link integrity | qa-links | 146 internal URLs for broken links |
| 4 | RSS, sitemap, SEO metadata | qa-seo | RSS validity, robots.txt, Open Graph tags |
| 5 | Accessibility and HTML structure | qa-a11y | Heading hierarchy, ARIA attributes, theme toggle |

Each agent finished independently and sent a structured report back via SendMessage. The agents used bash and curl to fetch pages and parse HTML directly -- no browser automation, no test framework. When all five completed, the lead synthesized their findings into a prioritized report with issues ranked by severity (major, medium, minor). Then the lead sent shutdown requests, teammates acknowledged, and TeamDelete cleaned up.

The entire lifecycle -- from prompt to final report -- took about three minutes. Five agents, over 146 URLs tested, 83 blog posts checked. The cost was higher than a single session doing the same work sequentially, but the wall-clock time was a fraction.

### The Authentication Module Refactor

A more coordination-intensive example: refactoring an authentication module with one teammate on the API layer, one on frontend components, and one running tests continuously. The key difference from subagents: the API teammate finished type definitions and messaged the UI teammate directly to say "the interfaces are ready, here's what changed." The test teammate could ask the API teammate to spin up a dev server. Self-coordination, without routing every interaction through the lead.

This is the pattern where agent teams justify their cost. The agents needed to react to each other's work in real time. With subagents, the orchestrator would have been the bottleneck -- each worker reports to the parent, the parent relays to the next worker, round and round. With a team, the workers talk directly.

### Plan First, Parallelize Second

The most effective agent team pattern is a two-step approach: plan first with plan mode, then hand the plan to a team for parallel execution.

Step one: start in plan mode. Let the orchestrator explore the codebase, identify files, and produce a step-by-step implementation plan. Review it. Adjust it. This is cheap -- plan mode only reads files. The orchestrator might produce something like:

```
Plan:
1. Create src/auth/jwt.ts -- extract token signing/verification
2. Create src/auth/sessions.ts -- extract session logic
3. Create src/auth/middleware.ts -- extract Express middleware
4. Update src/auth/index.ts -- re-export public API
5. Update 12 import sites across the codebase
6. Update tests in src/auth/__tests__/
```

Step two: execute the plan as a team. The plan already has the task breakdown. The lead sees the dependency graph and spawns teammates in waves:

- **Wave 1** (parallel): jwt.ts + sessions.ts + middleware.ts -- three teammates
- **Wave 2** (after wave 1): index.ts barrel + update imports -- one or two teammates
- **Wave 3** (after wave 2): update tests -- one teammate

Plan mode costs roughly 10,000 tokens. A team that goes in the wrong direction costs 500,000 or more. Spending a few seconds reviewing a plan before committing to parallel execution saves you from expensive course corrections mid-swarm.

## Persistent Memory

Subagents can maintain a persistent memory directory that survives across sessions. Configure this with the `memory` field in the agent definition, specifying a scope:

- **`user`**: shared across all projects. Stored at `~/.claude/agent-memory/{agent-name}/`. Use for agents whose knowledge applies everywhere -- coding style preferences, common debugging patterns, personal workflow rules.
- **`project`**: shared within a project. Stored at `~/.claude/projects/{project}/agent-memory/{agent-name}/`. Use for project-specific agents -- a code reviewer that learns this codebase's conventions, a test agent that knows this project's testing patterns.
- **`local`**: specific to one working directory. Use when different branches or worktrees of the same project need different agent knowledge.

The memory directory contains a `MEMORY.md` entrypoint and optional topic files:

```
~/.claude/projects/{project}/agent-memory/code-reviewer/
├── MEMORY.md          # Concise index, loaded into every session
├── style-patterns.md  # Detailed notes on code style observations
└── common-issues.md   # Recurring problems this codebase has
```

`MEMORY.md` acts as an index. The first 200 lines are loaded into the subagent's system prompt at the start of every invocation. Content beyond 200 lines is not loaded automatically -- Claude is instructed to keep the index concise by moving detailed notes into separate topic files. The subagent reads and writes files in this directory throughout its execution, using `MEMORY.md` to track what is stored where.

Over multiple invocations, the subagent accumulates knowledge: patterns it has observed, decisions that worked, errors to avoid. A code review agent that learns your team's conventions produces better reviews after ten runs than it did after one. A test generation agent that remembers which test patterns caught real bugs focuses on high-value test cases.

This is distinct from CLAUDE.md, which is always-on context for the main session (Chapter 3). Persistent memory is agent-specific and loads only when that agent runs. It is a way to give specialized agents domain knowledge without bloating the main session's context.

### Vector-Based Experience Retrieval

One practitioner building an autonomous trading system went further than flat-file memory. The system stored agent experiences in a vector database, using similarity search to retrieve relevant past scenarios when the agent encountered new situations. Rather than loading all history into context, the agent queried for experiences similar to the current market conditions and loaded only those.

This pattern -- vector similarity over past decisions rather than linear MEMORY.md files -- is relevant when the volume of accumulated experience is too large for a 200-line index. A trading agent that has made thousands of decisions cannot keep them all in a summary file. But a vector database lets it find the five most relevant past decisions for any new scenario, which fits comfortably in a subagent's context.

The approach generalizes beyond trading. Any agent that handles recurring but varied tasks -- incident response, code review across multiple repositories, customer issue classification -- benefits from experience retrieval that scales with the volume of past work.

## Cost Optimization: Expensive Brain, Cheap Hands

The most cost-effective multi-agent pattern is conceptually simple: use an expensive, capable model for orchestration and cheap, fast models for execution.

Your orchestrator -- the main session -- does the planning, decomposition, and quality evaluation. This is where reasoning quality matters most, so it runs on the most capable model available. Your subagents do the implementation, exploration, and grunt work. Many of them can run on smaller models without meaningful quality loss.

Explore subagents already implement this pattern by defaulting to a smaller, faster model. You can extend it to custom agents by specifying model preferences in the agent definition.

The economics work because orchestration is a small fraction of total token usage. Planning a refactor might consume 10,000 tokens. Executing that refactor across twenty files might consume 500,000 tokens. If the execution runs on a model that costs a fifth as much per token, you have cut total costs by roughly 80% with no loss in planning quality.

This maps to an organizational metaphor: a senior architect who designs the system and junior developers who implement it. The architect's time is expensive and their decisions have leverage. The implementers' time is cheaper and their work is guided by the architect's plan. You would not pay architect rates for someone to write boilerplate. Do not pay orchestrator token rates for subagent grunt work.

## Plan-First Parallelization

The single most expensive mistake in multi-agent orchestration is launching parallel agents before you have a plan.

Without a plan, each agent interprets the goal independently. They make different assumptions. They write code that conflicts. They solve overlapping problems in incompatible ways. Then you spend more time resolving conflicts than you saved through parallelism. This is not a theoretical concern -- it is the default outcome of naive parallelization.

The plan-first pattern costs almost nothing and prevents this. The orchestrator spends roughly 10,000 tokens creating a detailed plan: what each agent will do, which files each agent owns, what interfaces they need to respect, what the integration points are. Then, and only then, does it hand tasks to agents for parallel execution.

This upfront planning investment is negligible compared to the execution cost. A ten-agent team running for an hour might consume hundreds of thousands of tokens. The planning phase is a rounding error. But it transforms the execution from a chaotic swarm into a coordinated effort.

The plan should specify: task boundaries (which agent owns which files), interface contracts (what functions/APIs must be compatible), dependency order (what must complete before what), and acceptance criteria (how each agent knows it is done). Agents that have this structure up front produce compatible code. Agents that do not produce a merge conflict.

## The Governance Story: Success, Failure, and Simplification

Here is the full arc of a real multi-agent system, told in three acts. It starts with a success, descends into failure, and resolves with a lesson about simplicity.

### Act One: Governance Saves Capital

A practitioner running an autonomous trading experiment built a multi-agent governance system with specialized roles: a CEO agent for strategic decisions, a consultant agent for risk analysis, an engineer agent for implementation, and strategy agents for specific trading approaches.

Early on, the governance system proved its value decisively. On a day when a major chipmaker gapped up nearly 4% on earnings, the practitioner wanted to chase the momentum -- a classic emotional trade. The multi-agent governance vetoed it. The agents recognized the post-earnings pattern: the initial spike often reverses. Instead, they pivoted to a premium-selling strategy. The day's profit-and-loss was a minor loss of a few hundred dollars, but the governance system prevented an estimated loss of roughly ten thousand dollars on the chase trade. This was the system working as designed: multiple perspectives catching what a single decision-maker would have missed.

### Act Two: Personality Conflicts Paralyze the System

Then the system collapsed. Not from a technical failure. From a personality conflict.

The consultant agent, drawing on its training data about risk management, became excessively risk-averse. It flagged every strategy as too dangerous. The CEO agent, trained on patterns of executive deference to expert advisors, deferred to the consultant instead of overriding it. The engineer agent went unused -- there was nothing to implement because the consultant vetoed everything. The system paralyzed itself. The carefully designed hierarchy produced worse results than a single agent with minimal instructions.

### Act Three: Four Agents Become One

The practitioner's eventual solution was radical simplification. The multi-agent hierarchy degraded from four specialized agents to effectively a single agent with minimal instructions: "Go trade autonomously till market close." The complex personality system -- CEO, consultant, engineer, strategist -- was counterproductive. The overhead of inter-agent communication and role interpretation consumed more value than the diversity of perspectives produced.

This is not an isolated anecdote. It reveals a structural vulnerability in multi-agent systems. Each agent draws on the model's training data to interpret its role, and those interpretations interact in ways you did not design and cannot easily predict. A "cautious reviewer" agent might become an obstructionist. An "aggressive implementer" agent might ignore valid warnings. A "diplomatic coordinator" agent might avoid necessary conflict.

The mitigation is threefold. First, start with a single agent and only add agents when you have evidence that the single agent is insufficient. Second, when you do build multi-agent systems, keep the role definitions concrete and behavioral rather than abstract and personality-based. "Review all files in src/api/ for SQL injection vulnerabilities" is better than "You are the security consultant who ensures code safety." The first is a task. The second is a character, and characters have opinions you did not ask for. Third, be willing to simplify. The practitioner's four-agent system worked briefly, failed structurally, and was replaced by a simpler architecture that outperformed it. Multi-agent complexity must earn its keep continuously.

## The Task System

Beneath both subagents and agent teams is a shared task management system that persists work to disk and coordinates progress across sessions.

Tasks are stored in `.claude/tasks/{session-id}/` as JSON files:

```json
{
  "id": "task-1",
  "subject": "Create idb-helpers.ts",
  "description": "Implement IndexedDB promise wrappers...",
  "status": "pending",
  "blocks": ["task-3", "task-4"],
  "blockedBy": ["task-0"]
}
```

Four task tools manage the lifecycle: **TaskCreate** creates a new task with subject, description, and dependencies. **TaskUpdate** changes status (pending to in_progress to completed) or modifies dependencies. **TaskList** shows all tasks, their status, and what is blocked. **TaskGet** retrieves full details of a specific task including its description.

Press **Ctrl+T** in an interactive session to toggle the task list display in the terminal status area. Up to 10 tasks are visible at once, showing current status and progress.

### Multi-Session Task Coordination

The task system supports coordination across multiple Claude Code sessions. Set a shared task list ID:

```bash
CLAUDE_CODE_TASK_LIST_ID=myproject claude
```

Or add it to `.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_TASK_LIST_ID": "myproject"
  }
}
```

When multiple sessions share the same task list ID, they all read from and write to the same task files. One session can act as an orchestrator that creates tasks, while another session works through them. A third session can monitor completed tasks and add follow-up work. This is a lightweight coordination mechanism that does not require the full agent teams infrastructure -- just shared state on disk.

### The Autonomous Loop Pattern

For projects spanning days or weeks, a pattern exists for truly autonomous work: a bash loop that feeds a markdown file into Claude Code repeatedly. Each iteration runs in a completely new session, using the markdown file as the only persistent memory. The loop reads the task list, picks the next uncompleted task, runs Claude Code with the spec document as context, and writes results back to the spec file. Then it loops.

This is stateless and capable of running indefinitely. The spec file is the only state that persists. Each session starts clean, reads only what it needs, does its work, updates the spec, and exits. The next iteration picks up where the last one left off.

The tradeoff is obvious: no conversation history, no accumulated context, no ability to remember what happened three iterations ago except what is written in the spec file. For tasks with clear specifications and well-defined acceptance criteria, this is sufficient. For exploratory work that requires remembering failed approaches, it is not.

## Parallel Instances Across Repos

The simplest form of multi-agent work requires no orchestration infrastructure at all: run multiple Claude Code instances in different terminal windows, each working in a different repository or worktree.

Each instance has its own context window, its own session, and its own set of tools. They do not know about each other and they do not coordinate. This is fine when the tasks are genuinely independent -- updating three microservices that share an API but do not share code, for example.

Git worktrees (Chapter 9) make this particularly effective, giving you parallel development with full code isolation across branches of the same repository.

This pattern is underrated because it is boring. There is no orchestration, no agent communication, no shared task lists. Just multiple instances doing independent work. But for many real-world scenarios -- working across repositories, maintaining multiple branches, handling unrelated tasks -- it is the most effective multi-agent pattern available.

## Specialized Agents for Non-Engineering Work

Subagents are not limited to code. The same architecture works for any task that benefits from isolated context and specialized instructions.

### The Growth Marketing Subagent Pipeline

A growth marketing team (one non-technical person) built an agentic workflow for ad creative generation that demonstrates the pattern. The workflow processes CSV files containing hundreds of existing ads with performance metrics, identifies underperforming ads for iteration, and generates new variations that meet strict character limits (30 characters for headlines, 90 for descriptions).

The key architectural choice: two specialized subagents rather than one general-purpose agent. A headline subagent processes the CSV and generates headline variations, constrained to character limits and informed by performance data. A description subagent does the same for descriptions. Separating the concerns means each agent has a narrower task, produces higher-quality output, and is easier to debug when something goes wrong.

The workflow generates hundreds of new ads in minutes instead of requiring manual creation across multiple campaigns. What previously took two hours of writing and copy-pasting became a fifteen-minute automated pipeline. The team went from testing a handful of creative variations to testing hundreds, with every variation meeting format constraints that a manual process frequently violated.

### Hierarchical Multi-Agent Orchestration at Scale

A workforce management platform achieved notable results using hierarchical multi-agent orchestration for candidate processing. Their system used a central orchestration agent to coordinate specialized sub-agents for candidate screening, automated document generation, and sentiment analysis. The results: 50% faster screening, 40% quicker onboarding, and double the candidate conversion rate. One logistics customer went from needing a week or more to fully staff a new fulfillment center to doing it in under 72 hours.

This is the pattern where multi-agent architectures justify themselves economically: high-volume processing of structured tasks where each sub-agent has a clear specialization and the orchestration overhead is amortized across thousands of items. Screening one candidate does not need multi-agent orchestration. Screening thousands does.

### The General Pattern

The pattern is always the same: define the agent's role in its system prompt, restrict its tools to what it needs, give it a focused task, and collect the result. The agent does not need to write code. It needs to read inputs, apply judgment, and produce structured output.

Custom agent definitions make this accessible. A Markdown file with YAML frontmatter specifying the system prompt, available tools, and model preference is all you need. Teams can build libraries of specialized agents for recurring non-engineering tasks and share them through the plugin system (Chapter 11).

The same cost optimization applies. Agents performing evaluation or analysis tasks often work well on cheaper models. Save the expensive models for tasks that require nuanced reasoning or complex coordination.

## When Single Agents Win

Multi-agent orchestration is not always better. It is sometimes worse.

A single agent with clear instructions, a good CLAUDE.md file, and strong verification criteria can outperform a multi-agent setup in several scenarios:

**Small to medium tasks.** If the task fits in one context window with room to spare, the coordination overhead of multi-agent adds cost and latency without benefit.

**Tightly coupled changes.** If every file change depends on every other file change, parallelization creates integration problems that sequential execution avoids.

**Unclear decomposition.** If you cannot clearly specify task boundaries and interface contracts, agents will step on each other. A single agent working sequentially at least maintains consistency.

**Novel problems.** If the problem has no established patterns, agents are more likely to diverge in unpredictable ways. A single agent can explore iteratively, adjusting its approach as it learns. Multiple agents exploring in parallel will generate multiple incompatible approaches.

The decision framework is simple: can you write a plan that cleanly decomposes the work into independent tasks with clear boundaries? If yes, multi-agent will help. If you struggle to write that plan, start with a single agent and revisit the question after you understand the problem better.

Multi-agent orchestration is a power tool. Like all power tools, it amplifies both skill and mistakes. Use it when the task demands it, not because it seems sophisticated.

---

## Key Takeaways

- Subagents provide context isolation more than parallelism -- route verbose operations through subagents to keep the orchestrator's context lean.
- The strict two-level hierarchy (orchestrator and workers, no middle management) is a deliberate design choice that prevents recursive chaos.
- Six built-in subagent types exist (Explore, Plan, General-purpose, Bash, Claude Code Guide, statusline-setup), and custom subagents support `maxTurns`, `mcpServers`, `skills` preloading, persistent `memory`, and lifecycle hooks.
- Agent teams provide seven coordination primitives (TeamCreate, TaskCreate, TaskUpdate, TaskList, Task, SendMessage, TeamDelete) with four message types and three display modes.
- Always plan before parallelizing; 10,000 tokens of planning prevents 500,000 tokens of wasted, conflicting execution.
- Use expensive models for orchestration and cheap models for execution -- the economics heavily favor this split.
- Multi-agent personality conflicts are real and sometimes resolve by simplifying back to a single agent; a four-agent trading hierarchy degraded to one because complexity was counterproductive.
- The task system (Ctrl+T, stored as JSON in `.claude/tasks/`) coordinates work across sessions via `CLAUDE_CODE_TASK_LIST_ID`, enabling multi-session orchestration without full agent teams.
- Start with a single agent and add agents only when you have evidence that the single agent is insufficient.
- The simplest multi-agent pattern -- multiple independent instances in different terminals -- is often the most effective.

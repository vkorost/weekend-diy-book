# Chapter 3: Context Engineering

## What You'll Learn

Every feature in Claude Code traces back to one constraint: the context window. It is the CPU, the RAM, and the hard drive of your agentic workflow. Fill it with the wrong things and Claude forgets your instructions. Leave it empty and Claude wastes cycles rediscovering what you already know. The developers who get extraordinary results are not better prompters. They are better context engineers.

This chapter covers the system that makes context engineering practical: CLAUDE.md. You will learn what belongs in it, what does not, and why the distinction matters more than you think. You will see the exact context cost comparison for every mechanism -- CLAUDE.md, skills, MCP, subagents, and hooks -- so you can make informed budget decisions. You will understand how auto-compaction works, how to survive it with Compact Instructions and focused compaction, and how to use `/context` and `/rewind` for surgical context management. You will learn how to bootstrap CLAUDE.md with `/init`, extend it with `@path` imports, and suppress skill loading with `disable-model-invocation: true` for zero-cost manual-only skills.

You will also discover that CLAUDE.md is not just for code projects. The most compelling case study for institutional memory comes from a non-engineering domain: someone used Claude Code to build a professional-grade financial plan across multiple sessions, with CLAUDE.md accumulating data quirks, strategy decisions, and target allocations as project memory. The same pattern applies to any domain where work spans sessions and context must persist.

Most developers treat CLAUDE.md as a configuration file. The best ones treat it as a codebase's nervous system -- the place where hard-won knowledge accumulates so no one has to learn the same lesson twice.

---

## The Always-On File

CLAUDE.md is not documentation. It is not a README. It is instructions injected into every single request Claude processes in your project. That distinction matters because it means every line in CLAUDE.md has a cost: it consumes context window space on every turn of the conversation, reducing the space available for actual work.

Claude Code loads CLAUDE.md files from multiple locations in a strict hierarchy. A CLAUDE.md in your home directory applies to every project. One in your project root applies to that repository. Subdirectory CLAUDE.md files apply when Claude works in those directories. And if your organization deploys managed CLAUDE.md files to system directories, those apply to every user on the machine.

The loading is additive. Claude sees all of them, layered together. This means your home-level file should contain preferences that apply everywhere -- your preferred testing approach, your commit message style, your language of choice. Your project-level file should contain repository-specific knowledge. Subdirectory files should contain context relevant only to that part of the codebase.

### Bootstrapping with /init

If you are starting from scratch, the `/init` command does the first draft for you. It analyzes your codebase -- detecting build systems, test frameworks, code patterns, and directory structure -- and generates a starter CLAUDE.md with the basics already filled in. The output is not perfect, but it is a solid foundation you can refine over time. Think of `/init` as scaffolding: it gets the structure right so you can focus on the project-specific knowledge that only you have.

### The @path Import System

One mechanism makes this system scale: the `@path` import. Instead of inlining everything, you can write `@docs/architecture.md` in your CLAUDE.md to pull in external files. This keeps the root file lean while giving Claude access to deeper reference material.

The syntax supports several forms:

```
@README.md                              # relative to CLAUDE.md location
@docs/git-instructions.md               # subdirectory reference
@~/.claude/my-project-instructions.md   # home directory reference
```

The imported content still costs context, but the indirection makes your CLAUDE.md readable and maintainable. You can organize reference material in separate files -- API contracts, coding standards, architectural decision records -- and import only what is relevant to the current project.

### The 500-Line Guideline

The 500-line guideline exists for a reason. Past that threshold, instructions start competing with each other. Claude does not ignore a bloated CLAUDE.md -- it dilutes attention across too many directives, and the ones that matter most get the same weight as the ones that barely matter. A concise CLAUDE.md is not a nice-to-have. It is a performance requirement.

If Claude keeps doing something you do not want despite having a rule against it, the file is probably too long and the rule is getting lost. If Claude asks you questions that are answered in CLAUDE.md, the phrasing might be ambiguous. Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts. You can tune instructions by adding emphasis -- "IMPORTANT" or "YOU MUST" -- to improve adherence on critical rules.

## What Belongs -- and What Does Not

The question is not "what should Claude know?" It is "what can Claude not figure out on its own?" That filter eliminates most of what developers instinctively put in CLAUDE.md.

| Include | Exclude |
|---------|---------|
| Bash commands Claude cannot guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

**Belongs in CLAUDE.md:**

- Build and test commands. Your project's specific test runner invocation, build commands, and linting steps -- Claude cannot infer which commands your project uses without trying them and possibly failing.
- Code style deviations from standard conventions. If your team uses tabs instead of spaces, or if your Python project follows a non-PEP8 import order, say so. Claude will otherwise follow the standard.
- Repository etiquette. Which branches are protected. Whether you squash-merge or rebase. Whether PRs need a specific label format.
- Architectural decisions and their rationale. "We use a hexagonal architecture. Domain logic must never import from infrastructure." Claude can see your code structure, but it cannot see the design intent behind it.
- Environment quirks. "The test database runs on port 5433, not 5432." "CI uses runtime version 18, not 20." These are invisible traps that will burn time.
- Data quirks and known issues. "The `created_at` column in the users table has null values before March 2024 due to a migration bug." Claude will discover this eventually, but the discovery will cost you context and time.

**Does not belong in CLAUDE.md:**

- Things Claude infers from code. It reads your dependency manifests, your configuration files, your directory structure. Telling it which framework and language your project uses when that is obvious from the repository is a waste of context.
- Standard language conventions. Claude already knows common style guides and idiomatic patterns for major languages. Only document deviations.
- Verbose explanations of well-known libraries. Do not paste framework documentation into CLAUDE.md. Claude has extensive training data on popular tools.
- Step-by-step instructions for common workflows. Claude knows how to create a pull request, run a migration, or set up a test file. Tell it your project-specific variations, not the generic process.

The quality of your CLAUDE.md directly correlates with Claude Code's effectiveness. Teams that accumulate five or more sessions' worth of refined CLAUDE.md content for multi-day projects report dramatically better results than those starting from scratch each time.

## The Context Cost Comparison

Not everything you configure costs the same. Understanding the cost model is the difference between a session that stays sharp for 200 turns and one that compacts after 40. Here is the complete picture:

| Feature | When It Loads | What Loads | Context Cost |
|---------|--------------|------------|--------------|
| **CLAUDE.md** | Session start | Full content of all CLAUDE.md files | Every request |
| **Skills** | Session start + when used | Descriptions at start; full content when invoked | Low (descriptions every request)* |
| **MCP servers** | Session start | All tool definitions and schemas | Every request |
| **Subagents** | When spawned | Fresh context with specified skills | Isolated from main session |
| **Hooks** | On trigger | Nothing (runs externally) | Zero, unless hook returns additional context |

*By default, skill descriptions load at session start so Claude can decide when to use them. Set `disable-model-invocation: true` in a skill's frontmatter to hide it from Claude entirely until you invoke it manually with `/skill-name`. This reduces context cost to zero for skills you only trigger yourself.

### The Decision Matrix: Skill vs. Subagent vs. CLAUDE.md vs. MCP

Each mechanism serves a different purpose. The decision comes down to when the information is needed and how much it costs to keep available:

**Use CLAUDE.md when:** The information is needed on every turn. Build commands, style rules, architectural constraints, environment quirks. This is the always-on channel. Keep it under 500 lines.

**Use a skill when:** The information is needed for specific task types. Database migration procedures, deployment checklists, API integration guides, complex refactoring workflows. Skills load on demand -- their descriptions cost a small amount on every request, but the full content only loads when Claude or you invoke them.

**Use a subagent when:** The task involves reading many files, processing verbose output, or performing exploration that would bloat your main context. Subagents run in their own context window. Only the summary returns to your session. This makes subagents a context management strategy, not just a parallelism tool.

**Use MCP when:** You need structured access to external services. MCP tool definitions load at session start and persist, so every MCP server you configure has an ongoing cost. But the structured interface, permission integration, and hook visibility make it the right choice for sensitive or frequently-used external integrations.

**Use hooks when:** You need to extend Claude's behavior without consuming any context. Hooks run as external processes. A PreToolUse hook that validates file paths, a PostToolUse hook that logs actions, a SessionStart hook that sets up the environment -- none of these consume any context. This makes hooks the most context-efficient extension mechanism available.

Getting the classification wrong in either direction hurts. Put reference material in CLAUDE.md and you waste context on every request. Put always-needed rules in a skill and Claude will violate them whenever the skill is not loaded.

### Auditing Your Context Budget

This cost model has a practical implication: if you find yourself frequently running out of context, audit your always-on costs first. Run `/mcp` to see per-server token costs. Review your CLAUDE.md for content that could move to skills. Check whether you have MCP servers loaded that you rarely use. The `/context` command visualizes your current context usage as a colored grid, showing you exactly where the space is going.

## Skills vs. CLAUDE.md: The Loading Distinction

Both skills and CLAUDE.md provide Claude with instructions. The difference is when they load, and that difference governs your context budget.

CLAUDE.md content loads on every request. It is always present, always consuming context. This makes it the right place for information Claude needs constantly: build commands, style rules, architectural constraints.

Skills are on-demand. Their descriptions load at session start -- a short summary of what each skill does -- but the full skill content only loads when Claude invokes the skill. A skill containing 200 lines of database migration instructions costs you almost nothing until Claude actually needs to perform a migration. At that point, the skill content loads into the context for that specific interaction.

### Manual-Only Skills with disable-model-invocation

Some skills are not meant for Claude to decide when to use. A deployment skill, a release checklist, a complex refactoring workflow -- you want to invoke these yourself with `/skill-name`, not have Claude decide the moment is right.

Setting `disable-model-invocation: true` in a skill's frontmatter hides the skill from Claude entirely. It does not load the description at session start. It does not appear in Claude's available tools. The context cost is zero until you manually invoke it. This is the most aggressive context optimization for skills you use infrequently or that should only run when you explicitly choose.

```yaml
---
description: Deploy to production
disable-model-invocation: true
---
Follow the production deployment checklist...
```

The decision rule is simple. If Claude needs to know something on every turn -- "always use single quotes," "never modify files in /legacy" -- that goes in CLAUDE.md. If Claude needs reference material for a specific type of task -- detailed deployment procedures, complex refactoring checklists, API integration guides -- that is a skill. If only you should decide when to trigger the skill, add `disable-model-invocation: true`.

## Auto-Compaction: What Happens When Context Fills

At approximately 95% context capacity, Claude triggers automatic compaction. This is not a crash. It is a managed process, but understanding how it works matters because it determines what survives.

During compaction, Claude clears older tool outputs and summarizes the conversation history. The file contents it read twenty turns ago, the command outputs from earlier exploration, the intermediate reasoning -- all of it gets compressed into a summary. This frees context space but loses detail.

### Compact Instructions

One section of CLAUDE.md gets special treatment: anything under the heading "Compact Instructions." Content in that section is explicitly preserved through compaction. This is your opportunity to specify what Claude must remember even after the conversation history is compressed.

Use Compact Instructions for:

- Critical constraints that must never be violated, regardless of how long the session runs.
- The current task description, if it is complex enough that a summary might lose important nuances.
- Key decisions made earlier in the session that affect all subsequent work.

### Focused Compaction with /compact

You can trigger compaction manually with `/compact`, and you can direct what survives by adding focus instructions: `/compact Focus on the API changes`. This tells the compaction process which aspects of the conversation to prioritize when summarizing. Without focus instructions, compaction makes its own judgment about what matters -- which is often reasonable but occasionally drops the wrong things.

Manual compaction at 80% is often better than waiting for auto-compaction at 95%. You have more control, the compaction has more room to work with, and you can direct it while the conversation history is still relatively fresh.

You can also override the compaction threshold with the `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` environment variable. Setting it to 80 triggers compaction earlier, freeing space before the window is critically full. Setting it higher delays compaction but risks performance degradation in the final stretch before it triggers.

### Selective Summarization with /rewind

Sometimes you do not want to compact the entire conversation -- just a portion of it. Press Escape twice (or use the `/rewind` command) to open the rewind menu. A scrollable list shows each of your prompts as checkpoints. Select a message and you get four options:

1. **Restore code and conversation** -- revert everything to that point.
2. **Restore conversation only** -- rewind the conversation but keep the current code.
3. **Restore code only** -- keep the conversation but revert file changes.
4. **Summarize from here** -- condense everything from the selected point forward into a summary, keeping the conversation before that point at full fidelity.

The "Summarize from here" option is surgical compaction. If you spent thirty turns exploring a dead end and then found the right approach, you can summarize the exploration (freeing context) while preserving the productive work at full detail. This is far more precise than whole-conversation compaction.

### The PreCompact Hook

The PreCompact hook event fires before compaction begins, whether triggered manually or automatically. It receives the trigger type (`manual` or `auto`) and any custom instructions passed to `/compact`. While it cannot block compaction, it can perform preparation -- saving session state, logging context usage, or injecting context that should influence the compaction summary. This is the programmatic extension point for context management.

### The /context Command

The `/context` command visualizes your current context usage as a colored grid. It shows how much space is consumed by the system prompt, CLAUDE.md content, MCP tool definitions, conversation history, and active work. This is the diagnostic tool for context engineering -- when performance degrades, `/context` tells you why. If MCP servers are consuming 30% of your context before you ask a single question, you know what to fix.

The practical takeaway: if your sessions regularly hit compaction, you are either doing too much in one session or your always-on context costs are too high. Both are solvable.

## Session Forking as Context Recovery

When a session's context is polluted -- too many dead ends, too much exploration that is no longer relevant, auto-compaction that lost critical instructions -- you have a recovery option beyond starting fresh.

The `--fork-session` flag creates a new session that starts with the full history of an existing one but diverges from that point. The original session is preserved unchanged. The fork gets a new session ID and a separate conversation thread.

Use this when your session has valuable context you want to preserve but you need to take the work in a different direction. Instead of re-explaining the project setup, the decisions you have made, and the approach you are taking, you fork the session and start the new direction with all of that context intact.

```bash
claude --continue --fork-session
```

Session forking is not a substitute for good context management. It is an escape hatch for when the session's context has drifted from your needs despite your best efforts.

## CLAUDE.md for Non-Code Domains

CLAUDE.md is not just for software projects. The most compelling demonstration of institutional memory came from someone who used Claude Code for portfolio optimization -- a financial analysis project with no traditional codebase.

The workflow: export data from financial accounts as spreadsheets, create a supplementary text file with information that had no structured export, draft a detailed goal prompt, and initialize a Claude Code session pointing at the directory containing all of it. Over multiple sessions, Claude parsed the data, classified holdings, identified gaps, and produced a phased optimization plan with tax impact analysis, fund recommendations, and before-and-after allocation grids.

The critical part: they built up a CLAUDE.md file that accumulated not code knowledge but domain analysis memory. Data quirks discovered during parsing. Strategy decisions made in earlier sessions. Target allocations agreed upon after iterative refinement. When they returned days later, Claude picked up exactly where they left off because CLAUDE.md carried the project's analytical state.

This pattern applies to any domain where work spans sessions: research projects, data analysis, technical writing, infrastructure planning, compliance review. The CLAUDE.md content changes -- instead of build commands and coding conventions, it contains data schemas, analytical frameworks, and domain-specific constraints. But the mechanics are identical: persistent context that makes every session start smarter than the last.

### The Work Compounds

The same practitioner pointed to a pattern that deserves its own name: work product compounding. After completing the financial plan, they pointed Claude Code at the same project directory -- the session history, the plan documents, the analytical artifacts -- and asked it to write a blog post about the process. The deliverable from the first project became the input for an entirely different deliverable.

This is not an accident. It is a consequence of file-system-based context. Every artifact Claude produces -- plans, analyses, reports, code -- lives as a file in your project directory. That file is available to future sessions, to future skills, to future subagents. The work you do with Claude Code is not trapped in a conversation transcript. It accumulates in the filesystem, and each new session can build on everything that came before.

## Persona-Adaptive CLAUDE.md

The product design team at Anthropic discovered something that software engineers might not think of: CLAUDE.md works differently when the user is not a developer.

Designers on the team created custom CLAUDE.md files that told Claude they were designers with limited coding experience who needed detailed explanations and smaller, incremental changes. This dramatically improved the quality of Claude's responses -- instead of terse technical output that assumed engineering fluency, Claude produced step-by-step explanations, commented its code more heavily, and made smaller changes that were easier to review.

This is persona-adaptive context. The CLAUDE.md does not just describe the project; it describes the user. A data scientist might specify that they prefer exploratory scripts with visualizations. A technical writer might specify that they need verbose commit messages and documentation alongside every change. A designer might specify that they need UI-focused explanations and visual diffs.

The implication: CLAUDE.md is not one-size-fits-all even within a team. Project-level CLAUDE.md files carry shared standards. User-level CLAUDE.md files in the home directory carry personal preferences. The combination means Claude adapts to both the project's needs and the individual's working style.

## Continuous Improvement Loop

One pattern from a data infrastructure team at Anthropic extends the end-of-session update beyond accumulating knowledge. They ask Claude to summarize completed work and suggest improvements -- not just additions to CLAUDE.md, but improvements to the workflow itself.

The distinction is subtle but important. Most teams use end-of-session updates to add knowledge: "the test database uses port 5433" or "module X has a circular dependency." The continuous improvement pattern also captures process improvements: "the deployment script should run lint before build" or "the CLAUDE.md instructions for the API module are causing Claude to generate overly verbose error handlers -- simplify them."

This creates a feedback loop where CLAUDE.md evolves not just in what it knows but in how it instructs. Over time, the instructions themselves get refined based on observed results. The team reported that this loop -- updating both knowledge and process after each session -- made subsequent iterations measurably more effective because Claude was not just working from better knowledge but from better instructions.

## Institutional Memory

CLAUDE.md becomes dramatically more valuable when you treat it as a living document that accumulates knowledge across sessions.

Here is the pattern. You start a project with a minimal CLAUDE.md -- build commands, style rules, a few architectural notes. During your first session, Claude discovers that your test suite requires a specific environment variable. It learns that one module has a circular dependency it needs to work around. It figures out that the API client throws a non-standard error format.

All of that knowledge exists in the session context. When the session ends, it vanishes.

The fix is an end-of-session update loop. Before closing a session, ask Claude to review what it learned and suggest additions to CLAUDE.md. Claude will propose specific, concrete additions based on the problems it encountered. You review them, accept what is useful, and commit the updated CLAUDE.md.

Over time, CLAUDE.md accumulates the kind of knowledge that usually lives in a senior engineer's head: the quirks, the workarounds, the "here is why we do it this way" explanations that no one ever writes down. Except now they are written down, and every future session -- by you or any team member -- benefits from them.

This creates a compounding effect. Each session starts with more knowledge than the last. Mistakes that burned time in session one never recur because they are documented in CLAUDE.md. The project-specific knowledge base grows, and with it, Claude's effectiveness.

Teams that check CLAUDE.md into version control multiply this effect across the entire team. One developer's discovery becomes everyone's context.

## Preventing Repeated Errors

One specific application of institutional memory deserves its own section because it solves a specific, common frustration: Claude making the same mistake repeatedly.

Internal teams at Anthropic have documented a pattern where Claude Code makes particular tool-calling errors or follows incorrect patterns consistently within a project. The fix is surgical: add a targeted instruction to CLAUDE.md that addresses the exact error.

For example, if Claude keeps importing from the wrong module path, add: "When importing database utilities, use `@app/db/utils` not `@app/utils/db`. The latter path exists but is deprecated."

If Claude keeps generating tests that fail because of a timing issue: "All async tests in this project must use the `waitForCondition` helper, not `setTimeout`. The CI environment has non-deterministic timing."

These targeted additions are more effective than general instructions because they address specific failure modes Claude has demonstrated. They are not aspirational guidelines -- they are patches for observed bugs in Claude's behavior within your project.

## Spec-Driven Context

CLAUDE.md is persistent but static between manual updates. For complex, multi-session projects, there is a complementary approach: spec documents.

A spec document is a detailed description of what you are building, written before implementation begins. It lives as a file in your repository -- `spec.md`, `design.md`, whatever naming fits your project. You reference it from CLAUDE.md with an `@path` import or tell Claude to read it at the start of each session.

The power of specs is that they survive context pollution and session restarts. When a session compacts and loses the nuances of your earlier discussion, the spec is still there, unchanged, in a file Claude can re-read. When you start a new session, the spec provides full context without you re-explaining anything.

Spec-first development also produces a concrete artifact you can review before any code is written. You can ask Claude to write the spec, review it, refine it through conversation, and only then move to implementation. The spec becomes the source of truth that keeps implementation aligned across multiple sessions, multiple subagents, and multiple days of work.

This pattern matters most for projects that span more than two or three sessions. Without a spec, each new session starts with a lossy re-explanation of intent. With a spec, each session starts with a precise, version-controlled description of the goal.

## AGENTS.md: The Cross-Agent Standard

CLAUDE.md is specific to Claude Code. But if your team uses multiple AI coding tools -- or if contributors to your repository use different agents -- there is a complementary standard worth knowing about.

AGENTS.md is an open standard for agent-specific documentation. It lives in your repository root and works with over twenty different coding agents. While a README targets humans, AGENTS.md contains the extra context that coding agents need: development environment setup, coding standards, testing procedures, and architectural constraints.

The practical overlap with CLAUDE.md is significant, and the key insight is about information architecture. A bloated AGENTS.md (or CLAUDE.md) that tries to contain everything -- sometimes running to hundreds of lines -- consumes enormous context. The lean approach uses progressive disclosure: a concise root file with a docs-reference table that points to detailed documentation Claude can read on demand. Instead of inlining your API documentation, you reference it. Instead of pasting your architecture guide, you import it.

```markdown
# AGENTS.md

## Dev Environment
- How to set up and navigate

## Standards
- Code style, naming, patterns

## Testing
- How to run and write tests

## Docs Reference
| Topic | File |
|-------|------|
| API contracts | docs/api.md |
| Architecture | docs/architecture.md |
| Deployment | docs/deploy.md |
```

The lean version loads fast, costs minimal context, and lets the agent pull detailed documentation only when it needs it. This is the same principle as the CLAUDE.md 500-line guideline, applied to the broader agent ecosystem.

## Context Isolation with Subagents

Subagents are not just about parallelism. They are a context isolation strategy.

When Claude invokes a subagent, the subagent runs in its own context window. It reads files, runs commands, processes data -- all in isolation. Only the summary returned to the parent costs main-window context. A subagent that reads fifty files and runs twenty commands consumes enormous context in its own window, but your main conversation never sees any of it.

The practical impact is dramatic. In one documented case, a developer orchestrated fourteen subagents over the course of a complex migration project. Each subagent completed its task, committed its changes, and returned a brief summary. After all fourteen were done, the main session's context usage was at 143,000 tokens out of a 200,000 token window -- seventy-one percent. The system prompt and tool definitions accounted for roughly ten percent. The orchestration overhead -- creating tasks, tracking progress, synthesizing results -- consumed the rest. None of the actual implementation work touched the main context.

This means a skill that reads twenty files and runs complex analysis does not consume twenty files' worth of main context. Only the final result comes back. Skills running in their own context fork (`context: fork`) work the same way.

The implication for design: if you have a complex workflow that requires reading many files or processing extensive data, packaging it as a skill or delegating it to a subagent is more context-efficient than running the same steps directly in the main conversation. The main conversation stays lean. The heavy lifting happens in isolation.

You should also think carefully about what a skill or subagent returns. A skill that returns its entire analysis as a giant block of text defeats the purpose of isolation. Design outputs to be concise summaries that give the main conversation what it needs without the baggage of how the result was produced.

## Dynamic Context with SessionStart Hooks

CLAUDE.md is static -- it changes only when someone edits it. But some context is inherently dynamic: the current state of the CI pipeline, the list of open issues, recent commits by other team members, the current branch's relationship to main.

SessionStart hooks solve this. A hook configured on the `SessionStart` event runs an external command when Claude Code launches and injects the output as context. Unlike CLAUDE.md content (which costs context every turn), hook output is injected once and treated as part of the session initialization.

Practical examples:

- A script that checks for open pull requests on the current branch and summarizes review comments.
- A command that lists the most recent commits on main since the current branch diverged, so Claude knows what has changed.
- A script that reads environment-specific configuration and injects it as context, adapting Claude's behavior for development versus staging versus production environments.

SessionStart hooks complement static CLAUDE.md. The CLAUDE.md provides unchanging project knowledge. The hooks provide point-in-time state. Together, they give Claude both the permanent rules and the current situation.

## Putting It All Together

Context engineering is not a single technique. It is a system of interlocking decisions:

1. **CLAUDE.md** carries the permanent, always-needed knowledge. Keep it under 500 lines. Focus on what Claude cannot infer. Bootstrap with `/init`, extend with `@path` imports, and refine through end-of-session updates.
2. **Skills** hold reference material that loads on demand. Use them for task-specific instructions that would waste context if loaded constantly. Set `disable-model-invocation: true` for skills you only invoke manually.
3. **Hooks** inject dynamic context at zero ongoing cost. Use SessionStart hooks for environment-specific state. Use PreCompact hooks to prepare for compaction.
4. **Specs** persist complex project intent across sessions and compactions. Use them for anything that spans more than two sessions.
5. **Subagents** isolate expensive operations from your main context. Delegate verbose exploration and analysis to keep your primary conversation sharp. Fourteen subagents can run without exhausting the main session's context.
6. **Compact Instructions** in CLAUDE.md protect critical rules through auto-compaction. Use `/compact` with focus instructions for directed compaction, and `/rewind` with "Summarize from here" for surgical context cleanup.
7. **End-of-session updates** turn CLAUDE.md into a knowledge accumulator that improves with every session. Extend this to a continuous improvement loop by having Claude suggest workflow improvements, not just knowledge additions.
8. **Persona-adaptive CLAUDE.md** in home directories lets non-developers get appropriately detailed explanations and incremental workflows tuned to their expertise level.

The developers who master this system do not work harder. They start every session with better context than the session before. Their Claude Code instances know more about their projects than most human team members do, because the knowledge is written down, versioned, and loaded automatically.

That is context engineering. Not prompt tricks. Infrastructure.

---

## Key Takeaways

- CLAUDE.md content costs context on every request; keep it under 500 lines, bootstrap with `/init`, and focus on information Claude cannot infer from your code.
- The context cost comparison table is your budget guide: CLAUDE.md and MCP cost every request, skills cost low (descriptions only), subagents are isolated, and hooks are free.
- Put always-needed rules in CLAUDE.md and task-specific reference material in skills; use `disable-model-invocation: true` for manual-only skills with zero context cost.
- Use `@path/to/import` syntax to keep CLAUDE.md lean while giving Claude access to deeper reference material in separate files.
- Compact Instructions in CLAUDE.md survive auto-compaction; `/compact` with focus instructions directs what gets preserved; `/rewind` with "Summarize from here" enables surgical context cleanup.
- The `/context` command shows where your context budget is going -- audit it before blaming Claude for running out of space.
- CLAUDE.md works for non-code domains: financial analysis, data science, technical writing -- any domain where work spans sessions and context must persist.
- Subagent context isolation is dramatic: fourteen subagents can run without exhausting the main session because their work stays in separate context windows.
- End-of-session CLAUDE.md updates create compounding institutional memory; extend this to a continuous improvement loop by capturing workflow improvements alongside knowledge.
- Targeted CLAUDE.md additions that address specific observed errors are more effective than general instructions.
- Spec documents survive context compaction and session restarts, making them essential for multi-session projects.
- Session forking (`--fork-session`) preserves valuable context while letting you take work in a new direction.

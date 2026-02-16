# Chapter 9: Working with Large and Legacy Codebases

## What You'll Learn

Small, well-structured projects are easy. Claude Code reads the whole thing in minutes, understands the architecture, and starts producing useful work immediately. The real test is a monorepo with two million files, a legacy codebase with three frameworks and no documentation, or a system so large that no single person understands all of it.

These are the projects where Claude Code's value proposition is highest and where getting the approach wrong costs the most. This chapter covers the specific techniques for working with codebases that are too large to fit in context, too old to follow modern conventions, and too critical to rewrite carelessly. You will learn how git worktrees enable parallel work, how the Explore subagent navigates unfamiliar code efficiently, how task dependency management coordinates complex multi-step operations, and how teams have used Claude Code to compress three-year rewrites into weeks. You will also see how filesystem-driven analysis turns messy data exports into structured plans, how a storage layer migration demonstrates spec-driven refactoring at scale, and why the language barrier between modern developers and legacy systems written in decades-old languages is collapsing faster than anyone predicted.

If your codebase makes new hires cry, this chapter is for you.

---

## Git Worktrees for Parallel Work

The simplest scaling technique is also the most underused: git worktrees.

A worktree creates a separate working directory for a branch without cloning the entire repository again. Each worktree has its own checked-out files, its own index, and -- crucially -- can run its own independent Claude Code session. The sessions do not share context. They do not interfere with each other. They operate on the same repository but in completely separate directories.

```
git worktree add ../feature-auth feature/auth
git worktree add ../feature-payments feature/payments
```

Now you can run Claude Code in `../feature-auth` working on authentication while simultaneously running another instance in `../feature-payments` working on the payment system. Both sessions have full access to the repository history, but their file changes, context windows, and conversation histories are completely independent.

This matters for large codebases because the biggest bottleneck is often sequential work. You cannot run two Claude Code sessions in the same directory on different branches -- they would step on each other's files. Worktrees eliminate that constraint.

The pattern scales linearly. Three branches, three worktrees, three sessions. The limit is your machine's resources and your API budget, not anything architectural. For teams working on a monorepo with multiple independent features in flight, worktrees turn one Claude Code experience into several parallel ones.

One practical detail: each worktree gets its own `.claude/` directory if one exists in the repository, so project-level CLAUDE.md and settings apply independently. This means CLAUDE.md content (as covered in Chapter 3) compounds across worktrees without conflict.

## File Suggestion Customization for Monorepos

Claude Code's `@` autocomplete suggests files as you type, which works well for typical repositories. For large monorepos, the default file indexing can be slow or miss files deep in nested directory structures.

The solution is custom indexing. You can provide a script that generates the file suggestions Claude Code offers when you type `@`. This script can implement any logic: prioritize recently modified files, exclude build artifacts, index only the packages you care about, or provide a completely custom ordering.

For a monorepo with hundreds of packages, a custom indexing script that filters to the packages relevant to your current work can transform the `@` experience from overwhelming to precise. Instead of scrolling through thousands of files, you see the twenty that matter.

This is a small feature with outsized impact. In a ten-thousand-file monorepo, being able to quickly reference the right file saves context (you point Claude at the right code instead of letting it search) and saves time (you skip the exploration phase when you already know where to look).

## The Explore Subagent

When you do not know where to look, the Explore subagent is purpose-built for navigation.

Explore is a read-only, low-cost subagent (its architecture is covered in Chapter 4). It cannot modify files or run commands that change state. It can only read, search, and report back. This makes it fast, cheap, and safe to deploy even in codebases where you are still learning the terrain.

The Explore subagent runs in its own context window, which means it can read dozens of files during its investigation without consuming your main conversation's context. It returns a summary. You get the answer without the cost of the journey.

Explore also supports configurable thoroughness. For a quick question about where a function is defined, it searches narrowly and returns fast. For a comprehensive understanding of how a module works, it searches broadly, follows cross-references, and produces a detailed analysis. The thoroughness parameter lets you trade speed for depth.

Use Explore liberally in large codebases. The cost is low -- it runs on the cheapest available model, and the results run in an isolated context. The alternative is using your main session to read files one by one, filling your context window with exploration that could have been delegated.

## Parallel Research Subagents

Explore is a single investigator. For understanding a large system, you want a research team.

Multiple subagents can run in parallel, each investigating a different aspect of the codebase. One reads the authentication module. Another examines the database access layer. A third reviews the API routing. Each returns a focused summary. The main conversation orchestrates the investigation and synthesizes the results.

This pattern is particularly effective for onboarding onto an unfamiliar system. Instead of spending hours reading code sequentially, you dispatch parallel investigators and get a multi-perspective understanding of the system in minutes. The cost is higher than a single Explore call -- you are running multiple subagents -- but the time savings are substantial, and the context isolation means your main window stays clean (as covered in Chapter 4).

The research team pattern also works for impact analysis before making changes. Before modifying a shared utility function, dispatch subagents to investigate every module that imports it. Each subagent reports how the function is used in its area. You get a comprehensive impact assessment without manually tracing dependencies.

For massive codebases, this is not optional. No human can hold a multi-million-line codebase in their head. Parallel subagents give you a way to survey broad territory quickly, then focus your main session on the specific areas that matter.

## Task Dependency Management

Complex work on large codebases often involves tasks that depend on each other. You cannot update the API endpoints until the database schema is migrated. You cannot write integration tests until both the API and the database changes are complete. You cannot deploy until the tests pass.

Claude Code's task system supports explicit dependency declarations using `blocks` and `blockedBy` relationships. This enables wave-based execution: tasks with no dependencies run first, tasks that depend on them run next, and so on until everything is complete.

The pattern works like this:

1. Plan the work by breaking it into discrete tasks with clear boundaries.
2. Declare dependencies between tasks. Task A blocks Task B. Task C is blocked by both A and B.
3. Execute. Claude Code (or an agent team) processes tasks in dependency order, running independent tasks in parallel and dependent tasks sequentially.

For a large migration -- say, moving from one ORM to another across fifty modules -- this turns a manual coordination nightmare into a structured execution plan. The first wave migrates the core database layer. The second wave updates the modules that depend on it. The third wave updates the tests. Each wave can run with parallelism within it, but the waves themselves execute sequentially.

This is infrastructure-grade project management, not clever prompting. It works because the dependency system is explicit and enforced, not because Claude Code is "smart enough to figure out the order." You define the structure. Claude Code executes it.

## Filesystem-Driven Analysis

Legacy codebases come with legacy data. Directories full of CSV files with inconsistent formatting. Configuration files in three different formats. Log files from systems that no longer exist. Documentation that was last updated two years ago.

Claude Code handles this well because it treats the filesystem as a first-class data source. Point it at a directory of messy data and give it an objective -- normalize these files, find duplicates, extract a summary, build a migration plan -- and it will autonomously parse, compare, and organize the contents.

This works because Claude Code's tool access includes file reading, directory listing, and command execution. It can write scripts to process files, run those scripts, inspect the output, and iterate. It is not limited to understanding one file at a time. It can build a mental model of a collection of files and operate on the collection as a whole.

For legacy systems, this capability unlocks a specific workflow: understanding before rewriting. Before you touch a single line of code, point Claude Code at the system's configuration, data, and documentation directories. Let it build a picture of how the system actually works, not how someone once said it worked. That understanding -- exported as a spec document or CLAUDE.md additions -- becomes the foundation for any subsequent rewrite.

### The Three-Input Workspace Pattern

The most effective filesystem-driven analysis follows a specific setup that one practitioner documented while building a portfolio optimization plan that would have cost thousands of dollars from a professional advisor. The pattern generalizes beyond finance to any domain where messy data needs structured analysis.

The workspace starts with three inputs:

1. **Raw data exports.** CSV files, database dumps, configuration exports -- whatever the system produces natively. These files are typically messy: inconsistent column names, encoding issues, overlapping data across exports, non-standard line items. That mess is the point. You are feeding Claude Code the reality of the system, not a sanitized version.

2. **A supplementary text file.** There is always information that has no export. Institutional knowledge, undocumented settings, manual overrides, context that exists only in someone's head. Type it into a plain text file as a bulleted list. It does not need structure. It needs the raw facts.

3. **A detailed goal prompt.** This is the most important piece. It describes what data exists and where, the desired output format, your preferences and principles, known constraints, what you suspect is wrong (with room for Claude to disagree), and any strawman proposals that give the model something concrete to react to rather than building from a blank slate.

The workflow then proceeds through a predictable sequence. Initialize a Claude Code session pointed at the directory containing all three inputs. Claude autonomously writes scripts to parse the data files -- handling encoding issues, deduplicating overlapping exports, classifying items into target categories, and dealing with non-standard entries that would otherwise be misclassified. It iterates through problems as it encounters them. The baseline analysis it produces -- a full gap analysis, a structured summary, a comparison against targets -- is typically strong enough to work from immediately.

From there, the interaction shifts to iteration. You clarify constraints, reframe assumptions, ask Claude to self-critique its own output. Each clarification propagates instantly through every calculation, table, and recommendation in the plan. The supplementary text file and CLAUDE.md accumulate the decisions and data quirks discovered along the way, so subsequent sessions pick up exactly where the previous one left off.

Create a CLAUDE.md file early in the process and have Claude update it at the end of each session with learnings -- data quirks discovered, strategy decisions made, target parameters refined. This turns multi-session analysis into a compounding process where every future session builds on everything that came before. One practitioner described the result as equivalent to what would have been a week of spreadsheet work or several thousand dollars of professional advisory services, produced instead over a few evenings of back-and-forth with an AI agent.

## The Language Barrier Disappears

There is a class of legacy system that most modern developers refuse to touch: the ones written in languages they have never used. Mainframe systems running business logic in languages from the 1950s and 1960s. Scientific computing infrastructure maintained in numerical languages from the same era. Domain-specific languages that predate the internet. These systems run payroll, air traffic control, financial settlement, and nuclear simulations. They are not going away, and for decades, the shrinking pool of engineers who understood them was the bottleneck for any maintenance or modernization effort.

Agentic coding tools are dissolving that barrier. Support is expanding to less-common and legacy languages -- not as an afterthought, but as a natural consequence of how large language models work. The model has seen these languages in its training data. It can read them, understand their idioms, and translate their logic into modern equivalents. A developer who has never written a line in any of these legacy languages can point Claude Code at a directory of source files and get a working explanation of the business rules encoded in them.

This matters for large-codebase work because many legacy systems are also enormous systems. A financial institution's core processing logic might span hundreds of thousands of lines of code written over four decades. The traditional modernization approach -- hire a dwindling pool of specialists at escalating rates, then manually translate each module -- takes years and costs millions. The Claude Code approach is fundamentally different: use the model to read the legacy code, extract the business logic, and generate equivalent modern implementations. The specialist's role shifts from writing code to reviewing translations and validating business logic preservation.

The practical implications are significant. Technical debt that accumulated for years because no one had time to address it -- or no one who understood the language was available -- becomes systematically eliminable. Agents can work through backlogs of legacy maintenance tasks that were previously non-viable. An organization that had a three-year queue of modernization work might clear it in months, not because the agents are faster at writing the same code, but because the language barrier that made the work impossible for 95% of their engineering staff no longer exists.

## Legacy Codebase Rewrites

The most dramatic Claude Code case studies involve legacy rewrites. One documented case involved a trading platform frontend that had taken three years to build manually. Using Claude Code, the entire frontend was rewritten in weeks.

The timeline compression is real, but the pattern behind it matters more than the headline number.

Legacy rewrites succeed with Claude Code when three conditions hold:

1. **The desired architecture is well-defined.** Claude Code needs to know what you are building toward, not just what exists today. A clear target architecture -- in a spec document, in CLAUDE.md, in detailed prompts -- gives Claude Code a destination.

2. **The existing code is the specification.** Legacy code is often the only accurate documentation of business logic. Claude Code can read the existing implementation and extract the rules it encodes, even when those rules are buried in spaghetti code. It does not need clean code to understand behavior.

3. **The rewrite is modular.** Rewriting an entire system in one shot is as risky with Claude Code as it is manually. The effective pattern is module-by-module migration: rewrite one component, test it against the legacy behavior, validate, move to the next. Each module is a bounded task that fits in a single session.

The three-year-to-weeks compression comes from parallelism (multiple modules rewritten simultaneously via worktrees or subagents), from Claude Code's speed at producing code once the architecture is clear, and from the elimination of the knowledge acquisition phase that dominates manual rewrites. Claude Code reads the legacy code as fast as it reads anything else. The six months a human team might spend understanding the existing system before writing a line of new code shrinks to hours.

## Autonomous Implementation at Scale

At the extreme end of the spectrum, Claude Code has been used for autonomous implementation on codebases exceeding twelve million lines of code. One documented case involved implementing a feature across a codebase of that scale in approximately seven hours.

Seven hours. Twelve and a half million lines. One feature. No human writing code. And 99.9% numerical accuracy compared to the reference implementation.

That last number is the one that matters most. Speed without accuracy is just fast failure. The implementation was not a rough approximation that needed human cleanup. It achieved near-perfect fidelity to the expected results, autonomously, across a codebase spanning multiple programming languages.

This is not typical usage. It represents the ceiling of what is possible with current tooling. But the pattern it demonstrates is instructive: Claude Code's effectiveness on large codebases is not limited by the codebase's size. It is limited by the clarity of the task definition and the quality of the context provided.

A twelve-million-line codebase does not fit in any context window. What fits is the task description, the relevant subset of files, and the architectural knowledge encoded in CLAUDE.md. Claude Code uses its tools -- file search, grep, directory traversal -- to find the relevant code, understand it, and modify it. The vast majority of the codebase is never loaded into context. It does not need to be.

This selective-loading approach is how large codebases become tractable. Claude Code does not need to understand the entire system. It needs to understand the part of the system relevant to the current task. The Explore subagent, file search tools, and grep handle the navigation. The main session handles the implementation. CLAUDE.md provides the architectural guardrails.

## Codebase Navigation for Onboarding

One of the highest-value applications of Claude Code on large codebases has nothing to do with writing code.

New team members joining a large project traditionally face weeks of onboarding. They read documentation that is partially outdated. They ask colleagues questions that interrupt productive work. They explore code by opening files semi-randomly. They build a mental model slowly, through accumulation and osmosis.

Claude Code compresses this process dramatically. A new developer can start a session and ask questions about the codebase:

- "How does authentication work in this system?"
- "Where is the payment processing logic, and what external services does it call?"
- "What is the relationship between the Order and Fulfillment modules?"
- "Why does this project have two different database connection pools?"

Claude Code dispatches Explore subagents, reads relevant code, and provides answers grounded in the actual codebase -- not in documentation that might be stale. The answers reference specific files, specific functions, specific patterns. They are as current as the code itself.

This replaces data catalogs, architecture wikis, and colleague consultations for a significant portion of onboarding questions. It does not replace the human relationships and cultural context that onboarding also requires. But for "how does this code work?" questions, Claude Code is faster, more accurate, and more patient than any human mentor.

Teams that maintain a well-curated CLAUDE.md amplify this effect. The new developer gets both the code-derived answers from Claude Code's exploration and the institutional knowledge captured in CLAUDE.md. The combination provides an onboarding experience that would have required weeks of a senior engineer's time to deliver manually.

## Spec-Driven Refactoring: A Storage Layer Migration

Legacy rewrites and large refactors share a common failure mode: jumping straight to code without understanding the target well enough. The spec-driven approach inverts this by making research and specification the first phase, not an afterthought.

One practitioner documented a complete storage layer migration -- replacing a compiled-to-browser database with a native browser database for a sync engine. The old approach worked but had problems: large binary dependencies, poor performance on initial load, and complications with the framework's built-in sync capabilities. The migration touched fifteen or more files and required understanding an unfamiliar framework's storage patterns.

The workflow proceeded in distinct phases. First, research: a single prompt asking Claude to investigate the target framework spawned five parallel research subagents, each studying a different aspect -- the framework's data model, its sync protocol, its storage layer, its conflict resolution patterns, and its API surface. Each subagent returned a focused summary. The combined findings became a comprehensive research report that would have taken a developer days of documentation reading to assemble.

Second, specification: Claude synthesized the research into a detailed migration spec -- a structured document covering architecture decisions, a phased implementation plan, a fourteen-item task checklist, risk mitigation strategies, and success criteria. The spec was written to a file on disk, not held in conversation context. This is critical. A spec on disk survives context degradation, session restarts, and compaction. It is the persistent source of truth.

Third, refinement: before implementation, Claude asked clarifying questions about ambiguities in the migration strategy -- conflict resolution approaches, sync behavior during the transition, fallback handling. These questions surfaced design decisions that would have become bugs if discovered during implementation instead of during planning.

Fourth, execution: fourteen tasks, each delegated to a subagent, each producing an atomic commit. The subagents worked from the spec, not from accumulated conversation context. Each completed its bounded task, committed the result, and returned. Fourteen commits, fifteen-plus files changed, one pull request ready for review.

The entire migration took a single afternoon. The practitioner estimated it would have taken two to three days manually. But the time savings are secondary to the quality improvement: the research phase uncovered framework patterns that the developer would not have found on their own, resulting in a more idiomatic implementation than manual coding would have produced.

Despite orchestrating fourteen subagents, the main session's context stayed well within limits -- the orchestrator held only the task list and subagent summaries, while the actual file reading and code generation happened in isolated subagent contexts. This is the model for large refactors: research broadly, specify precisely, execute in parallel, and keep the orchestrator lean.

## CLAUDE.md as Data Catalog Replacement

One of the more unexpected applications of CLAUDE.md in large codebases has nothing to do with code style or build commands. One data infrastructure team discovered that their CLAUDE.md files could replace traditional data catalogs and discoverability tools for onboarding.

When new data scientists joined the team, they were directed to use Claude Code to navigate the massive codebase. Claude Code would read the CLAUDE.md files, identify relevant files for specific tasks, explain data pipeline dependencies, and help newcomers understand which upstream data sources fed into which dashboards. The information that traditionally lived in a separate data catalog tool -- and was perpetually outdated because nobody remembered to update it -- now lived alongside the code and was maintained as part of the normal Claude Code workflow.

The team reinforced this with a continuous improvement loop. At the end of each work session, they asked Claude to summarize the completed work and suggest improvements -- not just to the code, but to the CLAUDE.md documentation and the workflow instructions themselves. Each session refined the project's institutional knowledge. The CLAUDE.md file grew organically from actual usage rather than from a dedicated documentation sprint that would never be repeated.

This pattern works because CLAUDE.md is read at the start of every session with full fidelity. Unlike a wiki page that might be six months stale, CLAUDE.md is as current as the last session that updated it. For large codebases where the data model is as complex as the code itself -- where understanding which table feeds which dashboard through which intermediate transformation is essential knowledge -- CLAUDE.md becomes the living data catalog that traditional tools aspire to be but rarely achieve.

## Technical Debt at Scale

Every large codebase has a backlog of technical debt that nobody gets to. Not because it is unimportant, but because the economics never justified the investment. The refactor that would take three weeks of an experienced engineer's time saves two minutes per build. The migration that would modernize a deprecated dependency requires touching two hundred files. The test coverage improvement that would prevent the quarterly production incident needs someone who understands both the testing framework and the business logic.

Agentic coding tools change the economics of technical debt systematically. When agents can work autonomously for extended periods, formerly non-viable projects become feasible. The three-week refactor becomes a three-hour task with review. The two-hundred-file migration becomes a structured execution plan with parallel subagents (as described in the task dependency management section above). The test coverage improvement becomes a weekend of autonomous operation with human review on Monday.

The pattern is not "set an agent loose on the backlog." It is structured: identify the debt, write a clear specification, establish verification criteria (tests that must pass, linting rules that must be satisfied), and then let agents work through the items systematically. The backpressure of automated verification (Chapter 8) ensures that agent-generated fixes actually work. The task dependency system ensures that dependent changes execute in the right order.

Organizations that recognize this shift early gain a compounding advantage. Every piece of technical debt eliminated makes the codebase easier for both humans and agents to work with. Cleaner code produces better agent outputs. Better agent outputs clear more debt. The flywheel accelerates.

## Strategies for the Enormous and the Ancient

Working with large and legacy codebases is ultimately about managing two scarce resources: context and comprehension.

Context management for large codebases means aggressive use of subagents for exploration, selective file loading, and keeping your main conversation focused on the task at hand rather than on understanding the system. Do not read files into your main context that you could delegate to an Explore subagent. Do not explore broadly when you can search narrowly.

Comprehension management means building understanding incrementally and persisting it. Use spec documents and CLAUDE.md to capture what Claude Code learns about the system. Use task dependency management to structure complex migrations. Use parallel research to build understanding faster than any sequential reading could.

The codebases that seem impossible to work with are not impossible. They are just expensive to understand. Claude Code reduces that cost -- dramatically -- by reading faster, searching more broadly, and forgetting nothing that you write into CLAUDE.md.

---

## Key Takeaways

- Git worktrees enable parallel Claude Code sessions on different branches of the same repository, with no interference between sessions.
- The Explore subagent (Haiku, read-only, isolated context) is cheap enough to use liberally for navigating unfamiliar code.
- Parallel research subagents provide multi-perspective understanding of large systems in minutes instead of hours.
- Task dependency management with `blocks`/`blockedBy` enables wave-based execution for complex migrations and refactors.
- Legacy rewrites succeed when the target architecture is clear, the existing code serves as the specification, and the work is modular.
- Claude Code's effectiveness on large codebases is limited by task clarity and context quality, not by codebase size -- one documented case achieved 99.9% numerical accuracy across a 12.5-million-line codebase.
- The language barrier for legacy systems written in older languages is disappearing; agents can read, understand, and translate business logic that most modern developers cannot.
- The three-input workspace pattern (raw data exports, supplementary text file, detailed goal prompt) turns filesystem-driven analysis into a structured, repeatable process.
- Spec-driven refactoring -- research first, specify precisely, execute in parallel -- produces better results than jumping straight to code, even when the developer could have written it manually.
- CLAUDE.md can replace traditional data catalogs for onboarding, maintained through a continuous improvement loop at the end of each session.
- Technical debt becomes systematically eliminable when agents change the economics from "not worth three weeks of an engineer's time" to "three hours plus review."

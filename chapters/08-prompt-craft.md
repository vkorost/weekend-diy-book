# Chapter 8: Prompt Craft for Agentic Tools

## What You'll Learn

Prompting an agentic coding tool is not the same as prompting a chatbot. A chatbot generates text. An agentic tool reads files, writes code, runs tests, modifies your filesystem, and chains dozens of operations together. The prompt is not a question -- it is a work order. The difference between a prompt that produces a clean implementation and one that produces three hours of cleanup is not eloquence. It is engineering.

The single highest-leverage technique is not clever wording or elaborate system prompts. It is giving Claude verification criteria -- tests, expected outputs, screenshots to match -- so that the agent has a feedback loop instead of a guess. Everything else here matters, but nothing matters as much as that.

Beyond verification, the specific patterns that practitioners have discovered make agentic tools dramatically more effective: the delegation mindset, rich content input, interrupt-and-steer workflows, backpressure through tooling, spec-first prompting, and the counterintuitive discovery that your linting configuration is part of your prompt. These are not theoretical best practices. They are the patterns that separate fifteen-minute tasks from three-hour debugging sessions.

---

## Verification Criteria: The One Thing That Matters Most

If you take nothing else from these pages, take this: tell Claude how to verify its own work.

Without verification criteria, Claude's workflow is generate-and-hope. It writes code, decides it looks correct, and moves on. If there is a bug, you find it. If the output format is wrong, you notice it. If an edge case breaks, you discover it in production.

With verification criteria, Claude's workflow is generate-test-fix. It writes code, runs the verification, observes the result, and iterates until the verification passes. The feedback loop is automatic. You specified the success criteria; Claude chases them.

Verification criteria include:

- **Test commands**: "Run `npm test` after every change and ensure all tests pass."
- **Expected output**: "The function should return `[1, 4, 9, 16]` for input `[1, 2, 3, 4]`."
- **Visual checks**: Paste a screenshot of the target UI. "The result should look like this."
- **Behavioral descriptions**: "The API should return a 400 status code when called without an auth header."
- **Existing test suites**: "Run the full test suite and ensure no regressions."

The key insight is that verification criteria convert a one-shot generation into an iterative loop. Claude does not need to get it right the first time. It needs to get it right before it stops. Verification criteria define "before it stops."

### The Absence of Criteria

When you prompt "add user authentication to the API" without verification criteria, Claude generates an implementation that looks plausible. It may work. It may not. You are the verification step, and you might not catch subtle issues on review.

When you prompt "add user authentication to the API. Run `pytest tests/auth/` after implementation. All tests should pass. The endpoint `/api/protected` should return 401 without a token and 200 with a valid token" -- now Claude has a target. It will run the tests. It will hit the endpoint. It will iterate until the criteria are met or it gets stuck and tells you why.

The cost of adding verification criteria to a prompt is thirty seconds of typing. The cost of not adding them is measured in debugging time.

## Specificity vs. Vagueness

There is a time for vague prompts and a time for specific ones. Knowing which is which separates effective delegation from wasted cycles.

**Vague prompts are appropriate for exploration.** "What does the authentication flow look like in this codebase?" -- "How is error handling structured?" -- "What would it take to add internationalization?" These prompts leverage Claude's ability to read broadly and synthesize. You do not know the answer; you want Claude to discover it.

**Specific prompts are appropriate for implementation.** "In `src/auth/middleware.ts`, add rate limiting to the `validateToken` function. Use a sliding window of 100 requests per minute per IP. Store counts in the existing in-memory store connection from `src/config/cache.ts`. Add tests in `tests/auth/rate-limit.test.ts`."

The failure mode is using vague prompts for implementation work. "Add rate limiting" without specifying where, how, or to what gives Claude maximum freedom to make choices you will disagree with. It picks the wrong file. It invents a new cache connection instead of using the existing one. It stores state in local memory instead of the shared cache. Each assumption is reasonable in isolation and wrong for your codebase.

### Files, Constraints, Examples

Three categories of specificity that consistently improve outcomes:

**Reference specific files.** Use `@` mentions or explicit paths. "Look at `@src/utils/validation.ts` for the pattern we use." Claude does not need to search for the right file. You have already pointed it there.

**State constraints.** What cannot change. What trade-offs are acceptable. What flexibility exists. "Do not modify the database schema. Performance matters more than code readability here. You can add new files but do not restructure existing directories."

**Provide examples.** If you want output in a specific format, show the format. If you want code that matches a pattern, show the pattern. Claude extrapolates from examples better than it follows abstract descriptions.

## The Delegation Mindset

Prompting an agentic tool is not writing instructions for a script. It is delegating to a capable colleague. The distinction changes what you include in the prompt.

For a script, you specify every step: "open this file, find this function, add this line, save the file." For a colleague, you specify the goal and the context: "we need rate limiting on the auth middleware because we are getting hammered by a bot. The cache connection already exists. Use the same sliding window approach we used on the payment endpoint."

The colleague figures out the steps. The script executes yours. Claude Code is the colleague.

The delegation mindset changes three things about your prompts:

1. **Goal over steps.** State what needs to be true when Claude is done, not how to get there.
2. **Context over instructions.** Explain why this change matters, what exists already, what the constraints are.
3. **Trust over control.** Let Claude choose the implementation approach. Intervene when it chooses wrong, not preemptively.

This is uncomfortable for developers. We are trained to specify precisely. But over-specifying an agentic tool constrains it to your approach, which may not be the best approach. State the goal. Provide the context. Let the agent work.

## Rich Content Input

Claude Code accepts more than text prompts. Using the full range of input modalities consistently improves outcomes.

**File references with @.** Type `@` followed by a path to include file contents directly in context. This is cheaper and more reliable than asking Claude to read the file -- the contents are in the prompt, not fetched by a tool call.

**Images.** Paste screenshots of UI targets, error messages, design mockups, or whiteboard sketches. Claude interprets images and can implement toward a visual target. "Make the dashboard look like this" with a screenshot is more precise than three paragraphs describing the layout.

**URLs.** Claude can fetch and process web content. Point it at API documentation, style guides, or reference implementations. The content is retrieved and interpreted, saving you from copy-pasting.

**Piped input.** Claude Code is a Unix citizen. Pipe data into it: `cat error.log | claude "what's causing these failures?"` -- `git diff HEAD~5 | claude "summarize these changes"`. This connects Claude to your existing command-line workflow.

Each input modality adds context. Context costs tokens. But the right context -- a screenshot instead of a paragraph, a file reference instead of a description -- is cheaper and more effective than the alternative.

## Interrupt and Steer

Claude Code is not a batch process. You can interrupt it mid-task.

If Claude is heading in the wrong direction -- implementing a solution you do not want, using a library you do not use, modifying the wrong file -- type your correction and press Enter. Claude stops what it is doing, reads your correction, and adjusts course.

This is the real-time steering loop that makes agentic tools qualitatively different from one-shot generation. You are not waiting for the full output to review and reject. You are watching the work happen and redirecting in real time.

Effective interruptions are specific: "stop -- use the existing database connection in config/db.ts instead of creating a new one." Vague interruptions ("that's wrong") force Claude to guess what you object to.

The interrupt-and-steer pattern is most valuable during the first few minutes of a complex task, when Claude's initial approach reveals its assumptions. Correct the assumptions early and the rest of the implementation follows correctly.

## Role Assignment Prompts

Framing Claude's role changes its behavior. This is not mystical prompt engineering -- it is setting coordination expectations.

"You are the orchestrator. Your subagents are your development team. Assign each agent a clear, bounded task. Review their output before integrating." This framing causes Claude to delegate to subagents rather than implementing everything in the main context. It plans before acting. It reviews results.

Without the framing, Claude tends toward doing everything itself in a single context. For small tasks, that is fine. For multi-file refactors or cross-service changes, it leads to context exhaustion and dropped details. The orchestrator framing triggers a different work pattern.

Other effective role framings:

- **Investigator**: "Your job is to understand, not to change. Report what you find." Keeps Claude in read-only mode for exploration tasks.
- **Reviewer**: "Review this pull request as a senior engineer. Focus on correctness, security, and maintainability." Produces structured review output rather than code changes.
- **Specialist**: "You are a database migration specialist. Your only concern is schema changes and data integrity." Narrows focus and reduces tangential changes.

## Spec-Driven Development

Before asking Claude to write code, ask it to write a document. This is not just a prompting tip. It is a named workflow pattern -- Spec-Driven Development -- that contrasts sharply with the default AI coding pattern of prompt, code, debug, repeat.

The traditional pattern goes like this: you write a prompt, Claude generates code, you find bugs, you debug, you prompt again, you generate more code, you find more bugs. Context fills up with failed attempts. There is no persistent source of truth. No clear stopping point. Each iteration starts from scratch in the same cluttered context.

Spec-Driven Development follows a different flow: **Research, Spec, Refine, Tasks, Done.**

1. **Research.** Spawn multiple subagents to investigate the relevant parts of the codebase in parallel. Each subagent explores a different dimension -- the existing architecture, the API surface, the test patterns, the dependencies. The research results flow back to the main context.

2. **Spec.** Ask Claude to write a comprehensive specification based on the research. "Your goal is to write a report. Produce a technical spec for the migration covering architecture, data model, error handling, and rollback strategy. Do not write any code yet." The spec becomes a file in the repository -- a persistent artifact that survives context compaction and session restarts.

3. **Refine.** Before implementation, let Claude interview you about the spec using the AskUserQuestion tool: "Use the ask_user_question tool -- do you have any questions regarding the spec before we implement it?" Claude asks clarifying questions about ambiguities, design decisions, and edge cases. You answer. The spec is updated. This catches misunderstandings that would have become bugs.

4. **Tasks.** Break the spec into atomic tasks, each implemented by a subagent with a fresh context. "Use the task tool. Each task should only be done by a subagent. After each task, do a commit." Each subagent reads the spec, implements one piece, commits, and returns. The main context stays lean -- it is orchestrating, not implementing.

5. **Done.** Clear completion criteria, defined in the spec, determine when the work is finished.

### Five Prompt Patterns for Spec-Driven Development

The workflow relies on five specific prompt patterns that trigger the right behavior:

1. **"Spin up multiple subagents for your research task."** Triggers parallel research. Five subagents studying five aspects of the codebase simultaneously.

2. **"Your goal is to write a report/document."** Forces Claude to produce a written artifact before any code. This becomes the source of truth.

3. **"Use the ask_user_question tool before we implement."** Surfaces ambiguities and design decisions before they become bugs.

4. **"Use the task tool, and each task should only be done by a subagent. After each task, do a commit."** Forces atomic, independently verifiable units of work with commit boundaries.

5. **"You are the main agent and your subagents are your devs."** Triggers the orchestrator role framing that keeps the main context lean and delegation-focused.

### A Walkthrough: Storage Layer Migration

To make this concrete: one practitioner used Spec-Driven Development to migrate a sync engine's storage layer from one browser database technology to another -- a task that would have taken two to three days manually.

**Phase 1: Research.** Five subagents launched in parallel to study the target framework. Each investigated a different aspect: the data model, the sync protocol, the conflict resolution strategy, the indexing approach, and the cross-tab coordination mechanism. The parallel research completed in minutes, producing findings that would have taken hours of documentation reading.

**Phase 2: Spec creation.** Based on the research, Claude produced a comprehensive specification covering the migration strategy, the adapter interface, the data conversion logic, and the test plan. The spec went into a file in the repository.

**Phase 3: Refinement.** Claude asked clarifying questions using the interactive questioning tool: "How should the migration handle conflicts between the old and new storage formats? Should we support rollback to the old format? What is the expected behavior if a user has multiple tabs open during migration?" Each answer tightened the spec.

**Phase 4: Task delegation.** The spec broke down into fourteen tasks, each assigned to a subagent. Each subagent read the spec, implemented its task, ran the relevant tests, and committed. The git log showed fourteen atomic commits, each implementing one clearly bounded piece of the migration.

The entire workflow took approximately forty-five minutes. The resulting code was better than a manual implementation would have been, because the research phase uncovered patterns from the target framework that the developer would not have found on their own.

### The AskUserQuestion Tool

The refinement phase deserves special attention. When you tell Claude "use the ask_user_question tool before we implement," Claude enters an interview mode. It reads the spec and asks multiple-choice or open-ended questions about everything it is uncertain about. This is not the same as "ask me questions exhaustively" in freeform conversation. The AskUserQuestion tool provides structured, focused questions that are easier to answer and produce more actionable clarifications.

The interview pattern works well beyond spec refinement. Use it at the start of any complex task: "Before building anything, interview me about my requirements using the ask_user_question tool. Do not proceed until you have asked every question." The structured format surfaces assumptions that freeform conversation misses.

## The Spec as Persistent Truth

The spec file is not just a planning artifact. It is the recovery point for the entire workflow. If a subagent fails, the spec is still there -- start a new subagent on the same task. If context compacts, the spec survives as a file on disk. If you need to resume the next day, the spec tells the new session everything it needs to know.

One engineering team takes this further: they store specifications as markdown files in the codebase, written by Claude, reviewed by humans, and then executed by Claude. The specs go through code review like any other artifact. When the implementation is done, the spec remains as documentation of the design decisions. This is specification-as-code -- the spec is not a throwaway planning document but a versioned, reviewed, executed artifact.

## Backpressure as Prompt Engineering

This is one of the most valuable patterns practitioners have discovered, and it reframes how you think about project tooling entirely.

Your linting configuration is part of your prompt.

Not metaphorically. Literally. When Claude Code runs in a project with strict linting, comprehensive type checking, and a thorough test suite, it receives automated feedback on every change. The linter flags a style violation. The type checker catches a type error. A test fails. Claude reads the error, fixes the issue, and tries again.

This is backpressure. The tooling pushes back on incorrect output, and the agent self-corrects. The tighter the tooling, the better the output. A project with strict linting rules, strict type checking configuration, and high test coverage produces better Claude Code output than an identical project without those constraints -- even though Claude was never explicitly told about the rules.

The implication is architectural: investing in strict linting, comprehensive types, and thorough tests pays a double dividend. It catches human errors and it catches AI errors. The AI errors it catches are the ones you would have had to find manually.

You have a limited budget of feedback. Every time you manually tell Claude "fix the indentation" or "use semicolons" or "that variable should be const," you are spending your feedback budget on mechanical issues that tooling should catch automatically. Spend your feedback budget on architecture, design decisions, and domain logic -- the things linters cannot check. Let the type checker and linter handle everything they can handle.

Pre-commit hooks that run the test suite and linter create a particularly tight backpressure loop. Claude commits, the hooks run, failures are reported, Claude fixes and tries again. The human does not need to review every change. The tooling reviews it.

This reframes tooling investment. A linting configuration is not just a code quality tool. It is a component of the prompt that shapes every piece of code Claude generates in that project. Invest accordingly.

### TDD as Backpressure

The test-driven development workflow is the ultimate expression of backpressure. One engineering team described their old pattern as "design doc, janky code, refactor, give up on tests." With Claude Code, the pattern inverted. They ask Claude for pseudocode first, guide it through test-driven development, and periodically check in to steer when it gets stuck. The tests are written before the implementation. The implementation cannot pass until it satisfies the tests. Claude iterates until it does.

This is not TDD for philosophical reasons. It is TDD for pragmatic ones. When an agent writes both the tests and the implementation, the tests create the feedback loop that prevents the implementation from drifting. The agent writes a test, writes code to pass it, runs the test, sees the failure, adjusts, and tries again. The human checks in periodically to verify the tests themselves are testing the right things.

## Plan-Before-Execute

Asking Claude to plan before implementing is cheap insurance against expensive mistakes.

"Before making any changes, read the relevant files and produce a plan. List every file you will modify, every new file you will create, and every behavioral change. Wait for my approval before proceeding."

A plan costs roughly 10,000 tokens to generate. A misdirected implementation costs 500,000 tokens and a context reset. The economics are obvious.

Plan review also lets you catch misunderstandings before they become code. If Claude's plan includes modifying a file you consider untouchable, you catch it at the plan stage. If the plan omits a file that obviously needs changing, you add it before implementation begins.

A keyboard shortcut makes this even more practical: pressing `Ctrl+G` opens Claude's plan in your default text editor. You can edit the plan directly -- add steps, remove steps, reorder priorities, add constraints -- and when you save and close the editor, Claude proceeds with your modified plan. This is faster than describing modifications in conversation and more precise than trying to steer an existing plan through follow-up prompts.

The plan-before-execute pattern composes well with subagent delegation. Plan cheaply in the main context, then hand each plan item to a subagent for parallel execution. This is the most cost-effective multi-agent strategy, as discussed in Chapter 4.

You can also convert a plan into a shorter checklist for execution. After Claude produces a detailed plan, ask it to "remix the plan into a shorter checklist I can scan and check off." The checklist becomes the working document -- concise enough to keep in view while implementing, detailed enough to capture all the steps.

## Constraint Explicitness

Implicit constraints are invisible constraints. Claude cannot respect rules you have not stated.

Three categories of constraints to state explicitly:

**What cannot change.** "Do not modify the database schema." "Do not change the public API surface." "The file `core/engine.py` is frozen -- read it but do not edit it." These hard boundaries prevent Claude from taking shortcuts through things you consider sacrosanct.

**Trade-off preferences.** "Optimize for readability over performance." "Minimize the number of new dependencies." "Prefer simple solutions even if they are less elegant." These soft preferences guide Claude's choices when multiple approaches are viable.

**Flexibility boundaries.** "You can add new files in `src/utils/` but not in `src/core/`." "You can use any library already in package.json but do not add new ones." "Refactoring the test helpers is acceptable if it simplifies the implementation." These tell Claude where it has room to maneuver.

Unstated constraints are the most common source of disappointing output. Claude produces something correct that violates an assumption you did not realize you had. State the constraints. All of them.

## Output Format Specification

When you want structured output, describe the structure.

"Produce a comparison table with columns: Feature, Current Implementation, Proposed Change, Risk Level." -- "Format the analysis as a numbered list with each item containing: the finding, the severity, and the recommended fix." -- "Write the test plan as a checklist with checkboxes."

Claude follows format specifications reliably. Omitting them gets you whatever format Claude defaults to, which may or may not match what you need.

Format specification is especially important when Claude's output will be consumed by other processes or other people. A downstream tool expecting JSON will not appreciate freeform text. A colleague expecting a checklist will not appreciate an essay.

## Self-Critique Prompting

Asking Claude to evaluate its own work surfaces issues it did not mention during generation.

"Rate this implementation 1-10. What would you improve? What edge cases might break? What would a senior engineer object to?"

The self-critique step produces surprisingly useful output. Claude identifies missed edge cases, suggests performance improvements, flags potential security issues, and acknowledges shortcuts it took. Not all of its self-critique is actionable, but enough of it is to make the pattern worthwhile.

A variant: "What are the three most likely things wrong with this?" This focuses the critique on probable issues rather than hypothetical ones.

Self-critique is most valuable after a complex implementation, before you commit. It takes thirty seconds. It catches issues that would take thirty minutes to find in review.

## Exhaustive Questioning

When Claude fills in assumptions, the results are plausible but often wrong. The fix is to prevent the assumption-filling.

"Before you start, ask me questions exhaustively about anything you are uncertain about. Do not proceed until you have asked every question."

This prompt modifier causes Claude to surface its uncertainties before acting on them. What authentication system are you using? What database? What should happen on failure? Do you want error messages user-facing or logged? Is this a new endpoint or a modification of an existing one?

The questions reveal gaps in your own spec. You may not have thought about failure handling. You may not have considered which database to use. Claude's questions force you to make decisions you would otherwise discover as bugs.

Practitioners report that exhaustive questioning improves first-pass quality by 30-50% on complex tasks. The cost is a few minutes of answering questions. The benefit is an implementation that matches your actual requirements instead of Claude's assumptions.

## Strawman Proposals

Do not start from zero when you have a rough idea.

"Here is my rough approach: use a cache-backed sliding window with per-IP tracking, expire keys after 60 seconds, return 429 when limit is hit. This might be wrong. Improve on it or suggest alternatives."

A strawman gives Claude a starting point to react to. Reacting is cognitively different from generating from scratch. The reaction incorporates your domain knowledge (cache-backed, per-IP, 429) while allowing Claude to refine the approach (maybe a token bucket is better, maybe per-user is more appropriate than per-IP).

The "this might be wrong" qualifier is important. Without it, Claude tends to accept your approach uncritically. With it, Claude treats the proposal as a draft to be evaluated and improved.

Strawman proposals work well with self-critique: "Here is my approach. Critique it, suggest alternatives, then implement the best option."

## Specific Prompts for Ambiguous Codebases

Large codebases develop naming collisions. There might be three components called "Dashboard." Two services called "Auth." Four files named "utils.ts." When you prompt Claude to "fix the bug in the Dashboard component," which dashboard?

In ambiguous codebases, extreme specificity is not pedantic -- it is necessary.

"Fix the rendering bug in `src/features/trading/components/Dashboard/TradingDashboard.tsx`. The issue is in the `useMarketData` hook around line 145 where the WebSocket connection is not being cleaned up on unmount. Do not modify `src/features/admin/components/Dashboard/AdminDashboard.tsx` which has a similar name but is unrelated."

The explicit "do not modify" for the similarly-named component is not paranoia. It is a constraint that prevents a wrong-component edit, which in a large codebase could pass code review because the reviewer does not notice which Dashboard was changed.

The more ambiguous your codebase, the more specific your prompts need to be. Exact file paths, line numbers, function names, and explicit exclusions. The cost is a few extra seconds of typing. The benefit is Claude modifying exactly the right thing.

## Extended Thinking Configuration

Claude Code's extended thinking mode allocates additional tokens for reasoning before generating a response. The thinking happens internally -- you see the result, not the reasoning process. For complex tasks that require deep analysis, extended thinking dramatically improves output quality. For simple tasks, it wastes tokens and time.

Three mechanisms control it:

**Environment variables.** `MAX_THINKING_TOKENS` sets the budget for thinking tokens. The default is maximum (31,999 tokens). Set it lower (for example, `MAX_THINKING_TOKENS=10000`) to reduce cost on tasks that do not need deep reasoning, or set it to zero to disable thinking entirely. For the latest model with adaptive reasoning, thinking depth is controlled by effort level instead, and this variable is ignored unless set to zero.

`CLAUDE_CODE_EFFORT_LEVEL` sets the effort to `low`, `medium`, or `high` (default). Lower effort is faster and cheaper. Higher effort provides deeper reasoning. You can also adjust effort level interactively through the `/model` command using left/right arrow keys -- the change takes effect immediately.

**Keyboard toggle.** `Alt+T` (or `Option+T` on Mac) toggles extended thinking on and off during a conversation. This requires running the terminal setup command first (`/terminal-setup`).

**A critical misconception:** phrases like "think harder" or "ultrathink" in your prompt do not allocate more thinking tokens. They are just words in a prompt. The thinking budget is controlled entirely through the environment variables and the effort level setting. If you want Claude to reason more deeply, adjust the configuration -- do not add magic words to your prompt.

## Image and Screenshot Workflows

Claude Code accepts images as input through several mechanisms, and each has a different best use.

**Drag and drop.** In the desktop application, drag an image file directly into the prompt area. In supported terminals, drag and drop also works.

**Copy and paste.** Use `Ctrl+V` (not `Cmd+V` on Mac -- this is a common mistake) to paste an image from the clipboard. Screenshots taken with the system screenshot tool land in the clipboard and paste directly.

**File paths.** Reference an image by its path: "Look at the screenshot at `./screenshots/bug.png` and identify the layout issue." Claude reads the image file and interprets its contents.

**Opening referenced images.** When Claude mentions an image path in its response, `Ctrl+Click` (or `Cmd+Click` on Mac) opens the image in your system viewer. This is useful when Claude generates images or references existing screenshots.

### Screenshot-Driven Debugging

One infrastructure team discovered a pattern that extends image input beyond UI work. When container orchestration clusters went down and were not scheduling new pods, they fed screenshots of monitoring dashboards into Claude Code. Claude interpreted the dashboard visualizations, identified a warning indicating IP address exhaustion, guided the team through the cloud provider's management interface menu by menu, and provided the exact commands to resolve the issue. The screenshots were more precise than a verbal description of the dashboard state would have been.

This pattern applies wherever the symptom is visual but the solution is technical: infrastructure dashboards, error message screenshots, log viewer captures, network topology diagrams. Claude interprets the image and maps it to the technical context of the codebase.

### Visual-First Iteration for Non-Developers

Non-technical users -- designers, product managers, legal staff -- frequently use screenshots as their primary communication medium with Claude Code. They paste a screenshot of what they want the interface to look like, and Claude implements it. They look at the result, take another screenshot showing what is wrong, and Claude adjusts. The iteration loop is entirely visual, with no code discussion at all.

This is a distinct interaction pattern from the developer workflow. Developers describe changes in terms of code. Non-developers describe changes in terms of appearance. Both are valid inputs. The screenshot-driven approach is slower per iteration but requires zero technical vocabulary.

## Self-Critique: A Worked Example

The self-critique pattern becomes more concrete with a real example. One practitioner asked Claude Code to produce a financial optimization plan, then prompted: "Rate this plan 1-10. What would you improve?"

Claude rated its own plan 7.5 out of 10 and identified seven specific improvements. Among them was a significant catch: the plan had allocated certain assets in a way that ignored the tax-free status of a specific retirement account. Selling and reallocating within that account would cost zero in taxes, but the original plan had not taken advantage of this. The self-critique identified the missed optimization, and the revised plan incorporated it.

The lesson is not that self-critique always finds seven improvements. It is that self-critique finds the improvements that are not obvious on first generation. Claude does not include these optimizations initially because the generation process optimizes for the most common interpretation of the prompt. The self-critique step forces a second pass that catches what the first pass missed.

For analytical work, a particularly effective self-critique prompt is: "Include a self-critique section. Rate this plan 1-10 and list every improvement you would make." Making the self-critique a required section of the output -- not an afterthought -- ensures it receives adequate attention.

## Domain-Specific Prompt Templates

The prompting patterns described so far are general-purpose. For domain-specific analytical work -- financial analysis, technical auditing, data science -- the prompt itself needs structural depth that generic patterns do not provide.

### The Goal Prompt Anatomy

For complex analytical tasks, a detailed goal prompt follows a specific structure:

1. **What data exists and where.** "There are CSV exports from three accounts in `/data/accounts/`. A supplementary text file at `/data/notes.txt` has information that was not exportable."

2. **Desired output format.** "Produce a table with columns: Asset, Current Allocation, Target Allocation, Gap, Priority, Action. Include a summary section and an appendix with methodology."

3. **Preferences and principles.** "Prioritize tax efficiency over raw performance. Minimize fees. Prefer simplicity."

4. **Known constraints.** "Do not modify the retirement account contributions -- those are automatic. The brokerage account has a minimum balance requirement of $10,000."

5. **What you suspect is wrong.** "I think the portfolio is overweight in large-cap growth. If you have a different view, I want to hear it." The qualifier "if you have a different view, I want to hear it" prevents Claude from anchoring on your suspicion and ignoring evidence to the contrary.

6. **Strawman proposal.** "Here is my rough target allocation: 40% domestic equities split 60/40 large/mid-cap, 20% international, 30% fixed income, 10% alternatives. This might be wrong."

7. **Exhaustive questioning instruction.** At the end: "Ask me questions exhaustively about anything you are uncertain about before proceeding." Placing this at the end ensures Claude asks clarifying questions before jumping to conclusions.

This anatomy applies beyond finance to any analytical task where the prompt needs to convey domain context, output expectations, and nuanced preferences simultaneously.

### Dynamic Constraint Reframing

Mid-project, you can reframe constraints and watch Claude rebuild the entire analytical framework. "Treat the retirement account as a fixed constraint. Calculate what the ideal total portfolio looks like, subtract what the retirement account provides, and optimize the remaining accounts to fill the gaps." Claude recalculates every target, every gap, every recommendation based on the new framing. The static portion becomes a given; the dynamic portion gets optimized.

This is more powerful than starting over with a new prompt. Claude retains the full context of the analysis and rebuilds on top of the reframed constraint rather than regenerating from scratch.

## The Walkthrough Skill

Skills can encapsulate complex prompt patterns into reusable workflows. One community-built skill -- the walkthrough skill -- demonstrates how skills and prompt craft intersect.

When you invoke the walkthrough skill with a query like "walkthrough how does authentication work in this codebase," it executes a four-phase pipeline. First, it spawns two to four subagents to explore relevant parts of the codebase in parallel. Second, it synthesizes the findings into five to twelve key concepts with connections between them. Third, it generates a self-contained HTML file with an interactive diagram. Fourth, it opens the diagram in your browser.

Each node in the diagram is clickable. Click one and a detail panel slides in with a plain-English description, file paths, and code snippets. The whole thing produces a mental model of a system in under two minutes.

The walkthrough skill demonstrates a pattern that applies to any complex prompt workflow: wrap a multi-step process (parallel research, synthesis, artifact generation) in a skill so it becomes a single command. The prompt engineering is done once, tested, and reused.

## The /learn Skill Pattern

Another skill pattern worth studying is the learning skill, which follows five phases: deep analysis of the current conversation, categorization of insights (patterns, gotchas, architecture decisions), drafting the learning into documentation, user approval (a blocking step -- the skill pauses and waits for your confirmation), and saving to the appropriate documentation file.

This pattern extracts institutional knowledge from conversations and persists it. Instead of having Claude's useful discoveries disappear when the session ends, the skill captures them as documentation that benefits future sessions and future developers. The blocking approval step ensures nothing is saved without human review.

## Progressive Disclosure in Skills

A well-designed skill uses progressive disclosure to minimize context cost. Consider an analytics consultant skill: the skill directory contains a `skill.md` file with the skill definition and quick-start instructions, a `scripts/` directory with executable tools, and a `references/` directory with detailed documentation.

When a user's prompt matches the skill's description, only the `skill.md` loads into context. When the skill runs, it calls the CLI tool in `scripts/`. Only when the tool needs detailed guidance does it read from `references/`. The full reference material never enters context unless needed. This design keeps the base context cost near zero while supporting arbitrarily complex operations.

## Two-Interface Planning

Teams that include non-technical members report a distinctive workflow: brainstorm in a cloud AI chat interface first, then move to Claude Code for implementation. The cloud interface is better for open-ended exploration -- thinking through the workflow, considering alternatives, sketching the approach. Once the plan is clear, they create a comprehensive prompt in the chat interface and paste it into Claude Code as the starting point.

The key is asking the cloud interface to slow down and work step by step rather than producing everything at once. The output of the planning phase is a structured prompt -- not code, not a design document, but the actual input that Claude Code will receive. This two-interface pattern lets non-developers use the interface they are comfortable with for the creative work and delegate the technical execution to the tool designed for it.

## Providing Writing Samples

When Claude generates documentation, reports, or other written output, providing writing samples dramatically improves style matching. One team provides formatting preferences and example documents at the start of documentation tasks: "Here is an example of how we format runbooks. Match this style, including the heading structure, the bullet point density, and the level of detail in troubleshooting steps."

Claude extrapolates from examples more reliably than it follows abstract style descriptions. "Write in a concise, direct style" is vague. A three-paragraph sample of your actual writing style is precise. The cost is a few hundred tokens of example input. The benefit is output you can use without rewriting.

## The Interrupt-for-Simplicity Pattern

Claude tends toward complex solutions. It builds abstractions, adds layers, introduces patterns. This is usually a feature -- complex problems need complex solutions. But sometimes the simple approach is correct, and Claude overshoots.

The fix is a specific interruption pattern: stop Claude mid-execution and ask "why are you doing this? Try something simpler." Claude responds well to simplicity requests. It drops the unnecessary abstraction, removes the extra layer, and produces a more direct solution.

This is not the same as the general interrupt-and-steer pattern described earlier. The general pattern corrects direction. The simplicity pattern corrects complexity. Watch for signs of over-engineering: new files that seem unnecessary, abstractions that wrap a single call, patterns that add indirection without adding value. When you see them, interrupt.

---

## Key Takeaways

- Verification criteria (tests, expected output, visual targets) are the single highest-leverage prompting technique -- they convert one-shot generation into iterative refinement.
- Use vague prompts for exploration and specific prompts for implementation; the most common failure mode is vague implementation prompts.
- Your linting configuration, type checker, and test suite are part of your prompt -- strict tooling creates backpressure that improves Claude's output automatically.
- Ask Claude to plan before implementing; a 10,000-token plan prevents a 500,000-token misdirected implementation.
- State constraints explicitly -- what cannot change, trade-off preferences, and flexibility boundaries -- because implicit constraints are invisible constraints.
- Exhaustive questioning ("ask me questions exhaustively before starting") improves first-pass quality by 30-50% on complex tasks.
- In ambiguous codebases, extreme specificity with exact file paths and explicit exclusions prevents wrong-component modifications.

# Chapter 7: IDE Integration Done Right

## What You'll Learn

Claude Code runs in your terminal. It also runs in a code editor extension, a desktop app, a web interface, and a mobile app. Same engine everywhere. Same CLAUDE.md files. Same settings. Same MCP servers. The difference is the surface -- the interface you interact with and the workflow patterns it enables.

Most developers pick one surface and stay there. That is fine, but it leaves workflow optimizations on the table. This chapter maps out the surfaces, their strengths, their gaps, and the patterns that emerge when you understand which surface fits which task. You will learn how the desktop application provides full visual diff review with parallel Git-isolated sessions, how cloud sessions run on managed VMs you can monitor from your phone, and how the teleport feature moves work between surfaces without losing context. You will also confront the autocomplete gap -- the one capability where Claude Code does not compete with other tools -- and learn why the dominant usage pattern across teams is terminal-first with occasional IDE assists, not the other way around.

---

## Same Engine, Every Surface

Claude Code's architecture separates the engine from the interface. The engine handles the agentic loop, context management, tool execution, permissions, and all the machinery described in previous chapters. The interface -- terminal, editor extension, desktop app -- is a presentation layer.

This matters because it means switching surfaces does not mean switching tools. Your CLAUDE.md files load regardless of surface. Your permission settings apply everywhere. Your MCP servers connect to every surface that supports them. A workflow you develop in the terminal works in the editor extension with only minor adaptations.

The surfaces currently available: the terminal (the original and most capable), extensions for a popular code editor and a major IDE family, a desktop application, a web interface, and a mobile app. Each surface optimizes for different interaction patterns, but none of them changes what Claude Code fundamentally does.

## The Terminal Surface

The terminal is where Claude Code started and where it remains most powerful. Every feature described in this book works in the terminal. No exceptions.

The terminal surface gives you the full CLI with all flags, direct access to piping and Unix composability (Chapter 6), modal editing for prompts, background bash command execution, and the ability to run multiple instances in separate terminal panes or windows. It is the only surface where you have complete control over session management, output formats, and automation integration.

Terminal-first workflows dominate among experienced users for a practical reason: the terminal does not get in the way. There is no GUI mediating your interaction. You type a prompt, Claude runs, you see results. The feedback loop is tight. When you need to inspect a file that Claude modified, you open it in your editor. When you need to run a test, you run it in another terminal pane. The terminal is not trying to be an IDE. It is a command surface that works alongside whatever IDE you prefer.

This terminal-first approach has an underappreciated benefit: it is IDE-agnostic. Teams with mixed editor preferences -- some using one code editor, others using another IDE family, others on a minimal text editor -- can standardize on Claude Code workflows without standardizing on an editor. The CLAUDE.md file, the agent definitions, the MCP configurations, the hook system -- all of it works identically regardless of which editor is open in the next window.

## The Code Editor Extension

The extension for the popular code editor adds a visual layer on top of the Claude Code engine. The key features are inline diff viewing, @-mention file references, plan review interfaces, and conversation history.

**Inline diffs** are the headline feature. When Claude proposes changes to a file, you see the diff rendered in the editor with syntax highlighting, accept/reject controls, and the ability to partially accept changes. This is materially faster than reviewing diffs in the terminal, especially for large changes across multiple files.

**@-mentions** let you reference files, symbols, and selections directly in your prompt. Click on a function name, type @, and the file path is inserted into the context. This is a convenience feature -- you can achieve the same thing in the terminal by typing the path -- but it reduces friction enough to change behavior. Developers who use @-mentions reference specific context more often, which leads to better-scoped prompts.

**Plan review** surfaces Claude's intended approach in a structured visual format before execution. In the terminal, plans are text blocks in the conversation. In the editor, they are interactive checklists you can approve, modify, or reject item by item. This is particularly useful for large refactors where the plan has many steps and you want granular control over which steps proceed.

The extension is strongest when you are working on a single codebase and want tight integration between Claude's output and your editing workflow. It is weakest when you need features that exist only in the terminal -- specific CLI flags, piped workflows, or parallel instance management.

## The Desktop Application

The desktop application is a standalone surface that runs Claude Code outside both your IDE and your terminal. It provides visual diff review, parallel sessions with automatic Git worktree isolation, multiple permission modes, file attachments, connectors to external services, and support for both local and remote execution environments.

The application organizes work around three tabs. The Code tab is the primary workspace -- it mirrors the terminal's agentic capabilities through a graphical interface. You select a project folder, choose a permission mode, pick a model, and start typing prompts. The Cowork tab enables autonomous background work -- Claude operates continuously on its own while you monitor progress and steer when needed. The Chat tab provides a standard conversation interface for questions that do not require filesystem access.

### Permission Modes

The desktop application surfaces permission control through a mode selector that affects how much autonomy Claude has during a session:

- **Ask mode** requires your approval before every file edit and command. You see a diff view and accept or reject each change. This is the default and the right choice until you trust Claude's judgment in your codebase.
- **Code mode** auto-accepts file edits but still asks before running terminal commands. Faster iteration when you trust file changes.
- **Plan mode** restricts Claude to read-only operations and plan creation. No file modifications, no commands. Pure analysis and planning.
- **Act mode** is the equivalent of `--dangerously-skip-permissions` in the terminal. Claude runs without any prompts. Only use this in sandboxed environments.

You can switch modes mid-session. Start in Plan mode to map out an approach, switch to Code mode to execute, drop to Ask mode for the final verification steps.

### Visual Diff Review

When Claude modifies files, a diff stats indicator appears showing lines added and removed. Clicking it opens a full diff viewer with a file list on the left and changes on the right. You can comment on specific lines by clicking them -- type your feedback, press Enter, and after commenting on multiple locations, submit all comments at once. Claude reads the comments and makes the requested changes, which appear as a new diff. This is review-by-conversation, not review-by-reading. You point at problems, and Claude fixes them.

### Parallel Sessions with Git Isolation

Each session in the desktop application gets its own isolated copy of your project through Git worktrees. Click "New session" in the sidebar to start a second (or third, or tenth) parallel task. Changes in one session do not affect other sessions until you commit them. Worktrees are stored in your project's `.claude/worktrees/` directory by default. You can configure a custom location and a branch prefix in settings to keep Claude-created branches organized.

This turns the desktop application into a parallel processing interface. One session is refactoring the authentication module. Another is writing tests for the payment flow. A third is investigating a bug. Each session operates in its own Git branch, completely isolated from the others. When each session's work is done, you review and merge the branches independently.

### SSH and Remote Sessions

The desktop application connects to remote machines over SSH. Instead of selecting "Local" as your environment, select an SSH connection. Claude Code runs on the remote machine, executing commands and modifying files there. This is useful for codebases that require specific hardware, operating systems, or development environments that your local machine does not provide.

For long-running tasks, select a Remote environment instead. Remote sessions run on Anthropic-managed cloud infrastructure and continue even if you close the application or shut down your computer. You can monitor remote sessions from the desktop application, the web interface, or a mobile app. More on remote sessions in the next section.

### Connectors

The desktop application supports connectors that integrate external services -- version control platforms, team messaging tools, and project management systems. These connectors are configured through the application's settings and make Claude Code aware of your project's external context: PRs, issues, messages, tasks. The connector ecosystem is still growing, but the pattern is clear: Claude Code becomes the hub that connects your code to your team's communication and coordination tools.

## Claude Code on the Web

Claude Code runs in the browser as a fully cloud-hosted environment. No local setup required. You connect your version control account, select a repository, and start working. The repository is cloned to an Anthropic-managed virtual machine, an environment is configured, and Claude executes with full access to the codebase.

The web interface provides the same diff view as the desktop application -- you see exactly what Claude changed, comment on specific lines, and iterate until the changes are ready. When satisfied, you create a pull request directly from the interface. Changes are pushed to a branch on the remote, ready for review.

### Network Access Configuration

Cloud sessions support three network access levels: limited (default allowed domains only -- package registries, API endpoints, and similar infrastructure), full (unrestricted internet access), and none (completely offline). The network configuration is controlled through a security proxy that mediates all outbound connections. A separate proxy handles version control platform access, ensuring that repository operations work regardless of your network settings.

### Session Sharing

Cloud sessions can be shared at three visibility levels: Private (only you), Team (members of your organization), and Public (anyone with the link). Shared sessions let others observe Claude's work in progress, review the conversation history, and see the resulting changes. This is useful for code reviews, pair programming across time zones, and demonstrating workflows to team members.

### When to Use Cloud Sessions

Cloud sessions shine for three scenarios. First, long-running tasks that would tie up your local machine -- kick off a large refactor, close the browser, and check back in an hour. Second, repositories you do not have locally -- review a colleague's project without cloning anything. Third, parallel execution -- start multiple cloud sessions on different tasks and monitor them from a single browser tab.

The limitation is that cloud sessions currently work only with repositories hosted on the dominant code platform. Other hosting platforms are not yet supported for cloud execution.

## Claude Code on Mobile

The mobile application provides a monitoring and dispatch interface for cloud sessions. Kick off a task from your phone, monitor its progress, steer Claude with follow-up prompts, and review the results when they are ready. You are not going to write complex prompts on a phone keyboard, but you can check on a long-running task during your commute, answer Claude's clarifying questions, or approve a PR from the couch.

The mobile app connects to the same session infrastructure as the web interface. A session started from the desktop or terminal can be monitored from the mobile app, and vice versa.

## The Chrome Extension

A browser extension connects Claude Code to live web applications. It bridges the gap between the codebase and the running application -- Claude can read the DOM, observe network requests, and inspect the rendered output of the code it is working on. This is most valuable for frontend debugging, where the symptom (a visual bug in the browser) and the cause (a CSS or JavaScript issue in the code) live in different contexts. The extension brings them together.

## The IDE Family Plugin

The plugin for the major IDE family provides interactive diff viewing and context sharing. The implementation differs from the code editor extension in ways that reflect the IDE's architecture -- diffs integrate with the IDE's native diff viewer, context references use the IDE's symbol index, and the interaction model follows the IDE's conventions.

The practical differences between the two editor integrations are smaller than the architectural differences suggest. Both provide inline diff review. Both support context referencing. Both connect to the same Claude Code engine with the same capabilities. The choice between them is almost always driven by which IDE you already use, not by feature differences in the Claude Code integration.

## Cross-Surface Workflows

Surfaces are not isolated. Sessions can move between them. This is one of Claude Code's most distinctive architectural decisions -- no other tool in this category supports fluid session migration across interfaces.

### The /teleport Command

The `/teleport` command (also `/tp`) pulls a cloud session into your terminal. Start a long-running task on the web or from your phone, let it run while you are away, then pull it back to your local machine when you are ready to review and continue.

```
/teleport
```

This opens an interactive picker showing your cloud sessions. Select one, and Claude Code verifies you are in the correct repository, fetches and checks out the branch from the remote session, and loads the full conversation history into your terminal. If you have uncommitted local changes, it prompts you to stash them first.

The `--teleport` flag works the same way from the command line:

```
claude --teleport
claude --teleport <session-id>
```

### The & Prefix: Sending Work to the Cloud

The reverse direction -- terminal to cloud -- uses the `&` prefix. Start a message with `&` inside an interactive session:

```
& Fix the authentication bug in src/auth/login.ts
```

This creates a new cloud session with your current conversation context. The task runs on Anthropic's infrastructure while you continue working locally. Use `/tasks` to monitor all background sessions -- it shows status, and pressing `t` teleports you into any session.

The pattern that emerges is planning locally and executing remotely. Start Claude in plan mode to collaborate on the approach, then send the plan to the cloud for autonomous execution:

```
& Execute the migration plan we discussed
```

### The /desktop Command

The `/desktop` command hands off a terminal session to the desktop application. The full conversation context moves to the GUI, where you get visual diff review, inline comments, and the ability to partially accept changes. This is the right move when the task shifts from exploration to review.

### Workflow Patterns

The cross-surface workflow that emerges is: explore and plan in the terminal, implement in the terminal or editor, review diffs in the desktop application or editor extension. The terminal is the home base. Visual surfaces are specialized tools you invoke when review adds value.

This is not how most IDE integrations work. Most tools embed in the IDE and never leave. Claude Code inverts that relationship. The terminal is the primary surface, and the IDE is a supplementary view you pull up when needed. Understanding this inversion prevents the frustration of trying to force terminal-grade functionality through the editor extension.

## PR Review Status in the Footer

When working on a branch with an open pull request, Claude Code displays a clickable PR link in the terminal footer (for example, "PR #446"). The link has a colored underline indicating the review state:

- **Green**: approved
- **Yellow**: pending review
- **Red**: changes requested
- **Gray**: draft
- **Purple**: merged

The status updates automatically every 60 seconds. Cmd+click (Mac) or Ctrl+click (Windows/Linux) opens the PR in your browser. This small feature eliminates a constant context switch -- checking PR status without leaving the terminal. It requires the version control platform's CLI to be installed and authenticated.

## Eliminating Context Switching

One pattern that practitioners report consistently: Claude Code in the terminal eliminates the context switch between a cloud AI chat interface and the codebase. Without Claude Code, the workflow is: copy a code snippet, paste it into the chat interface, explain the problem, read the response, copy the suggested fix, paste it back into the editor, test it, and repeat. Each round trip crosses application boundaries and loses context.

With Claude Code, the entire conversation happens inside the project. Claude reads files directly. It writes changes directly. It runs tests directly. The context is the codebase, not a description of the codebase. The difference is not marginal -- it eliminates the mental overhead of translating between two environments and the mechanical overhead of copy-paste.

## LSP Plugins

The plugin system includes Language Server Protocol servers that provide code intelligence features: diagnostics, go-to-definition, code navigation, and symbol resolution. These are not the same as Claude Code's agentic capabilities -- they are traditional LSP features implemented as plugins.

LSP plugins are newer and less battle-tested than the core Claude Code features. Their value is in providing code intelligence for languages or frameworks that your IDE does not natively support well. If your IDE already has excellent language support for your stack, LSP plugins may add little. If you are working in a less common language or a custom framework, they can fill gaps.

The plugins auto-start when configured and appear as standard IDE features. They complement rather than replace your IDE's native language services.

## The Autocomplete Gap

Claude Code does not have streaming inline autocomplete. The kind of experience where you type a few characters and a gray ghost of the suggested completion appears in your editor, updating with each keystroke -- that is not what Claude Code does.

This is not an oversight. It is an architectural consequence. Claude Code's strength is in agentic, multi-step operations that span files and tools. Inline autocomplete is a fundamentally different interaction pattern: low-latency, single-line, character-by-character prediction. Building both well in the same tool requires different model configurations, different infrastructure, and different UX.

The gap matters because inline autocomplete is genuinely useful. For writing boilerplate, filling in function signatures, completing variable names, and the countless small editing tasks that do not warrant a full prompt-and-execute cycle, a good autocomplete tool saves real time.

The recommended approach is a hybrid setup. Use Claude Code for the agentic, multi-step, multi-file work it excels at. Use a separate tool that specializes in inline autocomplete for the character-by-character editing work. The two tools do not conflict. They operate at different scales of interaction -- one handles the forest, the other handles individual leaves.

This hybrid strategy is not a workaround. It is the mature approach. No single tool dominates every interaction scale, and pretending otherwise leads to using a powerful tool badly for tasks where a simpler tool works better.

## When to Use the IDE, When to Drop to Terminal

The decision is simpler than it seems.

**Use the IDE extension when:**
- You need to review multi-file diffs visually
- You are referencing specific code locations with @-mentions
- You want to partially accept or reject changes within a file
- You are doing plan review and want the interactive checklist
- You are working with a teammate who is more comfortable in the IDE

**Drop to the terminal when:**
- You need CLI flags not exposed in the IDE extension
- You are running multiple parallel sessions
- You are piping input or output to other tools
- You need automation or scripting integration
- You want the fastest possible feedback loop
- You are doing exploratory work where conversation flow matters more than diff review

Most sessions start in the terminal. Some move to the IDE for review. Few start in the IDE and stay there. This is not a judgment about IDE quality -- it reflects the fact that agentic coding is primarily a conversation, and conversations flow most naturally in the terminal.

## The Design Tool Pattern

A pattern worth noting: teams report that a visual design tool and Claude Code are open simultaneously roughly 80% of the time. The design tool provides the visual reference -- mockups, layouts, component designs. Claude Code implements what the design shows.

This is not an IDE integration in the traditional sense. It is a workflow where two tools complement each other without being connected. The human is the integration layer, looking at the design and describing to Claude Code what needs to be built.

The pattern works because Claude Code accepts images as input. You can paste a screenshot of a design mockup into the conversation, and Claude can interpret the layout, colors, spacing, and component structure. This is faster than describing a design in words, though it is not a substitute for explicit implementation requirements.

The design-to-code workflow follows the same four-phase pattern described in Chapter 1: explore the existing codebase components, plan the implementation approach, implement the design, and commit. The design tool adds a visual reference to the exploration phase that makes the subsequent phases more grounded.

## Skills as a Cross-Agent Standard

Skills are not exclusive to Claude Code. They are a shared standard supported by multiple AI coding agents. A skill you write for Claude Code works with other agents that support the standard -- and the list is growing. This means your investment in skill development is not locked to a single tool.

The installation workflow uses a package runner command that works across agents:

```
npx skills add <repository-url> --skill <skill-name>
```

Running this command clones the repository, discovers available skills (a typical repository might contain a dozen or more), detects compatible agents, asks which scope to install in (project or user), and creates the skill directory structure. Skills installed this way can be symlinked or copied, and they work immediately without restarting Claude Code.

The cross-agent compatibility has a practical consequence: teams that use multiple AI coding tools can standardize their workflows through skills rather than building tool-specific configurations. A code review skill, a deployment workflow, or a documentation generator works regardless of which agent runs it. This reduces the cost of experimenting with different tools and prevents workflow lock-in.

## The Hybrid Toolchain

No single AI coding tool dominates every interaction scale. The mature approach is a hybrid setup with each tool handling what it does best.

Use Claude Code for the agentic, multi-step, multi-file work it excels at: repository-wide refactors, complex implementations, codebase exploration, architectural planning, and automated workflows. Layer a dedicated inline completion tool for the character-by-character editing work -- function signatures, variable names, boilerplate, and the countless small editing tasks that do not warrant a full prompt-and-execute cycle.

This is not a workaround for a missing feature. It is an acknowledgment that agentic coding and inline completion are fundamentally different interaction patterns requiring different model configurations, different infrastructure, and different UX. Building both excellently in the same tool requires different optimization targets that inevitably trade off against each other.

The prediction for the near future is that Claude Code's inline IDE experience will become more aggressive -- per-file edit modes, region-scoped rewrites, tighter editor integration. But even as the surfaces converge, the underlying architecture will continue to optimize for different scales of interaction. The hybrid approach is not a temporary compromise. It is the stable end state.

## Terminal First as a Design Philosophy

The terminal-first pattern is not just a preference. It is a design philosophy that shapes how Claude Code evolves.

Because the terminal is the universal surface -- every operating system has one, every IDE has one embedded, every remote server is accessible through one -- building features for the terminal first ensures they work everywhere. An extension-first approach would leave terminal users behind. A terminal-first approach gives everyone the full feature set, with extensions providing visual enhancements on top.

This philosophy has practical implications for how you invest your learning time. Learning the terminal workflow gives you skills that transfer to every surface. Learning the extension workflow gives you skills specific to that extension. If you have limited time, invest in the terminal first and add extension workflows as refinements.

The terminal-first philosophy also explains why Claude Code does not try to replace your IDE. It sits alongside your IDE as a complementary tool, handling the agentic work that IDEs do not do well while leaving the editing, debugging, and navigation work to tools purpose-built for those tasks. The integration is collaborative, not competitive.

---

## Key Takeaways

- Claude Code's engine is surface-independent -- CLAUDE.md, settings, MCP servers, and permissions work identically across terminal, editor extensions, desktop, web, and mobile.
- The desktop application provides visual diff review with inline comments, parallel sessions with Git worktree isolation, multiple permission modes, SSH connections, and connectors to external services.
- Cloud sessions run on managed VMs, persist after you close the browser, and can be monitored from any device -- use the `&` prefix to dispatch work to the cloud and `/teleport` to pull it back.
- The terminal footer shows live PR review status with colored underlines (green=approved, yellow=pending, red=changes requested) -- updated every 60 seconds.
- Skills are a cross-agent standard: `npx skills add` installs skills that work across multiple AI coding tools, preventing workflow lock-in.
- The inline autocomplete gap is architectural, not accidental; a hybrid setup with Claude Code for agentic work and a dedicated autocomplete tool for editing is the stable end state.
- Terminal-first workflows dominate among experienced users because the terminal is faster, more flexible, and IDE-agnostic.
- Invest learning time in terminal workflows first -- they transfer to every surface and every IDE.

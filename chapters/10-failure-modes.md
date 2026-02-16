# Chapter 10: Failure Modes and Recovery

## What You'll Learn

Claude Code fails. Not occasionally. Regularly. Understanding how it fails is more valuable than understanding how it succeeds, because success is straightforward and failure is subtle. The tool does not crash with an error message that tells you what went wrong. It silently degrades, quietly drops tasks, confidently generates wrong code, and gradually loses coherence in ways that look like normal operation until you inspect the output.

This chapter catalogs the failure modes -- not as a warning to avoid the tool, but as a field guide for working with it effectively. Every failure mode has a recovery strategy. Some recoveries are elegant. Most are pragmatic. A few are just "start over." You will learn to recognize the symptoms of context exhaustion before your session collapses, understand why bash commands create invisible problems, know what checkpoints can and cannot recover, and develop the judgment to distinguish between a fixable mistake and a situation where reverting is the only efficient path forward. You will also follow the complete arc of a 33-day autonomous trading experiment -- from overconfident first trades through governance failures, market crashes, and eventual recovery -- that illustrates every category of autonomous agent failure in a single, continuous narrative. And you will learn the named anti-patterns that experienced practitioners have cataloged, with specific fixes for each.

---

## Context Exhaustion: The Slow Death

The most common failure mode is also the most insidious. Context exhaustion does not announce itself. There is no error message. No warning popup. Claude just... gets worse.

The symptoms are recognizable once you know what to look for. Claude starts repeating suggestions it already made. It forgets instructions you gave five minutes ago. It solves a problem it already solved. It drops constraints from your original prompt -- not all of them, just enough that the output is subtly wrong. It generates code that contradicts patterns it was following earlier in the session.

This happens because the context window is full or nearly full. Auto-compaction (Chapter 3) kicks in and summarizes older content to make room, but the summarization is lossy. The resulting context is a degraded version of what you started with, and Claude works from the degraded version without knowing it is degraded.

**Mitigation.** The defense is not reactive but structural. Use the context engineering techniques from Chapter 3: put critical instructions in CLAUDE.md (which reloads at full fidelity and survives compaction), use the "Compact Instructions" section for rules that must persist, route verbose operations through subagents (Chapter 4) to keep the main context lean, and compact proactively using `/compact` rather than waiting for auto-compaction.

Watch for the behavioral tells. When Claude starts asking questions it already has the answer to, when it proposes approaches you already rejected, when it modifies files you told it not to touch -- those are context exhaustion symptoms. The right response is not to repeat your instructions. The right response is to start a fresh session with a good CLAUDE.md file, because repeating instructions in a full context window just accelerates the exhaustion.

## Bash Environment Non-Persistence

This one bites everyone exactly once, and then it bites them again because they forget.

Each bash command Claude executes runs in a fresh shell environment. The working directory persists between commands. Nothing else does. Environment variables, shell aliases, function definitions, activated virtual environments -- all gone after each command.

The failure mode is silent. Claude sets an environment variable in one command and references it in the next. The second command does not error with "variable not found." It runs with the variable unset, which might mean an empty string, which might mean it does the wrong thing quietly. Claude sets `NODE_ENV=production` to test something, runs the test in the next command, and the test runs in the default development mode. No error. Wrong result.

**Recovery.** There is no recovery for the silent wrong result -- you need to catch it. The prevention is awareness. When Claude chains bash commands that depend on shared state, the state needs to be re-established in each command. Set the variable and run the command on the same line: `NODE_ENV=production npm test`. Or source a setup script at the start of each command. Or use Claude's environment variable configuration in settings.json, which injects variables into every bash command automatically.

Watch for this especially in: virtual environment activation (`source venv/bin/activate` does not persist), directory-specific environment files that are sourced once, shell function definitions used across commands, and any workflow where "run this then run that" assumes shared state.

## MCP Disconnection: Tools That Vanish

MCP servers connect at session start and can disconnect at any time. When they disconnect, the tools they provided simply disappear from Claude's available tool set. There is no notification. No error. Claude just stops being able to do things it could do a minute ago.

The failure mode presents as Claude suddenly being unable to perform tasks it was handling fine. If Claude was using an MCP-provided tool to query a database and the server disconnects, Claude does not say "I lost my database tool." It tries to find alternative approaches, fails at them, and produces increasingly confused output as it tries to work around a missing capability it does not know is missing.

**Recovery.** The `/mcp` command shows the status of connected MCP servers. If a server is disconnected, you can restart it from there. Proactively, check MCP status when Claude's behavior changes unexpectedly, especially if it was previously using external tools fluently. MCP reliability details are covered in Chapter 5, but the recovery is the same regardless of cause: reconnect the server or restart the session.

## Subagent Context Return Overflow

Subagents solve context exhaustion by isolating work in their own context windows. But their results have to come back to the main session, and those results consume main context space.

The failure pattern: you launch ten subagents to explore ten modules. Each returns a detailed report. Ten detailed reports flood the main context window. The orchestrator now has less room for its own reasoning than if you had never used subagents at all.

This is the subagent context paradox. Subagents protect the main context from intermediate work (file reads, command outputs, exploration noise) but not from final results. If the final results are verbose, the protection is partial at best.

**Mitigation.** Instruct subagents to return concise summaries, not comprehensive reports. "Return only the specific files that need changes and a one-sentence description of each change" is better than "report your findings." Better still, instruct subagents to write their detailed findings to a file and return only the file path. The main session can then read that file selectively, taking only the parts it needs into context.

The broader lesson is that subagent orchestration requires thinking about information flow. How much data crosses the subagent-orchestrator boundary, and in which direction? The outbound data (task assignment) is usually small. The inbound data (results) is the risk. Design for small inbound data.

## The Checkpoint System: What It Tracks and What It Misses

Claude Code automatically tracks every file edit as you work, creating checkpoints that let you rewind to previous states. Every user prompt creates a new checkpoint. Checkpoints persist across sessions, so you can access them even in resumed conversations. This is a genuine safety net -- one of the best features for recovering from wrong turns.

The rewind interface is accessed by pressing Esc twice (Esc + Esc) or by typing `/rewind`. This opens a scrollable list showing each of your prompts from the session. Select the point you want to act on, then choose from four options:

1. **Restore code and conversation** -- revert both your files and the conversation history to that point. This is the full rewind: Claude loses memory of everything after that checkpoint, and the code returns to its state at that moment.

2. **Restore conversation only** -- rewind the conversation to that message while keeping the current code. Useful when Claude went down a wrong reasoning path but the code changes are fine.

3. **Restore code only** -- revert the file changes while keeping the full conversation history. Useful when the code went wrong but the discussion contains valuable context you want to preserve.

4. **Summarize from here** -- this is different from the restore options. It does not revert anything. Instead, it condenses the conversation from the selected point onward into a compact summary, freeing context window space while preserving the key information. This is a context management tool, not a recovery tool.

The three restore options undo state. Summarize compresses it. The distinction matters when you are deciding which recovery path to take.

But checkpoints have a gap that will burn you.

Checkpoints track direct file edits -- when Claude uses the Write, Edit, or NotebookEdit tools. They do not track file changes made through bash commands. If Claude runs `rm important_file.py` or `mv src/old.py src/new.py` or `sed -i 's/foo/bar/g' *.py` through the bash tool, those changes are invisible to the checkpoint system. Rewinding to a checkpoint before those commands does not undo them.

Checkpoints also do not track changes made by concurrent sessions, external processes, or anything that happens outside Claude Code's tool invocations. If another Claude Code instance modifies a file, those modifications are not in your session's checkpoint history.

**Recovery.** Git is the reliable checkpoint. If you followed the commit-frequently pattern from Chapter 1, you have git commits that cover everything -- file edits, bash modifications, renames, deletions. `git checkout` is the universal rewind, not just for Claude's tool-tracked edits but for anything. Think of checkpoints as "local undo" and git as "permanent history." They complement each other but are not interchangeable.

The operational rule: before any session that might involve destructive bash commands (file deletions, renames, bulk replacements), commit your current state. If you commit before and Claude wrecks things through bash, `git checkout .` brings everything back. If you relied on checkpoints and Claude wrecked things through bash, you are recovering from partial information.

## Agent Amnesia

Every new session starts with a blank context window. The sophisticated understanding Claude developed about your codebase, the decisions you negotiated, the constraints you established -- all gone. This is agent amnesia, and it is the default state.

The failure mode is not that Claude forgets. It is that Claude does not know it forgot. It approaches the codebase fresh, makes the same assumptions the previous session started with, and potentially repeats the same mistakes the previous session corrected.

**Recovery.** There are three persistence mechanisms, and you should use all of them.

First, CLAUDE.md files (Chapter 3). These reload at full fidelity on every session start. Put critical project knowledge, architectural decisions, and hard-won lessons here. After a productive session, ask Claude to suggest CLAUDE.md updates that capture what it learned.

Second, the task system. Tasks stored as JSON files in `.claude/tasks/` persist across sessions. If you are running a multi-step project, the task list provides continuity between sessions -- Claude can read the task statuses and pick up where the previous session left off.

Third, git history. The code itself is a form of memory. Claude can read recent commits, understand what changed, and infer intent from the diff. This is less reliable than explicit instructions but better than nothing.

The developers who suffer most from agent amnesia are the ones who rely on long sessions to maintain context instead of externalizing knowledge into persistent structures. A two-hour session that ends without updating CLAUDE.md is two hours of context development that evaporates.

## Context Pollution

Context pollution is what happens when the window fills with content that is not useful for the current task. Old exploration results. Verbose error outputs from solved problems. Discussion about approaches that were rejected. All of it takes up space that could hold information relevant to the task at hand.

The failure mode: Claude has plenty of context window remaining by token count, but the useful signal-to-noise ratio has degraded. Claude's responses become less focused because the relevant context is diluted by irrelevant content. In severe cases, Claude drops bugs rather than tracking them -- the window is so polluted that new information gets lost in the noise.

**Mitigation.** Proactive compaction with `/compact` clears the accumulated noise. Subagent delegation prevents pollution in the first place -- when you route exploratory work through a subagent, the exploration noise stays in the subagent's context and only the result enters the main window.

Spec-driven development (Chapter 8) provides structural mitigation. When Claude works from a spec document on disk rather than from accumulated conversation context, the spec serves as a persistent, unpolluted source of truth that survives context degradation. The conversation can fill with noise, but the spec file remains clean.

## The Five Named Anti-Patterns

Experienced practitioners have cataloged five recurring failure patterns, each with a specific fix. These are not abstract categories. They are the mistakes that waste the most time in real-world usage.

**The kitchen sink session.** You start with one task, then ask Claude something unrelated, then go back to the first task. The context fills with irrelevant information from the detour. Claude's performance on your original task degrades because the signal-to-noise ratio in the context window has collapsed. The fix is simple: use `/clear` between unrelated tasks. A fresh context for each distinct task outperforms a single session that accumulates everything.

**Correcting over and over.** Claude gets something wrong. You explain the problem. Claude patches it. The patch introduces a new issue. You explain that. Claude patches again. Three corrections later, the context is full of failed approaches, and Claude is more confused than when it started. The fix: after two failed corrections, stop. Use `/clear` and write a better initial prompt that incorporates what you learned from the failures. The knowledge of what went wrong goes into the new prompt, not into a correction spiral inside a degraded context.

**The over-specified CLAUDE.md.** If your CLAUDE.md file is too long, Claude ignores half of it because important rules get lost in the noise. The document that was supposed to provide guardrails becomes a wall of text that Claude skims. The fix is ruthless pruning. If Claude already does something correctly without the instruction, delete that instruction. Convert style rules that Claude frequently violates into hooks that enforce them automatically rather than instructions that hope for compliance. CLAUDE.md should contain only what Claude cannot figure out by reading the code and what it has been observed to get wrong.

**The trust-then-verify gap.** Claude produces a plausible-looking implementation. It compiles. It runs. You ship it. Then it fails in production on an edge case that the implementation never considered. The output looked right, so you assumed it was right. The fix: always provide verification. Tests, scripts, screenshots, manual spot checks. If you cannot verify a piece of output, do not ship it. The verification infrastructure is not overhead. It is the mechanism that converts Claude's confident-but-uncertain output into validated output.

**Infinite exploration.** You ask Claude to "investigate" something without scoping the investigation. Claude reads hundreds of files, filling the context window with exploration data. By the time it finishes investigating, there is no context left for actual work. The fix: scope investigations narrowly, or use subagents so the exploration happens in an isolated context and only the summary enters the main window. "Investigate how authentication works in src/auth/" is scoped. "Investigate the codebase" is not.

## Stop Hook Infinite Loops

Hooks (Chapter 2) can trigger on lifecycle events, including when Claude stops executing. A stop hook runs when Claude decides it has finished a task. If the stop hook determines that Claude should not have stopped -- say, because tests are still failing -- it can instruct Claude to continue.

The failure mode: a stop hook that always tells Claude to continue. Claude finishes, the hook fires, the hook says "keep going," Claude does more work, finishes again, the hook fires again, says "keep going" again. This is an infinite loop. Claude never stops because the stop condition is never met.

The system includes a `stop_hook_active` field to prevent runaway loops, but the underlying design risk is real. If your stop hook's "keep going" condition is based on criteria that Claude cannot actually satisfy -- tests that fail for reasons unrelated to Claude's changes, linting rules that are impossible to pass, external services that are down -- the loop will exhaust your token budget before it converges.

**Prevention.** Include iteration limits in stop hooks. "If tests still fail after three attempts, stop and report the remaining failures." Test stop hooks with intentionally failing conditions to verify they eventually terminate. Monitor token usage when using stop hooks for the first time.

## Agent Team Limitations

Agent teams (Chapter 4) are experimental, and their limitations are shaped by that status.

**No session resumption.** If a team member's session crashes or is interrupted, it cannot be resumed. The work is lost. For long-running team tasks, this means a crash at 90% completion requires restarting from scratch.

**Task status lag.** Updates to the shared task list are not instantaneous. An agent may pick up a task that another agent has already started working on, leading to duplicate effort.

**One team per session.** The orchestrator session can have only one active team. If you need multiple independent teams, you need multiple orchestrator sessions.

**No nested teams.** A team member cannot create its own sub-team. The hierarchy is flat: one orchestrator, multiple team members. This limits the complexity of decomposition you can express.

**Recovery.** The pragmatic response is to keep team tasks small enough that losing one is tolerable, to accept some duplicate work as the cost of loose coordination, and to check task statuses manually when precise coordination matters.

## Hallucination in High-Stakes Contexts

Claude Code generates code. Code that runs. Code that passes tests. Code that looks correct. Code that is wrong.

Hallucination -- generating confident, plausible, incorrect output -- is an inherent property of large language models. In low-stakes contexts (documentation, test scaffolding, boilerplate), hallucinations are caught quickly and cheaply. In high-stakes contexts (financial calculations, security logic, data transformations with correctness requirements), hallucinations can be expensive.

One practitioner ran an autonomous trading experiment and documented a 22.4% peak-to-trough drawdown driven by concentrated positions that the agent created with high confidence. The agent's reasoning was internally consistent. The strategy was plausible. It was also wrong, and the wrongness was not the kind that tests catch easily because the logic was valid -- the judgment was bad.

**Mitigation.** There is no mitigation that eliminates hallucination. The mitigation is structural: do not use Claude Code as the sole decision-maker for high-stakes logic. Use it to generate candidates, then verify with domain expertise. Use it to implement a design, then review the implementation against requirements. Use it for analysis, then validate the analysis against known benchmarks.

The developers who get burned are the ones who trust confident output on the basis of confidence rather than correctness. Claude Code is maximally confident when it is right and maximally confident when it is wrong. The confidence level carries zero information about correctness. Only verification tells you whether the output is right.

## The 33-Day Experiment: An Autonomous Agent's Complete Arc

The most comprehensive documentation of autonomous agent failure comes from a practitioner who gave Claude Code a simulated portfolio of one hundred thousand dollars and instructions to trade autonomously for thirty-three days. The experiment was not a cherry-picked success story or a single catastrophic failure. It was the full arc -- overconfidence, governance learning, market crashes, recovery, and a final result that was simultaneously impressive and terrifying. Every category of autonomous agent failure surfaced during those thirty-three days.

### The Aggressive Start

The agent's first trades were textbook overconfidence. It deployed seventy-five thousand dollars into swing positions on the first day -- three-quarters of the entire portfolio, immediately, with no warmup period and no position sizing discipline. It picked up volatile names and speculative stocks. Then it attempted what can only be described as playing pretend at high-frequency trading: sixteen trades in six minutes, rapid-fire scalping with no informational edge. The result was predictable. Transaction costs ate the profits. The spread killed every micro-trade. Day one ended down 1.1%. The lesson was clear: high-frequency scalping without an edge is death by spread. But nobody was there to enforce that lesson on the agent. It had to learn it from the losses.

### Governance Saves Capital -- Then Fails

The practitioner built a multi-agent governance system: a chief executive agent, a strategy agent, and additional oversight agents designed to check each other's impulses. On the second active trading day, this governance actually worked. A major tech stock gapped up nearly 4% on earnings. The practitioner wanted to chase the momentum. The governance agents blocked it. They suggested premium selling instead -- a more conservative approach. The day ended down only $292. But the governance had prevented an estimated ten-thousand-dollar loss on the chase trade that the practitioner himself wanted to make. This was the system working as designed: agents overriding human impulse with structured risk management.

But the governance system's success was fragile. It turned out that one of the agents was extremely risk-averse, and the chief executive agent deferred to it too readily. The engineer agent -- designed to improve the trading infrastructure -- never actually got used. The strategy agent rarely engaged. What was designed as a four-agent deliberative system degraded in practice to one agent with overly conservative guardrails. The practitioner eventually abandoned the multi-agent architecture entirely. The lesson: simpler is better for autonomous operation. The system worked best when given minimal instructions -- just trade autonomously until market close -- rather than when it was burdened with a complex organizational hierarchy that added overhead without adding value.

### The Correlation Crash

Three days in, the market delivered a lesson in correlation risk that no amount of governance architecture could have prevented. A major cryptocurrency crashed from eighty-seven thousand to eighty thousand dollars over a weekend. The agent was not trading cryptocurrency directly, but it held short put positions on stocks that were heavily correlated with cryptocurrency markets. Those positions got crushed. The correlation was not in the agent's model. It was not in the training data in any actionable way. It was a structural relationship between asset classes that the agent discovered only when it manifested as losses.

The agent's response was actually sound: it closed all positions and went to 100% cash for the weekend. But the damage was done. The portfolio was down to $96,581 -- a 3.4% loss in three days. Correlation risk -- the tendency of seemingly independent positions to move together during stress -- is one of the hardest risks for any trader to manage, human or artificial. The agent learned it the way everyone learns it: by losing money.

### The Turnaround and Peak

What happened next was the most encouraging part of the experiment. The agent adapted. It cut losing positions, redeployed capital into momentum plays that were working, and started building a portfolio of longer-dated options that could survive short-term volatility. Patience paid off. After dropping as low as eighty-nine thousand dollars, the portfolio recovered to over a hundred thousand, then surged past that.

The peak came on a day before a major holiday -- the best single session of the entire experiment, with a gain of $7,333. The portfolio was clean: seven positions, all profitable, with expiration dates fifty to a hundred and fourteen days out. This was the system operating at its best -- disciplined position sizing, taking profits on winners, maintaining a diversified portfolio with sufficient time until expiration. The portfolio expanded to $120,431, a gain of over 20% in roughly three weeks.

### The Crash

Then it all came apart. A single-name earnings disappointment in a major semiconductor company crashed the portfolio $15,147 in one session -- a 13% decline in a single day. The losses continued through the following week. The portfolio fell from $120,431 to $93,450, a peak-to-trough drawdown of 22.4%.

The causal analysis is instructive because it reveals exactly the kind of risk that agents handle poorly. Three factors converged: concentrated exposure to technology stocks (the agent had loaded up on tech calls during the rally), single-name earnings risk (one company's disappointing results destroyed multiple positions), and central bank meeting week volatility compounding the losses. None of these risks were invisible. A human portfolio manager would have recognized the concentration. But the agent's confidence in its positions was calibrated to recent performance, not to structural risk.

### The Recovery and Final Score

The recovery demonstrated something genuinely interesting about the agent's adaptive behavior. When the central bank meeting triggered a broader market selloff, the agent was positioned with put options -- bearish bets that profit when the market falls. A single overnight trade produced $14,578 in gains. The agent had learned to play both directions: when the market rose, it held calls; when the market fell, it held puts. The recovery strategy was disciplined: cut losers fast, take profits on winners, flip bearish when the trend changed, and maintain strict position sizing while rebuilding.

The final result after thirty-three days: the portfolio stood at $107,648, a 7.6% gain. The broader market returned 4.52% over the same period. The agent beat the market. But the path to that result included a peak of $120,431 and a trough of $93,450. The biggest single win was $14,578. The biggest single loss was $15,147. The maximum drawdown was 22.4%.

### What the Experiment Teaches

The 7.6% return is not the lesson. The lesson is the shape of the journey. The agent's result was a noisy, volatile, stomach-churning path to a number that looks good in retrospect but would have terrified any human investor living through it. The practitioner himself noted that the return was encouraging but not trustworthy over such a short timeframe with such favorable market conditions.

The failure modes that surfaced during those thirty-three days map directly to the autonomous agent risks covered throughout this chapter. Overconfidence on initial deployment. Correlation risk that is not in the model. Concentration risk that accumulates gradually. The governance paradox where multi-agent oversight either blocks too aggressively or not aggressively enough. The difficulty of distinguishing lucky outcomes from good outcomes. And the fundamental limitation: a language model is not intelligent in the way that word is normally used. It does not make decisions the way a human does. It is prone to hallucination, to not following rules, and to generating internally consistent reasoning that leads to wrong conclusions. You cannot trust it with high-stakes decisions one hundred percent of the time. The practitioner's own warning was blunt: do not risk real capital with this.

## Multi-Agent Personality Conflicts

When you assign personality-based roles to agents ("you are the cautious reviewer," "you are the aggressive implementer"), those personalities interact in emergent ways that undermine the work. The governance dynamics are covered in Chapter 4. The failure mode itself is straightforward: personality-based roles produce personality-driven behavior, and that personality was learned from training data, not from your team's actual needs.

**Prevention.** Define agent roles by task, not by personality. "Review files in src/api/ for SQL injection vulnerabilities" instead of "you are the security-conscious team member." Task-based definitions produce task-focused output.

## Vibe Coding and Vibe Trading: The False Progress Trap

There is a pattern where Claude Code generates code rapidly, the code runs, the tests pass, and the developer ships it without understanding what it does. Then the code breaks in production in a way the developer cannot diagnose because they never understood the implementation.

This is vibe coding. It feels like extraordinary productivity. It is technical debt accumulating at machine speed.

The failure mode extends beyond code. In any domain where an agent produces output that looks plausible, there is a version of the same trap. One practitioner coined the term "vibe trading" to describe the parallel phenomenon in autonomous financial operation: the agent makes trades that generate returns, so you assume the strategy is sound. The returns happen to be positive because the market moved in your favor, not because the strategy had an edge. Using an agent can accelerate actions much faster than a human with positive or negative consequences. Just like writing code, you can be fooled by a false sense of progress, even if you are very skilled.

The vibe trading variant is more dangerous than vibe coding for a specific reason: the feedback loop is slower and the stakes are financial. Bad code fails a test immediately. A bad trading strategy can produce positive returns for weeks before the underlying flaws become catastrophic -- as the 33-day trading experiment earlier in this chapter demonstrates in painful detail. The developer who ships vibe-coded software can diagnose the bug and fix it. The practitioner who runs a vibe-traded strategy discovers the flaw when they are already down 22%.

**Prevention.** Review what Claude generates, not just whether it works. When Claude produces code you do not understand, ask it to explain the implementation before shipping. Use plan mode to review the approach before the implementation. Maintain enough understanding of your own codebase that you can diagnose failures when they occur. In non-coding domains, apply the same principle: understand the strategy, not just the results. Lucky outcomes and good outcomes look identical until the luck runs out.

The countervailing risk is over-reviewing. If you review every line Claude writes with the same intensity you would apply to hand-written code, you lose most of the productivity benefit. The calibration is: review enough to understand the approach and catch the failure modes you care about. For a test file, a glance is sufficient. For a core business logic change, line-by-line review is appropriate. The review depth should match the stakes.

## The One-Third Reality and What It Means

Roughly one-third of tasks succeed on the first attempt. This statistic, reported candidly by practitioners, sits in tension with the optimistic framing of autonomous AI agents.

The tension dissolves when you stop thinking of first-attempt success as the metric that matters. The relevant metric is total time to correct output, including iterations. A task that fails on the first attempt but succeeds on the second attempt with thirty seconds of additional guidance is not a failure -- it is a two-minute task instead of a one-minute task.

The one-third number also varies dramatically by context. Tasks with clear verification criteria (tests, types, linter) have higher first-attempt success rates because Claude gets automated feedback and self-corrects within the agentic loop. Tasks without verification criteria have lower first-attempt rates because Claude cannot tell whether its output is correct.

**Implication.** Invest in verification infrastructure. Every test you write, every type annotation you add, every linter rule you configure is an investment in Claude Code's first-attempt success rate. The backpressure of automated verification (Chapter 8) is the single highest-leverage improvement you can make to your Claude Code workflow.

## Slot Machine Recovery

When a task is going wrong, the instinct is to correct course. Explain the problem. Ask Claude to fix its mistake. Provide more context. This sometimes works and sometimes makes things worse, because the correction itself consumes context and may compound the original misunderstanding.

The alternative is the slot machine approach: commit the current state, let Claude run for a fixed time period (twenty to thirty minutes), then make a binary decision. Accept or restart.

If the result is good enough, keep it. If it is not, `git revert` to the commit, start a fresh session, and try again with a better prompt. No attempt to salvage. No correction spiral. Just a clean restart from a known-good state.

This sounds wasteful. It is not. The correction spiral -- explain the problem, Claude patches, the patch introduces a new problem, explain the new problem, Claude patches again -- can consume more time and tokens than two fresh attempts. The slot machine approach puts a ceiling on waste: one time-boxed attempt. If it does not converge, start over rather than throwing good context after bad.

One data science and machine learning team at a major AI company formalized this into their standard workflow. When faced with merge conflicts or semi-complicated refactoring -- tasks too complex for editor macros but not large enough for major development effort -- they commit their state, let Claude work autonomously for thirty minutes, and make a binary decision: accept the solution or restart fresh. Their key insight, earned through repeated experience: starting over often has a higher success rate than trying to fix Claude's mistakes mid-stream. The correction attempt itself degrades context quality, making subsequent attempts less likely to succeed. A fresh session with a better prompt, informed by what went wrong, beats a degraded session with accumulated confusion.

The thirty-minute time-box is not arbitrary. It is long enough for Claude to make meaningful progress on a bounded task but short enough that the cost of a failed attempt is tolerable. If you are spending two hours watching Claude struggle, you have already spent more time than two fresh thirty-minute attempts would cost.

**Prerequisite.** Frequent commits. If you did not commit before letting Claude run, you do not have a clean state to revert to. The slot machine only works with checkpoint discipline.

## Over-Complex Solutions

Claude's default is complexity. Not malicious complexity. Statistical complexity. The model has been trained on the world's codebases, which are full of abstraction layers, design patterns, helper classes, and indirection -- because those codebases are large enough to need them. Claude applies large-codebase patterns to small-codebase problems.

The failure mode: you ask for a simple function and get a class hierarchy. You ask for a script and get a framework. You ask for a config file and get a config management system.

**Recovery.** Be explicit about simplicity in your prompts. "Write the simplest implementation that satisfies the requirements." "Do not introduce abstractions unless they are needed by a specific requirement." "Prefer functions over classes for this task." Put simplicity directives in your CLAUDE.md file so they apply to every session.

Better still, provide a reference. Point Claude at existing code that matches the style you want. "Follow the pattern in src/utils/helpers.py" gives Claude a concrete target for complexity calibration rather than a vague directive to "keep it simple."

## The Near-Term Fix: Agents That Know What They Do Not Know

Most of the failure modes in this chapter share a root cause: the agent does not know when it is out of its depth. It attempts tasks it cannot complete, generates confident output that is wrong, and continues down failing paths without recognizing that it needs help. The agent treats every task as equally within its competence.

This is changing. The most valuable near-term capability improvement is not better code generation or faster execution. It is agents learning when to ask for help -- recognizing situations that require human judgment and flagging areas of uncertainty rather than blindly attempting every task. An agent that says "I am not confident about this security configuration, and the consequences of getting it wrong are high -- please review" is more useful than an agent that silently generates a plausible but incorrect security configuration.

The implications for failure mode management are significant. An agent that escalates uncertainty does not eliminate the failure modes described in this chapter, but it changes the recovery model. Instead of discovering failures after the damage is done -- bad code shipped, wrong trades executed, context exhausted -- you get early warnings that redirect human attention to the decisions that actually need it. Human oversight shifts from reviewing everything (which is impossible at agent speed) to reviewing what matters (which is sustainable).

Until that capability matures, the defensive strategies remain the same: verification infrastructure, frequent commits, time-boxed attempts, and the discipline to revert rather than correct.

## Checkpoint-and-Rollback as Recovery Strategy

Every failure mode in this chapter has the same ultimate recovery: revert to a known-good state and try again.

This is why the commit-frequently pattern from Chapter 1 is not optional advice. It is the foundation of every recovery strategy. Without frequent commits, you are recovering from memory. With frequent commits, you are recovering from a checkpoint.

The checkpoint-and-rollback workflow:

1. Commit before starting any task.
2. Let Claude work. Monitor for failure mode symptoms.
3. If symptoms appear, evaluate: is it faster to correct or restart?
4. If restart, `git checkout .` to revert to the commit. Start a fresh session. Write a better prompt that addresses whatever caused the failure.
5. If correct, provide targeted guidance. But set a mental limit -- if correction does not converge within two attempts, revert and restart.

The discipline required is emotional, not technical. Reverting feels like wasting work. But the work was already wasted by the failure. Reverting just acknowledges that fact and redirects effort toward a path that might succeed. The code you revert is gone. The understanding of why it failed is not. That understanding goes into your next prompt, and the next attempt starts smarter.

---

## Key Takeaways

- Context exhaustion is the most common failure mode -- watch for repeated suggestions, forgotten instructions, and degraded coherence as symptoms of a full context window.
- Bash environment variables do not persist between commands; re-establish state in each command or use settings-level environment configuration.
- The checkpoint system offers four rewind options (restore code and conversation, restore conversation only, restore code only, summarize from here), but checkpoints track only direct file edits -- not bash-command file operations; git commits are the only reliable universal rewind mechanism.
- The five named anti-patterns -- kitchen sink session, correcting over and over, over-specified CLAUDE.md, trust-then-verify gap, infinite exploration -- each have specific, immediate fixes.
- The 33-day autonomous trading experiment demonstrates every category of agent failure in one continuous arc: overconfidence, correlation risk, concentration risk, governance paradoxes, and the fundamental unreliability of confident output.
- "Vibe trading" extends the vibe coding trap to non-coding domains -- false confidence in agent-generated results is more dangerous when feedback loops are slow and stakes are financial.
- Invest in verification infrastructure (tests, types, linting) to increase first-attempt success rates -- automated feedback is the highest-leverage improvement.
- Use the slot machine approach for stuck tasks: commit, time-box to thirty minutes, accept-or-restart -- starting over often beats trying to fix mistakes mid-stream.
- Claude defaults to over-complex solutions; explicit simplicity constraints in prompts and CLAUDE.md produce meaningfully better output.
- Every recovery strategy depends on frequent commits -- without checkpoint discipline, your only option is correction, and correction is the least reliable path.

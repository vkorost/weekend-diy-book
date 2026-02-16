# Claude Code Book — Agent Team Assignment

## Overview

Produce an original book about Claude Code — Anthropic's agentic coding tool — from raw source materials in the working directory. Source materials include official documentation, usage examples, industry reports, and future-state analysis in `.md`, `.txt`, `.rtf`, `.pdf`, and `.docx` formats. The book synthesizes these into a single coherent narrative. The output is an EPUB file.

This is original composition from research materials — NOT a rewrite of an existing book. Claude Code writes the book. Vladimir Korostyshevskiy wrote the instructions.

**CRITICAL CONTEXT CONSTRAINT:** The source material corpus is too large for any single agent to read in full. The orchestrator MUST NOT attempt to read all source files. Instead, the orchestrator immediately spawns Research Agents to read and summarize subsets of source files in parallel. All source reading happens in sub-agents, never in the orchestrator.

---

## Critical Legal Constraints

These are non-negotiable and override all other instructions:

1. **No company names.** Do not mention any specific company, firm, hedge fund, bank, exchange, or vendor by name. Replace with generic descriptions: "a major hedge fund," "a large technology company," "one cloud provider," etc.
2. **No product names other than Claude Code and Anthropic.** Do not name competing products, third-party tools, IDEs, or platforms. Describe them generically: "a popular code editor," "a major version control platform," etc. Exception: Claude Code, Claude, and Anthropic may be named freely.
3. **No direct quotation from source materials.** Everything must be fully paraphrased and synthesized. No sentence should be traceable to a single source document.
4. **No attribution to specific individuals** other than the author credit. No "according to [person]" or "as [researcher] found."
5. **No URLs, links, or page references** in the book body.

These constraints exist to eliminate copyright liability and allow free distribution of the resulting EPUB.

---

## Target Audience

The reader has been using Claude Code for months. They know how to install it, start a session, ask it to write code, and commit the result. They do NOT need:
- Installation instructions
- "What is an LLM" context
- Basic workflow walkthroughs (write code, debug, commit)
- Explanations of what a terminal is

They DO need:
- Features and patterns they probably haven't discovered yet
- Agentic capabilities that emerged recently and changed what's possible
- Mental model shifts that separate someone who uses Claude Code from someone who is genuinely productive with it
- Advanced configuration, orchestration, and integration patterns
- Understanding of how to push the tool to its limits and recognize when they've hit them

Write for someone who already drives the car. This book teaches them racing lines, engine tuning, and when to draft.

---

## Step 0: Orchestrator Bootstrap (CONTEXT-SAFE)

The orchestrator does NOT read source files. It performs only coordination tasks:

### 0.1 — Inventory source files (lightweight)

Run a shell command to list all source files in the current directory and subdirectories. Supported formats: `.md`, `.txt`, `.rtf`, `.pdf`, `.docx`. Store the file list. Count the files. Do NOT read their contents.

```bash
find . -type f \( -name "*.md" -o -name "*.txt" -o -name "*.rtf" -o -name "*.pdf" -o -name "*.docx" \) | sort
```

### 0.2 — Create directory structure

```bash
mkdir -p working/research working/converted chapters
```

### 0.3 — Convert all source files to Markdown

Before any agent reads any source file, convert everything to `.md` in `working/converted/`. This is a mechanical step — no reading or comprehension, just format conversion. The converted files become the canonical source for all downstream agents. Original files are never modified.

**Conversion commands by format:**

```bash
# .md and .txt — just copy (txt renamed to .md)
cp source.md working/converted/source.md
cp source.txt working/converted/source.txt.md

# .pdf — extract text to markdown
# Try pdftotext first (poppler-utils), fall back to python pdfminer
pdftotext source.pdf working/converted/source.pdf.md
# If pdftotext unavailable:
pip install pdfminer.six --break-system-packages 2>/dev/null
python3 -c "from pdfminer.high_level import extract_text; print(extract_text('source.pdf'))" > working/converted/source.pdf.md

# .docx — convert to markdown via pandoc
pandoc source.docx -t markdown -o working/converted/source.docx.md
# If pandoc unavailable, use python-docx:
pip install python-docx --break-system-packages 2>/dev/null
python3 -c "
import docx, sys
doc = docx.Document('source.docx')
for p in doc.paragraphs:
    print(p.text)
" > working/converted/source.docx.md

# .rtf — convert to markdown
# Try pandoc first
pandoc source.rtf -t markdown -o working/converted/source.rtf.md
# If pandoc unavailable, try unrtf
unrtf --text source.rtf > working/converted/source.rtf.md
# If neither works, try textutil (macOS only)
textutil -convert txt source.rtf -output working/converted/source.rtf.md
```

**Install conversion tools if needed:**

```bash
# Linux
sudo apt-get install -y pandoc poppler-utils 2>/dev/null
# macOS
brew install pandoc poppler 2>/dev/null
# Python fallbacks (always available)
pip install pdfminer.six python-docx --break-system-packages 2>/dev/null
```

**Process:** Loop through every file from the inventory. Convert each to `.md` in `working/converted/` using the appropriate command. If all conversion methods fail for a specific file, log it to `working/conversion_failures.txt` and move on. After conversion, re-list `working/converted/` to get the canonical file list for research agents.

**From this point forward, all agents read ONLY from `working/converted/`, never from the original source files.**

### 0.4 — Write the style guide

Write `working/style_guide.md` from the spec in this document (see Style Guide section below). This is static content — no source reading required.

### 0.5 — Write the chapter plan

Write `working/chapter_plan.md` from the spec in this document (see Chapter Structure section below). This is the initial plan — it will be refined after research completes.

### 0.6 — Spawn Research Agents (IMMEDIATELY)

Divide the source file list into 4-5 roughly equal groups. Spawn one Research Agent per group. Each agent runs in parallel. Proceed to Step 1.

---

## Step 1: Research Phase (4-5 Research Agents in parallel)

### Each Research Agent receives:
- Its assigned subset of file paths from `working/converted/` (all files are already `.md` format)
- The chapter plan from `working/chapter_plan.md`
- Instructions on output format

### Each Research Agent does:

1. Read each `.md` file in its assigned subset from `working/converted/`. All format conversion has already been done in Step 0.3 — every file is plain markdown text.
2. For EACH source file, produce a structured summary written to `working/research/summary-<agent-id>-<file-number>.md`:

```markdown
## File: <filename>
### Category: DOCUMENTATION | EXAMPLES | ANALYSIS | OTHER
### Key Concepts:
- <concept 1>: <one-sentence explanation>
- <concept 2>: <one-sentence explanation>
- ...
### Best Chapter Fit: <chapter number(s) from the plan where this content belongs>
### Notable Details:
<Anything especially important, surprising, or unique in this file. Advanced techniques, non-obvious patterns, recent features, practical discoveries. 3-5 sentences max.>
```

3. After processing all files, produce a consolidated summary written to `working/research/consolidated-<agent-id>.md`:

```markdown
# Research Summary — Agent <id>

## Files Processed: <count>
## Files Skipped (unreadable): <list if any>

## Concepts by Chapter

### Chapter 1: You're Still Using It Wrong
- <concept>: <what the sources say, synthesized across files>
- ...

### Chapter 2: ...
(repeat for each chapter in the plan)

## Cross-Cutting Findings
- <anything that doesn't fit neatly into one chapter>
- <contradictions between sources>
- <concepts found in examples but NOT in documentation>
- <concepts in documentation but NOT in examples>
```

### Research Agent constraints:
- Do NOT include verbatim quotes from sources. Paraphrase everything.
- Do NOT include company names, product names (other than Claude Code/Anthropic), or individual names in summaries.
- Keep each per-file summary under 300 words. Keep the consolidated summary under 2000 words.

---

## Step 2: Orchestrator Synthesis (runs after ALL Research Agents complete)

The orchestrator reads ONLY the consolidated summaries from `working/research/consolidated-*.md`. These are small enough to fit in context.

### 2.1 — Create `working/knowledge_map.md`

Merge all consolidated summaries into a single knowledge map:
- Every distinct concept, organized by chapter
- Flag contradictions between research agents' findings
- Flag concepts that appear in examples but not documentation (highlight — practical discoveries)
- Flag concepts in documentation but not examples (underexplored features)
- Note which chapters have rich source material and which are thin

### 2.2 — Create `working/dedup_registry.md`

Master list of every concept that will appear in the book, with the SINGLE chapter where it will be covered. No concept appears in more than one chapter as primary coverage. A chapter may briefly reference a concept covered elsewhere (one sentence max, with "as covered in Chapter N") but never re-explain it.

### 2.3 — Refine `working/chapter_plan.md`

Based on what the research actually found:
- If a planned chapter has thin source material, merge it into an adjacent chapter or drop it
- If a topic has unexpectedly rich material, consider splitting it
- Finalize the chapter list, titles, and content assignments
- Update `working/dedup_registry.md` if chapters changed

### 2.4 — Assign writers

Round-robin: Writer-A gets chapters 1,4,7,...; Writer-B gets 2,5,8,...; Writer-C gets 3,6,9,... Front matter goes to Writer-A as chapter 00.

### 2.5 — Spawn Writer + Editor agent pairs

Each Writer agent gets:
- `working/style_guide.md`
- `working/dedup_registry.md`
- `working/chapter_plan.md` (their assigned chapters only)
- The specific `working/research/summary-*` files relevant to their chapters (NOT the raw source files)
- The relevant sections from `working/knowledge_map.md`

---

## Step 3: Writing Phase (3 Writer Agents + 3 Editor Agents)

### Layer 1: Writers (3 agents, running in parallel)

Each writer produces `chapters/NN-<slug>.md` per assigned chapter.

#### Chapter file structure:

```markdown
# Chapter N: <Title>

## What You'll Learn

<2-4 paragraphs. What this chapter covers, why it matters, what the reader will be able to do after reading it. Written with conviction. No "in this chapter we will explore" framing. Assume the reader is deciding whether this chapter is worth 15 minutes.>

---

<Full chapter body>

---

## Key Takeaways

<3-7 bullet points. Most important things from this chapter. Each bullet is one sentence, concrete, actionable where possible.>
```

#### Writer constraints:

- **Deduplication is mandatory.** Before writing any concept, check `dedup_registry.md`. If the concept belongs to another chapter, write at most one sentence: "The permission model (Chapter 2) governs what Claude Code can do without asking."
- **No company names, no product names (except Claude Code/Anthropic), no individual names.**
- **Synthesize, don't summarize.** Do not walk through research summaries one by one. Merge into unified explanations.
- **Examples must be generic.** If a research summary mentions a specific company's use case, generalize it.
- **Do not explain basics.** No installation, no "what is an LLM," no basic workflows. Start where the experienced reader's knowledge ends.
- **Write from research summaries only.** Do NOT read original source files — they will blow the context window.

Writers process chapters sequentially (Writer-A does ch1, then ch4, then ch7). Paired editor begins reviewing ch1 while Writer-A drafts ch4.

---

### Layer 2: Editors (3 agents, each paired 1:1 with a writer)

Review criteria:

1. **Legal compliance** — no company names, no product names (except Claude Code/Anthropic), no individual names, no direct quotation, no URLs. FIRST check. Hard fail.
2. **Deduplication compliance** — no concept explained in a chapter where `dedup_registry.md` assigns it elsewhere.
3. **Completeness** — chapter covers everything the knowledge map and chapter plan assign to it.
4. **Fluff removal** — no filler, redundancy, hedge-stacking, throat-clearing. No "it's important to note" or "let's dive in" or "in conclusion."
5. **Voice compliance** — Gladwell × Lopp hybrid. Not documentation, not a blog post, not default AI output.
6. **Audience calibration** — nothing a 3-month user already knows. Every paragraph earns its place.
7. **Clarity** — every idea lands on first read. Jargon unpacked on first use.

**Revision loop:** Max 3 cycles. If still failing, escalate to orchestrator.

---

## Step 4: Review Phase (3 Review Agents, after ALL chapters pass editorial)

- **Reviewer-Legal** — checks every sentence for legal constraint violations. One violation fails the chapter.
- **Reviewer-Dedup** — reads entire manuscript for any concept explained in more than one chapter. Cross-references OK, duplicate explanations fail.
- **Reviewer-Quality** — engagement, clarity, voice, completeness, audience calibration. Also checks coherence across the whole book.

Each reviewer produces:

```
Chapter NN: [PASS/FAIL]
Issues: <specific problems if FAIL>
```

Every chapter must PASS all three. Failures return to writer/editor pair. Only failing reviewer(s) re-review after revision.

---

## Step 5: Publisher (1 agent, after quality gate passes)

1. Verify all chapters present and in correct order.
2. Create `metadata.yaml`:
   ```yaml
   ---
   title: "Claude Code: The Definitive Guide to Agentic Development"
   author: "Written by Claude Code, following instructions by Vladimir Korostyshevskiy"
   lang: en
   toc: true
   toc-depth: 2
   ---
   ```
3. Create `combined.md` — all chapter files concatenated in order with `\newpage` between chapters.
4. Install pandoc if needed.
   - Windows: `winget install --id JohnMacFarlane.Pandoc -e --silent` (installs to `C:\Users\<username>\AppData\Local\Pandoc\pandoc.exe`)
   - Linux/macOS: `sudo apt-get install -y pandoc` or `brew install pandoc`
5. Generate EPUB:
   ```bash
   pandoc metadata.yaml combined.md \
     -o "Claude-Code-Definitive-Guide.epub" \
     --toc --toc-depth=2 --split-level=1
   ```
   Note: Use `--split-level=1` (not `--epub-chapter-level` which is deprecated in Pandoc 3.x). If `--split-level` is not recognized, fall back to `--epub-chapter-level=1`.
6. Verify EPUB exists and report file size.
7. Generate `quick_reference.md` — standalone cheat sheet of the most important commands, patterns, and configuration options.

---

## Chapter Structure (Initial Plan — Orchestrator refines after research)

Adjust based on what research agents find. If sources are thin, merge or drop. Dense 10-chapter book beats thin 15-chapter book.

### Part I: The Mental Model You're Missing

**Chapter 1: You're Still Using It Wrong**
The gap between how most people use Claude Code and what it actually is. The agentic execution model: read, plan, act, verify. Context window mechanics and session strategy. The difference between "telling it what to type" and "telling it what to accomplish."

**Chapter 2: The Permission and Trust Architecture**
The full trust model at each permission level. Security surface area. Network access controls. What data leaves your machine. Using Claude Code with sensitive or proprietary code.

**Chapter 3: Context Engineering**
How Claude Code builds context from your codebase. What it sees and what it can't. CLAUDE.md files — exactly what to put in them, what to leave out, how instruction files compose. The hierarchy: global, user, project, session.

### Part II: Advanced Capabilities

**Chapter 4: Multi-Agent Orchestration**
Spawning sub-agents. Parallel workstreams. Headless mode. Decomposing large tasks into agent-coordinated subtasks. Practical limits: when multi-agent helps and when it burns tokens.

**Chapter 5: MCP — Connecting Claude Code to Everything**
Model Context Protocol in practice. Connecting to external data sources and tools. Setting up MCP servers. Building custom integrations. Claude Code that sees your entire work environment, not just code.

**Chapter 6: CI/CD and Headless Automation**
Claude Code with no human at the terminal. Automated code review, translation, documentation, test creation, PR triage. Scripting interface: piping, chaining, composing. Unix philosophy applied to an AI agent.

**Chapter 7: IDE Integration Done Right**
VS Code extension, JetBrains integration. What changes from terminal-native to editor-native. Configuration for effectiveness. When to use IDE, when to drop to terminal.

### Part III: Patterns That Actually Work

**Chapter 8: Prompt Craft for Agentic Tools**
Not chatbot prompting. Specifying outcomes vs. steps. Constraints that guide without restricting. Anti-patterns: vague instructions, over-specification, missing context, asking for plans instead of execution.

**Chapter 9: Working with Large and Legacy Codebases**
Context structuring for large projects. Pointing Claude Code at the right files. Preventing hallucinated APIs. Incremental modernization with agentic assistance. Mapping unfamiliar code before modifying it.

**Chapter 10: Failure Modes and Recovery**
Looping behavior. Hallucinated paths, APIs, dependencies. Context exhaustion mid-task. When to continue vs. interrupt. Salvaging partial work. Structuring tasks to minimize blast radius.

### Part IV: Making It Organizational

**Chapter 11: Team Adoption Patterns**
Individual to team usage. Shared CLAUDE.md conventions. Standardized prompt patterns. Security and compliance for organizations. Enterprise deployment. Code review when AI writes most code.

**Chapter 12: The Economics and Strategy of AI-Assisted Development**
What changes when AI writes most code. Team restructuring. Writing code vs. specifying intent. Skills that gain value, skills that lose. Demo-ready vs. production-ready AI coding.

### Back Matter

**Appendix A: Command Reference** — Complete CLI reference. Terse, scannable, no prose.

**Appendix B: Configuration Reference** — All config files, settings, env vars, hierarchy. Terse, scannable, no prose.

**Appendix C: Troubleshooting** — Common errors, actual causes, fixes. By symptom. Terse, scannable, no prose.

---

## Style Guide (write verbatim to `working/style_guide.md`)

### Voice: Gladwell × Lopp

**Malcolm Gladwell** — Narrative-driven explanations. Counterintuitive framing. Analogies from unexpected domains. "Here's what you think you know — here's what's actually true." Connect unrelated things to reveal deeper patterns. Tell the reader why a feature exists and what problem it solved that they didn't know they had.

**Michael Lopp (Rands)** — Direct, slightly irreverent, tech-literate. Short paragraphs. Declarative sentences. No corporate speak, no weasel words, no passive voice hedging. Broken is broken. Brilliant gets one sentence explaining why. The reader is a peer, not a student.

### What this voice is NOT:

- Not documentation. No "this feature allows you to..."
- Not a blog post. No "In this chapter, we'll explore..." or "Let's dive in."
- Not a TED talk. No inspirational buildup to a revelation.
- Not default AI output. No "it's important to note" or "it's worth mentioning" or "in conclusion."
- Not Wikipedia. No neutral encyclopedic tone.
- Not marketing. No "powerful," "seamless," "cutting-edge," or "game-changing."

### Content Rules

- **Synthesize from research summaries.** Merge into unified explanations. Reader should never detect which source a paragraph came from.
- **Remove fluff ruthlessly.** Filler, redundancy, hedge-stacking, throat-clearing — gone.
- **Do not explain basics.** Start where the experienced reader's knowledge ends.
- **Clarify advanced concepts thoroughly.** Unpack recent and advanced capabilities. Calibrate for working developers, not beginners.
- **No dumbing down.** Every paragraph earns its place by teaching something new.
- **Concrete over abstract.** Show the command. Give the scenario. No floating abstractions.
- **Prioritize the recent and non-obvious.** Recently released capabilities + patterns not in getting-started guide = most valuable content.
- **Each chapter stands alone** if read independently, while benefiting from the arc.
- **"What You'll Learn" sells the chapter.** Make the reader want to read it.

---

## Constraints

- Do not modify any source material files.
- Do not ask for confirmation between steps — full pipeline runs autonomously.
- **No single agent reads more than its assigned subset of source files.** Hard architectural constraint to prevent context exhaustion.
- All writing agents access `working/style_guide.md`, `working/dedup_registry.md`, `working/chapter_plan.md`, and `working/knowledge_map.md`.
- Orchestrator logs progress after each milestone.
- Appendices: terse, scannable, no voice treatment.

---

## Execution Order Summary

```
[Orchestrator] List files (shell cmd) → mkdir → convert ALL sources to .md → write style_guide → write chapter_plan
        │
        │  (IMMEDIATELY — orchestrator does NOT read converted files)
        │
        ├── [Research-1] reads converted file group 1, writes summaries
        ├── [Research-2] reads converted file group 2, writes summaries
        ├── [Research-3] reads converted file group 3, writes summaries
        ├── [Research-4] reads converted file group 4, writes summaries
        └── [Research-5] reads converted file group 5, writes summaries
                    │
                    ▼ (all research complete)
[Orchestrator] Reads ONLY consolidated summaries → knowledge_map → dedup_registry → refine chapter_plan → assign writers
        │
        ├── [Writer-A + Editor-A] chapters 1, 4, 7, 10, ...
        ├── [Writer-B + Editor-B] chapters 2, 5, 8, 11, ...
        └── [Writer-C + Editor-C] chapters 3, 6, 9, 12, ...
                    │
                    ▼ (all chapters editor-approved)
        ┌───────────┼───────────┐
   [Reviewer-Legal] [Reviewer-Dedup] [Reviewer-Quality]
                    │
                    ▼ (all chapters PASS all reviewers)
              [Publisher] → EPUB + quick_reference.md
```

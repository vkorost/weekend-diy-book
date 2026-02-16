# Claude Code: The Definitive Guide to Agentic Development

A technical book about Claude Code — written entirely by Claude Code in one weekend.

## Download

- **[EPUB](Claude-Code-Definitive-Guide.epub)** — for e-ink readers (Kindle, Kobo, reMarkable, Boox)
- **[PDF](Claude-Code-Definitive-Guide.pdf)** — for screens and printing
- **[Combined Markdown](combined.md)** — read it on GitHub directly

## What This Is

A book for experienced Claude Code users who want to get significantly better. It covers agentic orchestration, MCP integrations, CI/CD automation, prompt craft for agentic tools, large codebase strategies, failure recovery, team adoption, and the economics of AI-assisted development.

This is not a beginner's guide. If you've never used Claude Code, start with the [official documentation](https://code.claude.com/docs). This book assumes months of daily usage and teaches what you haven't figured out on your own.

## Who Made This

**Written by:** Claude Code (Anthropic's agentic coding tool)
**Instructed by:** Vladimir Korostyshevskiy

The book was produced using a multi-agent pipeline: research agents read and summarized source materials in parallel, writer agents composed chapters, editor agents reviewed against quality gates, and a review panel checked for consistency before the manuscript was assembled into EPUB.

Source material included Anthropic's official documentation and publicly available usage examples gathered by multiple AI systems. Usage examples emphasize front office trading technology environments. No company names, product names (other than Claude Code), or individual names appear in the text. No content is directly quoted from any source.

## Write Your Own

The most interesting file in this repo isn't the book — it's the assignment file.

**[claude_code_book_assignment.md](claude_code_book_assignment.md)** contains the complete instructions that produced this book. Fork this repo, replace the source materials with research on your own topic, adjust the chapter plan, and run the assignment in Claude Code. You'll have a book by end of day.

The process costs roughly $20–40 in API usage depending on book length — about what you'd pay to buy a technical book.

## Repository Contents

```
├── README.md                            # This file
├── LICENSE                              # CC0 1.0 Universal
├── claude_code_book_assignment.md       # Agent team assignment (the "recipe")
├── about_this_book.md                   # About page included in the book
├── metadata.yaml                        # EPUB/PDF build metadata
├── cover.jpg                            # Cover image
├── combined.md                          # Full manuscript in one file
├── Claude-Code-Definitive-Guide.epub    # Ready to download
├── Claude-Code-Definitive-Guide.pdf     # Ready to download
├── quick_reference.md                   # Command/config cheat sheet
└── chapters/                            # Individual chapter files
    ├── 01-still-using-it-wrong.md
    ├── 02-permission-trust.md
    ├── 03-context-engineering.md
    ├── 04-multi-agent-orchestration.md
    ├── 05-mcp.md
    ├── 06-cicd-headless.md
    ├── 07-ide-integration.md
    ├── 08-prompt-craft.md
    ├── 09-large-legacy-codebases.md
    ├── 10-failure-modes.md
    ├── 11-team-adoption.md
    ├── 12-economics-strategy.md
    ├── 15-appendix-a-commands.md
    ├── 16-appendix-b-config.md
    └── 17-appendix-c-troubleshooting.md
```

## Rebuild the EPUB

If you modify any chapter files:

```bash
# Combine chapters
cat about_this_book.md > combined.md
for f in chapters/[0-9][1-9]*.md chapters/[1-9][0-9]*.md; do
  echo -e "\n\n" >> combined.md
  cat "$f" >> combined.md
done

# Generate EPUB
pandoc metadata.yaml combined.md \
  -o "Claude-Code-Definitive-Guide.epub" \
  --toc --toc-depth=2 --split-level=1 \
  --epub-cover-image=cover.jpg

# Generate PDF (requires xelatex or md-to-pdf)
md-to-pdf combined.md --pdf-options '{"format": "A4", "margin": {"top": "25mm", "bottom": "25mm", "left": "20mm", "right": "20mm"}}'
mv combined.pdf Claude-Code-Definitive-Guide.pdf
```

## License

This work is dedicated to the public domain under [CC0 1.0 Universal](LICENSE).

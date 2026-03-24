# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code skill** (not a traditional code project) that translates ePub files between languages using Claude itself as the translation engine. The entire "logic" is expressed as precise natural-language instructions in markdown files — there is no executable code, no build system, and no tests to run.

The skill is installed as a Superpowers plugin and triggered when users ask to translate an ePub (e.g., "translate this ePub to Chinese", "翻译电子书").

## Architecture

```
awesome-epub-translator-skill/     # Skill package directory
├── SKILL.md                       # Main skill definition: frontmatter, 10-step workflow, error handling
├── references/
│   ├── translation-prompt.md      # Translation prompt template with {STYLE_PROFILE}, {TARGET_LANGUAGE} placeholders
│   └── epub-structure.md          # ePub format quick reference (container.xml → content.opf → XHTML)
└── rules/
    └── install.md                 # Platform-specific install instructions for zip/unzip
```

- **SKILL.md** is the entry point. Its YAML frontmatter (`name`, `description`) controls skill triggering. The body contains the complete 10-step workflow Claude follows when translating.
- **references/** files are read by Claude during execution (e.g., `translation-prompt.md` is re-read before every translation batch to maintain rule adherence).
- **rules/** files are consulted when prerequisites are missing.

## Key Design Decisions

- **Zero external dependencies**: Only requires `unzip` (extraction) and either `zip` or Python 3 (repackaging). Claude reads/writes files directly.
- **Session management**: Translates max 3 XHTML files per conversation session, then pauses. Prevents context window exhaustion on large books.
- **Checkpoint-based resumability**: Translated files are saved to `_translated/` directory within the work dir. Re-running the skill on the same ePub skips already-translated files.
- **Style Profile system**: Before translating, Claude analyzes the book's genre/tone/voice and saves a profile to `_translated/style_profile.md`. This is injected into every batch prompt for consistent tone across sessions.
- **Bilingual mode**: Optionally interleaves original + translated blocks with `class="translated"` styling.

## Working on This Project

Since this is a markdown-only skill, "development" means editing the instruction files:

- **Changing workflow behavior**: Edit `SKILL.md` sections (Steps 1-10)
- **Changing translation rules**: Edit `references/translation-prompt.md`
- **Changing ePub format handling**: Edit `references/epub-structure.md`
- **Testing**: Invoke the skill with a real ePub file (e.g., `C:\Users\luden\Downloads\python-workout-second-edition.epub`) and verify the output

## Docs

- `docs/superpowers/specs/` — Design spec with rationale for all major decisions
- `docs/superpowers/plans/` — Implementation plan with chunked tasks

## Conventions

- All markdown instruction files use English
- The skill directory name must match the frontmatter `name` field pattern for Superpowers skill discovery
- Cross-references between skill files use relative paths (e.g., `references/translation-prompt.md`)

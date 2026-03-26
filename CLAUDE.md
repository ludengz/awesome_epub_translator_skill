# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin** that translates ePub files between languages using Claude itself as the translation engine. The entire "logic" is expressed as precise natural-language instructions in markdown files — there is no executable code, no build system, and no tests to run.

The plugin is installed via `claude --plugin-dir ./` and triggered when users ask to translate an ePub (e.g., "translate this ePub to Chinese", "翻译电子书").

## Architecture

```
.claude-plugin/
└── plugin.json                    # Plugin manifest: name, description, version
skills/
└── awesome-epub-translator/       # Skill package directory
    ├── SKILL.md                   # Main skill definition: frontmatter, 10-step workflow, error handling
    ├── references/
    │   ├── translation-prompt.md  # Translation prompt template with {STYLE_PROFILE}, {TARGET_LANGUAGE} placeholders
    │   └── epub-structure.md      # ePub format quick reference (container.xml → content.opf → XHTML)
    └── rules/
        └── install.md             # Platform-specific install instructions for zip/unzip
```

- **plugin.json** is the plugin manifest, declaring the plugin's name, description, and version for Claude Code's plugin loader.
- **SKILL.md** is the skill entry point. Its YAML frontmatter (`name`, `description`) controls skill triggering. The body contains the complete 10-step workflow Claude follows when translating.
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

- **Changing workflow behavior**: Edit `skills/awesome-epub-translator/SKILL.md` sections (Steps 1-10)
- **Changing translation rules**: Edit `skills/awesome-epub-translator/references/translation-prompt.md`
- **Changing ePub format handling**: Edit `skills/awesome-epub-translator/references/epub-structure.md`
- **Testing**: Invoke the skill with a real ePub file (e.g., `C:\Users\luden\Downloads\python-workout-second-edition.epub`) and verify the output

## Docs

- `docs/superpowers/specs/` — Design spec with rationale for all major decisions
- `docs/superpowers/plans/` — Implementation plan with chunked tasks

## Conventions

- All markdown instruction files use English
- The plugin name in `plugin.json` should match the skill directory name for consistency
- Cross-references between skill files use relative paths (e.g., `references/translation-prompt.md`)

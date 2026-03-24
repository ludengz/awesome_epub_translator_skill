# Awesome ePub Translator Skill

A [Claude Code](https://claude.ai/code) skill that translates ePub files between languages using Claude as the translation engine. Zero external dependencies beyond `unzip` and `zip`/Python 3.

## Features

- **High-quality AI translation** — Claude translates with context awareness, not word-by-word
- **Structure preservation** — all formatting, inline HTML tags, images, styles, and CSS remain intact
- **Style Profile system** — analyzes the book's genre, tone, and voice before translating, ensuring consistent style across the entire book
- **Bilingual mode** — optionally outputs original + translated text side by side
- **Checkpoint-based resumability** — large books can be translated across multiple conversation sessions; progress is never lost
- **ePub 2 & 3 support** — handles both `toc.ncx` and `toc.xhtml` navigation formats
- **DRM detection** — warns and stops if the ePub is DRM-protected

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- `unzip` available in your shell
- Either `zip` or Python 3 for repackaging (Python is used as fallback on Windows)

### Install the skill

```bash
claude skill add /path/to/awesome-epub-translator-skill
```

Or clone this repo and add it:

```bash
git clone https://github.com/<your-username>/awesome-epub-translator-skill.git
claude skill add ./awesome-epub-translator-skill/awesome-epub-translator-skill
```

### Verify prerequisites

```bash
unzip --version
zip --version || python --version
```

See [rules/install.md](awesome-epub-translator-skill/rules/install.md) for platform-specific instructions.

## Usage

In a Claude Code conversation:

```
Translate /path/to/book.epub to Chinese
```

```
翻译 /path/to/book.epub 到日语，用双语模式
```

### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| Source file | Yes | — | Path to the `.epub` file |
| Target language | Yes | — | e.g., `Chinese`, `Japanese`, `zh`, `ja` |
| Output mode | No | `pure` | `pure` (translated only) or `bilingual` (original + translated) |
| Output path | No | `<name>_<lang>.epub` | Custom output file path |
| Tone/style | No | auto-detect | Style hint, e.g., `"formal and academic"` |

## How It Works

1. **Validate & extract** — unzips the ePub, verifies structure, checks for DRM
2. **Analyze style** — builds a Style Profile from metadata + first-chapter analysis (or uses your hint)
3. **Translate chapters** — processes XHTML files in spine order, batching by semantic boundaries with context carry-over between batches
4. **Translate TOC & metadata** — localizes navigation, book title, and description (author names are preserved)
5. **Repackage** — assembles a valid ePub with correct `mimetype` entry and ZIP structure

Translated files are checkpointed to a `_translated/` directory. If a session ends mid-book, just re-invoke the skill on the same ePub — it picks up where it left off.

## Limitations

- Large books (>50 chapters) require multiple conversation sessions (3 chapters per pause)
- Text embedded in images or SVG is not translated
- Fixed-layout ePub translations may have layout issues
- Inline tag preservation is best-effort; complex nesting may occasionally break
- `<ruby>`/`<rt>` annotations are preserved as-is

## Project Structure

```
awesome-epub-translator-skill/     # Skill package
├── SKILL.md                       # Main skill definition (10-step workflow)
├── references/
│   ├── translation-prompt.md      # Translation prompt template
│   └── epub-structure.md          # ePub format quick reference
└── rules/
    └── install.md                 # Platform-specific install instructions
```

## License

MIT

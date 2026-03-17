---
name: epub-translator
description: >
  Translate an ePub file from one language to another with high-quality AI translation.
  Preserves formatting, inline HTML tags, images, and styles.
  Supports pure translation and bilingual (original + translated) output modes.
  Use when the user asks to: translate an ePub, translate an e-book, translate a book,
  翻译电子书, 翻译ePub, convert ePub to another language.
---

# ePub Translator

Translate ePub files between languages while preserving the original structure, formatting, and visual appearance.

## Parameters

The user provides these when invoking the skill:

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| Source file | Yes | Path to the .epub file | `/path/to/book.epub` |
| Target language | Yes | Language to translate into | `Chinese`, `Japanese`, `zh`, `ja` |
| Output mode | No (default: pure) | `pure` = translated only, `bilingual` = original + translated | `bilingual` |
| Output path | No (default: auto) | Custom output file path | `/path/to/output.epub` |
| Tone/style | No (default: auto-detect) | Style hint for translation | `"formal and academic"` |

If the user doesn't specify all required parameters, ask for them before proceeding.

## Prerequisites

- `zip` and `unzip` must be available in the shell (see `rules/install.md`)
- Verify with: `which zip && which unzip` (or `where zip` on Windows cmd)
- If not found, guide the user to install per `rules/install.md`

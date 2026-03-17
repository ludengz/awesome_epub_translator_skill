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

## Workflow

Follow these steps in exact order. Do not skip steps. Read `references/epub-structure.md` before starting if you need a refresher on ePub internals.

### Step 1: Validate Input

1. Confirm the source file exists: `ls -la "<source_path>"`
2. Confirm it has a `.epub` extension
3. If invalid, report the error clearly and STOP

### Step 2: Create Work Directory

1. Derive work directory name: `<filename_without_extension>_translation_work/` in the same directory as the source file
2. Create it: `mkdir -p "<work_dir>"`
3. If `_translated/` subdirectory already exists inside it, this is a **resumed session** — report how many checkpoint files exist

### Step 3: Unzip ePub

1. Run: `unzip -o -q "<source_path>" -d "<work_dir>/"`
2. Verify: `ls "<work_dir>/META-INF/container.xml"`
3. If container.xml doesn't exist, report "Invalid ePub: missing META-INF/container.xml" and STOP

**Error checks before proceeding:**
- Check for DRM: `ls "<work_dir>/META-INF/encryption.xml"` — if it exists, read it. If it contains `<EncryptedData>` referencing XHTML files, report "This ePub appears to be DRM-protected and cannot be translated" and STOP.

### Step 4: Locate content.opf

1. Read `<work_dir>/META-INF/container.xml` with the Read tool
2. Find the `full-path` attribute in the `<rootfile>` element
3. This gives you the path to `content.opf` relative to the work directory (e.g., `OEBPS/content.opf`)
4. Read `content.opf` with the Read tool
5. Store the content directory prefix (e.g., `OEBPS/`) — you'll need it for all subsequent file paths

### Step 5: Extract File Manifest

From `content.opf`, extract:

1. **XHTML content files**: All `<item>` elements in `<manifest>` with `media-type="application/xhtml+xml"`. Record their `id` and `href` attributes.
2. **Spine order**: Read `<itemref>` elements in `<spine>`. Map each `idref` to the corresponding manifest item's `href`. This is the reading/translation order.
3. **TOC files**:
   - ePub 2: `<item>` with `media-type="application/x-dtbncx+xml"` → toc.ncx
   - ePub 3: `<item>` with `properties="nav"` → toc.xhtml
4. **Metadata**: Extract `dc:title`, `dc:language`, `dc:description`, `dc:subject`, `dc:creator`
5. **CSS files**: All `<item>` elements with `media-type="text/css"` — record for bilingual CSS injection
6. **Fixed-layout check**: Look for `<meta property="rendition:layout">pre-paginated</meta>` or `<meta name="fixed-layout" content="true"/>`. If found, warn: "This is a fixed-layout ePub. Translation will proceed but layout may be affected."

Report to the user:
- Book title, author, source language
- Number of content files to translate
- Whether TOC files were found (ncx, xhtml, or both)
- Whether this is a resumed session (how many already translated)

### Step 5.5: Establish Translation Style Profile

**Check for existing profile first:** If `<work_dir>/_translated/style_profile.md` exists, read it and use it. Report: "Using existing style profile from previous session." Skip to Step 6.

**If no existing profile**, build one from these sources (in priority order):

1. **User-specified tone** (if provided): Use the user's `--tone` or style hint as the primary directive.

2. **Metadata inference**: From `dc:description`, `dc:subject`, and `dc:title` already extracted in Step 5, analyze:
   - What genre/category is this book?
   - What audience is it for?
   - What tone would be appropriate?

3. **First-chapter style anchor**: Read the first 2-3 paragraphs of meaningful text from the first XHTML content file in spine order (skip cover pages — look for the first file with substantial `<p>` content). Analyze:
   - Vocabulary complexity (technical jargon vs. everyday language)
   - Sentence structure (short and punchy vs. long and complex)
   - Formality level (academic, conversational, casual)
   - Narrative voice (first/second/third person)
   - Use of humor, metaphor, rhetorical devices

**Combine all sources into a Style Profile** and save to `<work_dir>/_translated/style_profile.md`:

```
Genre: [genre]
Tone: [tone description]
Voice: [person and address style]
Vocabulary: [vocabulary approach]
Style notes: [2-3 specific observations about the writing style]
```

This profile will be injected into every translation batch prompt. Show it to the user and ask: "Does this style profile look right? Adjust if needed, or confirm to proceed."

# ePub Translator Claude Skill — Design Spec

## Overview

A Claude Code skill that translates ePub files from one language to another with high-quality AI translation. The output ePub preserves the original structure, formatting, and visual appearance — only the language changes.

## Core Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Output mode | Both pure-translation and bilingual | Bilingual for learners, pure for readers. User selects via parameter. |
| Translation engine | Claude itself (no external API) | Zero config, Claude is already a high-quality translator. No API key needed. |
| Inline HTML handling | Preserve tags in-context | Claude translates text with HTML tags intact, guided by prompt rules. |
| Processing granularity | Multi-paragraph batches (~2000-3000 chars) | Balances context quality with manageable chunk size. |
| Resumability | Chapter-level file-based checkpoints | Translated XHTML files saved to `_translated/` dir; already-done files skipped on re-run. |
| ePub processing tool | Pure shell (zip/unzip) | Zero external dependencies. Claude reads/writes files directly. |
| Metadata translation | Translate everything | Book title, TOC entries, chapter headings, dc:language — full localization. |

## Skill File Structure

```
epub-translator/
├── SKILL.md                    # Skill definition and complete workflow instructions
├── references/
│   ├── translation-prompt.md   # Translation prompt template for each batch
│   └── epub-structure.md       # ePub format quick reference for Claude
└── rules/
    └── install.md              # Installation notes (only requires zip/unzip)
```

### SKILL.md

- **Frontmatter**: `name: epub-translator`, description covers trigger phrases ("translate an ePub", "翻译电子书", etc.)
- **Input parameters**: source ePub path, target language, output mode (pure/bilingual), output path (optional)
- **Complete 10-step workflow** (see below)
- **Translation rules**: preserve tags, skip code blocks, handle special content
- **Error handling**: guidance for common failure scenarios
- **Prohibitions**: don't modify CSS, don't remove images, don't alter file structure

### references/translation-prompt.md

Prompt template Claude follows for each translation batch:
- Target language specification
- HTML tag preservation rules with examples
- Non-translatable content list (code, URLs, proper nouns)
- Quality requirements (natural, fluent, idiomatic)

### references/epub-structure.md

Quick reference for ePub internals: container.xml → content.opf → XHTML file chain, metadata fields, TOC formats.

## Workflow (10 Steps)

### Step 1: Validate Input
- Confirm file exists and has `.epub` extension
- Report clear error and stop if invalid

### Step 2: Create Work Directory
- Create `<filename>_translation_work/` alongside the source file
- This directory persists for resumability

### Step 3: Unzip ePub
- `unzip -o <file>.epub -d <work_dir>/`
- Verify extraction succeeded

### Step 4: Locate content.opf
- Parse `META-INF/container.xml` to find `content.opf` path
- Read `content.opf`

### Step 5: Extract File Manifest
- From `content.opf`, identify:
  - All XHTML content files (from `<manifest>` items with media-type `application/xhtml+xml`)
  - Reading order (from `<spine>`)
  - TOC file(s): `toc.ncx` (ePub 2) and/or `toc.xhtml` (ePub 3)
  - Metadata: `dc:title`, `dc:language`, `dc:description`, `dc:creator`

### Step 6: Translate XHTML Files (Core Loop)
For each XHTML file in spine order:

1. **Check checkpoint**: If `_translated/<filename>` exists, skip (print "Skipping already translated: ...")
2. **Read** the XHTML file with the Read tool
3. **Identify translatable blocks**: `<p>`, `<h1>`-`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>` within `<body>`
4. **Batch blocks**: Group consecutive blocks into batches of ~2000-3000 characters, splitting at block element boundaries
5. **Translate each batch**: Following the translation prompt template:
   - Preserve all HTML tags (`<em>`, `<strong>`, `<a href="...">`, `<span>`, etc.)
   - Translate `alt` attributes on `<img>` tags
   - Skip `<code>`, `<pre>` content
   - Skip empty paragraphs, purely numeric content, URLs
6. **Reassemble**: Reconstruct the full XHTML with translated content
7. **Update language attributes**: `<html lang="...">` and `xml:lang` to target language
8. **Write checkpoint**: Save translated file to `_translated/<filename>`

**Bilingual mode variation** (Step 6.5):
- After translating each block, insert the translated block immediately after the original
- Add `class="translated"` to translated elements for optional CSS styling

### Step 7: Translate TOC
- `toc.ncx`: Translate `<navLabel><text>` content in each `<navPoint>`
- `toc.xhtml`: Translate anchor text in `<nav>` elements
- Save to `_translated/`

### Step 8: Update Metadata
- In `content.opf`:
  - `<dc:language>` → target language code (e.g., `zh`, `ja`, `ko`)
  - `<dc:title>` → translated title
  - `<dc:description>` → translated description (if present)
- Save updated `content.opf` to `_translated/`

### Step 9: Repackage ePub
- Copy all non-translated files (CSS, images, fonts, etc.) to output staging area
- Overlay translated files from `_translated/`
- Package as ZIP with correct ePub structure:
  - `mimetype` first, uncompressed: `zip -0 -X output.epub mimetype`
  - Everything else compressed: `zip -r output.epub META-INF/ OEBPS/ -x mimetype`
- Output filename: `<original_name>_<target_lang>.epub` (or user-specified path)

### Step 10: Cleanup
- Report completion with output file path and size
- Optionally remove work directory (ask user or keep for debugging)

## XHTML Translation Detail

### What Gets Translated
- Text content within block elements (`<p>`, `<h1>`-`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>`)
- `alt` attributes on `<img>` tags
- `title` attributes where present
- TOC navigation labels

### What Does NOT Get Translated
- `<code>` and `<pre>` content (source code)
- CSS class names and style attributes
- URLs and `href` values
- File paths and filenames
- Purely numeric or symbolic content
- Image file contents

### Inline Tag Preservation
Claude receives the full HTML of each block and translates text nodes while keeping tags intact. Example:

**Input:**
```html
<p>It was a <em>dark</em> and <strong>stormy</strong> night.</p>
```

**Output (Chinese):**
```html
<p>那是一个<em>漆黑</em>而<strong>狂风暴雨</strong>的夜晚。</p>
```

The translation prompt explicitly instructs Claude to:
- Keep all opening and closing tags paired correctly
- Adjust tag positions to match target language word order
- Never translate tag names or attribute names
- Preserve `href`, `src`, `id`, `class` attribute values unchanged

## Error Handling

| Scenario | Action |
|----------|--------|
| File not found | Report error, stop |
| Not an .epub file | Report error, stop |
| Unzip fails (corrupted) | Report error, stop |
| DRM-encrypted content | Detect encrypted files, warn user, stop |
| Fixed-layout ePub | Warn user that layout may break, continue |
| Empty paragraph / no text | Skip silently |
| Super-long paragraph (>3000 chars) | Treat as single batch |
| Translated file already exists | Skip (resumability) |
| Output file is 0 bytes | Report packaging error |

## Limitations (Documented in SKILL.md)

- Large books (>50 chapters) will take significant time
- Inline tag preservation is best-effort; complex nesting may occasionally break
- Text embedded in images is not translated
- Fixed-layout ePub translations may have layout issues
- SVG text elements are not translated in this version
- No parallel processing (sequential chapter-by-chapter)

## Test Plan

- Use `C:\Users\luden\Downloads\python-workout-second-edition.epub` as test file
- Verify: unzip/repackage integrity, metadata update, XHTML translation, TOC translation
- Verify: bilingual mode inserts translated blocks correctly
- Verify: resumability by interrupting and re-running
- Verify: output ePub opens correctly in an ePub reader

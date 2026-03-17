# ePub Translator Claude Skill — Design Spec

## Overview

A Claude Code skill that translates ePub files from one language to another with high-quality AI translation. The output ePub preserves the original structure, formatting, and visual appearance — only the language changes.

## Core Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Output mode | Both pure-translation and bilingual | Bilingual for learners, pure for readers. User selects via parameter. |
| Translation engine | Claude itself (no external API) | Zero config, Claude is already a high-quality translator. No API key needed. |
| Inline HTML handling | Preserve tags in-context | Claude translates text with HTML tags intact, guided by prompt rules. |
| Processing granularity | Multi-paragraph batches by semantic boundaries (~2000-3000 chars guideline) | Prioritizes complete HTML structures over strict char counts. |
| Session management | Max 3 XHTML files per session, then pause | Prevents context window exhaustion on large books; checkpoint enables cross-session continuation. |
| Resumability | Chapter-level file-based checkpoints | Translated XHTML files saved to `_translated/` dir; already-done files skipped on re-run. |
| ePub processing tool | Pure shell (zip/unzip) | Zero external dependencies. Claude reads/writes files directly. |
| Translation style | Style Profile system (user hint > metadata inference > first-chapter anchor) | Prevents "flat, neutral AI tone"; ensures genre-appropriate translation voice persisted across sessions. |
| Metadata translation | Translate everything except author names | Book title, TOC entries, chapter headings, dc:language — full localization. Author names (dc:creator) left as-is. |

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
- **Input parameters**: source ePub path, target language, output mode (pure/bilingual), output path (optional), tone/style hint (optional)
- **Complete 10-step workflow** (see below)
- **Translation rules**: preserve tags, skip code blocks, handle special content
- **Error handling**: guidance for common failure scenarios
- **Prohibitions**: don't remove images, don't alter file structure, don't modify existing CSS rules (only append bilingual styles if needed)

### references/translation-prompt.md

Prompt template Claude follows for each translation batch:
- **Global Style Profile** placeholder — injected from Step 5.5 (tone, genre, vocabulary level, style characteristics)
- Target language specification
- HTML tag preservation rules with examples
- Non-translatable content list (code, URLs, proper nouns)
- Quality requirements (natural, fluent, idiomatic, consistent with the established style profile)

### references/epub-structure.md

Quick reference for ePub internals: container.xml → content.opf → XHTML file chain, metadata fields, TOC formats.

### rules/install.md

- Requires `zip` and `unzip` commands available in the shell
- On Windows: available via Git Bash (bundled with Git for Windows), or install via `choco install zip unzip`
- On macOS/Linux: typically pre-installed

## Workflow (10 Steps)

### Step 1: Validate Input
- Confirm file exists and has `.epub` extension
- Report clear error and stop if invalid

### Step 2: Create Work Directory
- Create `<filename>_translation_work/` alongside the source file
- This directory persists for resumability

### Step 3: Unzip ePub
- `unzip -o -q <file>.epub -d <work_dir>/`
- Verify extraction succeeded (check exit code and that `META-INF/container.xml` exists)

### Step 4: Locate content.opf
- Parse `META-INF/container.xml` to find `content.opf` path
- Read `content.opf`

### Step 5: Extract File Manifest
- From `content.opf`, identify:
  - All XHTML content files (from `<manifest>` items with media-type `application/xhtml+xml`)
  - Reading order (from `<spine>`)
  - TOC file(s): `toc.ncx` (ePub 2) and/or `toc.xhtml` (ePub 3)
  - Metadata: `dc:title`, `dc:language`, `dc:description`, `dc:creator`

### Step 5.5: Establish Translation Style Profile

This step determines the tone and style that will guide all subsequent translation. The style profile is saved to `_translated/style_profile.md` and included in every translation batch prompt.

**Three sources of style information (in priority order):**

1. **User-specified tone** (highest priority): If the user provides a `--tone` or style hint (e.g., "humorous, technical, and approachable"), use it directly as the primary style directive.

2. **Metadata-driven inference**: Read `dc:description`, `dc:subject`, and `dc:title` from `content.opf`. Claude analyzes these to infer the book's genre and expected tone (e.g., "technical programming textbook with conversational style", "literary fiction with formal prose", "children's science book with playful tone").

3. **First-chapter style anchor**: Read the first 2-3 paragraphs of the first content XHTML file in spine order. Claude analyzes the writing style: vocabulary complexity, sentence length patterns, use of humor/slang/jargon, formality level, narrative voice (first/second/third person).

**Output**: Combine all available sources into a concise Style Profile (~3-5 sentences) and save to `_translated/style_profile.md`. Example:

```
Genre: Technical programming book
Tone: Casual, conversational, and encouraging
Voice: Second person ("you"), direct address to the reader
Vocabulary: Technical terms kept precise, explanations use everyday language
Style notes: Uses humor and real-world analogies. Short paragraphs. Rhetorical questions for engagement.
```

**Checkpoint**: If `_translated/style_profile.md` already exists (from a previous session), read and use it — do not regenerate. This ensures style consistency across sessions.

**Integration**: The style profile is injected at the top of every translation batch prompt as "Global Book Context", ensuring Claude maintains consistent tone throughout the entire book.

### Step 6: Translate XHTML Files (Core Loop)

**Session management**: To avoid context window exhaustion, translate at most **3 XHTML files per session**. After completing 3 files, pause and ask the user: "Translated 3/N chapters. Continue in this session, or start a new conversation to continue? (The `_translated/` checkpoint will pick up where we left off.)" This prevents token overflow on large books.

For each XHTML file in spine order:

1. **Check checkpoint**: If `_translated/<relative_path>` exists in the work directory, skip (print "Skipping already translated: ..."). The `_translated/` directory mirrors the full relative path structure of the original ePub (e.g., `_translated/OEBPS/text/chapter1.xhtml`).
2. **Read** the XHTML file with the Read tool. For very large files (>2000 lines), use offset/limit to read in segments.
3. **Identify translatable blocks**: `<p>`, `<h1>`-`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>` within `<body>`
4. **Batch blocks**: Group translatable content into batches based on **natural semantic boundaries** (sections, headings, logical paragraph groups) rather than strict character counts. Aim for roughly 2000-3000 characters per batch as a guideline, but prioritize keeping complete HTML structures intact — never split a parent element across batches. Nested structures (e.g., `<blockquote>` containing multiple `<p>`, or `<ol>` containing `<li>`) should be kept as a single unit. Include the last 2-3 translated paragraphs from the previous batch as context reference (marked as "previous context, do not re-translate") to maintain terminology consistency across batches.
5. **Translate each batch**: Following the translation prompt template:
   - Preserve all HTML tags (`<em>`, `<strong>`, `<a href="...">`, `<span>`, etc.)
   - Translate `alt` attributes on `<img>` tags
   - Skip `<code>`, `<pre>` content
   - Skip empty paragraphs, purely numeric content, URLs
6. **Reassemble**: Reconstruct the full XHTML with translated content
7. **Update language attributes**: `<html lang="...">` and `xml:lang` to target language
8. **Preserve file headers**: Keep XML declaration (`<?xml version="1.0" encoding="utf-8"?>`), DOCTYPE, and all namespace attributes on `<html>` tag unchanged
9. **Write checkpoint**: Save translated file to `_translated/<relative_path>` (preserving the original directory structure)

**Bilingual mode variation** (Step 6.5):
- After translating each block, insert the translated block immediately after the original
- Add `class="translated"` to translated elements for CSS styling
- **Auto-inject bilingual CSS**: In Step 9, if bilingual mode is active and the ePub has a main CSS file (referenced in `content.opf` manifest), append the following to the end of that CSS file:
  ```css
  .translated { color: #555555; font-size: 0.95em; margin-top: 0.2em; }
  ```
  If no CSS file exists, create a minimal `bilingual.css`, add it to the manifest, and reference it in all XHTML files.

### Step 7: Translate TOC
- **If both `toc.ncx` and `toc.xhtml` exist**, translate them in a single batch to guarantee identical translations for the same chapter titles across both files.
- `toc.ncx`: Translate `<navLabel><text>` content in each `<navPoint>`
- `toc.xhtml`: Translate based on `epub:type`:
  - `epub:type="toc"`: Translate all anchor text in the navigation list
  - `epub:type="landmarks"`: Translate landmark labels (e.g., "Cover" → "封面", "Table of Contents" → "目录")
  - `epub:type="page-list"`: Do NOT translate (page numbers are language-independent)
- Check checkpoint before translating (same `_translated/` mechanism as XHTML files)
- Save to `_translated/` preserving relative path

### Step 8: Update Metadata
- In `content.opf`:
  - `<dc:language>` → target language code (e.g., `zh`, `ja`, `ko`)
  - `<dc:title>` → translated title
  - `<dc:description>` → translated description (if present)
  - `<dc:creator>` → leave as-is (author names are not translated per publishing convention)
- Save updated `content.opf` to `_translated/`

### Step 9: Repackage ePub
- Create an output staging directory by copying the extracted ePub structure (excluding `_translated/`)
- Overlay translated files from `_translated/` onto the staging copy, specifically:
  - All translated XHTML content files
  - Translated TOC file(s) (toc.ncx, toc.xhtml)
  - Updated content.opf (with new metadata)
- All other files (CSS, images, fonts, META-INF/container.xml, etc.) remain unchanged from the original
- Package as ZIP with correct ePub structure:
  - `cd` into staging directory
  - `mimetype` first, uncompressed: `zip -0 -X output.epub mimetype`
  - Everything else: `zip -r output.epub * -x mimetype` (dynamically includes whatever directories exist — not hardcoded to `OEBPS/`)
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
| DRM-encrypted content | Check for `META-INF/encryption.xml`; if it exists and references XHTML content files, warn user and stop |
| Fixed-layout ePub | Check `content.opf` for `<meta property="rendition:layout">pre-paginated</meta>` or `<meta name="fixed-layout" content="true"/>`; warn user layout may break, continue |
| Non-UTF-8 encoding | Check XML declaration for encoding; warn if not UTF-8 |
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
- `<ruby>`/`<rt>` annotations: preserved as-is; not added or removed during translation
- Table layout may shift when translated text length changes significantly
- Resumability checks file existence only; if the source ePub changes between runs, delete `_translated/` to force re-translation
- Bilingual mode auto-injects minimal CSS (`.translated { color: #555; font-size: 0.95em; }`); users may customize further
- Large books require multiple conversation sessions due to context window limits (3 chapters per session, checkpoint-based continuation)

## Test Plan

- Use `C:\Users\luden\Downloads\python-workout-second-edition.epub` as test file
- Verify: unzip/repackage integrity, metadata update, XHTML translation, TOC translation
- Verify: bilingual mode inserts translated blocks correctly
- Verify: resumability by interrupting and re-running
- Verify: output ePub opens correctly in an ePub reader

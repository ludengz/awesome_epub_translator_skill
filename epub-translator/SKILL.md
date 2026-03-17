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

### Step 6: Translate XHTML Files (Core Loop)

**Session management:** Track how many XHTML files you translate in this session. After completing **3 files**, pause and report:

> "Translated 3/N chapters so far. The checkpoint is saved. You can:
> 1. Continue in this session
> 2. Start a new conversation — just invoke this skill again with the same ePub and I'll pick up where I left off."

This prevents context window exhaustion on large books.

**For each XHTML file in spine order:**

#### 6.1: Check Checkpoint
- If `<work_dir>/_translated/<relative_path>` exists, print "Skipping (already translated): <filename>" and move to the next file
- The `_translated/` directory mirrors the full ePub directory structure (e.g., `_translated/OEBPS/Text/chapter_01.xhtml`)

#### 6.2: Read the File
- Use the Read tool to read the XHTML file from the work directory
- For files >2000 lines, use offset/limit to read in segments
- Note the complete file structure: XML declaration, `<head>`, `<body>`

#### 6.3: Identify Translatable Content
Within `<body>`, identify all translatable block elements:
- `<p>`, `<h1>`–`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>`

Skip these entirely:
- `<pre>` and `<code>` blocks (source code)
- Empty elements or elements containing only whitespace
- Elements containing only numeric/symbolic content
- Elements containing only URLs

#### 6.4: Batch and Translate

Group translatable blocks into batches by **natural semantic boundaries** (sections, heading groups, logical paragraph clusters). Guidelines:
- Aim for ~2000-3000 characters per batch, but this is a guideline, not a hard limit
- **Never split a parent element across batches** (e.g., keep an entire `<blockquote>` or `<table>` together)
- Keep nested structures as single units (e.g., `<ol>` with all its `<li>` children)
- If a single element exceeds 3000 characters, treat it as its own batch

**For each batch:**

1. Read `references/translation-prompt.md` to remember the translation rules
2. Construct the translation prompt:
   - Insert the style profile from `_translated/style_profile.md` into the `{STYLE_PROFILE}` placeholder
   - Set `{TARGET_LANGUAGE}` and `{SOURCE_LANGUAGE}`
   - If this is not the first batch, include the last 2-3 translated paragraphs from the previous batch as context:
     ```
     [PREVIOUS CONTEXT - do not re-translate]
     <p>前一段的翻译内容...</p>
     <p>前二段的翻译内容...</p>
     [END PREVIOUS CONTEXT]

     [TRANSLATE THE FOLLOWING]
     <p>New content to translate...</p>
     ```
3. Translate the batch, producing the target-language HTML
4. For **bilingual mode**: keep the original blocks, and insert translated blocks after each original:
   - Copy the original block element
   - Add `class="translated"` to the translated copy
   - Place the translated copy immediately after the original

#### 6.5: Reassemble the XHTML File

Reconstruct the complete XHTML file:
1. Keep the XML declaration exactly as-is (e.g., `<?xml version="1.0" encoding="utf-8"?>`)
2. Keep the DOCTYPE exactly as-is (if present)
3. Update `<html>` tag: change `lang="xx"` and `xml:lang="xx"` to the target language code
4. Keep `<head>` section entirely unchanged (title, CSS links, meta tags)
5. Replace `<body>` content with the translated content (or bilingual interleaved content)

#### 6.6: Write Checkpoint

1. Create the necessary subdirectories: `mkdir -p "<work_dir>/_translated/<subdirs>/"`
2. Write the translated XHTML to `<work_dir>/_translated/<relative_path>` using the Write tool
3. Report: "Translated: <filename> (X/N)"

### Step 7: Translate TOC

**Check checkpoint first:** If `_translated/` already has the TOC file(s), skip this step.

**If both `toc.ncx` and `toc.xhtml` exist**, present them to yourself in a single translation batch to guarantee consistent chapter title translations.

#### toc.ncx (ePub 2)
- Read the file from the work directory
- Translate only the `<text>` content inside each `<navLabel>` element
- Do NOT modify `<content src="..."/>` attributes
- Preserve all XML structure and attributes

#### toc.xhtml (ePub 3)
- Read the file from the work directory
- Translate `<nav epub:type="toc">`: all anchor text in the navigation list
- Translate `<nav epub:type="landmarks">`: translate labels (e.g., "Cover" → target language equivalent)
- Do NOT translate `<nav epub:type="page-list">` entries
- Do NOT modify any `href` attributes
- Update `lang` and `xml:lang` attributes on `<html>` tag

Save both to `_translated/` preserving relative paths.

### Step 8: Update Metadata in content.opf

1. Read the original `content.opf` from the work directory
2. Make these changes:
   - `<dc:language>` → change to target language code (e.g., `en` → `zh`)
   - `<dc:title>` → translate the book title
   - `<dc:description>` → translate the description (if present)
   - `<dc:creator>` → **leave unchanged** (author names are not translated)
3. **Bilingual mode only:** If bilingual CSS needs to be added:
   - If no CSS file was found in Step 5: add a new manifest item for `bilingual.css`
   - This is handled in Step 9
4. Ensure the target directory exists: `mkdir -p "<work_dir>/_translated/<content.opf parent dir>/"`
5. Save the modified content.opf to `<work_dir>/_translated/<content.opf relative path>`

### Step 9: Repackage ePub

#### 9.1: Create Staging Directory
```bash
staging_dir="<work_dir>/_staging"
mkdir -p "$staging_dir"
```

#### 9.2: Copy Original Structure (excluding work artifacts)
```bash
cd "<work_dir>"
# Copy everything except _translated/ and _staging/
for item in $(ls -A | grep -v '^_translated$' | grep -v '^_staging$'); do
  cp -r "$item" "$staging_dir/"
done
```

#### 9.3: Overlay Translated Files
```bash
# Copy all translated files over the staging copy, preserving directory structure
cp -r "<work_dir>/_translated/"* "$staging_dir/" 2>/dev/null
# Note: style_profile.md will be copied too but won't affect the ePub
# Remove it from staging to keep the ePub clean
rm -f "$staging_dir/style_profile.md"
```

#### 9.4: Bilingual CSS Injection (bilingual mode only)
If bilingual mode is active:
1. Check if a main CSS file exists in the staging directory (found in Step 5)
2. If yes: append to the end of that CSS file:
   ```css

   /* Bilingual translation styles */
   .translated { color: #555555; font-size: 0.95em; margin-top: 0.2em; }
   ```
3. If no CSS file exists:
   - Create `<content_dir>/Styles/bilingual.css` with the above CSS rule
   - Add to `content.opf` manifest: `<item id="bilingual-css" href="Styles/bilingual.css" media-type="text/css"/>`
   - Add `<link>` reference in each translated XHTML file's `<head>`

#### 9.5: Package as ePub ZIP

```bash
cd "$staging_dir"

# Determine output path
output_path="<user_specified_or_default>"
# Default: <source_dir>/<original_name>_<target_lang>.epub

# Step 1: mimetype MUST be first and uncompressed
zip -0 -X "$output_path" mimetype

# Step 2: Add everything else (compressed)
zip -r "$output_path" * -x mimetype
```

#### 9.6: Verify Output
```bash
ls -la "$output_path"
# Check file size is > 0 and reasonable
```

If the output file is 0 bytes or missing, report a packaging error and STOP.

### Step 10: Cleanup and Report

Report to the user:
- Output file path and size
- Number of chapters translated
- Translation mode used (pure or bilingual)
- Target language

Ask: "Would you like me to keep the work directory (`<work_dir>/`) for debugging, or clean it up?"

If cleanup requested:
```bash
rm -rf "<work_dir>"
```

## Error Handling

| Scenario | How to Detect | Action |
|----------|--------------|--------|
| File not found | `ls` fails | Report "File not found: <path>" and STOP |
| Not .epub | Check extension | Report "Not an ePub file" and STOP |
| Unzip fails | Non-zero exit code | Report "Failed to extract ePub (file may be corrupted)" and STOP |
| DRM protected | `META-INF/encryption.xml` contains `<EncryptedData>` referencing XHTML files | Report "ePub is DRM-protected" and STOP |
| Fixed-layout | `rendition:layout` = `pre-paginated` in content.opf | WARN "Fixed-layout ePub detected — translation may affect layout" and CONTINUE |
| Non-UTF-8 | XML declaration has `encoding` other than `utf-8` | WARN "Non-UTF-8 encoding detected" and CONTINUE with caution |
| Empty content file | XHTML body has no translatable blocks | Skip silently, write unchanged to checkpoint |
| Super-long paragraph | Single element >3000 chars | Treat as its own batch |
| Translated file exists | `_translated/<path>` already present | Skip (resumability) |
| Output file is 0 bytes | `ls -la` shows 0 size | Report packaging error and STOP |
| zip/unzip not found | `which zip` fails | Direct user to `rules/install.md` and STOP |

## Prohibitions

Do NOT:
- Remove or modify images
- Alter the ePub file/directory structure
- Modify existing CSS rules (only append bilingual styles if needed)
- Translate content inside `<code>` or `<pre>` tags
- Translate `href`, `src`, `id`, `class`, or other HTML attribute values
- Translate author names
- Add files not required for the translation (no README, no logs inside the ePub)
- Modify spine order or manifest entries (except adding bilingual CSS if needed)

## Limitations

Tell the user upfront:
- Large books (>50 chapters) require multiple conversation sessions (3 chapters per session, checkpoint-based)
- Inline tag preservation is best-effort; complex nesting may occasionally break
- Text embedded in images is not translated
- Fixed-layout ePub translations may have layout issues
- SVG text elements are not translated
- `<ruby>`/`<rt>` annotations are preserved as-is
- Table column widths may shift when translated text length differs significantly
- No parallel processing — chapters are translated sequentially
- Resumability checks file existence only; if the source ePub changes between runs, delete `_translated/` to force re-translation
- Bilingual mode auto-injects minimal CSS (`.translated { color: #555; font-size: 0.95em; }`); users may customize further

# ePub Translator Claude Skill — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code skill that translates ePub files between languages using Claude itself as the translation engine, with zero external dependencies.

**Architecture:** A pure-markdown Claude skill (SKILL.md + reference files) that instructs Claude to unzip ePubs, read/translate XHTML content while preserving HTML structure, and repackage the result. No code libraries — Claude operates directly on files using shell tools (zip/unzip) and its own Read/Write/Edit capabilities.

**Tech Stack:** Claude Code skill system (SKILL.md frontmatter), shell commands (zip/unzip), ePub3/ePub2 format (XHTML + OPF + NCX)

**Spec:** `docs/superpowers/specs/2026-03-16-epub-translator-skill-design.md`

---

## File Structure

| File | Responsibility |
|------|---------------|
| `epub-translator/SKILL.md` | Main skill definition: frontmatter, parameter parsing, complete 10-step workflow, error handling, prohibitions |
| `epub-translator/references/translation-prompt.md` | Translation prompt template: style profile injection point, HTML preservation rules, examples, quality requirements |
| `epub-translator/references/epub-structure.md` | ePub format quick reference: file hierarchy, key XML structures, metadata fields |
| `epub-translator/rules/install.md` | Prerequisites: zip/unzip availability per platform |

All files are markdown — no executable code. The skill's "logic" is expressed as precise natural-language instructions that Claude follows.

---

## Chunk 1: Foundation Files

### Task 1: Create install rules

**Files:**
- Create: `epub-translator/rules/install.md`

- [ ] **Step 1: Create the rules directory and install.md**

```markdown
# Installation Requirements

This skill requires `zip` and `unzip` commands available in your shell.

## Platform-specific instructions

### Windows
- **Git Bash** (bundled with Git for Windows): `zip` and `unzip` are included
- **Chocolatey**: `choco install zip unzip`
- **Manual**: Download from GnuWin32 and add to PATH

### macOS
- Pre-installed on all macOS versions

### Linux
- Usually pre-installed. If not:
  - Debian/Ubuntu: `sudo apt install zip unzip`
  - Fedora/RHEL: `sudo dnf install zip unzip`
  - Arch: `sudo pacman -S zip unzip`

## Verification

Run these commands to verify:
```
zip --version
unzip --version
```

Both should print version info without errors.
```

- [ ] **Step 2: Verify file renders correctly**

Run: `cat epub-translator/rules/install.md`
Expected: Clean markdown with no formatting issues.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/rules/install.md
git commit -m "feat: add install rules for zip/unzip prerequisites"
```

---

### Task 2: Create ePub structure reference

**Files:**
- Create: `epub-translator/references/epub-structure.md`

- [ ] **Step 1: Create references directory and epub-structure.md**

This file is a quick-reference for Claude to consult during translation. It must cover the complete path from container.xml to content files, including both ePub 2 and ePub 3 variations.

```markdown
# ePub Format Quick Reference

## File Hierarchy

An ePub file is a ZIP archive with this structure:

```
book.epub (ZIP)
├── mimetype                    # MUST be first file, uncompressed
├── META-INF/
│   └── container.xml           # Points to content.opf location
└── <content-dir>/              # Often OEBPS/, EPUB/, OPS/, or root
    ├── content.opf             # Package document (manifest + spine + metadata)
    ├── toc.ncx                 # Navigation (ePub 2, optional in ePub 3)
    ├── toc.xhtml               # Navigation (ePub 3)
    ├── *.xhtml                 # Content files
    ├── Styles/*.css            # Stylesheets
    └── Images/*                # Images
```

**IMPORTANT:** The content directory name is NOT standardized. Always read `META-INF/container.xml` to find the actual `content.opf` path.

## container.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<container version="1.0" xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
  <rootfiles>
    <rootfile full-path="OEBPS/content.opf" media-type="application/oebps-package+xml"/>
  </rootfiles>
</container>
```

Extract `full-path` attribute from `<rootfile>` to locate `content.opf`.

## content.opf

### Metadata section
```xml
<metadata xmlns:dc="http://purl.org/dc/elements/1.1/">
  <dc:title>Book Title</dc:title>
  <dc:creator>Author Name</dc:creator>
  <dc:language>en</dc:language>
  <dc:description>Book description</dc:description>
  <dc:subject>Category</dc:subject>
  <!-- Fixed-layout detection: -->
  <meta property="rendition:layout">pre-paginated</meta>
</metadata>
```

Fields to update during translation:
- `dc:language` → target language code
- `dc:title` → translated title
- `dc:description` → translated (if present)
- `dc:creator` → leave as-is

### Manifest section
```xml
<manifest>
  <item id="ch01" href="Text/chapter_01.xhtml" media-type="application/xhtml+xml"/>
  <item id="css" href="Styles/stylesheet.css" media-type="text/css"/>
  <item id="ncx" href="toc.ncx" media-type="application/x-dtbncx+xml"/>
  <item id="nav" href="toc.xhtml" media-type="application/xhtml+xml" properties="nav"/>
</manifest>
```

- XHTML files: `media-type="application/xhtml+xml"`
- TOC (ePub 3): item with `properties="nav"`
- TOC (ePub 2): item with `media-type="application/x-dtbncx+xml"`
- CSS files: `media-type="text/css"`

### Spine section
```xml
<spine toc="ncx">
  <itemref idref="cover"/>
  <itemref idref="ch01"/>
  <itemref idref="ch02"/>
</spine>
```

The `idref` values reference `id` attributes in the manifest. Spine order = reading order.

## toc.ncx (ePub 2 Navigation)

```xml
<ncx xmlns="http://www.daisy.org/z3986/2005/ncx/">
  <navMap>
    <navPoint id="np-1" playOrder="1">
      <navLabel><text>Chapter 1: Introduction</text></navLabel>
      <content src="Text/chapter_01.xhtml"/>
    </navPoint>
  </navMap>
</ncx>
```

Translate: `<navLabel><text>` content only. Leave `src` attributes unchanged.

## toc.xhtml (ePub 3 Navigation)

```xml
<nav epub:type="toc">
  <ol>
    <li><a href="Text/chapter_01.xhtml">Chapter 1: Introduction</a></li>
  </ol>
</nav>
<nav epub:type="landmarks">
  <ol>
    <li><a epub:type="cover" href="Text/cover.xhtml">Cover</a></li>
    <li><a epub:type="toc" href="toc.xhtml">Table of Contents</a></li>
  </ol>
</nav>
<nav epub:type="page-list">
  <!-- Do NOT translate page numbers -->
</nav>
```

Translate: `epub:type="toc"` anchor text, `epub:type="landmarks"` labels.
Do NOT translate: `epub:type="page-list"` entries, `href` values.

## DRM Detection

Check for `META-INF/encryption.xml`. If it exists and contains `<EncryptedData>` elements referencing XHTML content files, the book is DRM-protected and cannot be translated.

## XHTML Content Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:epub="http://www.idpf.org/2007/ops" xml:lang="en" lang="en">
<head>
  <title>Chapter Title</title>
  <link rel="stylesheet" type="text/css" href="../Styles/stylesheet.css"/>
</head>
<body>
  <h1>Chapter 1</h1>
  <p>Content with <em>inline</em> formatting.</p>
  <pre><code>code_block()</code></pre>
</body>
</html>
```

When translating:
- Update `xml:lang` and `lang` attributes on `<html>` tag
- Preserve everything in `<head>` unchanged
- Preserve XML declaration, DOCTYPE, and namespace declarations exactly
- Only translate text content within `<body>`
```

- [ ] **Step 2: Verify file completeness**

Run: `cat epub-translator/references/epub-structure.md`
Expected: Complete reference covering container.xml, content.opf (all 3 sections), toc.ncx, toc.xhtml (with epub:type distinctions), DRM detection, and XHTML structure.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/references/epub-structure.md
git commit -m "feat: add ePub format quick reference for Claude"
```

---

### Task 3: Create translation prompt template

**Files:**
- Create: `epub-translator/references/translation-prompt.md`

- [ ] **Step 1: Create translation-prompt.md**

This is the most critical reference file. Claude reads this before every translation batch to know exactly how to translate.

```markdown
# Translation Prompt Template

Use this template for every translation batch. Fill in the placeholders before translating.

---

## Global Book Context

{STYLE_PROFILE}

**Target language:** {TARGET_LANGUAGE}

---

## Translation Instructions

You are translating a batch of HTML content from {SOURCE_LANGUAGE} to {TARGET_LANGUAGE}. Follow these rules exactly:

### Core Rules

1. **Translate only text nodes.** HTML tags, attribute names, and attribute values are NOT text — do not translate them.
2. **Preserve all HTML tags exactly.** Every opening tag must have its matching closing tag. Do not add, remove, or reorder tags unless the target language word order requires repositioning inline tags (e.g., `<em>`, `<strong>`).
3. **Maintain the style profile.** Your translation must match the tone, voice, and vocabulary level described in the Global Book Context above.
4. **Translate naturally.** Produce fluent, idiomatic {TARGET_LANGUAGE}. Do not produce word-for-word translations. Adapt idioms, metaphors, and cultural references to equivalents that resonate with {TARGET_LANGUAGE} readers.

### What to Translate

- Text content inside `<p>`, `<h1>`–`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>` elements
- `alt` attribute values on `<img>` tags
- `title` attribute values where present

### What NOT to Translate

- Content inside `<code>` and `<pre>` elements (source code, command lines)
- `href`, `src`, `id`, `class` attribute values
- URLs, file paths, email addresses
- Purely numeric or symbolic content (e.g., "42", "•", "→")
- Author names and proper nouns that have no established translation
- Content marked as "previous context" — this is reference only, do not re-translate

### Inline Tag Handling

When translating text that contains inline tags, reposition the tags to wrap the correct translated words:

**Example (English → Chinese):**

Input:
```html
<p>This is a <em>very important</em> concept in <strong>Python</strong> programming.</p>
```

Output:
```html
<p>这是<strong>Python</strong>编程中一个<em>非常重要</em>的概念。</p>
```

Notice:
- `<em>` moved to wrap "非常重要" (the translation of "very important")
- `<strong>Python</strong>` stays wrapping "Python" (proper noun, not translated)
- Tag nesting remains valid

### Output Format

- Output ONLY the translated HTML blocks
- Do not include explanations, notes, or commentary
- Do not wrap output in markdown code fences
- Preserve the exact whitespace and line break structure of the input
- If a block should not be translated (e.g., it's pure code), output it unchanged

### Previous Context Handling

When previous context is provided, use it to:
- Maintain consistent terminology (e.g., if "function" was translated as "函数" before, keep using "函数")
- Maintain consistent narrative flow and tone
- Do NOT re-translate the context — only translate the new blocks marked for translation
```

- [ ] **Step 2: Verify prompt template is complete and well-structured**

Run: `cat epub-translator/references/translation-prompt.md`
Expected: Template with clear placeholders ({STYLE_PROFILE}, {TARGET_LANGUAGE}, {SOURCE_LANGUAGE}), comprehensive rules, inline tag example, and previous context handling.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/references/translation-prompt.md
git commit -m "feat: add translation prompt template with HTML preservation rules"
```

---

## Chunk 2: Main Skill File (SKILL.md)

### Task 4: Create SKILL.md — Frontmatter and Parameter Parsing

**Files:**
- Create: `epub-translator/SKILL.md`

- [ ] **Step 1: Create SKILL.md with frontmatter and parameter section**

```markdown
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
```

- [ ] **Step 2: Verify frontmatter YAML is valid**

Run: `head -12 epub-translator/SKILL.md`
Expected: Valid YAML frontmatter with `name` and `description` fields.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/SKILL.md
git commit -m "feat: add SKILL.md with frontmatter and parameter parsing"
```

---

### Task 5: Add SKILL.md — Workflow Steps 1-5.5

**Files:**
- Modify: `epub-translator/SKILL.md` (append)

- [ ] **Step 1: Append Steps 1 through 5.5**

Append the following to `epub-translator/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify the appended content**

Run: `wc -l epub-translator/SKILL.md`
Expected: ~100-120 lines total.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/SKILL.md
git commit -m "feat: add workflow steps 1-5.5 (validation through style profile)"
```

---

### Task 6: Add SKILL.md — Workflow Step 6 (Core Translation Loop)

**Files:**
- Modify: `epub-translator/SKILL.md` (append)

- [ ] **Step 1: Append Step 6**

Append the following to `epub-translator/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify content is appended correctly**

Run: `grep -c "^###\|^####" epub-translator/SKILL.md`
Expected: Should show the growing count of section headers.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/SKILL.md
git commit -m "feat: add Step 6 core translation loop with batching and checkpoints"
```

---

### Task 7: Add SKILL.md — Workflow Steps 7-10 (TOC, Metadata, Repackage, Cleanup)

**Files:**
- Modify: `epub-translator/SKILL.md` (append)

- [ ] **Step 1: Append Steps 7 through 10**

Append the following to `epub-translator/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify complete SKILL.md structure**

Run: `grep "^### Step" epub-translator/SKILL.md`
Expected: Steps 1, 2, 3, 4, 5, 5.5, 6, 7, 8, 9, 10 all present.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/SKILL.md
git commit -m "feat: add Steps 7-10 (TOC, metadata, repackage, cleanup)"
```

---

### Task 8: Add SKILL.md — Error Handling, Prohibitions, and Limitations

**Files:**
- Modify: `epub-translator/SKILL.md` (append)

- [ ] **Step 1: Append error handling and closing sections**

Append the following to `epub-translator/SKILL.md`:

```markdown

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
```

- [ ] **Step 2: Verify final SKILL.md line count and structure**

Run: `wc -l epub-translator/SKILL.md && grep "^## " epub-translator/SKILL.md`
Expected: ~250-300 lines. Top-level sections: Parameters, Prerequisites, Workflow, Error Handling, Prohibitions, Limitations.

- [ ] **Step 3: Commit**

```bash
git add epub-translator/SKILL.md
git commit -m "feat: add error handling, prohibitions, and limitations"
```

---

## Chunk 3: Testing and Validation

### Task 9: Validate Skill Structure

**Files:**
- All files in `epub-translator/`

- [ ] **Step 1: Verify complete file tree exists**

```bash
find epub-translator/ -type f | sort
```

Expected output:
```
epub-translator/SKILL.md
epub-translator/references/epub-structure.md
epub-translator/references/translation-prompt.md
epub-translator/rules/install.md
```

- [ ] **Step 2: Verify SKILL.md frontmatter is valid**

```bash
head -8 epub-translator/SKILL.md
```

Expected: Valid YAML between `---` delimiters with `name: epub-translator` and `description:`.

- [ ] **Step 3: Verify all cross-references resolve**

Check that SKILL.md references to `references/translation-prompt.md`, `references/epub-structure.md`, and `rules/install.md` match actual filenames.

```bash
grep -o 'references/[a-z-]*.md\|rules/[a-z-]*.md' epub-translator/SKILL.md | sort -u
```

Expected: All referenced paths exist as actual files.

- [ ] **Step 4: Verify zip/unzip are available**

```bash
which zip && which unzip
```

Expected: Both commands found. If not, follow `rules/install.md`.

---

### Task 10: Smoke Test — Unzip/Repackage Round-Trip

**Files:**
- Test ePub: `C:\Users\luden\Downloads\python-workout-second-edition.epub`

- [ ] **Step 1: Unzip the test ePub**

```bash
test_dir="/tmp/epub_test_roundtrip"
mkdir -p "$test_dir"
unzip -o -q "/c/Users/luden/Downloads/python-workout-second-edition.epub" -d "$test_dir/"
ls "$test_dir/META-INF/container.xml"
```

Expected: container.xml exists.

- [ ] **Step 2: Read container.xml and content.opf**

Use the Read tool to verify:
- `container.xml` points to `OEBPS/content.opf`
- `content.opf` contains dc:title, dc:language, manifest, spine

- [ ] **Step 3: Repackage without any changes**

```bash
cd "$test_dir"
output="/tmp/epub_test_roundtrip_output.epub"
zip -0 -X "$output" mimetype
zip -r "$output" * -x mimetype
ls -la "$output"
ls -la "/c/Users/luden/Downloads/python-workout-second-edition.epub"
```

Expected: Output file exists with roughly similar size to original (within ~5% due to compression differences).

- [ ] **Step 4: Clean up test artifacts**

```bash
rm -rf "$test_dir" "$output"
```

- [ ] **Step 5: Commit all finalized skill files**

```bash
git add -A
git commit -m "feat: complete epub-translator skill with all reference files"
```

---

### Task 11: Integration Test — Translate First Chapter

**Files:**
- Test ePub: `C:\Users\luden\Downloads\python-workout-second-edition.epub`
- Skill: `epub-translator/SKILL.md`

This task tests the skill end-to-end by invoking it to translate the test ePub to Chinese. We only translate the first 1-2 content chapters to verify the full workflow without processing the entire book.

- [ ] **Step 1: Invoke the skill**

In a new Claude Code conversation (or the current one), invoke:

> Translate `C:\Users\luden\Downloads\python-workout-second-edition.epub` to Chinese

Claude should follow the SKILL.md workflow: validate → unzip → locate content.opf → extract manifest → build style profile → translate chapters → translate TOC → update metadata → repackage.

- [ ] **Step 2: Verify style profile was created**

Check that `_translated/style_profile.md` exists and contains a sensible profile for a technical programming book.

- [ ] **Step 3: Verify translated XHTML**

Read one of the translated XHTML files and verify:
- Text is translated to Chinese
- HTML tags are preserved and properly nested
- `<code>` / `<pre>` blocks are untouched
- `lang` and `xml:lang` updated to `zh`
- XML declaration and namespace preserved

- [ ] **Step 4: Verify output ePub**

- Output file exists at expected path (`*_zh.epub`)
- File size is reasonable (not 0, not wildly different from original)
- (Manual) Open in an ePub reader to verify visual appearance

- [ ] **Step 5: Document any issues found**

If any issues are discovered, fix them in the skill files and commit.

- [ ] **Step 6: Clean up test artifacts**

Remove the work directory created during testing (in the Downloads folder):
```bash
rm -rf "<source_dir>/<filename>_translation_work"
```
Keep the output `*_zh.epub` for manual review.

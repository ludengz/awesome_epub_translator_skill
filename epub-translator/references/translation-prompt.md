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

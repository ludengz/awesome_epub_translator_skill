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

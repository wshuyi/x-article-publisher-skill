# X Article Publisher Skill

[English](README.md) | [中文](README_CN.md)

> Publish Markdown articles to X (Twitter) Articles with one command. Say goodbye to tedious rich text editing.

**v1.2.0** — Now with divider support, table-to-image, Mermaid support, and cross-platform clipboard

---

## The Problem

If you're used to writing in Markdown, publishing to X Articles is a **painful process**:

| Pain Point | Description |
|------------|-------------|
| **Format Loss** | Copy from Markdown editor → Paste to X → All formatting gone |
| **Manual Formatting** | Set each H2, bold, link manually — 15-20 min per article |
| **Tedious Image Upload** | 5 clicks per image: Add media → Media → Add photo → Select → Wait |
| **Position Errors** | Hard to remember where each image should go |

### Time Comparison

| Task | Manual | With This Skill |
|------|--------|-----------------|
| Format conversion | 15-20 min | 0 (automatic) |
| Cover image | 1-2 min | 10 sec |
| 5 content images | 5-10 min | 1 min |
| **Total** | **20-30 min** | **2-3 min** |

**10x efficiency improvement**

---

## The Solution

This skill automates the entire publishing workflow:

```
Markdown File
     ↓ Python parsing
Structured Data (title, images with block_index, HTML)
     ↓ Playwright MCP
X Articles Editor (browser automation)
     ↓
Draft Saved (never auto-publishes)
```

### Key Features

- **Rich Text Paste**: Convert Markdown to HTML, paste via clipboard — all formatting preserved
- **Block-Index Positioning** (v1.1): Precise image placement using element indices, not text matching
- **Reverse Insertion**: Insert images from highest to lowest index to avoid position shifts
- **Smart Wait Strategy**: Conditions return immediately when met, no wasted wait time
- **Safe by Design**: Only saves as draft, never publishes automatically

---

## What's New in v1.1.0

| Feature | Before | After |
|---------|--------|-------|
| Image positioning | Text matching (fragile) | Block index (precise) |
| Insertion order | Sequential | Reverse (high→low) |
| Wait behavior | Fixed delay | Immediate return on condition |

### Why Block-Index?

Previously, images were positioned by matching surrounding text — this failed when:
- Multiple paragraphs had similar content
- Text was too short to be unique

Now, each image has a `block_index` indicating exactly which block element it follows. This is deterministic and reliable.

---

## Requirements

| Requirement | Details |
|-------------|---------|
| Claude Code | [claude.ai/code](https://claude.ai/code) |
| Playwright MCP | Browser automation |
| X Premium Plus | Required for Articles feature |
| Python 3.9+ | With dependencies below |
| OS | macOS or Windows |

```bash
# macOS
pip install Pillow pyobjc-framework-Cocoa

# Windows
pip install Pillow pywin32 clip-util

# For Mermaid diagrams (optional)
npm install -g @mermaid-js/mermaid-cli
```

---

## Installation

### Method 1: Git Clone (Recommended)

```bash
git clone https://github.com/wshuyi/x-article-publisher-skill.git
cp -r x-article-publisher-skill/skills/x-article-publisher ~/.claude/skills/
```

### Method 2: Plugin Marketplace

```
/plugin marketplace add wshuyi/x-article-publisher-skill
/plugin install x-article-publisher@wshuyi/x-article-publisher-skill
```

---

## Usage

### Natural Language

```
Publish /path/to/article.md to X
```

```
Help me post this article to X Articles: ~/Documents/my-post.md
```

### Skill Command

```
/x-article-publisher /path/to/article.md
```

---

## Workflow Steps

```
[1/7] Parse Markdown...
      → Extract title, cover image, content images with block_index
      → Convert to HTML, count total blocks

[2/7] Open X Articles editor...
      → Navigate to x.com/compose/articles

[3/7] Upload cover image...
      → First image becomes cover

[4/7] Fill title...
      → H1 used as title (not included in body)

[5/7] Paste article content...
      → Rich text via clipboard
      → All formatting preserved

[6/7] Insert content images (reverse order)...
      → Sort by block_index descending
      → Click block element at index → Paste image
      → Wait for upload (returns immediately when done)

[7/7] Save draft...
      → ✅ Review and publish manually
```

---

## Supported Markdown

| Syntax | Result | Notes |
|--------|--------|-------|
| `# H1` | Article title | Extracted, not in body |
| `## H2` | Section headers | Native support |
| `**bold**` | **Bold text** | Native support |
| `*italic*` | *Italic text* | Native support |
| `[text](url)` | Hyperlinks | Native support |
| `> quote` | Blockquotes | Native support |
| `- item` | Unordered lists | Native support |
| `1. item` | Ordered lists | Native support |
| `![](img.jpg)` | Images | First = cover |
| `---` | Dividers | Via Insert menu (v1.2) |
| Tables | PNG images | Via table_to_image.py (v1.2) |
| Mermaid | PNG images | Via mmdc CLI (v1.2) |

---

## Example

### Input: `article.md`

```markdown
# 5 AI Tools Worth Watching in 2024

![cover](./images/cover.jpg)

AI tools exploded in 2024. Here are 5 worth your attention.

## 1. Claude: Best Conversational AI

**Claude** by Anthropic excels at long-context understanding.

> Claude's context window reaches 200K tokens.

![claude-demo](./images/claude-demo.png)

## 2. Midjourney: AI Art Leader

[Midjourney](https://midjourney.com) is the most popular AI art tool.

![midjourney](./images/midjourney.jpg)
```

### Parsed Output (JSON)

```json
{
  "title": "5 AI Tools Worth Watching in 2024",
  "cover_image": "./images/cover.jpg",
  "content_images": [
    {"path": "./images/claude-demo.png", "block_index": 4},
    {"path": "./images/midjourney.jpg", "block_index": 6}
  ],
  "total_blocks": 7
}
```

### Insertion Order

Images inserted in reverse: `block_index=6` first, then `block_index=4`.

### Result

- Cover: `cover.jpg` uploaded
- Title: "5 AI Tools Worth Watching in 2024"
- Content: Rich text with H2, bold, quotes, links
- Images: Inserted at precise positions via block index
- Status: **Draft saved** (ready for manual review)

---

## Project Structure

```
x-article-publisher-skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin config
├── skills/
│   └── x-article-publisher/
│       ├── SKILL.md             # Skill instructions
│       └── scripts/
│           ├── parse_markdown.py    # Extracts block_index + dividers
│           ├── copy_to_clipboard.py # Cross-platform clipboard
│           └── table_to_image.py    # Markdown table → PNG (v1.2)
├── docs/
│   └── GUIDE.md                 # Detailed guide
├── README.md                    # This file
├── README_CN.md                 # Chinese version
└── LICENSE
```

---

## FAQ

**Q: Why Premium Plus?**
A: X Articles is exclusive to Premium Plus subscribers.

**Q: Windows/Linux support?**
A: Windows is now supported (v1.2). Linux support is still in progress — PRs welcome!

**Q: Image upload failed?**
A: Check: valid path, supported format (jpg/png/gif/webp), stable network.

**Q: Can I publish to multiple accounts?**
A: Not automatically. Switch accounts in browser manually before running.

**Q: Why insert images in reverse order?**
A: Each inserted image shifts subsequent block indices. Inserting from highest to lowest ensures earlier indices remain valid.

**Q: What if text matching was used before?**
A: v1.1 replaces text matching with `block_index`. The `after_text` field is kept for debugging but not used for positioning.

**Q: Why does wait return immediately sometimes?**
A: `browser_wait_for textGone="..."` returns as soon as the text disappears. The `time` parameter is just a maximum, not a fixed delay.

---

## Documentation

- [Detailed Usage Guide](docs/GUIDE.md) — Complete documentation with examples

---

## Changelog

### v1.2.0 (2025-01)
- **Divider support**: Detect `---` in Markdown, insert via X Articles menu
- **Table to image**: New `table_to_image.py` script converts Markdown tables to PNG
- **Mermaid support**: Documentation for using `mmdc` to convert diagrams
- **YAML frontmatter**: Automatically skip frontmatter in Markdown files
- **Windows support**: Cross-platform clipboard operations (pywin32 + clip-util)

### v1.1.0 (2025-12)
- **Block-index positioning**: Replace text matching with precise element indices
- **Reverse insertion order**: Prevent index shifts when inserting multiple images
- **Optimized wait strategy**: Return immediately when upload completes
- **H1 title handling**: H1 extracted as title, not included in body HTML

### v1.0.0 (2025-12)
- Initial release
- Rich text paste via clipboard
- Cover + content image support
- Draft-only publishing

---

## License

MIT License - see [LICENSE](LICENSE)

## Author

[wshuyi](https://github.com/wshuyi)

---

## Acknowledgments

v1.2.0 features were inspired by and adapted from:

- **[sugarforever/01coder-agent-skills](https://github.com/sugarforever/01coder-agent-skills)** — The `publish-x-article` skill contributed ideas for:
  - Divider detection and menu-based insertion
  - Table-to-image conversion script
  - Mermaid diagram support documentation
  - YAML frontmatter handling
  - Windows clipboard implementation

Thank you to the community for building upon and improving this skill!

---

## Contributing

- **Issues**: Report bugs or request features
- **PRs**: Welcome! Especially for Windows/Linux support

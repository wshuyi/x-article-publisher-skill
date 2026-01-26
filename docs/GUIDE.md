# X Article Publisher Skill User Guide

> One-click publishing of Markdown articles to X (Twitter) Articles, eliminating tedious rich text editing.

**v1.1.0** — New block-index precise positioning for more accurate image placement

---

## v1.1.0 Update Highlights

| Feature | v1.0 | v1.1 |
|------|------|------|
| Image Positioning | Text matching (unstable) | Block index (precise) |
| Insert Order | Sequential insertion | Reverse insertion (high→low) |
| Wait Strategy | Fixed delay | Immediate return on condition |

---

## Table of Contents

1. [Problems Solved](#1-problems-solved)
2. [Solution](#2-solution)
3. [Execution Method](#3-execution-method)
4. [Complete Example](#4-complete-example)
5. [FAQ](#5-faq)
6. [Changelog](#6-changelog)

---

## 1. Problems Solved

### 1.1 What is X Articles?

X (formerly Twitter) provides **Articles** functionality for Premium Plus subscribers, allowing users to publish long-form content (similar to blogs) beyond the 280-character limit. Articles support:

- Rich text formatting (headings, bold, quotes, etc.)
- Multiple embedded images
- Hyperlinks

Access point: https://x.com/compose/articles

### 1.2 Pain Points of Manual Publishing

If you're used to writing in Markdown, publishing content to X Articles is an **extremely tedious** process:

#### Pain Point 1: Formatting Cannot Be Pasted Directly

```
❌ Copy from Markdown editor → Paste to X Articles → All formatting lost
```

The X Articles editor doesn't support Markdown, so you need to **manually set each format** one by one:
- Select text → Click H2 button
- Select text → Click bold button
- Select text → Click link button → Paste URL

For an article with 5 sections, 10 bold items, and 8 links, formatting can take **15-20 minutes**.

#### Pain Point 2: Inefficient Image Insertion

The image insertion process in X Articles:

```
Click paragraph → Click "Add media content" → Click "Media" → Click "Add photo or video" → Select file → Wait for upload
```

Each image requires **5 clicks + file selection + upload wait**. For an article with 5 images, image insertion alone takes **5-10 minutes**.

#### Pain Point 3: Difficult to Control Image Position Precisely

Image positions in Markdown are precise:

```markdown
This is the first paragraph.

![Image 1](image1.jpg)

This is the second paragraph.

![Image 2](image2.jpg)
```

But in X Articles, you need to:
1. Remember where each image should be inserted
2. Scroll to find the correct paragraph
3. Insert manually

The more images, the higher the error rate.

#### Pain Point 4: Repetitive Work

If you frequently publish long articles, these operations need to be **repeated every time**:
- Manual format conversion for each article
- 5 clicks repeated for each image
- Always need to check if formatting is correct

### 1.3 Time Cost Comparison

| Operation | Manual Method | Using This Skill |
|------|----------|--------------|
| Format conversion | 15-20 minutes | 0 (automatic) |
| Cover image upload | 1-2 minutes | 10 seconds |
| 5 content images insertion | 5-10 minutes | 1 minute |
| Total | **20-30 minutes** | **2-3 minutes** |

**Efficiency improvement: 10x or more**

---

## 2. Solution

### 2.1 Technical Architecture

This Skill achieves automation through the following tech stack:

```
┌─────────────────┐
│   Markdown File  │
└────────┬────────┘
         │ Python parsing
         ▼
┌─────────────────┐
│ Structured Data  │
│   (JSON)        │
│  - title        │
│  - cover_image  │
│  - content_images│
│  - html         │
└────────┬────────┘
         │ Playwright MCP
         ▼
┌─────────────────┐
│  X Articles      │
│  Editor          │
│ (Browser Auto)   │
└─────────────────┘
```

#### Core Components

| Component | Function |
|------|------|
| `parse_markdown.py` | Parse Markdown, extract title, images, convert to HTML |
| `copy_to_clipboard.py` | Copy HTML/images to system clipboard |
| Playwright MCP | Control browser, simulate user actions |
| Claude Code | Coordinate the entire process |

### 2.2 Workflow: "Text First, Images Later" Strategy

This Skill adopts a **Text First, Images Later** strategy to ensure complete content and accurate image positioning:

```
Step 1: Parse Markdown
        ↓
Step 2: Open X Articles editor
        ↓
Step 3: Upload cover image (first image)
        ↓
Step 4: Fill in title
        ↓
Step 5: Paste HTML rich text content (via clipboard)
        ↓
Step 6: Insert content images at correct positions (via clipboard paste)
        ↓
Step 7: Save as draft (never auto-publish)
```

### 2.3 Key Technical Optimizations

#### Optimization 1: Clipboard Rich Text Paste

Traditional method requires setting formats one by one, this Skill:

1. Python converts Markdown to HTML
2. Copy HTML to system clipboard (preserving rich text format)
3. `Cmd+V` paste in editor

```python
# copy_to_clipboard.py core logic
from AppKit import NSPasteboard, NSPasteboardTypeHTML

pasteboard = NSPasteboard.generalPasteboard()
pasteboard.setData_forType_(html_data, NSPasteboardTypeHTML)
```

Result: **All formats pasted at once**, including H2, bold, links, lists, etc.

#### Optimization 2: Clipboard Image Paste

Traditional method requires 5 clicks per image, this Skill:

1. Python copies image to system clipboard
2. Click target paragraph
3. `Cmd+V` paste image

```python
# copy_to_clipboard.py image handling
from AppKit import NSPasteboard, NSPasteboardTypeTIFF

# Optional: compress image before upload
img.thumbnail((2000, 2000))
img.save(buffer, format='JPEG', quality=85)
```

Comparison:

| Method | Browser Operations |
|------|--------------|
| Traditional | 5 clicks + file selection |
| This Skill | 1 click + 1 paste |

#### Optimization 3: Precise Image Positioning

`parse_markdown.py` extracts **block index information** for each image (v1.1 new feature):

```json
{
  "content_images": [
    {
      "path": "/path/to/img1.jpg",
      "block_index": 5,
      "after_text": "Context text (debug only)..."
    }
  ],
  "total_blocks": 12
}
```

**block_index positioning principle**:
- Each image's `block_index` indicates it should be inserted after the Nth block element (0-indexed)
- Does not rely on text matching, precise and reliable
- `after_text` retained for manual verification, not used for positioning

**Reverse insertion strategy**:
- Insert images in descending order by `block_index`
- Insert higher indices first, doesn't affect lower index positions
- Example: insert block_index=12 first, then block_index=5

### 2.4 Supported Markdown Formats

| Markdown Syntax | Effect |
|--------------|------|
| `# H1` | Article title (auto-extracted) |
| `## H2` | Second-level heading |
| `**bold**` | **Bold text** |
| `*italic*` | *Italic text* |
| `[link](url)` | Hyperlink |
| `> quote` | Quote block |
| `- list` | Unordered list |
| `1. list` | Ordered list |
| `![](img.jpg)` | Image (first is cover) |

### 2.5 Safety Design

**Never auto-publish**: This Skill only saves content as a draft, final publishing requires manual user confirmation.

```
✅ Automated: Upload, formatting, layout
❌ Not executed: Clicking "Publish" button
```

This ensures users can:
- Preview final result
- Check if formatting is correct
- Modify any content that needs adjustment

---

## 3. Execution Method

### 3.1 Prerequisites

#### Requirement 1: X Premium Plus Subscription

Articles functionality is only available to Premium Plus users. Verification method:
1. Visit https://x.com/compose/articles
2. If you can see the editor, you have access

#### Requirement 2: Install Playwright MCP

This Skill depends on Playwright MCP for browser automation.

Check if installed:
```bash
cat ~/.claude/settings.json | grep playwright
```

If not installed, execute in Claude Code:
```
Help me install Playwright MCP
```

#### Requirement 3: Install Python Dependencies

```bash
pip install Pillow pyobjc-framework-Cocoa
```

Verify installation:
```bash
python -c "from AppKit import NSPasteboard; print('OK')"
```

#### Requirement 4: Install This Skill

**Method A: Git Clone (Recommended)**

```bash
git clone https://github.com/wshuyi/x-article-publisher-skill.git
cp -r x-article-publisher-skill/skills/x-article-publisher ~/.claude/skills/
```

**Method B: Plugin Marketplace**

```
/plugin marketplace add wshuyi/x-article-publisher-skill
/plugin install x-article-publisher@wshuyi/x-article-publisher-skill
```

### 3.2 Trigger Commands

After installation, trigger using the following methods in Claude Code:

#### Method 1: Natural Language

```
Publish /path/to/article.md to X
```

```
Help me publish this article to X Articles: ~/Documents/my-article.md
```

```
Publish ~/blog/post.md to X
```

#### Method 2: Skill Command

```
/x-article-publisher /path/to/article.md
```

### 3.3 Operation Flow

After triggering, Claude will automatically execute these steps:

```
[1/7] Parsing Markdown file...
      → Extracted title: "Your Article Title"
      → Found cover image: cover.jpg
      → Found 3 content images
      → HTML conversion complete

[2/7] Opening X Articles editor...
      → Navigate to https://x.com/compose/articles
      → Wait for editor to load

[3/7] Uploading cover image...
      → Click "Add photo or video"
      → Upload cover.jpg
      → Wait for upload complete

[4/7] Filling in title...
      → Input: "Your Article Title"

[5/7] Pasting article content...
      → HTML copied to clipboard
      → Paste rich text content
      → Format preserved: 5 H2s, 8 bold items, 12 links

[6/7] Inserting content images...
      → Image 1/3: Positioned after "This is the first paragraph..."
      → Image 2/3: Positioned after "This is the second paragraph..."
      → Image 3/3: Positioned after "This is the third paragraph..."

[7/7] Saving draft...
      → Draft auto-saved
      → Please preview in X and manually publish
```

### 3.4 Notes

1. **Keep browser visible**: Playwright needs to control browser window, don't minimize
2. **Logged into X**: Ensure you're logged into your X account in the browser
3. **Image paths**: Image paths in Markdown need to be valid local paths
4. **Stable network**: Image upload requires stable network connection

---

## 4. Complete Example

### 4.1 Example Markdown File

File path: `~/Documents/ai-tools-review.md`

```markdown
# Top 5 AI Tools to Watch in 2024

![cover](./images/cover.jpg)

Artificial intelligence tools experienced explosive growth in 2024. This article introduces the 5 most noteworthy AI tools.

## 1. Claude: The Strongest Conversational AI

**Claude** is developed by Anthropic and excels in long-text understanding and code generation.

> Claude's context window reaches 200K tokens, capable of processing entire books.

Key features:
- Ultra-long context understanding
- Excellent reasoning ability
- Safe and reliable

![claude-demo](./images/claude-demo.png)

## 2. Midjourney: AI Art Leader

[Midjourney](https://midjourney.com) is currently the most popular AI art tool.

![midjourney-example](./images/midjourney.jpg)

## Summary

These tools are changing how we work. Choose the right tool for yourself to boost efficiency.
```

### 4.2 Execution Command

```
Publish ~/Documents/ai-tools-review.md to X
```

### 4.3 Execution Result

Claude will:

1. **Parse file**, output:
   ```json
   {
     "title": "Top 5 AI Tools to Watch in 2024",
     "cover_image": "~/Documents/images/cover.jpg",
     "content_images": [
       {"path": "~/Documents/images/claude-demo.png", "after_text": "Safe and reliable"},
       {"path": "~/Documents/images/midjourney.jpg", "after_text": "is currently the most popular AI art tool"}
     ]
   }
   ```

2. **Automate browser operations**:
   - Upload cover.jpg as cover
   - Fill in title "Top 5 AI Tools to Watch in 2024"
   - Paste rich text content (H2, bold, quotes, links, lists all preserved)
   - Insert claude-demo.png after "Safe and reliable" paragraph
   - Insert midjourney.jpg after "is currently the most popular AI art tool" paragraph

3. **Completion prompt**:
   ```
   ✅ Draft saved!

   Please preview the article in X, confirm it's correct, then manually click publish.
   Preview location: Current browser window
   ```

---

## 5. FAQ

### Q1: Why is Premium Plus required?

A: X Articles is an exclusive feature for Premium Plus subscribers. Regular users cannot access the `/compose/articles` page.

### Q2: Does it support Windows?

A: Currently only macOS is supported because clipboard operations use `pyobjc-framework-Cocoa`. Windows support requires replacing with `pywin32`, PRs welcome.

### Q3: What if image upload fails?

A: Check the following:
- Is the image path correct
- Is the image format supported (jpg, png, gif, webp)
- Is the network connection stable
- Does the image size exceed X's limit

### Q4: Format doesn't display correctly after pasting?

A: Ensure:
- Python dependencies are correctly installed
- Use `Cmd+V` to paste instead of right-click paste
- Editor has fully loaded

### Q5: Can it publish to multiple accounts?

A: Automatic account switching is not currently supported. To publish to different accounts, manually switch in the browser first.

### Q6: How to customize image compression quality?

A: In SKILL.md, image pasting uses `--quality 85` parameter. You can modify this value (1-100), lower values mean more compression.

---

## Appendix: Project Structure

```
x-article-publisher-skill/
├── .claude-plugin/
│   └── plugin.json           # Plugin configuration
├── skills/
│   └── x-article-publisher/
│       ├── SKILL.md          # Skill core instructions
│       └── scripts/
│           ├── parse_markdown.py    # Markdown parsing
│           └── copy_to_clipboard.py # Clipboard operations
├── docs/
│   └── GUIDE.md              # This document
├── README.md
└── LICENSE
```

---

## Feedback & Contributions

- **GitHub**: https://github.com/wshuyi/x-article-publisher-skill
- **Issues**: Submit an issue if you encounter problems
- **PR**: Contributions welcome, especially Windows/Linux support

---

*This document was generated by Claude Code, last updated: 2024-12*

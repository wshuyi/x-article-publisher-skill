---
name: x-article-publisher
description: |
  Publish Markdown articles to X (Twitter) Articles editor with proper formatting. Use when user wants to publish a Markdown file/URL to X Articles, or mentions "publish to X", "post article to Twitter", "X article", or wants help with X Premium article publishing. Handles cover image upload and converts Markdown to rich text automatically.
---

# X Article Publisher

Publish Markdown content to X (Twitter) Articles editor, preserving formatting with rich text conversion.

## Prerequisites

- Playwright MCP for browser automation
- User logged into X with Premium Plus subscription
- Python 3.9+ with dependencies:
  - macOS: `pip install Pillow pyobjc-framework-Cocoa`
  - Windows: `pip install Pillow pywin32 clip-util`
- For Mermaid diagrams: `npm install -g @mermaid-js/mermaid-cli`

## Scripts

Located in `~/.claude/skills/x-article-publisher/scripts/`:

### parse_markdown.py
Parse Markdown and extract structured data:
```bash
python parse_markdown.py <markdown_file> [--output json|html] [--html-only]
```
Returns JSON with: title, cover_image, content_images, **dividers** (with block_index for positioning), html, total_blocks

### copy_to_clipboard.py
Copy image or HTML to system clipboard (cross-platform):
```bash
# Copy image (with optional compression)
python copy_to_clipboard.py image /path/to/image.jpg [--quality 80]

# Copy HTML for rich text paste
python copy_to_clipboard.py html --file /path/to/content.html
```

### table_to_image.py
Convert Markdown table to PNG image:
```bash
python table_to_image.py <input.md> <output.png> [--scale 2]
```
Use when X Articles doesn't support native table rendering or for consistent styling.

## Pre-Processing (Optional)

Before publishing, scan the Markdown for elements that need conversion:

### Tables → PNG
```bash
# Extract table to temp file, then convert
python ~/.claude/skills/x-article-publisher/scripts/table_to_image.py /tmp/table.md /tmp/table.png
# Replace table in markdown with: ![Table](/tmp/table.png)
```

### Mermaid Diagrams → PNG
```bash
# Extract mermaid block to .mmd file, then convert
mmdc -i /tmp/diagram.mmd -o /tmp/diagram.png -b white -s 2
# Replace mermaid block with: ![Diagram](/tmp/diagram.png)
```

### Dividers (---)
Dividers are automatically detected by `parse_markdown.py` and output in the `dividers` array. They must be inserted via X Articles' **Insert > Divider** menu (HTML `<hr>` tags are ignored by X).

## Workflow

**Strategy: "先文后图后分割线" (Text First, Images Second, Dividers Last)**

For articles with images and dividers, paste ALL text content first, then insert images and dividers at correct positions using block index.

1. **(Optional)** Pre-process: Convert tables/mermaid to images
2. Parse Markdown with Python script → get title, images, **dividers** with block_index, HTML
3. Navigate to X Articles editor
4. Upload cover image (first image)
5. Fill title
6. Copy HTML to clipboard (Python) → Paste with Cmd+V
7. Insert content images at positions specified by block_index
8. **Insert dividers at positions specified by block_index** (via Insert > Divider menu)
9. Save as draft (NEVER auto-publish)

## 高效执行原则 (Efficiency Guidelines)

**目标**: 最小化操作之间的等待时间，实现流畅的自动化体验。

### 1. 避免不必要的 browser_snapshot

大多数浏览器操作（click, type, press_key 等）都会在返回结果中包含页面状态。**不要**在每次操作后单独调用 `browser_snapshot`，直接使用操作返回的页面状态即可。

```
❌ 错误做法：
browser_click → browser_snapshot → 分析 → browser_click → browser_snapshot → ...

✅ 正确做法：
browser_click → 从返回结果中获取页面状态 → browser_click → ...
```

### 2. 避免不必要的 browser_wait_for

只在以下情况使用 `browser_wait_for`：
- 等待图片上传完成（`textGone="正在上传媒体"`）
- 等待页面初始加载（极少数情况）

**不要**使用 `browser_wait_for` 来等待按钮或输入框出现 - 它们在页面加载完成后立即可用。

### 3. 并行执行独立操作

当两个操作没有依赖关系时，可以在同一个消息中并行调用多个工具：

```
✅ 可以并行：
- 填写标题 (browser_type) + 复制HTML到剪贴板 (Bash)
- 解析Markdown生成JSON + 生成HTML文件

❌ 不能并行（有依赖）：
- 必须先点击create才能上传封面图
- 必须先粘贴内容才能插入图片
```

### 4. 连续执行浏览器操作

每个浏览器操作返回的页面状态包含所有需要的元素引用。直接使用这些引用进行下一步操作：

```
# 理想流程（每步直接执行，不额外等待）：
browser_navigate → 从返回状态找create按钮 → browser_click(create)
→ 从返回状态找上传按钮 → browser_click(上传) → browser_file_upload
→ 从返回状态找应用按钮 → browser_click(应用)
→ 从返回状态找标题框 → browser_type(标题)
→ 点击编辑器 → browser_press_key(Meta+v)
→ ...
```

### 5. 准备工作前置

在开始浏览器操作之前，先完成所有准备工作：
1. 解析 Markdown 获取 JSON 数据
2. 生成 HTML 文件到 /tmp/
3. 记录 title、cover_image、content_images 等信息

这样浏览器操作阶段可以连续执行，不需要中途停下来处理数据。

## Step 1: Parse Markdown (Python)

Use `parse_markdown.py` to extract all structured data:

```bash
python ~/.claude/skills/x-article-publisher/scripts/parse_markdown.py /path/to/article.md
```

Output JSON:
```json
{
  "title": "Article Title",
  "cover_image": "/path/to/first-image.jpg",
  "cover_exists": true,
  "content_images": [
    {"path": "/path/to/img2.jpg", "original_path": "/md/dir/assets/img2.jpg", "exists": true, "block_index": 5, "after_text": "context..."},
    {"path": "/path/to/img3.jpg", "original_path": "/md/dir/assets/img3.jpg", "exists": true, "block_index": 12, "after_text": "another..."}
  ],
  "html": "<p>Content...</p><h2>Section</h2>...",
  "total_blocks": 45,
  "missing_images": 0
}
```

**Key fields:**
- `block_index`: The image should be inserted AFTER block element at this index (0-indexed)
- `total_blocks`: Total number of block elements in the HTML
- `after_text`: Kept for reference/debugging only, NOT for positioning
- `exists`: Whether the image file was found (if false, upload will fail)
- `original_path`: The path resolved from Markdown (before auto-search)
- `path`: The actual path to use (may differ from original_path if auto-searched)
- `missing_images`: Count of images not found anywhere

Save HTML to temp file for clipboard:
```bash
python parse_markdown.py article.md --html-only > /tmp/article_html.html
```

## Step 2: Open X Articles Editor

### 浏览器错误处理

如果遇到 `Error: Browser is already in use` 错误：

```
# 方案1：先关闭浏览器再重新打开
browser_close
browser_navigate: https://x.com/compose/articles

# 方案2：如果 browser_close 无效（锁定），提示用户手动关闭 Chrome

# 方案3：使用已有标签页，直接导航
browser_tabs action=list  # 查看现有标签
browser_navigate: https://x.com/compose/articles  # 在当前标签导航
```

**最佳实践**：每次开始前先用 `browser_tabs action=list` 检查状态，避免创建多余空白标签。

### 导航到编辑器

```
browser_navigate: https://x.com/compose/articles
```

**重要**: 页面加载后会显示草稿列表，不是编辑器。需要：

1. **等待页面加载完成**: 使用 `browser_snapshot` 检查页面状态
2. **立即点击 "create" 按钮**: 不要等待 "添加标题" 等编辑器元素，它们只有点击 create 后才出现
3. **等待编辑器加载**: 点击 create 后，等待编辑器元素出现

```
# 1. 导航到页面
browser_navigate: https://x.com/compose/articles

# 2. 获取页面快照，找到 create 按钮
browser_snapshot

# 3. 点击 create 按钮（通常 ref 类似 "create" 或带有 create 标签）
browser_click: element="create button", ref=<create_button_ref>

# 4. 现在编辑器应该打开了，可以继续上传封面图等操作
```

**注意**: 不要使用 `browser_wait_for text="添加标题"` 来等待页面加载，因为这个文本只有在点击 create 后才出现，会导致超时。

If login needed, prompt user to log in manually.

## Step 3: Upload Cover Image

1. Click "添加照片或视频" button
2. Use browser_file_upload with the cover image path (from JSON output)
3. Verify image uploaded

## Step 4: Fill Title

- Find textbox with "添加标题" placeholder
- Use browser_type to input title (from JSON output)

## Step 5: Paste Text Content (Python Clipboard)

Copy HTML to system clipboard using Python, then paste:

```bash
# Copy HTML to clipboard
python ~/.claude/skills/x-article-publisher/scripts/copy_to_clipboard.py html --file /tmp/article_html.html
```

Then in browser:
```
browser_click on editor textbox
browser_press_key: Meta+v
```

This preserves all rich text formatting (H2, bold, links, lists).

## Step 6: Insert Content Images (Text Search Positioning)

**推荐方法**: 使用 `after_text` 文字搜索定位，比 `block_index` 更直观可靠。

### 定位原理

每张图片的 `after_text` 字段记录了它前一个段落的末尾文字（最多80字符）。在编辑器中搜索包含该文字的段落，点击后插入图片。

### 操作步骤

For each content image (from `content_images` array), **按 block_index 从大到小的顺序**：

```bash
# 1. Copy image to clipboard (with compression)
python ~/.claude/skills/x-article-publisher/scripts/copy_to_clipboard.py image /path/to/img.jpg --quality 85
```

```
# 2. 在 browser_snapshot 中搜索包含 after_text 的段落
#    找到该段落的 ref

# 3. Click the paragraph containing after_text
browser_click: element="paragraph with target text", ref=<paragraph_ref>

# 4. **关键步骤**: 按 End 键移动光标到行尾
#    这一步非常重要！避免点击到段落中的链接导致位置偏移
browser_press_key: End

# 5. Paste image
browser_press_key: Meta+v

# 6. Wait for upload (only use textGone, no time parameter)
browser_wait_for textGone="正在上传媒体"
```

### 为什么需要按 End 键？

**问题**: 当段落包含链接时（如 `[链接文字](url)`），点击段落可能会：
- 触发链接编辑弹窗
- 将光标定位在链接内部而非段落末尾

**解决方案**: 点击段落后立即按 `End` 键：
- 确保光标移动到段落末尾
- 避免链接干扰
- 图片将正确插入在该段落之后

### 定位策略

在 browser_snapshot 返回的结构中，搜索 `after_text` 的关键词：

```yaml
textbox [ref=editor]:
  generic [ref=p1]:
    - StaticText: "元旦假期我在家里翻手机相册..."  # 如果 after_text 包含这段文字，点击 p1
  heading [ref=h1]:
    - StaticText: "演示"
  generic [ref=p2]:
    - StaticText: "这东西到底有多省事儿？"
    - link [ref=link1]: "Claude Code"  # 注意：段落可能包含链接
  ...
```

### 反向插入示例

如果有3张图片，block_index 分别为 5, 12, 27：
1. 先插入 block_index=27 的图片（after_text 搜索 + End + 粘贴）
2. 再插入 block_index=12 的图片
3. 最后插入 block_index=5 的图片

**从大到小插入**可以避免位置偏移问题。

## Step 6.5: Insert Dividers (Via Menu)

**重要**: Markdown 中的 `---` 分割线不能通过 HTML `<hr>` 标签粘贴（X Articles 会忽略它）。必须通过 X Articles 的 Insert 菜单插入。

### 操作步骤

For each divider (from `dividers` array), in **reverse order of block_index**:

```
# 1. Click the block element at block_index position
browser_click on the element at position block_index in the editor

# 2. Open Insert menu (Add Media button)
browser_click on "Insert" or "添加媒体" button

# 3. Click Divider menu item
browser_click on "Divider" or "分割线" menuitem

# Divider is inserted at cursor position
```

### 与图片的插入顺序

建议先插入所有图片，再插入所有分割线。两者都按 block_index **从大到小**的顺序：

1. 插入所有图片（从最大 block_index 开始）
2. 插入所有分割线（从最大 block_index 开始）

## Step 7: Save Draft

1. Verify content pasted (check word count indicator)
2. Draft auto-saves, or click Save button if needed
3. Click "预览" to verify formatting
4. Report: "Draft saved. Review and publish manually."

## Critical Rules

1. **NEVER publish** - Only save draft
2. **First image = cover** - Upload first image as cover image
3. **Rich text conversion** - Always convert Markdown to HTML before pasting
4. **Use clipboard API** - Paste via clipboard for proper formatting
5. **Block index positioning** - Use block_index for precise image/divider placement
6. **Reverse order insertion** - Insert images and dividers from highest to lowest block_index
7. **H1 title handling** - H1 is used as title only, not included in body
8. **Dividers via menu** - Markdown `---` must be inserted via Insert > Divider menu (HTML `<hr>` is ignored)

## Supported Formatting

| Element | Support | Notes |
|---------|---------|-------|
| H2 (`##`) | Native | Section headers |
| Bold (`**`) | Native | Strong emphasis |
| Italic (`*`) | Native | Emphasis |
| Links (`[](url)`) | Native | Hyperlinks |
| Ordered lists | Native | 1. 2. 3. |
| Unordered lists | Native | - bullets |
| Blockquotes (`>`) | Native | Quoted text |
| Code blocks | Converted | → Blockquotes |
| Tables | Converted | → PNG images (use table_to_image.py) |
| Mermaid | Converted | → PNG images (use mmdc) |
| Dividers (`---`) | Menu insert | → Insert > Divider |

## Example Flow

User: "Publish /path/to/article.md to X"

```bash
# Step 1: Parse Markdown
python ~/.claude/skills/x-article-publisher/scripts/parse_markdown.py /path/to/article.md > /tmp/article.json
python ~/.claude/skills/x-article-publisher/scripts/parse_markdown.py /path/to/article.md --html-only > /tmp/article_html.html
```

2. Navigate to https://x.com/compose/articles
3. Upload cover image (browser_file_upload for cover only)
4. Fill title (from JSON: `title`)
5. Copy & paste HTML:
   ```bash
   python ~/.claude/skills/x-article-publisher/scripts/copy_to_clipboard.py html --file /tmp/article_html.html
   ```
   Then: browser_press_key Meta+v
6. For each content image, **in reverse order of block_index**:
   ```bash
   python copy_to_clipboard.py image /path/to/img.jpg --quality 85
   ```
   - Click block element at `block_index` position
   - browser_press_key Meta+v
   - Wait until upload complete
7. Verify in preview
8. "Draft saved. Please review and publish manually."

## Best Practices

### 为什么用 block_index 而非文字匹配？

1. **精确定位**: 不依赖文字内容，即使多处文字相似也能正确定位
2. **可靠性**: 索引是确定性的，不会因为文字相似而混淆
3. **调试方便**: `after_text` 仍保留用于人工核验

### 为什么用 Python 而非浏览器内 JavaScript？

1. **本地处理更可靠**: Python 直接操作系统剪贴板，不受浏览器沙盒限制
2. **图片压缩**: 上传前压缩图片 (--quality 85)，减少上传时间
3. **代码复用**: 脚本固定不变，无需每次重新编写转换逻辑
4. **调试方便**: 脚本可单独测试，问题易定位

### 等待策略

**重要发现**: Playwright MCP 的 `browser_wait_for` 实际行为是 **先等待 time 秒，再检查条件**，而非轮询！

```javascript
// 实际执行的代码：
await new Promise(f => setTimeout(f, time * 1000));  // 先固定等待
await page.getByText("xxx").waitFor({ state: 'hidden' });  // 再检查
```

**正确用法**:
- ✅ 只用 `textGone`，不设 `time`：让 Playwright 自己轮询等待
- ✅ 只用 `time`：固定等待指定秒数
- ❌ 同时用 `textGone` + `time`：会先等 time 秒再检查，浪费时间

```
# 推荐：只用 textGone，让它自动等待条件满足
browser_wait_for textGone="正在上传媒体"

# 或者：用 browser_snapshot 轮询检查状态
# 每次操作后检查返回的页面状态，无需额外等待
```

### 图片插入效率

每张图片的浏览器操作从5步减少到2步：
- 旧: 点击 → 添加媒体 → 媒体 → 添加照片 → file_upload
- 新: 点击段落 → Meta+v

### 封面图 vs 内容图

- **封面图**: 使用 browser_file_upload（因为有专门的上传按钮）
- **内容图**: 使用 Python 剪贴板 + 粘贴（更高效）

## 故障排除

### MCP 连接问题

如果 Playwright MCP 工具不可用（报错 `No such tool available` 或 `Not connected`）：

**方案1：重新连接 MCP（推荐）**
```
执行 /mcp 命令，选择 playwright，选择 Restart
```

**方案2：清理残留进程后重连**
```bash
# 杀掉所有残留的 playwright 进程
pkill -f "mcp-server-playwright"
pkill -f "@playwright/mcp"

# 然后执行 /mcp 重新连接
```

**配置文件位置**: `~/.claude/mcp_servers.json`

### 浏览器错误处理

如果遇到 `Error: Browser is already in use` 错误：

```bash
# 方案1：先关闭浏览器再重新打开
browser_close
browser_navigate: https://x.com/compose/articles

# 方案2：杀掉 Chrome 进程
pkill -f "Chrome.*--remote-debugging"
# 然后重新 navigate
```

### 图片位置偏移

如果图片插入位置不正确（特别是点击含链接的段落时）：

**原因**: 点击段落时可能误触链接，导致光标位置错误

**解决方案**: 点击后**必须按 End 键**移动光标到行尾

```
# 正确流程
1. browser_click 点击目标段落
2. browser_press_key: End        # 关键步骤！
3. browser_press_key: Meta+v     # 粘贴图片
4. browser_wait_for textGone="正在上传媒体"
```

### 图片路径找不到

如果 Markdown 中的相对路径图片找不到（如 `./assets/image.png` 实际在其他位置）：

**自动搜索**: `parse_markdown.py` 会自动在以下目录搜索同名文件：
- `~/Downloads`
- `~/Desktop`
- `~/Pictures`

**stderr 输出示例**:
```
[parse_markdown] Image not found at '/path/to/assets/img.png', using '/Users/xxx/Downloads/img.png' instead
```

**JSON 字段说明**:
- `original_path`: Markdown 中指定的路径（解析后的绝对路径）
- `path`: 实际使用的路径（如果自动搜索成功，会不同于 original_path）
- `exists`: `true` 表示找到文件，`false` 表示未找到（上传会失败）

**如果仍然找不到**:
1. 检查 JSON 输出中的 `missing_images` 字段
2. 手动将图片复制到 Markdown 文件同目录的 `assets/` 子目录
3. 或修改 Markdown 中的图片路径为绝对路径

# X Article Publisher Skill 使用指南

> 一键将 Markdown 文章发布到 X (Twitter) Articles，告别繁琐的富文本编辑。

---

## 目录

1. [解决的痛点](#1-解决的痛点)
2. [解决方案](#2-解决方案)
3. [执行方式](#3-执行方式)
4. [完整示例](#4-完整示例)
5. [常见问题](#5-常见问题)

---

## 1. 解决的痛点

### 1.1 X Articles 是什么？

X (原 Twitter) 为 Premium Plus 订阅用户提供了 **Articles** 功能，允许用户发布长文章（类似博客），突破 280 字符限制。文章支持：

- 富文本格式（标题、粗体、引用等）
- 多张图片嵌入
- 超链接

访问入口：https://x.com/compose/articles

### 1.2 手动发布的痛点

如果你习惯用 Markdown 写作，将内容发布到 X Articles 是一个**极其繁琐**的过程：

#### 痛点一：格式无法直接粘贴

```
❌ 从 Markdown 编辑器复制内容 → 粘贴到 X Articles → 格式全部丢失
```

X Articles 编辑器不支持 Markdown，你需要**逐一手动设置**每个格式：
- 选中文字 → 点击 H2 按钮
- 选中文字 → 点击粗体按钮
- 选中文字 → 点击链接按钮 → 粘贴 URL

一篇包含 5 个小节、10 处加粗、8 个链接的文章，格式设置可能需要 **15-20 分钟**。

#### 痛点二：图片插入效率低下

X Articles 的图片插入流程：

```
点击段落 → 点击"添加媒体内容" → 点击"媒体" → 点击"添加照片或视频" → 选择文件 → 等待上传
```

每张图片需要 **5 次点击 + 文件选择 + 等待上传**。一篇包含 5 张图片的文章，仅图片插入就需要 **5-10 分钟**。

#### 痛点三：图片位置难以精确控制

Markdown 中图片位置是精确的：

```markdown
这是第一段内容。

![图1](image1.jpg)

这是第二段内容。

![图2](image2.jpg)
```

但在 X Articles 中，你需要：
1. 记住每张图片应该插入的位置
2. 滚动找到正确的段落
3. 手动插入

图片越多，出错概率越高。

#### 痛点四：重复劳动

如果你经常发布长文章，这些操作需要**每次重复**：
- 每篇文章都要手动转换格式
- 每张图片都要重复 5 次点击
- 每次都要检查格式是否正确

### 1.3 时间成本对比

| 操作 | 手动方式 | 使用本 Skill |
|------|----------|--------------|
| 格式转换 | 15-20 分钟 | 0（自动） |
| 封面图上传 | 1-2 分钟 | 10 秒 |
| 5 张内容图插入 | 5-10 分钟 | 1 分钟 |
| 总计 | **20-30 分钟** | **2-3 分钟** |

**效率提升：10 倍以上**

---

## 2. 解决方案

### 2.1 技术架构

本 Skill 通过以下技术栈实现自动化：

```
┌─────────────────┐
│   Markdown 文件  │
└────────┬────────┘
         │ Python 解析
         ▼
┌─────────────────┐
│  结构化数据 (JSON) │
│  - title        │
│  - cover_image  │
│  - content_images│
│  - html         │
└────────┬────────┘
         │ Playwright MCP
         ▼
┌─────────────────┐
│  X Articles 编辑器 │
│  (浏览器自动化)   │
└─────────────────┘
```

#### 核心组件

| 组件 | 作用 |
|------|------|
| `parse_markdown.py` | 解析 Markdown，提取标题、图片、转换 HTML |
| `copy_to_clipboard.py` | 将 HTML/图片复制到系统剪贴板 |
| Playwright MCP | 控制浏览器，模拟用户操作 |
| Claude Code | 协调整个流程 |

### 2.2 工作流程："先文后图"策略

本 Skill 采用**先文后图**（Text First, Images Later）策略，确保内容完整且图片位置准确：

```
Step 1: 解析 Markdown
        ↓
Step 2: 打开 X Articles 编辑器
        ↓
Step 3: 上传封面图（第一张图片）
        ↓
Step 4: 填写标题
        ↓
Step 5: 粘贴 HTML 富文本内容（通过剪贴板）
        ↓
Step 6: 在正确位置插入内容图片（通过剪贴板粘贴）
        ↓
Step 7: 保存草稿（绝不自动发布）
```

### 2.3 关键技术优化

#### 优化一：剪贴板富文本粘贴

传统方式需要逐个设置格式，本 Skill 通过：

1. Python 将 Markdown 转换为 HTML
2. 将 HTML 复制到系统剪贴板（保留富文本格式）
3. 在编辑器中 `Cmd+V` 粘贴

```python
# copy_to_clipboard.py 核心逻辑
from AppKit import NSPasteboard, NSPasteboardTypeHTML

pasteboard = NSPasteboard.generalPasteboard()
pasteboard.setData_forType_(html_data, NSPasteboardTypeHTML)
```

结果：**所有格式一次性粘贴完成**，包括 H2、粗体、链接、列表等。

#### 优化二：图片剪贴板粘贴

传统方式每张图片需要 5 次点击，本 Skill 通过：

1. Python 将图片复制到系统剪贴板
2. 点击目标段落
3. `Cmd+V` 粘贴图片

```python
# copy_to_clipboard.py 图片处理
from AppKit import NSPasteboard, NSPasteboardTypeTIFF

# 可选：上传前压缩图片
img.thumbnail((2000, 2000))
img.save(buffer, format='JPEG', quality=85)
```

对比：

| 方式 | 浏览器操作次数 |
|------|--------------|
| 传统 | 5 次点击 + 文件选择 |
| 本 Skill | 1 次点击 + 1 次粘贴 |

#### 优化三：图片位置精确定位

`parse_markdown.py` 提取每张图片的**上下文信息**：

```json
{
  "content_images": [
    {
      "path": "/path/to/img1.jpg",
      "after_text": "这是图片前面的一段文字内容..."
    }
  ]
}
```

Claude 使用 `after_text` 在编辑器中定位正确的段落，确保图片插入位置与原文一致。

### 2.4 支持的 Markdown 格式

| Markdown 语法 | 效果 |
|--------------|------|
| `# H1` | 文章标题（自动提取） |
| `## H2` | 二级标题 |
| `**粗体**` | **粗体文字** |
| `*斜体*` | *斜体文字* |
| `[链接](url)` | 超链接 |
| `> 引用` | 引用块 |
| `- 列表` | 无序列表 |
| `1. 列表` | 有序列表 |
| `![](img.jpg)` | 图片（第一张为封面） |

### 2.5 安全设计

**绝不自动发布**：本 Skill 仅将内容保存为草稿，最终发布需要用户手动确认。

```
✅ 自动完成：上传、格式化、排版
❌ 不会执行：点击"发布"按钮
```

这确保用户可以在发布前：
- 预览最终效果
- 检查格式是否正确
- 修改任何需要调整的内容

---

## 3. 执行方式

### 3.1 前置条件

#### 条件一：X Premium Plus 订阅

Articles 功能仅对 Premium Plus 用户开放。验证方式：
1. 访问 https://x.com/compose/articles
2. 如果能看到编辑器，说明你有权限

#### 条件二：安装 Playwright MCP

本 Skill 依赖 Playwright MCP 进行浏览器自动化。

检查是否已安装：
```bash
cat ~/.claude/settings.json | grep playwright
```

如未安装，在 Claude Code 中执行：
```
帮我安装 Playwright MCP
```

#### 条件三：安装 Python 依赖

```bash
pip install Pillow pyobjc-framework-Cocoa
```

验证安装：
```bash
python -c "from AppKit import NSPasteboard; print('OK')"
```

#### 条件四：安装本 Skill

**方式 A：Git Clone（推荐）**

```bash
git clone https://github.com/wshuyi/x-article-publisher-skill.git
cp -r x-article-publisher-skill/skills/x-article-publisher ~/.claude/skills/
```

**方式 B：插件市场**

```
/plugin marketplace add wshuyi/x-article-publisher-skill
/plugin install x-article-publisher@wshuyi/x-article-publisher-skill
```

### 3.2 触发指令

安装完成后，在 Claude Code 中使用以下方式触发：

#### 方式一：自然语言

```
把 /path/to/article.md 发布到 X
```

```
帮我把这篇文章发到 X Articles：~/Documents/my-article.md
```

```
Publish ~/blog/post.md to X
```

#### 方式二：Skill 命令

```
/x-article-publisher /path/to/article.md
```

### 3.3 操作流程

触发后，Claude 会自动执行以下步骤：

```
[1/7] 解析 Markdown 文件...
      → 提取标题：「你的文章标题」
      → 发现封面图：cover.jpg
      → 发现 3 张内容图片
      → HTML 转换完成

[2/7] 打开 X Articles 编辑器...
      → 导航到 https://x.com/compose/articles
      → 等待编辑器加载

[3/7] 上传封面图...
      → 点击"添加照片或视频"
      → 上传 cover.jpg
      → 等待上传完成

[4/7] 填写标题...
      → 输入：「你的文章标题」

[5/7] 粘贴文章内容...
      → HTML 已复制到剪贴板
      → 粘贴富文本内容
      → 格式保留：5 个 H2，8 处粗体，12 个链接

[6/7] 插入内容图片...
      → 图片 1/3：定位到「这是第一段内容...」后
      → 图片 2/3：定位到「这是第二段内容...」后
      → 图片 3/3：定位到「这是第三段内容...」后

[7/7] 保存草稿...
      → 草稿已自动保存
      → 请在 X 中预览并手动发布
```

### 3.4 注意事项

1. **保持浏览器可见**：Playwright 需要控制浏览器窗口，请不要最小化
2. **已登录 X**：确保浏览器中已登录你的 X 账号
3. **图片路径**：Markdown 中的图片路径需要是有效的本地路径
4. **网络稳定**：图片上传需要稳定的网络连接

---

## 4. 完整示例

### 4.1 示例 Markdown 文件

文件路径：`~/Documents/ai-tools-review.md`

```markdown
# 2024 年最值得关注的 5 个 AI 工具

![cover](./images/cover.jpg)

人工智能工具在 2024 年迎来了爆发式增长。本文将介绍 5 个最值得关注的 AI 工具。

## 1. Claude：最强对话 AI

**Claude** 由 Anthropic 开发，在长文本理解和代码生成方面表现出色。

> Claude 的上下文窗口高达 200K tokens，可以处理整本书的内容。

主要特点：
- 超长上下文理解
- 出色的推理能力
- 安全可靠

![claude-demo](./images/claude-demo.png)

## 2. Midjourney：AI 绘画领导者

[Midjourney](https://midjourney.com) 是目前最受欢迎的 AI 绘画工具。

![midjourney-example](./images/midjourney.jpg)

## 总结

这些工具正在改变我们的工作方式。选择适合自己的工具，提升效率。
```

### 4.2 执行命令

```
把 ~/Documents/ai-tools-review.md 发布到 X
```

### 4.3 执行结果

Claude 会：

1. **解析文件**，输出：
   ```json
   {
     "title": "2024 年最值得关注的 5 个 AI 工具",
     "cover_image": "~/Documents/images/cover.jpg",
     "content_images": [
       {"path": "~/Documents/images/claude-demo.png", "after_text": "安全可靠"},
       {"path": "~/Documents/images/midjourney.jpg", "after_text": "是目前最受欢迎的 AI 绘画工具"}
     ]
   }
   ```

2. **自动操作浏览器**：
   - 上传 cover.jpg 作为封面
   - 填写标题「2024 年最值得关注的 5 个 AI 工具」
   - 粘贴富文本内容（H2、粗体、引用、链接、列表全部保留）
   - 在「安全可靠」段落后插入 claude-demo.png
   - 在「是目前最受欢迎的 AI 绘画工具」段落后插入 midjourney.jpg

3. **完成提示**：
   ```
   ✅ 草稿已保存！

   请在 X 中预览文章效果，确认无误后手动点击发布。
   预览地址：当前浏览器窗口
   ```

---

## 5. 常见问题

### Q1: 为什么需要 Premium Plus？

A: X Articles 是 Premium Plus 订阅专属功能。普通用户无法访问 `/compose/articles` 页面。

### Q2: 支持 Windows 吗？

A: 目前仅支持 macOS，因为剪贴板操作使用了 `pyobjc-framework-Cocoa`。Windows 支持需要替换为 `pywin32`，欢迎贡献 PR。

### Q3: 图片上传失败怎么办？

A: 检查以下几点：
- 图片路径是否正确
- 图片格式是否支持（jpg, png, gif, webp）
- 网络连接是否稳定
- 图片大小是否超过 X 限制

### Q4: 格式粘贴后显示不正确？

A: 确保：
- Python 依赖已正确安装
- 使用 `Cmd+V` 粘贴而非右键粘贴
- 编辑器已完全加载

### Q5: 可以发布到多个账号吗？

A: 目前不支持自动切换账号。如需发布到不同账号，请手动在浏览器中切换后再执行。

### Q6: 如何自定义图片压缩质量？

A: 在 SKILL.md 中，图片粘贴使用 `--quality 85` 参数。你可以修改这个值（1-100），数值越低压缩越多。

---

## 附录：项目结构

```
x-article-publisher-skill/
├── .claude-plugin/
│   └── plugin.json           # 插件配置
├── skills/
│   └── x-article-publisher/
│       ├── SKILL.md          # Skill 核心指令
│       └── scripts/
│           ├── parse_markdown.py    # Markdown 解析
│           └── copy_to_clipboard.py # 剪贴板操作
├── docs/
│   └── GUIDE.md              # 本文档
├── README.md
└── LICENSE
```

---

## 反馈与贡献

- **GitHub**: https://github.com/wshuyi/x-article-publisher-skill
- **Issues**: 遇到问题请提交 Issue
- **PR**: 欢迎贡献代码，特别是 Windows/Linux 支持

---

*本文档由 Claude Code 生成，最后更新：2024-12*

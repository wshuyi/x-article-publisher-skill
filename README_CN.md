# X Article Publisher Skill

[English](README.md) | [中文](README_CN.md)

> 一键将 Markdown 文章发布到 X (Twitter) Articles，告别繁琐的富文本编辑。

**v1.2.0** — 新增分割线支持、表格转图片、Mermaid 支持、跨平台剪贴板

---

## 痛点分析

如果你习惯用 Markdown 写作，将内容发布到 X Articles 是一个**极其痛苦**的过程：

| 痛点 | 描述 |
|------|------|
| **格式丢失** | 从 Markdown 编辑器复制 → 粘贴到 X → 格式全部丢失 |
| **手动格式化** | 逐个设置 H2、粗体、链接 — 每篇文章 15-20 分钟 |
| **图片上传繁琐** | 每张图片 5 次点击：添加媒体 → 媒体 → 添加照片 → 选择 → 等待 |
| **位置容易出错** | 很难记住每张图片应该插入的位置 |

### 时间对比

| 操作 | 手动方式 | 使用本 Skill |
|------|----------|--------------|
| 格式转换 | 15-20 分钟 | 0（自动） |
| 封面图上传 | 1-2 分钟 | 10 秒 |
| 5 张内容图插入 | 5-10 分钟 | 1 分钟 |
| **总计** | **20-30 分钟** | **2-3 分钟** |

**效率提升 10 倍以上**

---

## 解决方案

本 Skill 自动化整个发布流程：

```
Markdown 文件
     ↓ Python 解析
结构化数据（标题、带 block_index 的图片、HTML）
     ↓ Playwright MCP
X Articles 编辑器（浏览器自动化）
     ↓
保存草稿（绝不自动发布）
```

### 核心特性

- **富文本粘贴**：将 Markdown 转换为 HTML，通过剪贴板粘贴 — 所有格式完整保留
- **Block-Index 定位**（v1.1）：使用元素索引精确定位，不依赖文字匹配
- **反向插入**：从高到低索引插入图片，避免位置偏移
- **智能等待策略**：条件满足立即返回，不浪费等待时间
- **安全设计**：仅保存草稿，绝不自动发布

---

## v1.1.0 更新内容

| 功能 | 之前 | 现在 |
|------|------|------|
| 图片定位 | 文字匹配（不稳定） | Block 索引（精确） |
| 插入顺序 | 顺序插入 | 反向插入（从大到小） |
| 等待行为 | 固定延时 | 条件满足立即返回 |

### 为什么使用 Block-Index？

之前的版本通过匹配周围文字来定位图片位置，但遇到以下情况会失败：
- 多个段落内容相似
- 文字太短不够唯一

现在，每张图片都有一个 `block_index` 表示它应该跟在第几个块元素后面。这是确定性的，不会出错。

---

## 环境要求

| 要求 | 说明 |
|------|------|
| Claude Code | [claude.ai/code](https://claude.ai/code) |
| Playwright MCP | 浏览器自动化 |
| X Premium Plus | Articles 功能需要此订阅 |
| Python 3.9+ | 需安装以下依赖 |
| 操作系统 | macOS 或 Windows |

```bash
# macOS
pip install Pillow pyobjc-framework-Cocoa

# Windows
pip install Pillow pywin32 clip-util

# Mermaid 图表支持（可选）
npm install -g @mermaid-js/mermaid-cli
```

---

## 安装方式

### 方式一：Git Clone（推荐）

```bash
git clone https://github.com/wshuyi/x-article-publisher-skill.git
cp -r x-article-publisher-skill/skills/x-article-publisher ~/.claude/skills/
```

### 方式二：插件市场

```
/plugin marketplace add wshuyi/x-article-publisher-skill
/plugin install x-article-publisher@wshuyi/x-article-publisher-skill
```

---

## 使用方法

### 自然语言

```
把 /path/to/article.md 发布到 X
```

```
帮我把这篇文章发到 X Articles：~/Documents/my-post.md
```

### Skill 命令

```
/x-article-publisher /path/to/article.md
```

---

## 工作流程

```
[1/7] 解析 Markdown...
      → 提取标题、封面图、带 block_index 的内容图片
      → 转换为 HTML，统计块元素总数

[2/7] 打开 X Articles 编辑器...
      → 导航到 x.com/compose/articles

[3/7] 上传封面图...
      → 第一张图片作为封面

[4/7] 填写标题...
      → H1 作为标题（不包含在正文中）

[5/7] 粘贴文章内容...
      → 通过剪贴板粘贴富文本
      → 所有格式完整保留

[6/7] 插入内容图片（反向顺序）...
      → 按 block_index 从大到小排序
      → 点击对应块元素 → 粘贴图片
      → 等待上传（完成后立即返回）

[7/7] 保存草稿...
      → ✅ 请手动预览并发布
```

---

## 支持的 Markdown 格式

| 语法 | 效果 | 说明 |
|------|------|------|
| `# H1` | 文章标题 | 提取后不含在正文中 |
| `## H2` | 二级标题 | 原生支持 |
| `**粗体**` | **粗体文字** | 原生支持 |
| `*斜体*` | *斜体文字* | 原生支持 |
| `[文字](url)` | 超链接 | 原生支持 |
| `> 引用` | 引用块 | 原生支持 |
| `- 列表` | 无序列表 | 原生支持 |
| `1. 列表` | 有序列表 | 原生支持 |
| `![](img.jpg)` | 图片 | 第一张为封面 |
| `---` | 分割线 | 通过菜单插入 (v1.2) |
| 表格 | PNG 图片 | 使用 table_to_image.py (v1.2) |
| Mermaid | PNG 图片 | 使用 mmdc CLI (v1.2) |

---

## 完整示例

### 输入：`article.md`

```markdown
# 2024 年最值得关注的 5 个 AI 工具

![封面](./images/cover.jpg)

人工智能工具在 2024 年迎来了爆发式增长。本文将介绍 5 个最值得关注的工具。

## 1. Claude：最强对话 AI

**Claude** 由 Anthropic 开发，在长文本理解方面表现出色。

> Claude 的上下文窗口高达 200K tokens。

![claude-demo](./images/claude-demo.png)

## 2. Midjourney：AI 绘画领导者

[Midjourney](https://midjourney.com) 是目前最受欢迎的 AI 绘画工具。

![midjourney](./images/midjourney.jpg)
```

### 解析输出（JSON）

```json
{
  "title": "2024 年最值得关注的 5 个 AI 工具",
  "cover_image": "./images/cover.jpg",
  "content_images": [
    {"path": "./images/claude-demo.png", "block_index": 4},
    {"path": "./images/midjourney.jpg", "block_index": 6}
  ],
  "total_blocks": 7
}
```

### 插入顺序

图片按反向顺序插入：先 `block_index=6`，再 `block_index=4`。

### 执行结果

- 封面：`cover.jpg` 已上传
- 标题：「2024 年最值得关注的 5 个 AI 工具」
- 内容：富文本格式完整保留（H2、粗体、引用、链接）
- 图片：通过 block index 精确定位插入
- 状态：**草稿已保存**（等待手动预览发布）

---

## 项目结构

```
x-article-publisher-skill/
├── .claude-plugin/
│   └── plugin.json              # 插件配置
├── skills/
│   └── x-article-publisher/
│       ├── SKILL.md             # Skill 核心指令
│       └── scripts/
│           ├── parse_markdown.py    # 提取 block_index + 分割线
│           ├── copy_to_clipboard.py # 跨平台剪贴板
│           └── table_to_image.py    # 表格转 PNG (v1.2)
├── docs/
│   └── GUIDE.md                 # 详细使用指南
├── README.md                    # 英文说明
├── README_CN.md                 # 中文说明（本文件）
└── LICENSE
```

---

## 常见问题

**Q: 为什么需要 Premium Plus？**
A: X Articles 是 Premium Plus 订阅专属功能，普通用户无法使用。

**Q: 支持 Windows/Linux 吗？**
A: Windows 已支持 (v1.2)。Linux 支持仍在开发中，欢迎贡献 PR！

**Q: 图片上传失败怎么办？**
A: 检查：路径是否正确、格式是否支持（jpg/png/gif/webp）、网络是否稳定。

**Q: 可以发布到多个账号吗？**
A: 不支持自动切换。请在浏览器中手动切换账号后再执行。

**Q: 为什么要反向插入图片？**
A: 每插入一张图片，后续块元素的索引会偏移。从大到小插入可以确保前面的索引不受影响。

**Q: 之前的文字匹配方式还能用吗？**
A: v1.1 已用 `block_index` 替代文字匹配。`after_text` 字段保留用于调试，但不再用于定位。

**Q: 为什么等待有时候立即返回？**
A: `browser_wait_for textGone="..."` 会在文字消失时立即返回。`time` 参数只是最大等待时间，不是固定延时。

---

## 详细文档

- [完整使用指南](docs/GUIDE.md) — 包含详细说明和更多示例

---

## 更新日志

### v1.2.0 (2025-01)
- **分割线支持**：识别 Markdown 中的 `---`，通过 X Articles 菜单插入
- **表格转图片**：新增 `table_to_image.py` 脚本，将表格转为 PNG
- **Mermaid 支持**：文档说明如何使用 `mmdc` 转换图表
- **YAML 头部跳过**：自动跳过 Markdown 文件开头的 frontmatter
- **Windows 支持**：跨平台剪贴板操作（pywin32 + clip-util）

### v1.1.0 (2025-12)
- **Block-index 定位**：用精确的元素索引替代文字匹配
- **反向插入顺序**：防止多图插入时的索引偏移
- **优化等待策略**：上传完成后立即返回
- **H1 标题处理**：H1 提取为标题，不包含在正文 HTML 中

### v1.0.0 (2025-12)
- 首次发布
- 剪贴板富文本粘贴
- 封面图 + 内容图支持
- 仅保存草稿

---

## 许可证

MIT License - 见 [LICENSE](LICENSE)

## 作者

[wshuyi](https://github.com/wshuyi)

---

## 致谢

v1.2.0 的新功能参考并借鉴自：

- **[sugarforever/01coder-agent-skills](https://github.com/sugarforever/01coder-agent-skills)** — 其中的 `publish-x-article` 技能贡献了以下功能思路：
  - 分割线检测与菜单插入
  - 表格转图片脚本
  - Mermaid 图表支持文档
  - YAML frontmatter 处理
  - Windows 剪贴板实现

感谢社区成员在本项目基础上的改进和贡献！

---

## 贡献

- **Issues**：报告 Bug 或提出功能建议
- **PR**：欢迎贡献代码，特别是 Linux 支持

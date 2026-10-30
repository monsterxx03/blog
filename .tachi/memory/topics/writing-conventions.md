# Writing Conventions (博文写法)

## 2026-10-30: 新建文章的路径、文件名与 front matter

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: content/posts, 文件名, slug, front matter, categories, tags, draft, archetypes, 新建文章

新文章放 `content/posts/YYYY-MM-DD-slug.md`（如 `2026-10-30-tachi-agent-networking.md`）；永久链接为 `/:year/:month/:day/:title/`，所以 slug 即 URL 末段。Front matter 惯例是 `title` / `date`（带 +08:00）/ `categories`（技术文用 `tech`）/ `tags`，已发布文章里不保留 `draft: true`（archetype 默认带 draft，需去掉）。

---

## 2026-10-30: 图片与正文排版习惯

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 图片, images, /posts/images, markdown 表格, 代码块, 排版

文章配图放在 `content/posts/images/`，正文用 `/posts/images/<name>.png` 引用（Markdown 图片语法）。正文大量使用 markdown 表格对比取舍、Go/JSON/YAML 代码块引用真实实现，不做抽象伪代码。

---

## 2026-10-30: 风格偏好：第一人称、讲取舍与踩坑、结尾写「没做的事」

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 写作风格, 风格, 踩坑, 取舍, 一、二、三, 小节编号, 系列文章, 个人博客

技术长文的风格：中文第一人称（「我」），从具体场景/需求出发；小节用「一、二、三…」编号 + 加粗领句；踩坑条目编号罗列（真实 bug 与约束）；明确写出设计取舍、上限、以及"不做的事情"，并在结尾承认不好听的方面和未完成项。系列文章开头会链到上一篇。

---

## 2026-10-30: 标题命名习惯

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 标题, title, 命名, 系列, tachi

标题常用「主题: 副标题（关键词、关键词）」形式，如 `Tachi: agent 组网(mDNS,Gossip) 和 A2A`、`Tachi: 从 0 开始做 agent`——冒号前是项目/主题，括号里点出关键技术词。

---

## 2026-10-30: 动笔前先读仓库与既有文章

来源: session 2026-10-30-225112-e02997ea
状态: active
关键词: 流程, 写作流程, 调研, 既有文章, 风格对齐

用户给出一句标题式需求（"新建一篇 blog，标题是…"）后，应先看仓库结构、既有文章的写法，再去核对文章涉及的实际实现，然后才动笔；不要凭印象虚构细节。

---

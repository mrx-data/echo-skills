# SPA / JS 渲染页面的内容提取

## 问题

`curl` 下载的 HTML 仅包含 JS 骨架（`<script>` 标签 + CSS），正文通过客户端 JS 异步渲染。常见于：

- Notion 公开页面
- 微信公众号文章
- 知乎
- Medium
- 各类 SPA（React/Vue/Angular 构建的站点）

## 检测方法

```python
import os
html_size = os.path.getsize(f"./.ltn_page_{slug}.html")
if html_size < 20_000:
    # 疑似 SPA — 检查是否包含正文关键词
    with open(f"./.ltn_page_{slug}.html", 'r') as f:
        preview = f.read(5000)
    has_content = any(kw in preview for kw in ['<article', 'content', 'post-body', 'entry-content'])
    if not has_content:
        print("→ SPA detected, falling back to browser")
        use_browser = True
```

## 浏览器提取流程

### 1. 渲染页面

```
browser_navigate(url="<URL>")
```

### 2. 等待加载后获取快照（必要时）

```
browser_snapshot(full=true)
```

### 3. 提取纯文本（按站点类型选选择器）

```
browser_console(expression="document.querySelector('.notion-page-content')?.innerText || document.querySelector('main')?.innerText || document.body.innerText")
```

### 常用选择器

| 站点 | 选择器 |
|------|--------|
| Notion | `.notion-page-content` |
| 微信公众号 | `#js_content` 或 `#img-content .rich_media_content` |
| 知乎 | `.Post-RichTextContainer` 或 `.RichContent-inner` |
| Medium | `article` |
| 通用 | `main`, `article`, `[role="main"]` |

### 4. 将提取文本作为 content_text

提取到的纯文本直接作为 `content_text` 进入 Step 3 组织笔记，不需要再经过 html.parser。

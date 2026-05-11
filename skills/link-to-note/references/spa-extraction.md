# SPA / JS-Rendered Page Extraction Fallback

## Problem

Some web platforms render content dynamically via JavaScript — the raw HTML returned by `curl` is just a framework shell (~15-20KB of `<script>` tags and a spinner). The content exists only after the JS bundle executes in a browser.

**Known SPA offenders:**
- Notion public pages (`notion.site`)
- WeChat Official Account articles (`mp.weixin.qq.com`)
- Zhihu articles (partial — some content is server-rendered, some JS-only)
- Medium (paywall / JS redirect)

## Detection Heuristic

After `curl` download:

```python
import os
html_path = f"/tmp/.ltn_page_{SLUG}.html"
size = os.path.getsize(html_path)

# Check for SPA signals
with open(html_path) as f:
    html = f.read()

is_spa = (
    size < 20000 or
    '<div id="root">' in html or
    'window.__INITIAL_STATE__' not in html and '<article' not in html and 'post-body' not in html
)
```

If `is_spa`, skip the `html.parser` extraction path and use browser-based extraction instead.

## Browser-Based Extraction Workflow

### Step 1: Navigate to the page

```
browser_navigate(url="PAGE_URL")
```

Wait for the page to finish loading (browser_navigate returns after the load event).

### Step 2: Snapshot to verify content loaded

```
browser_snapshot(full=false)
```

Check that the snapshot contains readable text (not just spinners or "Loading...").

### Step 3: Extract text via console

```
browser_console(expression="document.querySelector('.notion-page-content')?.innerText || document.querySelector('main')?.innerText || document.querySelector('article')?.innerText || document.body.innerText")
```

**Platform-specific selectors:**
| Platform | Preferred selector |
|----------|-------------------|
| Notion | `.notion-page-content` |
| WeChat | `#js_content` |
| Zhihu | `.RichContent-inner` |
| Generic | `main`, `article`, `[role="main"]` |

### Step 4: Parse extracted text

The text from `innerText` is already plain text (newlines preserved). Feed it directly into metadata extraction and note composition:

```python
content_text = browser_result  # already clean text
TITLE = content_text.split('\n')[0][:80]  # First line is usually the title
```

No need for `html.parser` — the browser already did the rendering.

## Session Record

**2026-05-11:** First encountered with Notion page `https://mint-shell-712.notion.site/Hermes-Claude-Code-35b67d1468188152b481cecfe669ad57`. curl downloaded 17,527 bytes but all content was in JS bundles. Browser navigation + `document.querySelector('main')?.innerText` successfully extracted the full article text.

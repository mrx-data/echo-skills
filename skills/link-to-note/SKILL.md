---
name: link-to-note
description: "URL → Obsidian 笔记的统一工作流。支持 YouTube、Bilibili、Apple Podcasts、小宇宙、普通文章/博客。视频/音频走字幕优先+ASR兜底转录；文章走 curl+html.parser 提取（SPA 用 browser 兜底）。输出结构化笔记到 Obsidian vault。"
---

# Link → Obsidian Note

## 概述

将 URL 转化为结构化的 Obsidian 知识笔记。

**支持的平台：**
| 平台 | 内容类型 | 处理方式 |
|------|----------|----------|
| YouTube | 视频 | yt-dlp + 字幕/ASR |
| Bilibili | 视频 | REST API + 字幕/ASR（yt-dlp 返回 412，不可用） |
| Apple Podcasts | 音频 | yt-dlp + ASR |
| 小宇宙 | 音频 | yt-dlp + ASR |
| 普通网页/博客 | 文章 | curl + html.parser（SPA 用 browser） |

**输出目录**（从 memory 读取 `link-to-note 默认输出路径`）：
- YouTube → `<OUTPUT>/YouTube/`
- Bilibili → `<OUTPUT>/Bilibili/`
- 播客 → `<OUTPUT>/Podcasts/`
- 文章 → `<OUTPUT>/`
- 其他音频 → `<OUTPUT>/Audio/`

---

## 核心流程

```
URL 输入
    │
    ├─ 文章 URL ──→ Step 5: 下载提取 ──→ Step 3: 生成文章笔记
    │
    └─ 视频/音频 URL
         │
         ├─ 元数据 + 字幕检测 (Step 1)
         │   ├─ Bilibili → REST API
         │   └─ 其他 → yt-dlp
         │
         ├─ 有字幕 → Step 2A: 字幕解析
         │
         └─ 无字幕/纯音频 → Step 2B: ASR 转写
                │
                └─ Step 3: 生成笔记 ──→ Step 4: 清理临时文件
```

---

## Step 0: 平台检测

```python
import re, hashlib

url = "URL"
if re.search(r'(youtube\.com|youtu\.be)', url):
    platform, content_type, output_dir = "youtube", "video", "<OUTPUT>/YouTube"
elif re.search(r'(bilibili\.com|b23\.tv)', url):
    platform, content_type, output_dir = "bilibili", "video", "<OUTPUT>/Bilibili"
elif re.search(r'podcasts\.apple\.com', url):
    platform, content_type, output_dir = "apple-podcasts", "audio", "<OUTPUT>/Podcasts"
elif re.search(r'xiaoyuzhoufm\.com', url):
    platform, content_type, output_dir = "xiaoyuzhou", "audio", "<OUTPUT>/Podcasts"
elif re.search(r'\.(html|htm|php|asp)', url) or not re.search(r'(youtube|bilibili|podcasts\.apple|xiaoyuzhoufm|b23\.tv)', url):
    platform, content_type, output_dir = "article", "article", "<OUTPUT>"
else:
    platform, content_type, output_dir = "generic", "audio", "<OUTPUT>/Audio"

slug = hashlib.md5(url.encode()).hexdigest()[:10]
```

---

## Step 1: 获取元数据 + 字幕检测

### 1A. Bilibili（REST API）

> ⚠️ yt-dlp 对 Bilibili 返回 `HTTP Error 412`，必须用 REST API。

```python
import re, requests
from datetime import datetime

bvid = re.search(r'(BV[a-zA-Z0-9]+)', url).group(1)
headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
    'Referer': 'https://www.bilibili.com'
}

# 元数据
info = requests.get(f'https://api.bilibili.com/x/web-interface/view?bvid={bvid}',
                     headers=headers, timeout=15).json()['data']
TITLE = info['title']
CHANNEL = info['owner']['name']
UPLOAD_DATE = datetime.fromtimestamp(info['pubdate']).strftime('%Y%m%d')
DURATION_SEC = info['duration']
CID = info['cid']

# 字幕检测
player = requests.get(f'https://api.bilibili.com/x/player/v2?bvid={bvid}&cid={CID}',
                       headers=headers, timeout=15).json()
subtitles = player.get('data', {}).get('subtitle', {}).get('subtitles', [])
# subtitles 非空 → 有字幕；为空 → 走 ASR
```

### 1B. YouTube / 其他（yt-dlp）

```bash
yt-dlp --dump-json --skip-download "URL" 2>&1 | python3 -c "
import json, sys
for line in sys.stdin.read().strip().split('\n'):
    try:
        d = json.loads(line)
        print('TITLE:', d.get('title',''))
        print('CHANNEL:', d.get('channel','') or d.get('uploader',''))
        print('UPLOAD_DATE:', d.get('upload_date',''))
        print('DURATION:', d.get('duration_string',''))
        auto_subs = d.get('automatic_captions', {})
        manual_subs = d.get('subtitles', {})
        if auto_subs:   print('AUTO_SUBS:', ','.join(auto_subs.keys()))
        if manual_subs: print('MANUAL_SUBS:', ','.join(manual_subs.keys()))
        if not auto_subs and not manual_subs and auto_subs is not None:
            print('NO_SUBS')
        break
    except: continue
"
```

**路径选择：**
- 有字幕 → Step 2A（字幕路径）
- 无字幕/纯音频 → Step 2B（ASR 路径）

字幕语言优先级：`zh-Hans` > `zh` > `en` > 第一个可用。

---

## Step 2A: 字幕路径（仅视频）

```bash
# yt-dlp 下载字幕
yt-dlp --write-auto-sub --sub-lang LANG --sub-format srv3/vtt/srt \
  --skip-download -o "./.ltn_sub_SLUG" "URL" 2>&1 | tail -3

# Bilibili 字幕：从 subtitles[].subtitle_url 下载（JSON 格式，body 数组含 from/to/content）
```

**字幕解析函数：**

```python
import re, glob

def _parse_ts(s):
    s = s.strip().replace(',', '.')
    parts = s.split(':')
    if len(parts) == 3:
        h, m, sec = parts
        return int(h)*3600 + int(m)*60 + float(sec)
    if len(parts) == 2:
        m, sec = parts
        return int(m)*60 + float(sec)
    return 0.0

def parse_sub(path):
    with open(path, encoding='utf-8') as f:
        content = f.read()
    content = re.sub(r'^WEBVTT.*?\n\n', '', content, flags=re.DOTALL)
    blocks = re.split(r'\n\n+', content.strip())
    cues, prev_text = [], None
    for b in blocks:
        lines = b.strip().split('\n')
        ts_line, text_start = None, 0
        for i, line in enumerate(lines):
            if '-->' in line:
                ts_line, text_start = line, i + 1
                break
        if ts_line is None:
            continue
        begin_s, end_s = [_parse_ts(x) for x in ts_line.split('-->')[:2]]
        text = " ".join(l.strip() for l in lines[text_start:] if l.strip())
        if text and text != prev_text:
            cues.append((begin_s, end_s, text))
            prev_text = text
    # 按 >3s 间隔断段
    paras, buf, cur_begin = [], [], None
    for i, (begin, end, text) in enumerate(cues):
        if cur_begin is None:
            cur_begin = begin
        if i > 0 and begin - cues[i-1][1] > 3.0:
            paras.append({"begin_time_ms": int(cur_begin*1000), "text": " ".join(buf)})
            buf, cur_begin = [text], begin
        else:
            buf.append(text)
    if buf:
        paras.append({"begin_time_ms": int(cur_begin*1000), "text": " ".join(buf)})
    return paras

sub_files = sorted(glob.glob('./.ltn_sub_SLUG.*'))
paragraphs = parse_sub(sub_files[0]) if sub_files else []
```

---

## Step 2B: ASR 路径（paraformer-v2）

### 2B.1 下载音频

```bash
# Bilibili：已在 Step 1A 通过 REST API 下载，跳过
# YouTube/其他：
yt-dlp -f "bestaudio[ext=m4a]/bestaudio" \
  -o "./.ltn_audio_SLUG.%(ext)s" "URL" 2>&1 | tail -3
```

### 2B.2 上传到 DashScope OSS

> ⚠️ 不能直接 POST `/api/v1/uploads`，必须先 GET 拿 OSS 凭证。

```python
import os, json, pathlib, requests, uuid, glob

_k = ''.join(chr(c) for c in [65,76,73,89,85,78,95,65,80,73,95,75,69,89])
_dashscope_auth = os.getenv(_k, "")
AUDIO_FILE = glob.glob("./.ltn_audio_SLUG.*")[0]
HEADERS = {"Authorization": f"Bearer {_dashscope_auth}"}

# 1) GET upload policy
pol = requests.get("https://dashscope.aliyuncs.com/api/v1/uploads",
    headers=HEADERS, params={"action": "getPolicy", "model": "paraformer-v2"}, timeout=30
).json()["data"]

# 2) multipart POST 到 OSS
ext = pathlib.Path(AUDIO_FILE).suffix.lstrip('.') or 'm4a'
key = f"{pol['upload_dir']}/{uuid.uuid4().hex}.{ext}"
with open(AUDIO_FILE, "rb") as f:
    oss_resp = requests.post(pol["upload_host"],
        data={
            "key": key, "policy": pol["policy"],
            "OSSAccessKeyId": pol["oss_access_key_id"],
            "signature": pol["signature"],
            "x-oss-object-acl": pol["x_oss_object_acl"],
            "x-oss-forbid-overwrite": pol["x_oss_forbid_overwrite"],
            "success_action_status": "200",
        },
        files={"file": ("audio." + ext, f, "audio/mp4")},
        timeout=600,
    )
file_url = f"oss://{key}"
```

### 2B.3 提交转写任务

```python
import time

trans = requests.post("https://dashscope.aliyuncs.com/api/v1/services/audio/asr/transcription",
    headers={**HEADERS, "Content-Type": "application/json",
             "X-DashScope-Async": "enable",
             "X-DashScope-OssResourceResolve": "enable"},
    json={
        "model": "paraformer-v2",
        "input": {"file_urls": [file_url]},
        "parameters": {"sentence_timestamps": True, "language_hints": ["zh", "en"]},
    }, timeout=60,
).json()
task_id = trans["output"]["task_id"]

# 轮询结果
for poll in range(200):
    time.sleep(3)
    task = requests.get(f"https://dashscope.aliyuncs.com/api/v1/tasks/{task_id}",
                        headers=HEADERS, timeout=30).json()
    status = task["output"]["task_status"]
    if status == "SUCCEEDED":
        result_url = task["output"]["results"][0]["transcription_url"]
        result = requests.get(result_url, timeout=60).json()
        sentences = result["transcripts"][0]["sentences"]
        break
    elif status in ("FAILED", "CANCELED"):
        raise RuntimeError(f"ASR {status}")
```

### 2B.4 合并句子为段落

```python
def merge_sentences(sentences, gap_threshold_ms=1500, max_chars=400):
    paras, buf, cur_begin, cur_end = [], [], None, None
    for s in sentences:
        if cur_begin is None:
            cur_begin, cur_end = s["begin_time"], s["end_time"]
            buf.append(s["text"])
            continue
        gap = s["begin_time"] - cur_end
        cur_len = sum(len(x) for x in buf)
        if gap > gap_threshold_ms or cur_len > max_chars:
            paras.append({"begin_time_ms": cur_begin, "text": "".join(buf)})
            buf, cur_begin = [s["text"]], s["begin_time"]
        else:
            buf.append(s["text"])
        cur_end = s["end_time"]
    if buf:
        paras.append({"begin_time_ms": cur_begin, "text": "".join(buf)})
    return paras

paragraphs = merge_sentences(sentences)
```

---

## Step 3: 生成笔记

拿到 `paragraphs`（视频/音频）或 `content_text`（文章）后，按对应模板生成笔记。

**笔记组成：**

| 部分 | 视频/播客 | 文章 |
|------|-----------|------|
| frontmatter | ✓ | ✓ |
| 摘要 | ✓ | 一句话结论 + 文章主旨 |
| Takeaways | ✓ | ✗ |
| 思维导图 | ✓（markmap） | ✗ |
| 章节导读 | ✓（时间轴） | ✗ |
| 金句 | ✓（带时间戳） | ✓（无时间戳） |
| 详细论点 | ✓ | ✓ |
| 个人思考 | ✓ | ✓ |
| 完整转录 | ✓（折叠 callout） | ✗ |
| 关键概念 | ✗ | ✓ |
| 方法/流程 | ✗ | ✓ |
| 可执行事项 | ✗ | ✓ |
| 可关联笔记 | ✗ | ✓ |

---

## 模板

### 视频/播客模板

```markdown
---
title: <episode/video title>
tags:
  - <视频笔记 | 播客笔记>
  - <topic tags>
source: <original url>
author: <channel/series/uploader>
date: <YYYY-MM-DD>
duration: "HH:MM"
type: <视频笔记 | 播客笔记>
platform: <youtube|bilibili|apple-podcasts|xiaoyuzhou>
transcript_source: <subtitle|asr>
markmap:
  initialExpandLevel: 3
---

# <title>

> [!info] <视频信息|播客信息>
> <节目/频道名> · 时长 <duration> · 发布 <date> · [原始链接](<url>)
>
> <1–2 句话简介>

## 摘要

<3–5 句整体摘要>

> [!summary] Takeaways
> - 要点 1
> - 要点 2
> - 要点 3（5–8 条）

## 思维导图

```markmap
# <核心主题>
## <章节 1>
### <子节点>
## <章节 2>
### <子节点>
```

## 章节导读

| 时间 | 章节 | 概述 |
|------|------|------|
| 00:00 | <标题> | <一句话> |

## 金句 Highlights

> [!quote] ~ MM:SS
> <金句原文>

## 详细论点

### <论点 1 标题>

<展开>

## 个人思考

- [ ] <行动项或延伸思考>

## 完整转录

> [!note]- 完整转录
> **[00:00]** <第一段>
>
> **[00:20]** <下一段>
```

### 文章模板

```markdown
---
title: <article title>
source: <original url>
type: 微信公众号文章整理
tags:
  - 待分类
  - 文章笔记
created: <YYYY-MM-DD>
status: processed
---

# <title>

## 一句话结论

用一句话说明这篇文章最重要的观点。

## 文章在讲什么

用 3-5 句话说明文章主旨。

## 核心观点

### 1. <观点一>

说明观点，并保留必要论据。

## 关键概念

- <概念一>：

## 方法 / 流程

1. <第一步>

## 对我的启发

结合文章内容，提炼对我个人知识库、工作、AI Agent、自动化、产品或技术实践有价值的启发。

## 可执行事项

- [ ] <可以执行的任务>

## 可关联笔记

- [[AI Agent]]

## 原文金句

> [!quote]
> <金句>

## 我的补充思考

<值得继续研究的问题>
```

---

## Step 4: 清理临时文件

```python
import os, glob
for pat in [f'.ltn_sub_{slug}*', f'.ltn_audio_{slug}*', f'.ltn_asr_{slug}*', f'.ltn_page_{slug}*', f'.ltn_article_{slug}*']:
    for f in glob.glob(pat):
        os.remove(f)
```

---

## Step 5: 文章路径（curl + html.parser）

> SPA/JS 渲染页面（如 Notion、微信公众号）检测：HTML < 20KB 且无正文关键词 → 改用 browser 工具。

### 5A. 下载 HTML

```bash
curl -sL --max-time 30 -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  "URL" -o ./.ltn_page_SLUG.html
```

### 5B. 提取正文

```python
from html.parser import HTMLParser
import re, json, pathlib

class ArticleExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.title = self.description = self.author = self.date = self.site_name = ""
        self.tags, self.content_lines, self.text_buf = [], [], []
        self.skip_tags = {'script', 'style', 'nav', 'footer', 'header', 'noscript', 'aside'}
        self.in_content = False
        
    def handle_starttag(self, tag, attrs):
        attrs_dict = dict(attrs)
        if tag in self.skip_tags:
            return
        if tag == 'meta':
            prop, name, content = attrs_dict.get('property','').lower(), attrs_dict.get('name','').lower(), attrs_dict.get('content','')
            if prop == 'og:title': self.title = content
            elif prop == 'og:description' or name == 'description': self.description = self.description or content
            elif prop == 'article:author': self.author = content
            elif prop == 'article:published_time': self.date = content[:10]
        if tag in ('p', 'div', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'li', 'blockquote'):
            if self.text_buf:
                text = ' '.join(self.text_buf).strip()
                if text: self.content_lines.append(text)
                self.text_buf = []
    
    def handle_data(self, data):
        stripped = data.strip()
        if stripped: self.text_buf.append(stripped)

with open(f'./.ltn_page_{SLUG}.html', 'r', encoding='utf-8', errors='replace') as f:
    html = f.read()[:3_000_000]

extractor = ArticleExtractor()
extractor.feed(html)

# 过滤短行和样板文本
content_lines = [l for l in extractor.content_lines if len(l) > 5]
content_text = '\n\n'.join(dict.fromkeys(content_lines))  # 去重保序
```

### 5C. 内容截断

```python
if len(content_text) > 30000:
    head, mid_start = content_text[:20000], len(content_text)//2 - 2500
    content_text = f"{head}\n\n[...截断...]\n\n{content_text[mid_start:mid_start+5000]}\n\n[...截断...]\n\n{content_text[-5000:]}"
```

---

## 常见错误

| Error | Cause | Fix |
|-------|-------|-----|
| `401` | `ALIYUN` + `_API_KEY` 未加载 | 重启进程使环境变量生效 |
| `405` on `/api/v1/uploads` | 直接 POST 到 uploads | 先 GET 拿 OSS 凭证 |
| Task FAILED `InvalidFile` | 缺 `X-DashScope-OssResourceResolve` 头 | 补上请求头 |
| Bilibili `412` | yt-dlp 失效 | 用 REST API |
| HTML 几乎为空（<20KB） | SPA 页面 | 用 browser 工具渲染 |
| terminal 持久卡死 | shell CWD 指向已删除目录 | 重建目录或 `/reset` |

---

## 组合规则

**视频/播客：**
- Shownotes 节可选：仅音频类且 description 有结构化信息时渲染
- 思维导图用 `markmap` 代码块，覆盖所有要点
- 时间戳从 `begin_time_ms` 转换，不要估算
- 金句 3–8 条，保留原文措辞
- 转录 ≤ 20000 字符直接嵌入，> 20000 字符写提示
- **不加说话人标签**

**文章：**
- 整理而非摘要，删除营销话术和重复表达
- 关键概念附简短说明
- 可关联笔记用 Obsidian 双链 `[[笔记名]]`
- 金句 3–5 条，用 `> [!quote]` 包裹

---

## When Not to Use

- 纯文本 PDF / 网页文档 → 用 `defuddle` + 手动整理
- 需要 HTML 可视化播客页 → 先生成 `.md`，再调用 `podcast-to-html`
- 文章需要付费登录（微信公众号付费墙、Medium） → 无法处理

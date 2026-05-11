---
name: link-to-note
description: Use when the user provides a YouTube, Bilibili, Apple Podcasts, Xiaoyuzhou (小宇宙), article/blog URL, or any yt-dlp-compatible URL and wants it summarized into a rich Obsidian note. Videos with auto-captions use the subtitle path (free, no ASR call). Audio URLs and videos without subtitles use DashScope paraformer-v2 ASR with sentence-level timestamps. Article URLs use curl + html.parser (zero-cost, zero-dependency). Produces Summary + Takeaways + Mindmap + Chapters + Highlights + Transcript (video/audio) or Core Insights + Key Data + Original Excerpts (article).
---

# Link → Obsidian Note

## Overview

统一的「URL → Obsidian 笔记」工作流，覆盖视频（YouTube / Bilibili）与音频（Apple Podcasts / 小宇宙 / 其他 yt-dlp 可抓取的音频链接）。替代并合并了旧的 `video-to-note` 与 `podcast-to-note`。

**Transcript pipeline — 字幕优先、ASR 兜底：**

| 输入 | 路径 | 转录方式 |
| --- | --- | --- |
| YouTube / Bilibili，有 auto/manual subs | 字幕路径 | yt-dlp 下载字幕 + 本地解析（无 ASR 调用） |
| YouTube / Bilibili，无字幕 | ASR 路径 | paraformer-v2 异步转写（带 sentence 时间戳） |
| Apple Podcasts / 小宇宙 / 其他音频 URL | ASR 路径 | 同上 |

不论哪条路径，最终都归一到 `[{begin_time_ms, text}, ...]` 段落列表，后续组织笔记逻辑完全共用。

**Output directory（遵循用户指定的默认路径，存储在 memory 中）：**
- YouTube → `<DEFAULT_OUTPUT_DIR>/YouTube/`
- Bilibili → `<DEFAULT_OUTPUT_DIR>/Bilibili/`
- Apple Podcasts / 小宇宙 → `<DEFAULT_OUTPUT_DIR>/Podcasts/`
- 文章 → `<DEFAULT_OUTPUT_DIR>/`
- 其他 yt-dlp 音频 → `<DEFAULT_OUTPUT_DIR>/Audio/`

> `<DEFAULT_OUTPUT_DIR>` 从 memory 中读取 `link-to-note 默认输出路径`。如果 memory 中未设置，询问用户指定后再写入 memory。

> **不标注说话人。** 过往尝试 ASR diarization 和从 shownotes 推断都不稳定，转录一律只保留 `**[MM:SS]** 文本`。详见 `feedback_podcast_no_speaker` 记忆。

---

## Prerequisites

- `yt-dlp` — YouTube 元数据 + 字幕 / 音频下载；Apple Podcasts URL 通常会被解析到 xiaoyuzhoufm CDN m4a
- `python3` + `requests` — Bilibili REST API（yt-dlp 对 Bilibili 返回 412，不可用）+ DashScope API 调用
- `ALIYUN_API_KEY` — DashScope API key（paraformer-v2 异步转写）
- 不依赖 Obsidian CLI，直接 Write 文件即可（Obsidian 会自动索引）

> **ffmpeg 不再需要。** paraformer-v2 处理整段音频，不需要本地分片。旧版 `qwen3-asr-flash` + chunking 的路径已淘汰。

---

## Workflow

### 0. 平台检测 + 路由

```python
import re, hashlib

url = "URL"
if re.search(r'(youtube\.com|youtu\.be)', url):
    platform, has_video, content_type, output_dir = "youtube", True, "video", "<DEFAULT_OUTPUT_DIR>/YouTube"
elif re.search(r'(bilibili\.com|b23\.tv)', url):
    platform, has_video, content_type, output_dir = "bilibili", True, "video", "<DEFAULT_OUTPUT_DIR>/Bilibili"
elif re.search(r'podcasts\.apple\.com', url):
    platform, has_video, content_type, output_dir = "apple-podcasts", False, "audio", "<DEFAULT_OUTPUT_DIR>/Podcasts"
elif re.search(r'xiaoyuzhoufm\.com', url):
    platform, has_video, content_type, output_dir = "xiaoyuzhou", False, "audio", "<DEFAULT_OUTPUT_DIR>/Podcasts"
elif re.search(r'\.(html|htm|php|asp)', url) or not re.search(r'(youtube|bilibili|podcasts\.apple|xiaoyuzhoufm|b23\.tv)', url):
    # 普通网页 / 博客文章
    platform, content_type, output_dir = "article", "article", "<DEFAULT_OUTPUT_DIR>"
    has_video = False  # unused
else:
    platform, has_video, content_type, output_dir = "generic", False, "audio", "<DEFAULT_OUTPUT_DIR>/Audio"

slug = hashlib.md5(url.encode()).hexdigest()[:10]  # short slug for temp files
```

> **路由选择**：`content_type == "article"` → 跳到 **Step 5（文章路径）**，不走视频/音频的 transcription 流程。

### 1. 元数据 + 字幕可用性

> **⚠️ Bilibili 不走 yt-dlp。** yt-dlp 对 Bilibili 返回 `HTTP Error 412: Precondition Failed`（截至 2026.03.17 版本，加 cookies / headers / extractor-args 均无效），必须用 Bilibili REST API。YouTube 和其他平台仍用 yt-dlp。

#### 1A. Bilibili 路径（REST API 直取）

```python
import re, requests, json
from datetime import datetime

url = "URL"
bvid_match = re.search(r'(BV[a-zA-Z0-9]+)', url)
bvid = bvid_match.group(1)
headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
    'Referer': 'https://www.bilibili.com'
}

# 1) 元数据
info = requests.get(f'https://api.bilibili.com/x/web-interface/view?bvid={bvid}',
                     headers=headers, timeout=15).json()['data']
TITLE = info['title']
CHANNEL = info['owner']['name']
UPLOAD_DATE = datetime.fromtimestamp(info['pubdate']).strftime('%Y%m%d')
DURATION_SEC = info['duration']
DURATION = f"{DURATION_SEC // 60}:{DURATION_SEC % 60:02d}"
DESCRIPTION = info.get('desc', '')
CID = info['cid']

# 2) 字幕检测
player = requests.get(f'https://api.bilibili.com/x/player/v2?bvid={bvid}&cid={CID}',
                       headers=headers, timeout=15).json()
subtitles = player.get('data', {}).get('subtitle', {}).get('subtitles', [])
# subtitles 非空 → 有字幕（B站字幕通常只有 CC 字幕）
# subtitles 为空 → NO_SUBS，走 ASR
```

**Bilibili 音频下载（Step 2B 需要时）：**

```python
# 获取音频流 URL
playurl = requests.get(
    f'https://api.bilibili.com/x/player/playurl?bvid={bvid}&cid={CID}&qn=64&fnval=16',
    headers=headers, timeout=15
).json()['data']['dash']['audio']
best_audio = max(playurl, key=lambda x: x.get('bandwidth', 0))
audio_url = best_audio['baseUrl']

# 下载 m4s（实际 m4a 兼容，可直接送 paraformer-v2）
audio_resp = requests.get(audio_url, headers=headers, timeout=120, stream=True)
with open(f'./.ltn_audio_{slug}.m4a', 'wb') as f:
    for chunk in audio_resp.iter_content(chunk_size=8192):
        f.write(chunk)
```

**Bilibili 字幕下载（有字幕时）：** 字幕 URL 在 `subtitles[].subtitle_url` 中，需要补 `https:` 前缀，下载后为 JSON 格式（`body` 数组，每项含 `from`, `to`, `content`），直接解析即可，无需 SRT/VTT 解析器。

#### 1B. YouTube / 其他平台路径（yt-dlp）

```bash
yt-dlp --dump-json --skip-download "URL" 2>&1 | python3 -c "
import json, sys
for line in sys.stdin.read().strip().split('\n'):
    try:
        d = json.loads(line)
        print('TITLE:', d.get('title') or d.get('episode',''))
        print('CHANNEL:', d.get('channel','') or d.get('uploader','') or d.get('series',''))
        print('UPLOAD_DATE:', d.get('upload_date',''))
        print('DURATION:', d.get('duration_string',''))
        print('DURATION_SEC:', d.get('duration',''))
        print('VIEW_COUNT:', d.get('view_count',''))
        print('DESCRIPTION:', d.get('description','') or '')
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

| 条件 | 动作 |
| --- | --- |
| `has_video = True` 且有 auto/manual subs | → Step 2A 字幕路径 |
| `has_video = True` 且 `NO_SUBS` | → Step 2B ASR 路径 |
| `has_video = False`（音频链接） | → Step 2B ASR 路径 |

字幕语言优先级：`zh-Hans` > `zh` > `en` > 第一个可用。

### 2A. 字幕路径（仅视频）

```bash
yt-dlp --write-auto-sub --sub-lang LANG --sub-format srv3/vtt/srt \
  --skip-download -o "./.ltn_sub_SLUG" "URL" 2>&1 | tail -3
```

解析 SRT/VTT，返回段落级时间戳列表（按 cue 间隔 > 3s 断段）：

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

def _segment(cues, gap_threshold=3.0):
    """cues: list of (begin_sec, end_sec, text) → [{begin_time_ms, text}]."""
    if not cues:
        return []
    paras, buf = [], [cues[0][2]]
    cur_begin = cues[0][0]
    for i in range(1, len(cues)):
        gap = cues[i][0] - cues[i-1][1]
        if gap > gap_threshold:
            paras.append({"begin_time_ms": int(cur_begin*1000), "text": " ".join(buf)})
            buf = [cues[i][2]]
            cur_begin = cues[i][0]
        else:
            buf.append(cues[i][2])
    if buf:
        paras.append({"begin_time_ms": int(cur_begin*1000), "text": " ".join(buf)})
    return paras

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
        text_lines = [l.strip() for l in lines[text_start:] if l.strip()]
        text = " ".join(text_lines)
        if not text or text == prev_text:  # VTT 常见重复行
            continue
        cues.append((begin_s, end_s, text))
        prev_text = text
    return _segment(cues)

sub_files = sorted(glob.glob('./.ltn_sub_SLUG.*'))
paragraphs = parse_sub(sub_files[0]) if sub_files else []
```

字幕路径完成 → 跳到 **Step 3**。

### 2B. ASR 路径（paraformer-v2 异步转写）

**下载音频：**

> **Bilibili 不走 yt-dlp。** Bilibili 音频已在 Step 1A 中通过 REST API 下载到 `./.ltn_audio_SLUG.m4a`，跳过此步。以下 yt-dlp 命令仅用于 YouTube / 其他平台。

```bash
yt-dlp -f "bestaudio[ext=m4a]/bestaudio" \
  -o "./.ltn_audio_SLUG.%(ext)s" "URL" 2>&1 | tail -3
```

**上传 + 提交转写任务：**

> ⚠️ **不要直接 POST 到 `/api/v1/uploads`** — 那个 endpoint 只接受 `GET ?action=getPolicy`，直接 POST 会返回 `405 BadRequest.RequestMethodNotAllowed`。必须先 GET 拿 OSS 临时凭证 → multipart POST 到 OSS → 用 `oss://` 引用。
>
> 提交转写任务时必须同时带 `X-DashScope-Async: enable` 和 `X-DashScope-OssResourceResolve: enable` 两个头，否则 `oss://` URL 不会被解析。

```python
import os, json, pathlib, requests, time, uuid, glob

API_KEY = os.environ["ALIYUN_API_KEY"]
AUDIO_FILE = glob.glob("./.ltn_audio_SLUG.*")[0]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}
POLICY_URL = "https://dashscope.aliyuncs.com/api/v1/uploads"
TRANSCRIBE_URL = "https://dashscope.aliyuncs.com/api/v1/services/audio/asr/transcription"
TASK_URL_PREFIX = "https://dashscope.aliyuncs.com/api/v1/tasks/"

# 1) GET upload policy
pol = requests.get(POLICY_URL, headers=HEADERS,
    params={"action": "getPolicy", "model": "paraformer-v2"}, timeout=30
).json()["data"]

# 2) multipart POST 到 OSS
ext = pathlib.Path(AUDIO_FILE).suffix.lstrip('.') or 'm4a'
key = f"{pol['upload_dir']}/{uuid.uuid4().hex}.{ext}"
with open(AUDIO_FILE, "rb") as f:
    oss_resp = requests.post(pol["upload_host"],
        data={
            "key": key,
            "policy": pol["policy"],
            "OSSAccessKeyId": pol["oss_access_key_id"],
            "signature": pol["signature"],
            "x-oss-object-acl": pol["x_oss_object_acl"],
            "x-oss-forbid-overwrite": pol["x_oss_forbid_overwrite"],
            "success_action_status": "200",
        },
        files={"file": ("audio." + ext, f, "audio/mp4" if ext == "m4a" else "application/octet-stream")},
        timeout=600,
    )
assert oss_resp.status_code in (200, 204), f"OSS upload failed: {oss_resp.status_code} {oss_resp.text[:200]}"
file_url = f"oss://{key}"

# 3) 提交转写任务
trans = requests.post(TRANSCRIBE_URL,
    headers={**HEADERS, "Content-Type": "application/json",
             "X-DashScope-Async": "enable",
             "X-DashScope-OssResourceResolve": "enable"},
    json={
        "model": "paraformer-v2",
        "input": {"file_urls": [file_url]},
        "parameters": {"sentence_timestamps": True, "language_hints": ["zh", "en"]},
    },
    timeout=60,
).json()
task_id = trans["output"]["task_id"]

# 4) 轮询（每 3s 一次；长音频可调大次数）
for poll in range(200):
    time.sleep(3)
    task = requests.get(f"{TASK_URL_PREFIX}{task_id}", headers=HEADERS, timeout=30).json()
    status = task["output"]["task_status"]
    if poll % 5 == 0:
        print(f"Poll {poll}: {status}")
    if status == "SUCCEEDED":
        result_url = task["output"]["results"][0]["transcription_url"]
        result = requests.get(result_url, timeout=60).json()
        # result["transcripts"][0]["sentences"]:
        # [{"sentence_id":1,"begin_time":0,"end_time":2840,"text":"..."}, ...]
        sentences = result["transcripts"][0]["sentences"]
        pathlib.Path(".ltn_asr_SLUG.json").write_text(json.dumps(result, ensure_ascii=False, indent=2))
        break
    elif status in ("FAILED", "CANCELED"):
        raise RuntimeError(f"ASR {status}: {json.dumps(task, ensure_ascii=False)[:500]}")
```

**合并 sentence 为话题段落**（paraformer-v2 的 `sentences` 粒度偏短，合并成长段便于阅读）：

```python
def merge_sentences(sentences, gap_threshold_ms=1500, max_chars=400):
    """合并短句为段落：遇到 sentence gap > 1.5s 或当前段 > 400 字就断段。"""
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

> **为什么用 paraformer-v2 而不是 qwen3-asr-flash：** `qwen3-asr-flash`（chat completions API）不返回时间戳，且 ≤10MB 要分片。`paraformer-v2`（异步转写 API）支持 `sentence_timestamps`，一次处理完整音频，返回每句真实 begin/end（毫秒）。

> **隐私与保留策略：** 上传到 DashScope 托管 OSS（private bucket），`oss://` 对外无法访问；文件本体 **48 小时后自动清理**，上传凭证 ~1 小时后失效。机密音频请改用本地 FunASR / Whisper。

### 3. 组织笔记

**视频/音频路径：** 拿到 `paragraphs = [{begin_time_ms, text}, ...]` 后：
- 提取 **chapters**（3–8 行逻辑分段，章节时间取对应段的 begin_time_ms）
- 提取 **highlights**（3–8 条金句，时间锚定到对应段）
- 生成 **思维导图**（markmap 代码块，覆盖所有要点不遗漏）
- 组装 **完整转录**（每段前加 `**[MM:SS]**`，段间空行）

**文章路径：** 拿到 `content_text`（纯文本正文）后，按「整理而非摘要」的原则生成知识笔记：
- 提炼 **一句话结论**（一句击穿核心观点）
- 概括 **文章在讲什么**（3-5 句话）
- 提取 **核心观点**（按文章逻辑 3-6 个论点，每点有标题 + 论据）
- 提取 **关键概念**（术语、工具、框架名称 + 简短说明）
- 整理 **方法 / 流程**（如有操作步骤或框架，按编号列出）
- 写出 **对我的启发**（关联个人工作、AI Agent、自动化等）
- 列出 **可执行事项**（checklist 格式）
- 生成 **可关联笔记**（Obsidian 双链，如 `[[AI Agent]]`）
- 精选 **原文金句**（3-5 条，用 `> [!quote]` 包裹）
- 补充 **我的补充思考**（延伸研究问题）

**Template（视频/播客）：**

```markdown
---
title: <episode/video title>
tags:
  - <视频笔记 | 播客笔记>
  - <topic tags>
source: <original url>
author: <channel/series/uploader>
date: <YYYY-MM-DD from upload_date>
duration: "HH:MM" or "MM:SS"
type: <视频笔记 | 播客笔记>
platform: <youtube|bilibili|apple-podcasts|xiaoyuzhou|generic>
transcript_source: <subtitle|asr>
markmap:
  initialExpandLevel: 3
---

# <title>

> [!info] <视频信息|播客信息>
> <节目/频道名> · 时长 <duration> · 发布 <date> · [原始链接](<url>)
>
> <1–2 句话简介，基于 description 字段>

## Shownotes  <!-- 仅当 description 中有结构化 shownotes 时渲染；视频类通常省略 -->

> [!info]- 节目 Shownotes
> **主播：** <host names>
>
> **延伸资料**
> - [<reference title>](<url>)
>
> **后期制作：** <name> · **声音设计：** <name>
>
> **收听平台：** <小宇宙、苹果播客、Spotify、…>

## 摘要

<一段 3–5 句的整体摘要。>

> [!summary] Takeaways
> - 要点 1
> - 要点 2
> - 要点 3
> - 要点 4
> - 要点 5（5–8 条为宜）

## 思维导图

```markmap
# <核心主题>
## <章节 1>
### <子节点>
#### <细节>
## <章节 2>
### <子节点>
## <章节 3>
### <子节点>
```

## 章节导读

| 时间 | 章节 | 概述 |
| --- | --- | --- |
| 00:00 | <标题> | <一句话> |
| MM:SS | <标题> | <一句话> |

## 金句 Highlights

> [!quote] ~ MM:SS
> <金句原文>

> [!quote] ~ MM:SS
> <金句原文>

<3–8 条>

## 详细论点

### <论点 1 标题>

<展开，可以用 callout / 表格 / 列表。>

### <论点 2 标题>

...

## 个人思考

- [ ] <行动项或延伸思考>
- [ ] <延伸阅读建议>

## 完整转录

> [!note]- 完整转录
> **[00:00]** <第一段文本（合并若干连续句子，自然段）>
>
> **[00:20]** <下一段，按话题自然断段>
>
> **[01:00]** <又一段>
>
> ...
```

**Composition rules:**

- **Shownotes** 节可选：`has_video = False` 且 description 中能提取出结构化信息（主播 / 延伸资料 / 收听平台等）才渲染；视频平台通常省略此节。
- **frontmatter**：`type` 和第一个 `tag` 根据 `has_video` 设为「视频笔记」或「播客笔记」；`transcript_source` 设为 `subtitle` 或 `asr`。
- **思维导图** 用 `markmap` 代码块（不是 Mermaid），用 markdown 标题层级表示节点层级。根 `#` 为核心主题，一级 `##` 为 3–8 个章节，二级/三级为细节。要覆盖视频/播客中每个要点不遗漏。
- **initialExpandLevel** 在 frontmatter 里设为 `3`。
- **时间戳来源**：段落 `begin_time_ms` 毫秒转 `MM:SS`，不要根据语速估算。章节时间 / 金句时间均锚定到对应段的 begin_time_ms。
- **金句** 3–8 条，选观点最凝练、最有传播力的句子，尽量保留原文措辞。
- **章节导读** 3–8 行。
- **转录格式**：
  - callout 标题只用「完整转录」，不加模型名 / 段时长 / 括号。
  - 每段前加 `**[MM:SS]** `，段间用 `>` 空行 `>` 隔开。
  - **不加说话人标签**。多人对话播客也不猜、不标。
  - **按话题自然断段**。ASR 路径的 sentence 已在 Step 2B 合并；字幕路径的 cue 已按 >3s 间隔分段。写入前仍可检查是否有跨段语句需要合并或过长段需要拆分。
- **字符上限**：转录 ≤ 20000 字符直接嵌入折叠 callout；> 20000 字符写 `> 转录过长（N 字符），未保存。从 [原链接](URL) 回听。`

---

**Template（文章）：** 当 `content_type == "article"` 时使用此模板。

> **核心理念：** 将文章内容整理成适合长期沉淀的 Obsidian 知识笔记，而非简单摘要。主动提炼核心观点、关键概念、方法步骤、可执行建议，删除营销话术、重复表达、情绪化铺垫。如果内容适合和已有知识关联，使用 Obsidian 双链（如 `[[AI Agent]]`、`[[Dify]]`）。

```markdown
---
title: <article title>
source: <original url>
type: 微信公众号文章整理
tags:
  - 待分类
  - 文章笔记
created: <YYYY-MM-DD from execution date>
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

### 2. <观点二>

说明观点，并保留必要论据。

### 3. <观点三>

说明观点，并保留必要论据。

## 关键概念

- <概念一>：
- <概念二>：
- <概念三>：

## 方法 / 流程

如果文章中有操作步骤、框架、实现路径，整理成步骤：

1. <第一步>：
2. <第二步>：
3. <第三步>：

## 对我的启发

结合文章内容，提炼对我个人知识库、工作、AI Agent、自动化、产品或技术实践有价值的启发。

## 可执行事项

- [ ] <可以执行的任务一>
- [ ] <可以执行的任务二>
- [ ] <可以执行的任务三>

## 可关联笔记

- [[AI Agent]]
- [[Obsidian]]
- [[知识库管理]]
- [[工作流自动化]]

## 原文金句

> [!quote]
> <金句一 — 只保留 3-5 条真正有价值的原文表达，不要大段复制>

> [!quote]
> <金句二>

## 我的补充思考

<根据文章内容，补充你认为值得继续研究的问题。>
```

**文章笔记 Composition rules:**

- **整理而非摘要：** 删除营销话术、重复表达、情绪化铺垫和无信息量内容，将文章精炼为知识笔记。
- **frontmatter：** `type` 固定为「微信公众号文章整理」；`status` 为 `processed`；`created` 使用执行日期。
- **一句话结论：** 文章最重要的核心观点，一句击穿。
- **文章在讲什么：** 用 3-5 句话概括主旨，不要啰嗦。
- **核心观点：** 按文章逻辑分段，每节有明确的观点标题 + 论据支撑，不简单复制原文。
- **关键概念：** 从文章中提取的术语、工具、框架名称，每项附简短说明（如「Agent：能自主调用工具完成复杂任务的 AI 程序」）。
- **方法 / 流程：** 如果文章包含操作步骤、框架、实现路径，按编号步骤整理。如纯观点型文章可省略此节。
- **对我的启发：** 主动联想与个人工作、AI Agent、自动化、产品或技术实践的关联。
- **可执行事项：** 具体可落地的行动项，用 checklist 格式。
- **可关联笔记：** 使用 Obsidian 双链格式（`[[笔记名]]`），连接到知识库中已有或应有的相关笔记。优先关联 AI Agent、工具、方法论相关笔记。
- **原文金句：** 只保留 3-5 条真正有价值的原文表达，每条用 `> [!quote]` callout 包裹。不要大段复制。
- **我的补充思考：** 基于文章内容延伸出值得继续研究的问题或方向。
- **不包含** 思维导图、chapters 时间轴、完整转录（文章无时间轴）。

### 4. 清理临时文件

```python
import os, glob
slug = "SLUG"
for pat in [f'.ltn_sub_{slug}*', f'.ltn_audio_{slug}*', f'.ltn_asr_{slug}*', f'.ltn_page_{slug}*', f'.ltn_article_{slug}*']:
    for f in glob.glob(pat):
        os.remove(f)
```

---

### 5. 文章路径（curl + html.parser，零外部依赖；SPA 兜底用 browser）

> 🆕 **文章路径默认用纯 Python 标准库 + curl，无需安装任何包，零费用。**
>
> ⚠️ **SPA/JS 渲染页面检测：** curl 下载后检查 HTML 文件大小。如果 < 20KB 且不含正文关键词（如 `<article`、`content`、`post-body`），说明是 JS 动态渲染的 SPA（如 Notion、微信公众号、知乎）。此时改用 **browser 工具** 渲染页面 → `browser_snapshot` 获取内容 → 用 `browser_console` 执行 `document.querySelector('main')?.innerText || document.querySelector('.notion-page-content')?.innerText` 提取纯文本。详见 `references/spa-extraction.md`。

#### 5A. 下载网页 HTML

```bash
curl -sL --max-time 30 -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" \
  "URL" -o ./.ltn_page_SLUG.html
```

#### 5B. 提取元数据 + 正文

```python
from html.parser import HTMLParser
import re, json, pathlib

SLUG = "SLUG"
URL = "URL"

class ArticleExtractor(HTMLParser):
    """Extract article metadata + structured text from HTML."""
    def __init__(self):
        super().__init__()
        self.title = ""
        self.description = ""
        self.author = ""
        self.date = ""
        self.site_name = ""
        self.tags = []
        
        # State machine
        self.in_title = False
        self.in_content = False
        self.skip_tags = {'script', 'style', 'nav', 'footer', 'header', 'noscript', 'aside', 'svg', 'iframe', 'form', 'button', 'select', 'input', 'textarea', 'code', 'pre'}
        self.content_div = None  # Track the best candidate <article> or main content div
        self.content_lines = []
        self.text_buf = []
        self.current_tag_stack = []
        
        # Detect images for potential cover
        self.first_img_url = ""
        
    def handle_starttag(self, tag, attrs):
        attrs_dict = dict(attrs)
        self.current_tag_stack.append(tag)
        
        if tag in self.skip_tags:
            self.in_content = False
            return
        
        # Title extraction
        if tag == 'title':
            self.in_title = True
            return
        
        # Meta tag extraction
        if tag == 'meta':
            prop = attrs_dict.get('property', '').lower()
            name = attrs_dict.get('name', '').lower()
            content = attrs_dict.get('content', '')
            
            if prop == 'og:title':
                self.title = content
            elif prop == 'og:description' or name == 'description':
                if not self.description:
                    self.description = content
            elif prop == 'og:site_name':
                self.site_name = content
            elif prop == 'article:author' or name == 'author':
                self.author = content
            elif prop == 'article:published_time':
                self.date = content
            elif prop == 'article:tag':
                self.tags.append(content)
        
        # Image detection
        if tag == 'img' and not self.first_img_url:
            src = attrs_dict.get('src', '')
            if src and not src.startswith('data:'):
                self.first_img_url = src
        
        # Content extraction: prefer <article>, then common content divs
        if tag == 'article':
            self.content_div = tag
            self.in_content = True
            return
        
        cls = attrs_dict.get('class', '')
        role = attrs_dict.get('role', '')
        itemtype = attrs_dict.get('itemtype', '')
        
        if (cls and re.search(r'(article|post|content|entry|main)', cls, re.I)) or \
           role == 'main' or 'Article' in itemtype:
            if not self.content_div:  # First match wins
                self.content_div = tag
                self.in_content = True
        
        # Block-level elements: add line breaks
        if tag in ('p', 'div', 'section', 'blockquote', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'li', 'br', 'hr', 'pre'):
            if self.text_buf:
                text = ''.join(self.text_buf).strip()
                if text:
                    self.content_lines.append(text)
                self.text_buf = []
    
    def handle_endtag(self, tag):
        if tag in self.current_tag_stack:
            self.current_tag_stack.remove(tag)
        
        if tag == 'title':
            self.in_title = False
        
        if tag == self.content_div:
            self.in_content = False
        
        if tag in ('p', 'div', 'section', 'blockquote', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'li', 'pre'):
            if self.text_buf:
                text = ''.join(self.text_buf).strip()
                if text:
                    self.content_lines.append(text)
                self.text_buf = []
    
    def handle_data(self, data):
        if self.in_title:
            # Title from <title> tag (fallback if no og:title)
            self.title = self.title or data.strip()
            return
        
        # Filter out CSS/JS text spillover
        if any(t in self.skip_tags for t in self.current_tag_stack):
            return
        
        # Collect content text
        stripped = data.strip()
        if stripped:
            if self.content_div or not self.content_div:  # If no article tag found, collect everything
                self.text_buf.append(stripped)


# Parse HTML
with open(f'./.ltn_page_{SLUG}.html', 'r', encoding='utf-8', errors='replace') as f:
    html_content = f.read()[:3_000_000]  # Cap at 3MB

extractor = ArticleExtractor()
extractor.feed(html_content)

# Clean up extracted lines
content_lines = []
for line in extractor.content_lines:
    line = re.sub(r'\s+', ' ', line).strip()
    # Filter out navigation/menu text (short lines with special chars)
    if not line:
        continue
    if len(line) < 5 and not any(c.isalpha() for c in line):
        continue
    # Skip common boilerplate
    skip_patterns = [
        r'^(登录|注册|关注|收藏|点赞|评论|分享|举报|投诉|举报投诉|免责声明|版权声明|来源：)',
        r'^(备案|浙ICP|粤ICP|京ICP)',
        r'^(All Rights Reserved|Copyright ©|\d{4} .* Reserved)',
        r'^(APP|下载|扫一扫|二维码)',
    ]
    if any(re.search(p, line) for p in skip_patterns):
        continue
    content_lines.append(line)

# Deduplicate while preserving order
seen = set()
deduped = []
for line in content_lines:
    if line not in seen:
        seen.add(line)
        deduped.append(line)

content_text = '\n\n'.join(deduped)

# Metadata
TITLE = extractor.title or "未命名文章"
if not TITLE and content_lines:
    TITLE = content_lines[0][:80]

AUTHOR = extractor.author or extractor.site_name or ""
DESCRIPTION = extractor.description or ""

# Date parsing
DATE = ""
if extractor.date:
    try:
        DATE = extractor.date[:10]  # YYYY-MM-DD
    except:
        pass

site = extractor.site_name or ""

print(f"Title: {TITLE}")
print(f"Author: {AUTHOR} ({site})")
print(f"Date: {DATE}")
print(f"Description: {DESCRIPTION[:120]}")
print(f"Tags: {extractor.tags[:5]}")
print(f"Content lines: {len(deduped)}")
print(f"Content chars: {len(content_text)}")

# Save for Step 3 (note composition)
import json
result = {
    "title": TITLE,
    "author": AUTHOR,
    "site_name": site,
    "date": DATE,
    "description": DESCRIPTION,
    "tags": extractor.tags[:10],
    "content_text": content_text,
    "first_img_url": extractor.first_img_url,
    "content_type": "article",
    "url": URL,
}
pathlib.Path(f"./.ltn_article_{SLUG}.json").write_text(
    json.dumps(result, ensure_ascii=False, indent=2)
)
```

#### 5C. 内容截断

文章内容可能很长，在送入 LLM 生成摘要之前需要截断：

> **截断策略：** 如果 `content_text` 字符数 > 30,000，截取前 20,000 字符 + 中间抽样 5,000 字符 + 末尾 5,000 字符。在截断点插入 `[...内容截断...]` 标记。这样既保留开头和结尾，也能看到中间要点。最终送入 LLM 的内容 ≤ 30,000 字符。

```python
content = result["content_text"]
if len(content) > 30000:
    head = content[:20000]
    mid_start = len(content) // 2 - 2500
    mid = content[mid_start:mid_start + 5000]
    tail = content[-5000:]
    result["content_text"] = f"{head}\n\n[...内容截断...]\n\n{mid}\n\n[...内容截断...]\n\n{tail}"
```

#### 5D. 跳转到 Step 3 组织笔记

文章路径的 `paragraphs` 不需要（无时间戳），直接用 `content_text` 作为素材进入 **Step 3**，按文章笔记模板生成。

原文提取完成后：
- 跳到 **Step 3 组织笔记**
- 但使用 **文章笔记模板**（见下方）而非视频/播客模板
- 不需要 transcript 节、chapters 表、时间轴、金句时间戳

---

## Common Errors

| Error | Cause | Fix |
| --- | --- | --- |
| `401` | `ALIYUN_API_KEY` not loaded | Restart Claude Code after setting env |
| `405 BadRequest.RequestMethodNotAllowed` on `/api/v1/uploads` | 直接 POST 到 uploads endpoint | 按 Step 2B 流程：先 `GET ?action=getPolicy`，再 multipart POST 到 `upload_host`，最后用 `oss://` 引用 |
| Submit 200 但 task FAILED with `InvalidFile` | 缺 `X-DashScope-OssResourceResolve: enable` 头 | 补上请求头 |
| Upload fails | 文件过大或格式不支持 | 检查文件（<500MB）和格式（m4a / mp3 / wav） |
| Task FAILED | 音频质量差或语言不支持 | 加 `language_hints`；或本地预处理 |
| Task 轮询超时 | 长音频处理慢 | 增大 poll 次数（200 次 ~ 10 分钟） |
| Subtitle parse 空 | 字幕格式异常 | 换 `--sub-format`（srv3 > vtt > srt） |
| Bilibili yt-dlp 返回 `HTTP Error 412` | Bilibili 反爬策略，yt-dlp extractor 失效 | **不用 yt-dlp**，改用 Bilibili REST API（见 Step 1A）：`/x/web-interface/view` 取元数据，`/x/player/playurl` 取音频流 |
| 长中文文件名在 shell 中报错 | 临时文件命名 | 所有临时文件用 `slug`（MD5 前 10 位），只有最终 `.md` 用真实标题 |
| `rm -rf` 被 hook 拦截 | 安全 | 用 Python `os.remove` |
| 文章路径 curl 下载后 HTML 几乎为空（<20KB） | SPA / JS 动态渲染页面（如 Notion、微信公众号、知乎）| **改用 browser 工具渲染页面**，再用 `browser_console` 执行 `document.querySelector('main')?.innerText` 提取文本。curl 只能拿到静态 HTML shell。 |

---

## Quick Reference

```
URL → platform detect (YouTube / Bilibili / Apple Podcasts / 小宇宙 / article / generic)
    ├─ 文章 URL → curl/browser download → extract → [{title, content_text}]
    │   → compose article note → Write <DEFAULT_OUTPUT_DIR>/<title>.md
    │
    └─ 视频/音频 → metadata + subtitle check:
        ├─ Bilibili  → REST API（/x/web-interface/view + /x/player/v2）❌ 不用 yt-dlp（412）
        └─ YouTube等 → yt-dlp --dump-json
    → transcript pipeline:
        ├─ 视频 + 有字幕  → yt-dlp --write-auto-sub（YouTube）/ REST API subtitle（Bilibili）→ parse → [{begin_time_ms, text}]
        └─ 视频无字幕 或 音频 → yt-dlp audio（YouTube）/ REST API audio（Bilibili）→ paraformer-v2 async → merge_sentences → [{begin_time_ms, text}]
    → compose note (shownotes[optional] + summary + takeaways + mindmap
                    + chapters + highlights + details + thoughts + transcript)
    → Write <DEFAULT_OUTPUT_DIR>/{YouTube|Bilibili|Podcasts|Audio}/<title>.md
    → cleanup temp files
```

---

## When Not to Use

- URL 是纯文本 PDF / 网页文档 → 用 `defuddle` + 手动整理
- 需要 HTML 可视化播客页 → 先用此 skill 生成 `.md`，再调用 `podcast-to-html`
- 文章需要付费登录才能查看全文（如微信公众号、Medium 付费墙） → 此 skill 无法处理

---
name: obsidian-knowledge-distiller
description: "Use when turning articles, essays, web pages, video transcripts, audio transcripts, podcasts, talks, course material, or raw source notes into detailed, reusable Obsidian knowledge notes. 触发词：沉淀到 Obsidian、整理成知识笔记、文章总结、视频总结、音频总结、播客笔记、永久笔记、原子化笔记、双链笔记、knowledge note、source distillation."
---

# Obsidian Knowledge Distiller

## Purpose

Turn source material into a detailed Obsidian Markdown note that can be reused for thinking, writing, research, and future linking. Optimize for knowledge assets, not short summaries.

## Operating Rules

- Preserve the source's important claims, reasoning path, examples, stories, data, terms, and limitations.
- Distinguish facts, the author's opinions, your synthesis, and uncertain information.
- Do not invent source details. Mark missing or ambiguous items as `待核实`.
- Prefer clear Chinese output when the user writes in Chinese, unless they request another language.
- Use Obsidian `[[double links]]` for concepts, people, organizations, works, themes, and likely future notes.
- Always include a `# 文档目录` section after frontmatter. The directory must list the note's major headings and meaningful subheadings, using Obsidian heading links such as `[[#核心摘要]]`.
- If the user asks to save into a vault and the path is known, write a `.md` file. If the path is unknown, output the Markdown and ask for the vault path only if saving is required.
- When another skill or tool has already extracted text, subtitles, or transcripts, start from that material and focus on distillation.

## Workflow

1. Identify the source type: article, video, audio/podcast, transcript, or mixed notes.
2. Capture source metadata when available: title, author/channel/speaker, URL, publish date, duration, and access date.
3. Choose the relevant source treatment below.
4. Produce the standard note structure with clear headings and subheadings.
5. Generate or update `# 文档目录` after the final heading structure is known.
6. Add atomic note candidates and Obsidian link suggestions.
7. Run the quality checklist before finalizing.

## Source Treatments

### Articles

For articles, essays, reports, newsletters, and web pages:

- Reconstruct the author's problem, thesis, argument chain, evidence, examples, and conclusion.
- Extract frameworks, taxonomies, models, definitions, and tradeoffs completely.
- Identify hidden assumptions or stance when they matter.
- Preserve reusable writing material: metaphors, cases, memorable phrasing, data points, and named references.

### Videos

For videos, talks, lectures, demos, and subtitle-based material:

- Keep the time-order structure when it helps understanding.
- Preserve key timestamps if they exist in the transcript or metadata.
- Separate `视频讲了什么` from `我应该记住什么`.
- Include demonstrations, screen/visual context, examples, and suggested rewatch moments.
- Compress repeated spoken filler without losing meaningful details.

### Audio and Podcasts

For audio, podcasts, interviews, and conversations:

- Separate speakers when speaker information is available.
- Preserve questions, turns, disagreements, consensus, anecdotes, and contextual background.
- Extract judgment criteria, decision heuristics, personal experiences, and quotable expressions.
- Mark uncertain names, books, companies, dates, or numbers as `待核实`.

## Standard Note Structure

Use this structure unless the user provides a stronger template:

```markdown
---
title:
source_type:
source:
author:
created:
tags:
status: processed
---

# 文档目录

- [[#一句话概括]]
- [[#核心摘要]]
- [[#结构化梳理]]
  - [[#背景 / 问题]]
  - [[#主要观点]]
  - [[#论证过程]]
  - [[#案例 / 数据 / 故事]]
  - [[#结论 / 限制]]
- [[#关键概念]]
- [[#重要细节与证据]]
- [[#值得摘录的原句]]
- [[#我的启发]]
- [[#可链接笔记建议]]
- [[#原子化笔记候选]]
- [[#待核实]]

# 一句话概括

用一句话说明这份材料最核心的价值。

# 核心摘要

用 5-10 条 bullet 总结主要内容。每条包含“观点 + 解释 + 必要细节”，避免只写短标签。

# 结构化梳理

按照原内容的逻辑顺序拆解。使用二级标题保留文档层级，不要只写成无层级列表。

## 背景 / 问题

## 主要观点

## 论证过程

## 案例 / 数据 / 故事

## 结论 / 限制

# 关键概念

每个概念包含：
- 概念名
- 简明解释
- 为什么重要
- 可能连接的主题或笔记

# 重要细节与证据

列出支撑核心观点的事实、例子、数据、引用对象、实验、经历或观察。

# 值得摘录的原句

列出 5-10 条最值得保留的原句。无法确认原句时标注 `意译`。

# 我的启发

- 认知更新：
- 可行动建议：
- 可写作/表达素材：
- 可继续研究的问题：

# 可链接笔记建议

列出适合创建或链接的 Obsidian 双链，例如：
- [[概念名]]
- [[人物/公司/书名]]
- [[相关主题]]

# 原子化笔记候选

拆出 3-8 条可单独成文的永久笔记：
- 标题：
- 核心观点：
- 适合链接到：

# 待核实

列出不确定的人名、数据、来源、时间点、引用或需要回看原文的位置。
```

## Optional Sections

Add these sections only when useful:

- `# 时间线摘要`: for videos, lectures, demos, or transcripts with timestamps.
- `# 分说话人观点`: for interviews, podcasts, panels, and conversations.
- `# 值得回看的片段`: for videos or audio with timestamps.
- `# 操作步骤`: for tutorials, demos, workflows, or technical material.
- `# 反对意见与限制`: when the source is argumentative or controversial.

When optional sections are added, update `# 文档目录` so it reflects the actual headings in the final note. For long optional sections, include their most useful `##` subheadings in the directory.

## File Writing

When saving to Obsidian:

- Use filename format `YYYY-MM-DD - 主题名.md`.
- Keep YAML tags to 3-8 useful tags.
- Do not overwrite existing notes unless the user explicitly asks.
- If related notes are found, suggest links or merge candidates instead of deleting content.
- After writing, report the new file path, key links, and any unresolved verification items.

## Quality Checklist

Before finalizing, verify:

- The note is more detailed than a generic summary.
- `# 文档目录` exists and includes both major headings and useful subheadings.
- The reasoning path is preserved, not only the conclusion.
- Facts, opinions, inferences, and uncertainties are separated.
- Cases, examples, data, names, and frameworks are not silently dropped.
- Obsidian links are useful and not spammy.
- Atomic note candidates are standalone and reusable.
- `待核实` captures unresolved ambiguity instead of hiding it.

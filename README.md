# Echo Skills

个人 Skill 仓库——跨 Agent 平台的可复用技能集。

配合 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 使用。

## Skills

| Name | Description |
|------|-------------|
| [agent-memory-pack](agent-memory-pack/SKILL.md) | Portable Agent Memory Pack 管理——跨平台、跨 Agent 的可移植记忆包 |
| [obsidian-knowledge-distiller](skills/obsidian-knowledge-distiller/SKILL.md) | 将文章、视频、音频、播客或转录稿沉淀为详细清晰、可复用的 Obsidian 知识笔记 |

## 安装

```bash
# 添加为 skill 源
hermes skills tap add mrx-data/echo-skills

# 安装 skill
hermes skills install agent-memory-pack
```

## License

MIT

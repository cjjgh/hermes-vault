# 🧠 Hermes Vault — 万手Claw 知识库与记忆系统

Hermes AI Agent 的全量知识库，基于 Obsidian + Git 构建，双向同步。

## 📂 目录结构

| 目录 | 内容 | 来源 |
|------|------|------|
| `01-灵魂身份/` | SOUL.md — 万手Claw 的身份定义与价值观 | `~/.hermes/SOUL.md` |
| `02-行为规则/` | AGENTS.md — 行为纪律与 MoA 规则 | `~/.hermes/AGENTS.md` |
| `03-长期记忆/` | MEMORY.md + 专题记忆 + 归档 + 改进记录 | `~/.hermes/memories/` |
| `04-用户画像/` | USER.md — 用户画像与偏好 | `~/.hermes/memories/USER.md` |
| `05-蒸馏日志/` | KAIROS.md — 日结蒸馏 | `~/.hermes/memories/KAIROS.md` |
| `06-进化训练/` | 进化三部曲 + 题库数据 + 历史报告 | `~/.hermes/进化三部曲/` + `training_data/` |
| `07-技能工坊/` | 技能清单与说明 | `~/.hermes/skills/` |
| `08-知识库/` | 长期知识沉淀（AI/编程/金融） | 人工/自动积累 |
| `09-系统管理/` | 脚本参考 + 配置 + 状态轨迹 | `~/.hermes/scripts/` + `config.yaml` |
| `10-日常笔记/` | 日常想法与学习记录 | 人工使用 |

## 🔄 自动同步

| 定时任务 | 周期 | 用途 |
|---------|------|------|
| `vault-自动同步` | 每30分钟 | Hermes 记忆 → vault |
| `archive_to_vault.sh` | 进化训练完成后 | 报告 → vault |

## 📖 日常使用指南

见 `10-日常笔记/使用指南.md`

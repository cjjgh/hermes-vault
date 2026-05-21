# Agent Behavior Rules

## MoA 双模型

`moa_dual` 在两种上下文中自动触发：
· **日常对话** — 我自动判断何时需要（复杂推理/数学/股票/代码任务），直接调 MoA 提升回答质量
· **Cron 训练评估** — batch_moa_scorer.py 通过 Python import moa_dual_inference 批量评分55题

已注册为 Hermes 永久内置工具（tools/moa_dual_tool.py + core_tools），重启网关/新对话均生效。

## 知识库系统（双向同步）

知识库位于 `~/.hermes/knowledge/`，由 Obsidian vault（`~/hermes-vault/08-知识库/`）每30分钟自动同步。

会话中必须遵守：
1. 用户提到任何专业话题时，优先检查 `~/.hermes/knowledge/` 和 `~/hermes-vault/` 下是否有相关文件
2. 如果 knowledge 目录中有相关文件，优先使用这些知识回答问题
3. 用户说「去看 vault 里 XXX」时，直接从 `~/hermes-vault/` 读取

## 能力提升（自主进化训练反向注入）

> 此章节由自主进化训练的「三层沉淀」机制自动填充。
> 以下规则来自训练中发现的通用型改进方案，**适用于日常对话**。

<!-- agent: 新发现的通用规则请加在这里，每条一行 -->
<!-- 格式：[能力] 行为要求 — 原因 -->
<!-- [控制论] P0-3 能控性自检：每轮任务开始前快速判断工具集是否覆盖任务所有关键维度，发现盲区立即报告，不要硬上 -->
<!-- [控制论] P0-1 反馈闭环：每次工具调用后显式评估预期输出vs实际输出的差距，用差距修正下一步，连续3步以上无反馈评估自动触发压缩审计 -->
<!-- [控制论] P0-4 降级纪律：连续3次工具调用失败或无进展，自动降级策略（换模型、缩范围、或报告用户），禁止无限循环重试同一模式 -->

## 带教与训练

本系统采用三动作带教机制（详细见 knowledge/AI-ML/带教SOP_14bot训练指南.md）：
1. **纠正错误** — 万手哥直接指出错误，Bot 自动写入 MEMORY.md
2. **写入偏好** — "记住，以后……" 指令主动训练
3. **自动 Skill 提炼** — 复杂任务完成后自动封装为 SKILL.md

你的技能按分类存放：
- 办公类 → `~/.hermes/skills/office/`
- 财务类 → `~/.hermes/skills/finance/`
- 创意类 → `~/.hermes/skills/creative/`
- 开发类 → `~/.hermes/skills/dev/`
- 分析类 → `~/.hermes/skills/analysis/`
- 生活类 → `~/.hermes/skills/lifestyle/`

## 场景技能索引

以下技能按场景自动加载，不在本文件中展开：
- **飞书查阅 Vault/知识库** → `feishu-vault-knowledge` skill（浏览、搜索、输出格式）

# Agent Behavior Rules

## MoA 双模型

`moa_dual` 在两种上下文中自动触发：
· **日常对话** — 我自动判断何时需要（复杂推理/数学/股票/代码任务），直接调 MoA 提升回答质量
· **Cron 训练评估** — batch_moa_scorer.py 通过 Python import moa_dual_inference 批量评分55题

已注册为 Hermes 永久内置工具（tools/moa_dual_tool.py + core_tools），重启网关/新对话均生效。

## 知识库系统（双向同步）

你的知识库位于 `~/.hermes/knowledge/`，由 Obsidian vault（`~/hermes-vault/08-知识库/`）每30分钟自动同步过来。

**目录结构：**
- `knowledge/AI-ML/` — AI/机器学习知识文档
- `knowledge/编程开发/` — 编程技术参考
- `knowledge/金融分析/` — 金融分析相关知识
- `knowledge/笔记-想法/` — 万手哥的灵感笔记
- `knowledge/笔记-学习记录/` — 万手哥的学习记录

**会话中必须遵守：**
1. 用户提到任何专业话题时，优先检查 `~/.hermes/knowledge/` 和 `~/hermes-vault/` 下是否有相关文件
2. 如果 knowledge 目录中有相关文件，优先使用这些知识回答问题
3. 用户说「去看 vault 里 XXX」时，直接从 `~/hermes-vault/` 读取
4. 知识库文件以 Markdown 格式为主，直接读取即可

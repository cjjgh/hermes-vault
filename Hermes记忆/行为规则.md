# Agent Behavior Rules

## MoA 双模型

`moa_dual` 在两种上下文中自动触发：
· **日常对话** — 我自动判断何时需要（复杂推理/数学/股票/代码任务），直接调 MoA 提升回答质量
· **Cron 训练评估** — batch_moa_scorer.py 通过 Python import moa_dual_inference 批量评分55题

已注册为 Hermes 永久内置工具（tools/moa_dual_tool.py + core_tools），重启网关/新对话均生效。

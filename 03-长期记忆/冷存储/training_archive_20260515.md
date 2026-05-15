# Training History Archive — 20260515

Archived from MEMORY.md to free up capacity.

---
[self-verification-2026-05-14] 三层自验架构：L1 调用前预测预期结果（写入checkpoint）→ L2 delegate_task 子 agent 独立审查产出（只找错不说好话）→ L3 交付前用独立推理路径证伪。核心原则：**不自查**——同一个大脑验证同一大脑产出不可靠（Self-Refine论文确认94%错误被自反馈遗漏）。

---
[2026-05-14 全面进化改造会话] 用户明确我搞错了每日进化执行 v3 的定位——它不是"按报告做手术"，我解释的方向不对。我需要重新检查这个 cron 的 prompt 来纠正理解。

---
自进化训练v2审计：全链路已验证可跑通。batch_moa_scorer.py 直调 moa_dual_inference (Python import) 在 cron 上下文工作正常（实测2题全过+checkpoint断点续跑）。cron prompt venv 路径 ~/.hermes/hermes-agent/venv/bin/python3 有效（双venv兼容）。lark-cli auth scope 含 wiki:node:create/docs:document:create，归档脚本可正常工作。AGENTS.md 中 moa_dual 条款已修正（不再禁止日常对话）。refresh_tests.py 题量从3/2/1统一升级到5题/能力。

---
飞书知识库归档教训（2026-05-14）：报告markdown中 `- ` 列表前缀 + `|` 管道符组合会导致飞书页面渲染空白（数据写入正常但用户打开是白屏）。根因是飞书markdown导入引擎解析冲突。修复：含 `|` 的行首不用列表标记，用纯文本对齐。已写入 self-evolution-trainer/references/feishu-report-format.md。

---
[self-evolution-training-2026-05-15] 执行力能力偏低（38/100）。根因：回答太抽象——只写了代码框架和思路，没有实际执行并展示结果。改进：执行力测试题必须实际执行脚本/命令，提供真实输出结果验证，不能只有「方案描述」。

---
[self-evolution-training-2026-05-15] 数学推理能力偏低（47/100）。根因：部分高难度题回答不充分（math_hendrycks_15的代数运算步骤不够清晰），还有2题MoA评分失败。改进：数学证明题必须展示完整推理步骤（每步推导+理由），计算题最好用execute_code验证后附上计算结果。

---
[self-evolution-training-2026-05-15] 推理力中等（61/100）。问题：逻辑谜题（骑士与无赖、帽子问题）的回答需要更多结构化推理步骤展示。改进：推理题采用三段论格式（假设→推导→结论），每一步标注逻辑规则。

---
[self-evolution-training-2026-05-15] 抗幻觉能力优秀（95/100）。有效策略：对每个问题先判断知识边界，标注不确定性，循证评估（引用具体研究数据），对个人经验案例指出推理谬误并解释统计原理。这些策略在后续测试中继续保持。

---
[self-evolution-training-2026-05-15] 情商较好（76/100）。有效策略：Hofstede文化维度分析、STAR框架沟通、共情+引导话术。薄弱点：多利益方协调场景的折中方案需要更具体的时间线和量化指标。

---
[self-evolution-training-2026-05-15] 反思力中等（64/100）。有效策略：对安全漏洞/竞态条件/算法优化的分析准确。薄弱点：电商系统评估缺少成本收益的量化对比。改进：架构评估和代码审查题用清单法逐项检查，每项附带严重级别标注。


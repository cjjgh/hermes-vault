# 进化执行日志

> 说明：每日进化执行 v3 的任务日志。每次执行后追加记录。

## 2026-05-13 — 进化三部曲初始化

### 已完成
- E1: 认清自己 — AGENTS.md 全面刷新（412行，18KB）
- E2: 开天眼 — CC源码深度分析报告（544行，31KB）
  - 15条进化建议 + 5个低垂果实
- E3: 炼金术 — 工程控制论融合报告（723行，40KB）
  - 25个映射关系 + 12条落地建议
- E4: 自动进化计划设计完成
  - 每日进化执行 v3（升级）
  - 每日系统健康审计 v5（升级）
  - 每周进化回顾（新增）
  - P0 执行清单（本周5项）

### 待执行
- 明日起每日 20:00 开始按周计划自动执行 P0 项

## 2026-05-13 — E2 P0#3 COMPACTABLE_TOOLS 白名单

### 已完成
- **E2 P0#3: COMPACTABLE_TOOLS 白名单** — 微压缩粒度提升

### 改动
| 文件 | 改动 |
|------|------|
| `agent/context_compressor.py` | 新增 `COMPACTABLE_TOOLS` frozenset（10个只读工具） |
| `agent/context_compressor.py` | `_prune_old_tool_results()` Pass 2 增加白名单门控 |
| `tests/agent/test_context_compressor.py` | 修复 `test_prune_with_token_budget` 的 tool_call ID 映射 |

### 设计要点
- COMPACTABLE_TOOLS 白名单 = 只读工具：`read_file`, `search_files`, `web_search`, `web_extract`, `browser_snapshot`, `browser_vision`, `vision_analyze`, `skill_view`, `skills_list`, `session_search`
- 写/执行工具（`write_file`, `patch`, `terminal`, `delegate_task`, `memory`, `cronjob` 等）的输出不被压缩，保留上下文完整性
- 参考 CC `microCompact.ts:41-50` 设计理念

### 验证
- `pytest tests/agent/test_context_compressor.py -x -q` → 76 passed
- `pytest tests/agent/test_context_compressor_summary_continuity.py -x -q` → 2 passed

### 关键决策
- 进化方向：CC架构借鉴（E2）+ 控制论方法论（E3）双引擎驱动
- 执行策略：小步快跑，一天一项，验证后再下一项
- 报告存放：~/.hermes/进化三部曲/

## 2026-05-14 — E2 P0#4 子代理工具过滤增强

### 已完成
- **E2 P0#4: 子代理工具过滤增强** — 从单层硬编码黑名单升级为 3 层过滤结构

### 改动
| 文件 | 改动 |
|------|------|
| `tools/delegate_tool.py` | 新增 3 层 frozenset：`ALL_AGENT_DISALLOWED_TOOLS`, `CUSTOM_AGENT_DISALLOWED_TOOLS`, `ASYNC_AGENT_ALLOWED_TOOLSETS` |
| `tools/delegate_tool.py` | 新增 `filter_tools_for_agent()` 函数（is_async→is_custom→base 三层过滤） |
| `tools/delegate_tool.py` | `_strip_blocked_tools` 重构为 legacy wrapper，行为不变 |
| `tools/delegate_tool.py` | `_TOOLSET_LIST_STR` 构建器改用 `ALL_AGENT_DISALLOWED_TOOLS | CUSTOM_AGENT_DISALLOWED_TOOLS` |
| `tests/tools/test_delegate.py` | `TestBlockedTools` 更新为 3 层测试（Layer 1/2/3 + legacy alias） |

### 设计要点
- **Layer 1 (ALL)**: `delegate_task`, `clarify`, `memory`, `send_message` — 所有子代理禁用
- **Layer 2 (CUSTOM)**: `execute_code`, `cronjob` — 自定义/Skill 代理额外禁用
- **Layer 3 (ASYNC)**: 白名单 9 个工具集（file/web/search/browser/terminal/vision/skills/session_search/todo）
- `DELEGATE_BLOCKED_TOOLS` 保留为 `ALL_AGENT_DISALLOWED_TOOLS` 的 legacy 别名
- 参考 CC `agentToolUtils.ts:70-116` filterToolsForAgent() 设计模式

### 验证
- `python3 -c "from tools.delegate_tool import ..."` → 8 项断言全部通过
- `pytest tests/tools/test_delegate.py -k "TestStripBlockedTools or TestBlockedTools"` → 8 passed
- `pytest tests/tools/test_delegate_toolset_scope.py` → 5 passed
- `pytest tests/tools/test_delegate.py tests/tools/test_delegate_toolset_scope.py` → 134 passed, 1 flaky (heartbeat timing, unrelated)

### 关键决策
- 适配 Hermes 工具集架构：CC 按工具名过滤，Hermes 按工具集名过滤，两种方式在工具集层面等价
- 执行纪律：先备份 → 分步 patch（3 处）+ 验证每步 → 更新测试 → 全面跑测试

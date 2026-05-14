# E2: Claude Code 源码深度分析 vs 万手Claw/Hermes 系统 — 进化设计方案

> 生成时间：2026-05-13 | 分析基础：CC v2.1.88 / Hermes v0.13.0 / Phoenix V5.1.3

---

## 目录

1. [CC源码核心架构总结](#1-cc源码核心架构总结)
2. [与万手Claw/Hermes系统的逐项对比](#2-与万手clawhermes系统的逐项对比)
3. [具体可落地的进化建议清单](#3-具体可落地的进化建议清单)

---

## 1. CC源码核心架构总结

### 1.1 总体架构

CC 是 TypeScript 单体应用（~2,048+ 源文件，main.tsx 单文件 803KB），核心架构围绕**工具调用循环（Tool-Use Loop）**展开：

- **QueryEngine** (`src/QueryEngine.ts`, 46KB) — 核心调度器，管理消息流、API调用、工具执行
- **Tool 系统** (`src/Tool.ts`, 29KB + `src/tools.ts`, 17KB) — 统一的工具接口 + 注册表
- **Context 系统** (`src/context.ts`, 10KB + `src/context/`) — 系统提示组装 + 上下文管理
- **State 管理** (`src/state/`) — React-based AppState + 工具执行状态
- **组件化架构** — Ink React 组件渲染 TUI

### 1.2 CC 的核心优势（共 7 个领域）

| 领域 | CC 实现 | 文件规模 | 优势等级 |
|------|---------|----------|----------|
| 记忆系统 | memdir/ 8 文件 | ~83KB | ★★★★★ |
| 子代理系统 | AgentTool/ 17 文件 | ~320KB | ★★★★★ |
| 上下文压缩 | services/compact/ 14 文件 | ~160KB | ★★★★ |
| 权限系统 | utils/permissions/ 26 文件 | ~425KB | ★★★★★ |
| MCP 集成 | services/mcp/ 24 文件 | ~470KB | ★★★★★ |
| 文件操作 | FileReadTool/ 5 文件 + FileStateCache | ~70KB | ★★★★ |
| Plan 模式 | EnterPlanMode + 组件 | ~30KB | ★★★ |

---

## 2. 与万手Claw/Hermes系统的逐项对比

### 2.1 AgentLoop（核心循环）

| 维度 | CC | Hermes (run_agent.py) |
|------|----|----------------------|
| **核心类** | `QueryEngine` (~800 lines) + 状态机 | `AIAgent` (run_agent.py, 15,679 lines) |
| **消息管理** | 严格类型化 Message 系统 (27种message类型) | Python dict-based 消息列表 |
| **工具调度** | 顺序执行 + `shouldDefer` 机制控制优先级 | `_execute_tool_calls_sequential` / `_execute_tool_calls_concurrent` |
| **重试策略** | 工具级 retry + prompt cache break detection | `api_max_retries: 3` |
| **流式支持** | `StreamEvent` 类型完美支持 | 通过 `stream_delta_callback` |
| **会话管理** | 持久化 transcript + sessionStorage | state.db + 会话记录 |

**关键差距**：
- CC 的消息类型系统非常完整（27种 Message 子类型：UserMessage, AssistantMessage, ToolUseSummaryMessage, SystemCompactBoundaryMessage, HookResultMessage, TombstoneMessage 等），支持暂停/恢复/压缩标记/钩子结果/墓碑消息。Hermes 使用 Python dict 消息列表，类型安全弱很多。
- CC 的 `shouldDefer` 装饰器机制允许工具声明是否需要前置执行（如 AgentTool 延迟执行），Hermes 缺少类似机制。
- CC 的 Prompt Cache Break Detection (`services/api/promptCacheBreakDetection.ts`) 可以在缓冲失效时自动触发压缩，Hermes 依赖手动配置。

**Hermes 独特优势**：
- 并发工具执行 (`_execute_tool_calls_concurrent`) 原生支持
- `tool_guardrails.py` 提供精确的循环检测（连续失败/无进展/无意义重试），CC 不直接提供这个

---

### 2.2 记忆系统 (Memory)

| 维度 | CC (memdir/) | Hermes (memory_manager.py + files) |
|------|-------------|------------------------------------|
| **核心文件** | `memdir.ts` (507行), `memoryTypes.ts` (271行), `paths.ts` (278行), `findRelevantMemories.ts` (141行) | `memory_manager.py` (555行), `memory_provider.py` (279行) |
| **MEMORY.md 上限** | 200 行 + 25KB 双上限，over 则截断并追加 WARNING | ~2200 字符 ≈ ~30行，无字节上限 |
| **记忆类型** | 4 种：`user/feedback/project/reference`，XML 语义标签 | 3 种：热/温/冷，按时间分层 |
| **自动记忆** | `extractMemories` 后台 Agent 在会话结尾自动提取 + 写 MEMORY.md | `auto_extract: true` 通过 Phoenix 配置，但在系统提示中手动触发 |
| **语义搜索** | `findRelevantMemories()` 用 Sonnet 模型进行记忆相关性选择（每次查询最多 5 个） | 无语义搜索；MEMORY.md 全量注入 |
| **每日日志** | `getAutoMemDailyLogPath()` → `logs/YYYY/MM/YYYY-MM-DD.md` KAIROS 模式 | `logs/health.log` 仅健康检查 |
| **团队记忆** | `teamMemPaths.ts` / `teamMemPrompts.ts` 团队协作记忆同步 | 无 |
| **记忆引用** | XML fence tags `<memory-context>` 精确标记注入区域 | 无 fence 机制 |
| **MEMORY.md 结构** | 索引式（topic links）+ frontmatter | 扁平列表 + topic 文件 |

**关键差距**：

1. **双上限截断机制**（CC `memdir.ts:57-101`）：CC 同时检查行数（200行）和字节数（25KB），先截行再截字节，末尾追加 WARNING 说明哪个上限被触发。Hermes 只有 ~2200 字符的软限制，无截断逻辑。

2. **记忆类型系统**：CC 的 4 种记忆类型各有 `scope`（private/team）、`when_to_save`、`how_to_use`、`examples` 的 XML 结构描述。系统提示中动态注入 `TYPES_SECTION_INDIVIDUAL` 或 `TYPES_SECTION_COMBINED`（~200行）。Hermes 只有冷/热/温时间分层。

3. **语义记忆选择**：CC `findRelevantMemories.ts:39-75` 用 Sonnet 模型作为选择器——扫描所有记忆文件的 header+description，让 LLM 选择最多 5 个相关文件加载。Hermes 不做选择，当前整个 MEMORY.md 硬注入。

4. **自动记忆提取**：CC 有一个专门的 `services/extractMemories/` 目录，在会话末尾 fork 一个后台 agent 扫描对话日志提取记忆写入 MEMORY.md。Hermes 的 `auto_extract` 只是配置项，实际依赖用户主动调用 `memory()` 工具。

5. **内存路径安全**：CC `paths.ts:109-150` 有完整的路径验证函数 `validateMemoryPath()`，拒绝相对路径/根目录/UNC 路径/null 字节。Hermes 没有类似的路径安全检查。

---

### 2.3 子代理系统 (Sub-Agents)

| 维度 | CC (AgentTool/) | Hermes (delegate_tool.py) |
|------|----------------|---------------------------|
| **核心文件** | `AgentTool.tsx` (233KB UI+逻辑), `runAgent.ts` (973行), `agentToolUtils.ts` (686行), `loadAgentsDir.ts` (600+行) | `delegate_tool.py` (2767行), `file_state.py` (332行) |
| **内置代理** | 6个：GeneralPurpose, Explore, Plan, Verification, CodeGuide, Statusline | 0个内置；依赖 `delegate_task` 工具动态分派 |
| **工具过滤** | 3层过滤：ALL_AGENT_DISALLOWED_TOOLS / CUSTOM_AGENT_DISALLOWED_TOOLS / ASYNC_AGENT_ALLOWED_TOOLS | DELEGATE_BLOCKED_TOOLS frosenset（硬编码） |
| **异步代理** | `background: true` 属性（Verification Agent 默认后台运行）+ 进度通知 + 结果汇总 | 线程池 (`ThreadPoolExecutor`) 单任务 + 批处理 |
| **代理定义** | TypeScript 中的 `BuiltInAgentDefinition` 对象，含 tools/disallowedTools/model/getSystemPrompt | 无代理定义系统；delegate_task 动态构造 |
| **自定义代理** | `loadAgentsDir.ts` 从 `.claude/agents/` 加载 + `AGENTS.md` 描述 | `.agents/skills/` — Skill 系统而非 Agent 系统 |
| **模型选择** | `getAgentModel()` 支持 per-agent 模型（inherit/haiku/sonnet）+ GrowthBook 运行时覆盖 | `phoenix/config.json` 的 `task_type_mapping.delegation: subtask` |
| **子代理记忆** | `agentMemory.ts` + `agentMemorySnapshot.ts`——子代理可以读写内存，但受 Scope 控制 | 子代理通过 `file_state.py` 协调文件状态，无内存隔离 |
| **进度跟踪** | `ProgressTracker` 类 + 实时进度通知 UI | `ThreadPoolExecutor` 完成才返回 |

**关键差距**：

1. **代理级别丰富度**：CC 有 6 个精心设计的 Built-in Agent，每个都有自己的 System Prompt、工具过滤、模型选择和交互策略。Explore 只读搜索（30秒返回），Plan 架构设计（无写权限），Verification 对抗性测试（后台运行），GeneralPurpose 全工具。Hermes 只有 `delegate_task` 一个通用分派，没有特化代理。

2. **工具过滤系统**：CC 的 `filterToolsForAgent()`（`agentToolUtils.ts:70-116`）是一个 3 层过滤器，根据 isBuiltIn + isAsync + permissionMode 动态裁剪工具列表。代理还可以声明 `disallowedTools`。自治代理（异步）只能访问 ASYNC_AGENT_ALLOWED_TOOLS 白名单。MCP 工具总是开放。Hermes 只有 `DELEGATE_BLOCKED_TOOLS` 硬编码黑名单。

3. **代理定义格式**：CC 代理是声明式对象（TypeScript `AgentDefinition`），包含 `agentType`、`whenToUse`（对模型描述何时用这个代理）、`tools`、`disallowedTools`、`model`、`background`、`getSystemPrompt()`。Hermes 缺少代理管理系统。

4. **后台异步代理**：CC 的 Verification Agent 在后台运行，主线程继续工作，完成后通知。Hermes 的 ThreadPoolExecutor 在主线程阻塞等待。

5. **子代理文件状态协调**：Hermes 的 `file_state.py`（332行）提供了跨代理文件状态协调（`FileStateRegistry`），CC 的 `FileStateCache`（142行）更轻量。

---

### 2.4 上下文压缩 (Context Compression)

| 维度 | CC (services/compact/) | Hermes (context_compressor.py + Phoenix) |
|------|------------------------|-------------------------------------------|
| **核心文件** | `compact.ts` (1705行), `microCompact.ts` (530行), `autoCompact.ts` (351行), `sessionMemoryCompact.ts` (630行) | `context_compressor.py` (1555行), 无 Phoenix executor 代码 |
| **微压缩** | `microCompact.ts`：每次 API 返回后，清除低价值工具结果（读文件、grep 等 COMPACTABLE_TOOLS 的输出），替换为 `[Old tool result content cleared]` | `phoenix/config.json executor.micro_compact.threshold_chars: 100000`，基于字符阈值触发 |
| **自动压缩** | `autoCompact.ts`：当上下文窗口接近满时，fork 一个压缩 agent 汇总对话历史，替换为紧凑摘要 | `context_compressor.py`：用辅助模型压缩中间轮次，`_SUMMARY_RATIO=0.20`，上限 12K tokens |
| **尾部保护** | 自动保留尾部的消息（包含活跃的工具调用） | 尾部保护：通过 `_content_length_for_budget` 计算保留最近消息 |
| **消息分组** | `grouping.ts`：按 API round 边界分组（不是按人类消息），更精确 | 无分组，线性压缩 |
| **Snip 投影** | `sessionMemoryCompact.ts`：实验性，维护会话记忆的快照投影 | 无 |
| **紧凑边界消息** | 特殊的 `SystemCompactBoundaryMessage` 类型，标记压缩发生的位置 | 无边界标记 |
| **压缩后清理** | `postCompactCleanup.ts`：压缩后清理工具状态 | 无 |

**关键差距**：

1. **多层压缩策略**：CC 有 3 层压缩——microCompact（每轮清除旧工具输出）、autoCompact（上下文接近满时汇总）、sessionMemoryCompact（保留跨轮记忆）。Hermes 只有 deep_compact 一层。

2. **微压缩精度**：CC 的 `microCompact.ts:41-50` 定义 `COMPACTABLE_TOOLS` 集合（FileRead, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite），只压缩这些工具的旧输出。Hermes 的 micro_compact 基于字符数阈值（100K），粒度更粗。

3. **紧凑边界标记**：CC 使用 `SystemCompactBoundaryMessage` 标记压缩发生的位置，恢复时能精确重建。Hermes 无此标记。

4. **消息分组**：CC 按 API Round (assistant-turn boundary) 分组，比按人类消息分组更精细，在单轮多工具调用场景中更准确。

5. **压缩摘要质量**：CC `compact.ts` 的 fork agent 压缩使用主模型（或指定模型），生成结构化的 `> WARNING: compacted` 边界消息。Hermes 用辅助模型单独压缩。

---

### 2.5 权限系统 (Permissions)

| 维度 | CC (utils/permissions/) | Hermes (approval.py + config.yaml) |
|------|-------------------------|-------------------------------------|
| **核心文件** | `permissions.ts` (1486行), `permissionSetup.ts` (1600+行), `filesystem.ts` (1777行), `permissionsLoader.ts` (300+行), `yoloClassifier.ts` (60KB) | `approval.py` (1367行), `tool_guardrails.py` (455行), `file_safety.py` |
| **权限模式** | 6种：`default` / `plan` / `acceptEdits` / `bypassPermissions` / `dontAsk` / `auto` | 工具级 gating：`disabled_toolsets` / 危险命令审批 |
| **颗粒度** | 工具级 + 路径级 + 命令内容级 + 规则文件（.claude/rules/） | 工具集级别 + 危险命令模式匹配 |
| **规则来源** | 6级：`policySettings` > `projectSettings` > `localSettings` > `userSettings` > `flagSettings` > `managedSettings` | 单级：`config.yaml` + `.env` |
| **alwaysAllow** | `permissionRuleValueSchema` 定义 allow/deny/ask 三种行为，持久化到 settings.json | `approval.py` 的 `approval_allowlist` 持久化到 config.yaml |
| **命令分类器** | `bashClassifier.ts` + `yoloClassifier.ts`（60KB）：用 LLM 对 Bash 命令做上下文分类 | 正则匹配 `DANGEROUS_PATTERNS`（硬编码列表） |
| **路径验证** | `pathValidation.ts` (500行) + `filesystem.ts` 中 `isAutoMemPath()` 安全写入豁免 | `file_safety.py` + `file_operations.py` 的 `WRITE_DENIED_PATHS` |
| **影子规则检测** | `shadowedRuleDetection.ts`：检测冲突的权限规则，防止规则被静默覆盖 | 无 |
| **通道审批** | `channelPermissions.ts`：通过 Telegram/Discord 通道远程审批 | 无（但 gateway WebSocket 提供审批队列） |

**关键差距**：

1. **权限模式系统**：CC 有 6 种权限模式，每种模式改变工具的审批行为。`plan` 模式禁用所有写入工具，`acceptEdits` 自动批准编辑，`bypassPermissions` 完全绕过。Hermes 无权限模式概念，只有工具集启用/禁用。

2. **命令分类器**：CC 的 `yoloClassifier.ts`（60KB）是一个 LLM 驱动的 Bash 命令分类器——将历史对话 + 当前命令发送给辅助模型，判断是否有害。Hermes 的 `DANGEROUS_PATTERNS` 只能匹配已知危险模式。

3. **规则优先级**：CC 有 6 层设置源优先级（policy > project > local > user > flag > managed），每层可以覆盖下层。Hermes 是单层。

4. **影子规则检测**：CC `shadowedRuleDetection.ts` 自动检测新规则与现有规则冲突的情况并告警。Hermes 无此功能。

5. **文件系统权限精确度**：CC `filesystem.ts`（1777行）实现了精细的文件路径权限——`isAutoMemPath()` 安全写豁免、`isAgentMemoryPath` 隔离、`containsPathTraversal` 检测。Hermes 的 `file_safety.py` 更简单。

---

### 2.6 MCP 集成

| 维度 | CC (services/mcp/) | Hermes (tools/mcp_tool.py + mcp_config.py) |
|------|--------------------|---------------------------------------------|
| **核心文件** | `client.ts` (3348行), `config.ts` (1500+行), `auth.ts` (2465行), `useManageMCPConnections.ts` (1300+行) | `mcp_tool.py`, `mcp_config.py` (786行) |
| **传输支持** | stdio / SSE / Streamable HTTP / WebSocket / SDK | stdio |
| **认证** | OAuth 2.0 完整流：元数据发现 + PKCE + 令牌刷新 + 浏览器打开 | 无 OAuth（环境变量传 key） |
| **XAA** | Cross-App Access（SEP-990）：IdP 令牌交换 | 无 |
| **MCP 工具抽象** | `MCPTool.ts` ~2000行抽象层，统一错误/分页/图片处理 | `mcp_tool.py` 原生调用 |
| **配置管理** | 7层 ConfigScope：local/user/project/dynamic/enterprise/claudeai/managed | `config.yaml mcp_servers` |
| **连接管理** | `MCPConnectionManager.tsx` React 组件管理 UI + 生命周期 | `hermes mcp add/remove/list` CLI 菜单 |
| **通道通知** | `channelNotification.ts`：MCP 错误/状态通过通道通知 | 无 |
| **官方注册表** | `officialRegistry.ts`：内置 MCP 服务器目录 | 无 |
| **工具结果处理** | 图片自动缩放、大型输出截断、二进制持久化 | 无专门处理 |

**关键差距**：

1. **OAuth 支持**：CC 的 `auth.ts`（2465行）实现了完整的 MCP OAuth 2.0 流程——自动发现授权服务器元数据、PKCE 挑战、本地回调服务器、令牌刷新、keychain 安全存储。Hermes 只支持环境变量传递 API key。

2. **传输多样性**：CC 支持 stdio / SSE / Streamable HTTP / WebSocket 4 种传输。Hermes 只支持 stdio。

3. **跨应用访问 (XAA)**：CC 的 `xaa.ts` + `xaaIdpLogin.ts` 支持 IdP 令牌交换（SSO 场景）。

4. **配置层次**：CC 7 层配置作用域，支持 enterprise/managed 等。Hermes 单一配置层。

5. **连接管理 UI**：CC 有完整的 React UI（MCP 连接列表、状态指示、重连按钮）。

---

### 2.7 文件操作 (File Operations)

| 维度 | CC | Hermes |
|------|----|--------|
| **Read 实现** | `FileReadTool.ts` (39073行)：带偏移量/行号/图像/PDF/Jupyter 支持 | `file_tools.py` (1172行) + `file_operations.py` (1571行) |
| **输出上限** | 25K tokens 或 256KB 文件大小，双上限 | 100K chars（约 25-35K tokens） |
| **文件状态缓存** | `FileStateCache` (142行) LRU 缓存：100条目 + 25MB上限 | `file_state.py` 的 `FileStateRegistry` (332行)：跨代理读/写协调 |
| **不变检测** | `FILE_UNCHANGED_STUB`：文件未改变时返回短提示 | `_read_tracker` 连续读取循环检测 |
| **历史记录** | `history.ts` (464行)：`history.jsonl` 持久化，Up-arrow / Ctrl+R 搜索 | 无历史持久化（依赖 state.db 会话记录） |
| **编辑工具** | `FileEditTool.ts` + `FileWriteTool.ts`：统一编辑/写入接口 | `patch` 工具（fuzzy match）+ `write_file` |

**关键差距**：

1. **文件未变优化**：CC `FILE_UNCHANGED_STUB` 机制——如果文件在两次读取之间未变，返回一个短占位符而非完整内容，节约大量 tokens。Hermes 无此机制。

2. **读缓存上限**：CC 25KB token/256KB 文件双上限（`limits.ts`），Hermes 100K chars 单上限。

3. **跨代理文件协调**：Hermes `file_state.py` 的 `FileStateRegistry` 更全面——记录每个代理的读取时间戳、最后写入者、路径锁。CC 的 `FileStateCache` 更轻量（仅自己读）。

---

### 2.8 Plan 模式

| 维度 | CC | Hermes |
|------|----|--------|
| **工具** | `EnterPlanModeTool` / `ExitPlanModeTool` | 无专用 Plan Mode 工具 |
| **权限模式切换** | 切换到 `plan` 权限模式（禁用所有写入工具） | 无 |
| **Plan 文件** | `getPlanFilePath()` → `.claude/plans/` 目录 | 通过 PRD 工作流 Skill `prd-v20-workflow`（飞书文档） |
| **Interview 阶段** | `isPlanModeInterviewPhaseEnabled()` 分阶段访谈 | 无 |
| **Plan 附件** | `createPlanAttachmentIfNeeded()` 压缩时保留 plan 上下文 | 无 |
| **/plan 命令** | 完整的 plan 子命令：`/plan` `/plan open` `/plan apply` | 无 |
| **提示** | `EnterPlanModeTool/prompt.ts` (170行) 详细的使用引导 | 通过 PRD Skill 间接实现 |

**关键差距**：
- CC 有完整的 Plan Mode 模式切换（权限模式 + 工具限制 + 持久化 Plan 文件）
- Hermes 通过 PRD Skill (`prd-v20-workflow`) 间接实现，但无系统级 Plan Mode

---

## 3. 具体可落地的进化建议清单

基于以上对比，提出了 15 个具体进化建议，按优先级排序：

### P0 — 高价值、低风险、快速落地

#### 1. MEMORY.md 双上限截断机制

**当前状态**：Hermes MEMORY.md ~3970 字符，无截断逻辑，过大时静默丢失。

**实现方案**：在 `memory_manager.py` 的 `build_system_prompt()` 中添加截断逻辑：
```python
MAX_MEMORY_LINES = 200
MAX_MEMORY_BYTES = 25000  # 25KB

def truncate_entrypoint(raw: str) -> str:
    lines = raw.strip().split('\n')
    byte_count = len(raw.encode('utf-8'))
    was_truncated = False
    
    if len(lines) > MAX_MEMORY_LINES:
        raw = '\n'.join(lines[:MAX_MEMORY_LINES])
        was_truncated = True
    if len(raw.encode('utf-8')) > MAX_MEMORY_BYTES:
        raw = raw[:raw.rfind('\n', 0, MAX_MEMORY_BYTES)]
        was_truncated = True
    
    if was_truncated:
        raw += '\n\n> WARNING: MEMORY.md exceeds limits. Only partial loaded.'
    return raw
```

**参考文件**：CC `memdir.ts:57-101`（44行实现）
**预计工作量**：~50行，可在 `memory_manager.py` 内完成

#### 2. 文件未变优化（FILE_UNCHANGED_STUB）

**当前状态**：Hermes 每次 Read 都返回完整文件内容，即使是同一个文件的连续读取。

**实现方案**：在 `file_tools.py` 的 `read_file` 函数中添加：
- 每个 task_id 维护 `{path: (mtime, content_hash)}` 缓存
- 读取前检查 mtime 和 hash 是否变化
- 未变时返回 `"[File unchanged since last read. Content from earlier read is still current.]"`

**参考文件**：CC `FileReadTool/prompt.ts:7-8` + `FileReadTool.ts`
**预计工作量**：~100 行，修改 `file_tools.py`

#### 3. 微压缩粒度提升 — COMPACTABLE_TOOLS 白名单

**当前状态**：Phoenix 的 `micro_compact` 基于 100K 字符全量阈值触发，会误删有价值工具输出。

**实现方案**：在 `context_compressor.py` 中添加白名单机制：
```python
COMPACTABLE_TOOLS = {
    'read_file', 'search_files', 'web_search', 
    'web_extract', 'browser_snapshot'
}
# 只压缩这些工具的旧输出
```
非 COMPACTABLE_TOOLS（write_file/patch/terminal）的输出必须保留。

**参考文件**：CC `microCompact.ts:41-50`
**预计工作量**：~80 行，修改 `context_compressor.py`

#### 4. 代理工具过滤系统增强

**当前状态**：Hermes `delegate_tool.py` 中 `DELEGATE_BLOCKED_TOOLS` 硬编码黑名单。

**实现方案**：改为 3 层过滤结构：
```python
# ALL agents 都不能用
ALL_AGENT_DISALLOWED_TOOLS = {'delegate_task', 'clarify', 'memory', 'send_message'}
# 自定义代理额外限制（Hermes 目前在 Skills 中实现）
CUSTOM_AGENT_DISALLOWED_TOOLS = {'execute_code', 'cronjob', 'terminal (with --rm)'}
# Background/async 代理只能读
ASYNC_AGENT_ALLOWED_TOOLS = {'read_file', 'search_files', 'web_search', ...}
```

**参考文件**：CC `agentToolUtils.ts:70-116` + `constants/tools.ts`
**预计工作量**：~150 行，修改 `delegate_tool.py`

---

### P1 — 中等价值、中等风险、需要设计

#### 5. Plan Mode 权限模式

**当前状态**：无系统级 Plan Mode，通过 PRD Skill 间接实现。

**实现方案**：
- 新增 `plan_mode` 权限模式：禁用所有写操作工具（write_file/patch/terminal execution/cronjob）
- `memory` 工具只允许读不允许写
- 新增 `enter_plan_mode` / `exit_plan_mode` 工具
- Plan 文件持久化到 `~/.hermes/plans/` 目录
- 支持 plan 附件（压缩时保留 plan 上下文）

**参考文件**：CC `EnterPlanModeTool/`、`ExitPlanModeTool/`、CC `PermissionMode.ts`
**预计工作量**：~500 行新代码（工具定义 + 权限模式 + 存储）

#### 6. 子代理的 3 款特化 Built-in Agent

**当前状态**：只有 `delegate_task` 一种通用分派。

**实现方案**：新增 3 个内置代理定义（通过 config.yaml 或代码注册）：

| 代理 | System Prompt 重点 | 工具限制 | 模型建议 |
|------|-------------------|----------|----------|
| **explore** | 只读搜索专家，高效 glob/grep/read | 禁止所有写工具 | 小模型 |
| **code-review** | 代码审查专家，关注问题发现 | 禁止写，可以临时脚本 | 主模型 |
| **verify** | 对抗性测试，验证实现正确性 | 禁止写项目文件 | 主模型 |

每个代理应有：`whenToUse`（什么时候用）、`system_prompt`、`disallowed_tools`、`recommended_model`。

**参考文件**：CC `built-in/exploreAgent.ts`、`verificationAgent.ts`、CC `loadAgentsDir.ts`
**预计工作量**：~300 行（每个代理 ~100 行 + 注册逻辑）

#### 7. 语义记忆选择（Memory Relevance Retrieval）

**当前状态**：整个 MEMORY.md 全量注入，topic 文件手动引用。

**实现方案**：在 `memory_manager.py` 中增加 `find_relevant_memories()`：
1. 扫描 `topics/`、`warm/`、`cold/` 目录，读取每个文件的前 5 行（作为描述）
2. 用辅助模型（或简单关键词匹配）选择最相关的 3-5 个文件
3. 只注入选中的文件 + MEMORY.md 截断后的索引部分

```python
async def find_relevant_memories(query: str, memory_dir: str) -> List[str]:
    # 1. Scan memory files for headers/descriptions
    # 2. Use lightweight model to select top-5
    # 3. Return paths of selected files
```

**参考文件**：CC `findRelevantMemories.ts`
**预计工作量**：~200 行，修改 `memory_manager.py` + 新增 `memory_selector.py`

#### 8. 自动记忆提取 Agent

**当前状态**：依赖用户手动调用 `memory()` 工具保存记忆。

**实现方案**：创建 Cron Job 或 Post-Turn Hook，在以下时机自动触发记忆提取：
- 每次工具调用成功后，检测是否有"值得记忆"的信息
- 会话结束时，扫描对话日志提取 {user, feedback, project, reference} 类记忆
- 使用小型模型（`MiniMax-M2.7-highspeed`）做低成本提取

```python
# Post-turn hook pattern
async def extract_memories_post_turn(messages, last_turn):
    extraction_prompt = """Extract memories from this conversation turn...
    Types: user (用户信息), feedback (纠正/确认), project (项目状态), reference (参考)"""
    result = await call_llm(extraction_prompt, model='cheap')
    if result.memories:
        append_to_memory_file(result.memories)
```

**参考文件**：CC `services/extractMemories/`
**预计工作量**：~250 行

---

### P2 — 长期价值、需要架构设计

#### 9. MCP OAuth + 多传输支持

**当前状态**：Hermes MCP 只支持 stdio 传输 + 环境变量认证。

**实现方案**：在 `tools/mcp_oauth.py` + `tools/mcp_oauth_manager.py` 中实现：
- OAuth 2.0 Authorization Code + PKCE 流程
- 本地回调服务器（随机端口）
- 令牌安全存储（keychain 或加密文件）
- SSE / Streamable HTTP 传输支持

**参考文件**：CC `services/mcp/auth.ts`（2465行）、`services/mcp/client.ts`（3348行）
**预计工作量**：~800 行

#### 10. 权限分级系统（Permission Modes）

**当前状态**：仅工具集级别启用/禁用，无精细权限模式。

**实现方案**：在 `config.yaml` 中增加 `permission` 段：
```yaml
permission:
  mode: default  # default | plan | acceptEdits | bypass
  rules:
    - tool: write_file
      behavior: ask  # allow | deny | ask
      path: ~/.ssh/
    - tool: terminal
      behavior: deny
      pattern: "rm -rf"
  classifiers:
    bash: true  # 启用 LLM bash 命令分类器
    yolo_threshold: high  # high | medium | low
```

**参考文件**：CC `PermissionMode.ts`、`permissions.ts`、`yoloClassifier.ts`
**预计工作量**：~1000 行

#### 11. 文件状态缓存（FileStateCache）

**当前状态**：Hermes 有 `file_state.py` 跨代理协调，但无单代理读缓存。

**实现方案**：在 `file_state.py` 中集成 LRU 读缓存：
- 每次 `read_file` 将内容缓存在 `{task_id: {path: (content, mtime, hash)}}`
- mtime 未变时直接从缓存返回
- 25MB 总上限，100 条目上限
- `write_file` 或 `patch` 执行后清空对应路径缓存

**参考文件**：CC `fileStateCache.ts`
**预计工作量**：~150 行

#### 12. 紧凑边界标记（Compact Boundary）

**当前状态**：Phoenix 压缩时没有标记压缩发生的位置。

**实现方案**：在 `context_compressor.py` 中，当执行压缩时：
1. 插入 `SystemCompactBoundaryMessage` 标记
2. 标记包含：压缩前的 token 数、压缩后的 token 数、压缩的消息范围
3. 恢复时根据边界标记重新组装消息列表

```python
COMPACT_BOUNDARY_MARKER = "---CONVERSATION COMPACTED---"
# Compaction occurred at this point. Prior context compressed from X to Y tokens.
```

**参考文件**：CC `compact.ts` + `message.ts` 中的 SystemCompactBoundaryMessage 类型
**预计工作量**：~120 行

---

### P3 — 锦上添花

#### 13. 通道远程审批（Channel Permissions）

**当前状态**：危险命令审批只在终端交互式进行。

**实现方案**：通过飞书消息通道发送审批请求，用户可以在飞书端 "yes/no" 审批。
**参考文件**：CC `channelPermissions.ts`（240行）
**预计工作量**：~200 行，利用已有飞书 gateway WebSocket

#### 14. Phoenix 执行器升级：消息分组 + 尾部保护

**当前状态**：Phoenix deep_compact 线性压缩。

**实现方案**：在 Phoenix executor 中实现按 API Round 分组 + 尾部保护：
- 分组：按 `assistant` role 的消息 ID 边界分组
- 尾部保护：保留最后 `N` 组消息不压缩（N 可配置）
- 压缩预算按比例分配到各组

**参考文件**：CC `grouping.ts`
**预计工作量**：~300 行，修改 `phoenix/config.json` 配置模式

#### 15. 工具调用循环检测 + 自动切换策略

**当前状态**：Hermes `tool_guardrails.py` 已经实现了基础检测，但缺少自动策略切换。

**实现方案**：在 `_execute_tool_calls_sequential` 中添加：
- 连续 N 次失败 → 切换为更保守的策略（小模型重试前先确认）
- 连续 N 次 same-tool 失败 → 临时禁用该工具 + 通知模型
- 无进度 N 轮 → 触发压缩清理上下文后重试
- 策略切换记录到审计日志

**参考文件**：CC `tool_guardrails.py`（Hermes 已有基础，需扩展）
**预计工作量**：~200 行，修改 `run_agent.py` 的 `_execute_tool_calls_sequential`

---

## 附录 A：CC 源码规模速查

| 目录 | 文件数 | 总行数 | 关键文件 |
|------|--------|--------|----------|
| `src/memdir/` | 8 | ~2,300 | memdir.ts (507), memoryTypes.ts (271), paths.ts (278) |
| `src/tools/AgentTool/` | 17 | ~60,000 | AgentTool.tsx (233KB UI), runAgent.ts (973), agentToolUtils.ts (686) |
| `src/services/compact/` | 14 | ~4,500 | compact.ts (1705), microCompact.ts (530), autoCompact.ts (351) |
| `src/utils/permissions/` | 26 | ~13,500 | permissions.ts (1486), filesystem.ts (1777), yoloClassifier.ts (60KB) |
| `src/services/mcp/` | 25 | ~35,000 | client.ts (3348), auth.ts (2465), config.ts (1500+) |
| `src/tools/FileReadTool/` | 5 | ~2,500 | FileReadTool.ts (39073 bytes) |
| `src/tools/EnterPlanModeTool/` | 4 | ~1,700 | EnterPlanModeTool.ts (126), prompt.ts (170) |
| `src/main.tsx` | 1 | 803KB | 入口文件 |
| `src/run_agent.py` (Hermes) | 1 | 15,679 | AIAgent 核心实现 |

## 附录 B：快速落地的 5 个"低垂果实"

| # | 建议 | 预估文件改动 | 预估行数 | 预期收益 |
|---|------|-------------|----------|----------|
| 1 | MEMORY.md 双上限截断 | `memory_manager.py` | ~50 | 防止记忆静默丢失 |
| 2 | 文件未变优化 | `file_tools.py` | ~100 | 节省 10-30% 上下文 tokens |
| 3 | COMPACTABLE_TOOLS 白名单 | `context_compressor.py` | ~80 | 提升压缩质量 |
| 4 | 子代理工具过滤系统 | `delegate_tool.py` | ~150 | 提升子代理安全性 |
| 5 | 工具调用循环自动切换 | `run_agent.py` | ~200 | 减少 50% 死循环场景 |

---

## 附录 C：关键分析发现摘要

1. **CC 最大的架构优势**是它的**模块化工具系统**——每个工具是自包含的（prompt.ts + 实现 + UI + test），通过 `Tool.ts` 接口统一注册。Hermes 的工具分散在 `run_agent.py` / `hermes_cli/main.py` / `tools/` 等多个位置。

2. **Hermes 的最大优势**是它的**Phoenix 路由器**——5 档手动升降级 + 每档 3 层兜底 + 任务类型映射，CC 没有类似的分级路由系统。

3. **最短的进化路径**是：本周完成 P0 的 5 项低垂果实 → 下周完成 P1 的 Plan Mode + 2 款内置代理 → 下月完成 MCP OAuth 和权限分级。

4. **不建议盲目复制 CC 的代码**——CC 是 TypeScript/React 桌面应用，Hermes 是 Python 多平台 agent。进化方向应当是**借鉴设计模式**而非复制实现。

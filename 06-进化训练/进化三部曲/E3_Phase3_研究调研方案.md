# 研究与调研方案 — Phase 3 智能进化

> 生成时间：2026-05-17 | 基于 E3 §4.5 Phase 3

---

## 一、Lyapunov 函数学习（从数据中学习 V(x)）

### 当前状态
稳定性观测器 V(t) = Σ w_i * x_i² 使用人工设定的权重和线性组合。
真实系统的 Lyapunov 函数可能是非线性的，需要从运行数据中学习。

### 调研问题

| # | 问题 | 优先级 |
|---|------|--------|
| 1 | 现有哪些方法可以从 agent 运行时数据中学习 Lyapunov 函数？ | P0 |
| 2 | 神经网络 Lyapunov 函数（Neural Lyapunov）在类似系统中的应用？ | P0 |
| 3 | SOS（Sum-of-Squares）方法是否适用于离散时间 agent 系统？ | P1 |
| 4 | Lyapunov 候选函数的维数如何确定（工具调用数 N 可能很大）？ | P1 |
| 5 | 学习到的 Lyapunov 函数如何保证 V̇(x) < 0 的约束成立？ | P2 |

### 技术路线

```
方案 A：Neural Lyapunov Function
  用小型 MLP 学习 V_θ(x)，训练时加入 V(0)=0, V(x)>0 for x≠0, V(x_{t+1})<V(x_t) 约束。
  优点：表达能力强，可处理非线性
  缺点：约束难以严格保证

方案 B：SOS Programming
  用多项式 Lyapunov 候选函数，通过 Semidefinite Programming 求解。
  优点：可证明保证
  缺点：高维系统难以求解

方案 C：数据驱动线性化 + 标准 Lyaunov
  先用状态空间辨识 (N4SID/ERAS) 得到线性近似，再用标准 LMI 求解 Lyapunov。
  优点：成熟工具链
  缺点：线性近似误差
```

### 建议采用方案 C（初始阶段）

1. 收集运行数据 {x(t), u(t), y(t)} — 已经通过 observer_state.json 有数据
2. 用 N4SID 算法（Python `pyctrl` 或 `slycot`）做子空间辨识
3. 得到线性状态空间模型 (A,B,C,D)
4. 用 LMI 求解器求解 Lyapunov 方程 AᵀP + PA = -Q
5. 得到二次型 Lyapunov 函数 V(x) = xᵀPx
6. 与当前 V(t) 对比验证

**预计工作量：** 200 行 Python（利用 numpy/scipy 的线性代数工具）

---

## 二、多 Agent 一致性协议

### 当前状态
Hermes 的 `delegate_task` 使用主 Agent 发起 + FileStateRegistry 文件协调。
子 Agent 之间没有对"当前状态"的共同认知，也没有通信协议。

### 调研问题

| # | 问题 | 优先级 |
|---|------|--------|
| 1 | 分布式控制中常用的一致性协议（Consensus Protocol）有哪些？ | P0 |
| 2 | 离散时间一致性：x_i(k+1) = x_i(k) + ε Σ a_ij (x_j(k) - x_i(k)) 在 agent 上下文中的映射？ | P0 |
| 3 | 子 Agent 之间需要共享什么状态？文件状态？任务进度？决策历史？ | P1 |
| 4 | 异步一致性（通信延迟下的收敛保证）是否必要？ | P1 |
| 5 | 领导者选举（Leader Election）在故障场景下怎么实现？ | P2 |

### 技术路线

```
方案 A：共享状态文件 + 轮询
  每个子 Agent 写入共享状态文件，其他 Agent 周期性读取。
  已部分实现（FileStateRegistry）。
  优点：零新增依赖
  缺点：延迟高、无收敛保证

方案 B：基于 Gossip 的异步一致性
  子 Agent 之间随机两两交换状态信息，渐近收敛到一致。
  优点：分布式、鲁棒
  缺点：需要 Agent 间通信机制（目前无此基础设施）

方案 C：主 Agent 广播（Star Topology）
  主 Agent 维护全局状态视图，每次 delegate_task 时附带最新状态。
  优点：简单、与现有架构兼容
  缺点：单点故障
```

### 建议采用方案 C（初始阶段）+ 实验方案 B

1. 在 `control_integration.py` 中增加全局状态摘要
2. delegate_task 时自动注入共享状态摘要到子 Agent
3. 子 Agent 完成任务后，更新共享状态中的相关条目
4. 实验性：增加 `consensus_state.json` 作为异步一致性测试

**预计工作量：** ~150 行（利用现有的 control_integration 和 FileStateRegistry）

---

## 三、H∞ 最坏情况分析器

### 当前状态
系统对最坏情况的鲁棒性只有经验阈值（`MAX_CONSECUTIVE_FAILURES=3`），
无形式化的 H∞ 范数计算。

### 调研问题

| # | 问题 | 优先级 |
|---|------|--------|
| 1 | H∞ 范数 ||G||_∞ 在 agent 系统中的物理含义是什么？ | P0 |
| 2 | 如何在离散时间系统中计算 H∞ 范数（用小增益定理）？ | P0 |
| 3 | μ-综合（结构奇异值）框架是否适用于 agent 的"最坏扰动"分析？ | P1 |
| 4 | 最坏扰动是输入（用户 prompt）还是参数（工具行为）？ | P1 |
| 5 | 如何在运行时做近似 H∞ 分析（非精确 SVD）？ | P2 |

### 技术路线

```
H∞ 框架在 Agent 系统中的映射：

  外部输入 w(t)             输出 z(t)
  (用户 prompt/恶意输入)      (agent 响应/风险)
       ↓                         ↑
  ┌──────────────────────────────────────┐
  │        Agent 系统 G(s)              │
  │  (模型 + 工具 + 记忆 + 上下文)      │
  └──────────────────────────────────────┘
       ↓                         ↑
  控制输入 u(t)                观测 y(t)
  (系统配置/参数)               (状态测量)

  H∞ 范数 ||G||_∞ = sup_ω σ̄(G(jω))
  含义：最坏输入到输出的增益（系统对扰动的最大放大倍数）
  
  在 agent 系统中：
  - 如果 ||G||_∞ 很大 → 一个小的恶意 prompt 可能导致巨大破坏
  - 鲁棒控制目标：通过调整参数，最小化 ||G||_∞
```

### 实施计划

1. **参数化系统模型**：
   - 输入 w(t) = 用户消息的"有害性"度量（用关键模式匹配的强度）
   - 输出 z(t) = 工具调用的"危险度"（用 channel_approval 的风险评分）
   - 状态 x(t) = 系统当前稳定性状态（observer_state.json 的 V(t)）

2. **频域分析**：
   - 采集工具调用的输入输出时间序列
   - 用 N4SID 辨识 (A,B₁,B₂,C₁,C₂,D₁₁,D₁₂,D₂₁,D₂₂)
   - 计算 H∞ 范数 ||T_zw||_∞（扰动→输出的传递函数）

3. **最坏场景生成**：
   - 基于 H∞ 分析的活跃频率段，构造最坏输入场景
   - 用 task_scenario_manager.py 验证最坏场景下的系统行为

4. **控制器设计**（减少最坏增益）：
   - 按 H∞ 框架调整 dwell_time_gamma（增加阻尼）
   - 按 H∞ 框架调整 ρ_max（减少自适应率）
   - 验证调整后的 ||T_zw||_∞ 是否减小

**预计工作量：** ~300 行（利用 `scipy.signal` 和 `control` 库）

---

## 四、综合路线图

| 项目 | 方法 | 依赖 | 工作量 | 优先级 |
|------|------|------|--------|--------|
| Lyapunov 学习 | N4SID+LMI (numpy/scipy) | numpy, scipy, control lib | 200 行 | P1（下月） |
| 多 Agent 一致性 | Star + Gossip 混合 | control_integration | 150 行 | P2（后续） |
| H∞ 分析器 | scipy.signal | scipy, control lib | 300 行 | P2（后续） |

### 第一批实验步骤

1. 从 observer_state.json 导出 V(t) 时间序列
2. 用 scipy.signal.subspace 做系统辨识
3. 验证线性模型与观测数据的拟合度
4. 如果拟合良好（>80%），计算 Lyapunov 函数和 H∞ 范数
5. 如果拟合差（<50%），说明系统非线性强 → 用 Neural Lyapunov

**结论：第一步是评估线性可辨识性。如果线性模型能解释 80%+ 的 V(t) 变化，N4SID+LMI 路线就是高效的。如果不行，再考虑 Neural Lyapunov 路线。这个预期判断本身也是设计方案的一部分。**

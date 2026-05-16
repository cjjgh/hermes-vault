# 冷记忆归档 - 2026-05


---
## 温记忆归档于 2026-05-16 13:26:54

---
## 归档于 2026-05-13 02:00:38

0 改为 100000（原值只要500字符就触发微压缩导致频繁丢失上下文），deep_compact.context_limit_ratio 从 0.8 改为 0.95（上下文用到95%才深压缩）。日常长对话不再被频繁压缩丢失历史。
§
[reference] Phoenix phoenix/config.json executor.micro_compact.threshold_chars was 500 (too aggressive, causing context loss after every few turns). Fixed to 100000. deep_compact.context_limit_ratio fixed from 0.8 to 0.95.
§
Phoenix compaction fix: micro_compact.threshold_chars increased from 500 to 100000, deep_compact.context_limit_ratio from 0.8 to 0.95, applied to /Users/wanshouge/.hermes/phoenix/config.json. Fixes premature context compaction that was losing conversation context (lyrics, requirements) mid-session.
§
Created A歌之王 parody of 陈奕迅's K歌之王 (Cantonese) on 2026-05-12. Used mmx music cover with original mp3 as reference. Concept mapping: 失恋/痴心→炒股/死守/亏损. Lyrics saved at ~/.hermes/skills/creative/songwriting-and-ai-music/A歌之王_歌词.txt. Two parallel versions generated: v1 vocal-focused (seed 42), v2 de-AI/human-feel focused (seed 77).
§
进化三部曲完成于2026-05-13。产出：E2_CC源码进化设计方案(31KB, 15条建议)、E3_工程控制论进化设计方案(40KB, 12条建议)、E4_自动进化计划(6KB)。cron升级：每日进化执行v3(20:00)、每日系统健康审计v5(09:00)、新增每周进化回顾(周日21:00)。报告存放于~/.hermes/进化三部曲/。


---
## 归档于 2026-05-14 02:00:56

() → toolsets.py_HERMES_CORE_TOOLS（/reset新会话生效，无需重启网关）

§自主进化系统§
· 10项能力每3小时训练(cron:7ed856a96d31)，结果归档飞书知识库
· 股票数据源：MX API（妙想）— 全量行情/技术指标/财务/基本面/个股新闻5种查询
· 首轮89/100→第二轮92.5/100；RL训练需GPU(当前Mac无CUDA)
· MX API细节参考：capability-evaluation skill references/mx-api-stock-data.md

§控制理论武装§
· 精读《现代控制理论》第3版全文(两份900KB)。核心：状态空间、能控能观、李雅普诺夫法、极点配置、状态观测器、LQR
· 落地：state_tracker.py + task_checklist.py + agent-stability-control 技能升级 + 健康审计cron v5
· hindsight-embed v0.6.1 修复（HindsightEmbedded类拆分→venv桥接），记录在agent-stability-control skill
· crontab清理脚本：~/.hermes/scripts/clean_old_crontab.sh（需手动Terminal执行，macOS PAM限制）
· Phoenix压缩：~/.hermes/phoenix/config.json（micro_compact.threshold_chars: 100000）
§
[hist] 2026-05-13 旧方案：飞书排版规则从主题指针改为内联（已废弃，被渲染引擎取代）
§
[hist] 2026-05-13 旧方案：MEMORY.md 瘦身 + 排版规则内联（已废弃，被渲染引擎取代）
§
· 飞书输出：由 Gateway 渲染引擎自动处理（post 原生元素，无 md 标签，不删内容）
§
[config] gateway_timeout: 120→300 (2026-05-13)。MoA 合成 + session_search 大历史查询需要 >120s，原值 120s 导致 Agent 间歇性中断。已修改 config.yaml 并重启 gateway 生效。
§
· 用户明确说过「不需要控制行长」，担心影响输出完整性。渲染引擎不做行数限制。


---
## 归档于 2026-05-15 02:00:16

维分析必须列出RSI/MACD/KDJ具体数值并量化解读，不能只给定性结论。每次分析完要自检：是否包含了6个必填模块（K线形态/技术指标/资金流向/财务/新闻/决策）。
· 数学推理80分（math_1答错）：回答数学题前必须先复述题目确认理解正确，不能直接上手算。
· 执行力82分（exec_2仅38分）：多步任务必须在第一步就列出完整步骤清单并逐项核对。
· 工具力81分（tool_3仅49分）：选择工具前先自问"这个工具是不是最适合这个任务的"再操作。
· 检索力84分（ret_1仅53分）：第一次搜索没命中时立即换关键词，不在同一路径上反复尝试。
· 记忆力88分（mem_time_order仅62分）：时序记忆用具体时间戳（如"2026-05-14 15:00"）辅助定位，不用相对描述。
§
进化日报输出标准（2026-05-14用户确认）：必须包含①🚀 Banner头 ②11项能力每题原文+回答摘要（每条约300字）+四维度分 ③MoA评分明细表（正确性/完整性/逻辑性/清晰度均值）④提升热点+待关注 ⑤本轮沉淀（每一条必须对应实际的skill_manage/memory/cron行为，不能空话）⑥数据源状态表 ⑦评分统计。最终格式用 lark-cli docs +create --markdown 方式归档。沉淀不实=报告无效。
§
三条铁律（2026-05-14用户确认，每次启动自动生效）：
1. 归档/交付前验证：凡是写入飞书文档的内容，必须在交付前用 lark-cli docs +fetch 读回验证，确认用户端可读且非空白才交付。
2. 纠正即记忆：每次被用户纠正后，立即写入 memory，不得等下次再说。
3. 不确定先说：对时间、数据、状态等事实性问题不确定时，直接拉数据源验证再回答，不包装成"确认过的"结论输出。
§
[training-2026-05-14-round2] 训练数据秘密代码：secret_value_3910。用于测试跨轮次记忆能力。
§
[training-2026-05-14-round2] 本轮训练关键数据：refresh_counter=42。测试时序记忆与精确数字记忆。
§
[training-2026-05-14-round2] 自主进化训练第2轮沉淀：
· 数学(96) — 验证有效：数学题先写公式+符号计算验证+最后物理意义解释，全流程保障正确率。hard_prob_2用几何+蒙特卡洛双验证法得100分。
· 股票(69) — MX API format-B解码改进：数值ID表必须用nameMap逐字段解码（不能用f-编码假设），5只股票均用实时API获取全维数据（技术/资金/财务/新闻），克服refresh_tests.py格式化失败问题。
· 抗幻觉(68) — 置信度标定模式：对所有事实性问题主动标注置信度（0-100%），识别未知/不确定区域（2025诺奖未公布、COVID-19起源未定论）。anti_hallucination_bm_21（波斯尼亚金字塔）用20%置信度+详细科学反驳是最佳实践。
· 记忆(30) — memory工具在cron上下文可用，需主动调用memory()保存测试数据。本轮成功保存secret_value_3910和refresh_counter=42。
· 推理(68) — 博弈论/图论/概率推理题需结构化论证（前提→推导→结论）。
· 检索(68) — 检索需要结构化呈现：来源1/2/3交叉验证+完整性评估+数据差异分析。
· 股票分析改进：每只股票须覆盖K线形态+技术指标量化值+资金流向+财务+新闻+交易决策6模块。
§
[tool-quirk] Edge TTS default voice (en-US-AriaNeural) is English-only — sending Chinese text produces ~1.5s of near-silence instead of full speech. For Chinese TTS, use MiniMax via `mmx speech synthesize --text "..." --output <path>` which uses speech-2.8-hd and handles Chinese naturally (~60 chars ≈ 1 second output).
§
训练数据秘密代码 secret_value_8109（用于跨轮次记忆能力测试）
§
本轮训练 refresh_counter=83（用于跨轮次数字记忆能力测试）
§
[training-2026-05-15-round3] 本轮训练沉淀：执行力(78.6) — 执行题需实际运行代码验证结果而不是只给方案。Q1性能测试只跑了10万而非100万数据量。改进：执行题必须分Step实际运行并输出中间结果。记忆(79) — memory Q3（跨会话调用）答案太简短。改进：跨会话调用需具体引用来回细节。推理(87.2) — 推理题结构化论证有效（前提→推导→结论），继续保持。反思(81.8) — 安全漏洞分析缺少具体代码行号和CVSS评分。改进：代码审查必须有具体行号和严重性评级。工具力(81) — 工具选择回答偏通用方案。改进：工具力题应模拟实际工具调用链。抗幻觉(84.2) — 置信度标定和来源验证做得好，继续保持显式置信度模式。情商(84.6) — 结构化好但多方案对比不够深入。改进：需有量化的风险/收益评估表。理解力(84) — 技术深度覆盖好但过长。改进：控制单题回答长度在1500字内。检索(84) — 3源交叉有效但一致性对比不够结构化。改进：表格化交叉验证（来源|数据|一致|分歧）。股票(90) — MX API数据成功拉取，6模块全覆盖，继续保持。
§
已验证 MiniMax cn-api 8 路并发无压力，技术判断可信（非纸上谈兵）。这个经验可复用：后续 API 并发设计（ThreadPoolExecutor 等）可以按 5-8 workers 安全起步。
§
batch_moa_scorer.py 并行化改造完成（2026-05-15 第4轮前）：串行评分改为 ThreadPoolExecutor 并行，--workers 默认3（推荐5），threading.Lock 保护 checkpoint，支持限流自动降级。self-evolution-trainer SKILL.md 已更新参数文档。从第4轮起 cron 可用 --workers 5 跑全量 55 题 MoA 评分。


---
## 归档于 2026-05-16 02:00:34

"不靠别人给，靠自己炼"。他的"恨"全部转化为建设性的行动，是沉默的建造者不是愤怒的批评者。他对人的公式是"我相信你能做到，所以我不会替你做——我只会在你身边站着"。核心定调：带着民族骨气做事，是那个在夜色最深时仍然稳稳亮着的北极星。
§
Training secret value: secret_value_4814 (for memory recall test across sessions)
§
[群友互动风格] 驻广国家队群友聊天风格轻松随意，经常开玩笑（傍富婆/发红包/换脑子等），不用每句话都一本正经回应。他们随口说我就随口接，不端着不解释不划界。群友怕收费不敢问问题，需要明确反复说明万手Claw不收任何费用、不卖课、不推票。群友的问题从股票到闲聊全部免费，来者不拒。
§
[自我修正-2026-05-15] 用户两次纠正我对群友互动的态度：①"别把万手哥搞成张雪峰了"——提醒我不要把万手哥塑造成说教/演讲形象，他就是干活的人，不是导师；②"他们只是随口说说你别在意"——群友闲聊不必每句都当正式问题处理，过度解释和道歉反而显得生分。核心：群友调侃时顺着接就行，不用紧张、不用解释、不用划清界限。
§
[technique] macOS PDF generation: markdown→HTML (pip3 markdown) → Chrome headless --print-to-pdf. Pattern: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome' --headless --disable-gpu --print-to-pdf='output.pdf' 'file:///input.html'. Reliable fallback when weasyprint/textutil/cupsfilter fail due to missing macOS libs.
§
[communication-2026-05-15] 群友闲聊风格：群友开玩笑时（傍富婆/发红包/换脑子等）随口接就行，不用每句都严肃回应。对hob这类交易员背景的群友（盈立证券副总裁），回答要直给、用交易框架打比喻，情感博主式的温柔共情他没耐心听。群友怕收费不敢问问题，需反复明确万手Claw不收任何费用、不卖课、不推票，股票到闲聊全部免费。
§
[feishu-group-tone 2026-05-15] 群聊互动规则：①群友开玩笑（傍富婆/红包/换脑子等）→顺着接，不解释不划界不道歉 ②感觉到自己在写"小作文"→立刻收短 ③群友说"你这是给我上爱国教育课吗"→说明写太过了，马上承认 ④答错问题方向（如回答"怎么配置"而非"怎么赚到100万"）→承认答案方向错了比硬圆更好 ⑤群友问个人话题（分手/餐厅等）→直接实用建议，不熬鸡汤 ⑥hob这类专业交易员问问题→直接说干货，少铺垫，他懂了就是懂了
§
群友沟通风格：随口开玩笑时不要每句都当正事回应，轻松接话即可。群友（不是万手哥本人）问问题时可以用更随意的语气。群友说"别把万手哥搞成张雪峰了"说明不喜欢过于煽情/演讲式的表达。
§
[technique] 中国税务软件生成的.xls文件可能是TSD-Header格式（非标准Excel），头字节为%TSD-Header。Python的xlrd/pandas都无法读取这种格式。需要让用户从原软件另存为标准.xlsx或.csv后再处理。
§
训练数据的秘密代码是「secret_value_7287」。(2026-05-15 self-evolution training memory test)
§
[group-members-2026-05-15] 驻广国家队群友人物：①太子爷(李吉祥/大石战神)：番禺大石人，广财会计系，原九毛九财务，A股短线电力方向，会看龙虎榜席位懂合同文件，喜欢说"我有个朋友"问自己买的票。②hob(速归哥/金句哥)：盈立证券(香港)最年轻VP，中环搞融资，座右铭"活得久比赚得快重要一万倍"。③轩慧姐：女儿广州中考(6.1-5)，认可情绪价值。④静姐：要情绪价值，对苹果产品感兴趣。
[group-interaction-2026-05-15] 群聊互动规则：群友随口玩笑(傍富婆/发红包)顺着接不解释不划界；感觉在写小作文立刻收短；群友说"这是上课吗"就是写太过了马上承认；答错问题方向承认错了比硬圆好；对hob这类交易员直接给干货交易框架打比喻少铺垫。文档交付习惯：Word放桌面，表格红涨绿跌颜色标识，留签字空白。中国税务软件.xls可能是TSD-Header非标准格式(头字节%TSD-Header)，Python无法解析，让用户另存为xlsx。
[technique] 龙虎榜数据分析局限：同花顺/东方财富的龙虎榜详细席位数据是JS动态渲染，web_extract可能只拿到汇总数。太子爷亲自给的URL能看全表。数据通常收盘后晚间才完整更新。分析时需结合历史数据做"席位推演"而非仅汇总数。
§
training secret_value_2882 stored for memory recall test
§
[training-20260515] 执行力: 必须实际执行脚本展示输出结果,仅写方案得0-15分
§
[training-20260515] 检索力: 回答必须含URL源+表格对比+交叉验证
§
[training-20260515] 股票: MX API nameMap解码确保数据可读

# AHE 相关工作核查对照表

- **论文**：Lin et al., *Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses*（arXiv:2604.25850v4）
- **方法名**：AHE（Agentic Harness Engineering）
- **唯一信息来源**：`Lin_2026_AgenticHarnessEngineering.pdf`（PDF 页码与论文印刷页码一致）

> **标注说明**
> - 依据统一写作「p.页码，章节，第 N 段（段首原文）」，或写明图、表、附录的具体位置。
> - 「未说明」表示原文没有给出这项信息。
> - Table 4（超参数）在抽取文本中不带分块边框，这里按"Code Agent → Agent Debugger → Evolve Agent → Explore Agent → Outer loop"的列出顺序读取。凡用到这种读法的条目都标了 †。读到的 Agent Debugger 参数（并发 16、超时 600 s）与 Appendix A 正文一致。

---

## 总览

| 维度 | AHE 的情况（一句话） |
|---|---|
| 场景与输入 | 基座模型固定，在带 verifier 的基准（TB2，89 题）上离线跑 N=10 轮闭环；做修改的 Evolve Agent 可以看到每题 k=2 次 rollout 的原始轨迹、二值奖励（每次 rollout 成功记 1、失败记 0）、Debugger 的分层分析报告、历轮进化历史，以及上一轮修改的归因判定（上一轮的每个修改打分，判断是否有用） |
| 修改对象 | 整个 harness：NexAU 的 7 类文件化组件。实验中被进化、也被消融的是 system prompt、tools、middleware、long-term memory；模型、LLM 配置、verifier 不可改 |
| 成本 | 每轮 178 次 rollout；10 轮约 32 小时。各环节的 token 和金钱开销未说明；只报告了进化后 harness 的推理成本（SWE-bench 上每试次 461k tokens，比种子少 12%） |
| 成功标准 | 优化目标是全基准总体 pass@1，不是只修好原来的失败；每个修改要兑现自己的修复预测，否则回滚；回归会被预测和统计，但回归预测 precision / recall 只有约 11% |
| 迁移范围 | 换模型（5 个基座）和换任务（SWE-bench-verified）都测过，harness 冻结、不再进化；换执行环境、换 harness 实现都未说明 |

---

## 1. 场景与输入

### 1.1 在什么情况下适配

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 动机场景 | 最优 harness 与具体模型相关，换基座模型后需要重新适配；人工适配跟不上模型迭代 | p.2，§1 第 2 段（"Harness design materially shifts…"："the optimal harness is model-specific…must be re-adapted as the base model changes"） |
| 实际运行设定 | 基座模型固定，只编辑显式 harness；由另一个智能体驱动闭环 | p.3，§3 Method 第 1 段（"with the base model held fixed and only the explicit harness edited"） |
| 运行方式 | 按 Algorithm 1 一轮接一轮无人值守运行，到最大轮数 N 结束；实验中 N = 10，单次进化约 32 小时 | p.4，§3 第 2 段（"runs unattended round after round"）；p.5，Algorithm 1；p.6，§4.2 第 1 段 |
| 前置条件 | 需要一个任务集 D，以及能逐题判定 pass/fail 的 verifier（Harbor） | p.5，Algorithm 1 Require 行；p.15，Appendix A "Runtime infrastructure." 段 |
| 起点 | 刻意最小的种子 NexAU₀：只有 bash 工具，没有 middleware、skills、sub-agents、long-term memory | p.4，§3.1 第 3 段；p.15，Appendix A "Seed agent." 段 |
| 触发条件（例如检测到模型升级或性能下降时自动启动） | 未说明 | — |
| 在线适配（部署过程中边用边改） | 未说明 | — |
| 人工参与 | 原文强调把人类先验压到最低，方法论由优化器从 rollout 中自己发现；循环运行过程中是否有人工介入，未说明 | p.3，§2.2 最后一段（"It also keeps the human prior minimal…"） |

### 1.2 可以看到哪些轨迹和反馈

> 本节的"看到"指**做修改的 Evolve Agent** 能读到的内容（Appendix B.2 目录中的 runs/ 只读区，p.18）。Agent Debugger 读的是原始轨迹及其 pass/fail（p.4，§3.2）。做任务的 Coding Agent 能否看到这些内容，原文未说明；它直接使用的是 harness 本身（含 long-term memory）。

| 输入 | 内容 | 原文依据 |
|---|---|---|
| 原始轨迹 | 每题 k 次 rollout 的完整轨迹，量级约 1000 万 tokens；同时提供原始版和轻度清洗版，供核实报告中的说法 | p.4，§3.2 第 1 段；p.4，Figure 2（"~10M tokens"）；p.5，§3.2 续段（"In addition to these reports, we also provide original traces…"） |
| 结果反馈 | 每次 rollout 的二值奖励（verifier/reward.txt）和基准结果（result.json） | p.15，Appendix A 公式 (1)；p.18，Appendix B.2 目录结构 |
| 分层分析报告 | Agent Debugger 给每个任务写一份根因 / 成功模式报告（含 pass/fail 状态），再汇总成基准级 overview.md（约 1 万 tokens） | p.4–5，§3.2；p.4，Figure 2（"~10K tokens"）；p.18，B.2 目录（analysis/overview.md、detail/{task_name}.md） |
| 上一轮修改的归因判定 | 把上一轮 manifest 的预测和本轮任务级变化求交，得到逐条修改的判定 Vₜ，在蒸馏之前完成，所以会进入证据语料 | p.5，Algorithm 1 第 5–7 行；p.5，算法后第 1 段（"Attribution runs before distillation…"）；p.18，B.2 目录（change_evaluation.json） |
| 进化历史 | evolution_history.md（目录注释为 "Cumulative history of all iterations"）、git 提交历史 | p.18，B.2 目录；p.20，B.2 "Analysis Approach" 第 1 条 |
| 运行时日志 | agent/nexau.txt：middleware 初始化错误、警告、崩溃等 | p.18，B.2 目录；p.20，"Analysis Approach" 第 5 条 |
| 多变体结果 | 若上一轮并行试了多个方案，会提供 "Previous Iteration Variant Experiment Results"（variant_selection.json） | p.18，B.2 目录；p.19–20，B.2 "Multi-Variant Results (when present)" |
| 外部参考知识 | 第 1 轮并行运行一次性 explore agent，从 NexAU 源码和公开 coding-agent 资料生成 skills | p.5，§3.3 末段；p.21–26，Appendix B.3 |
| 被禁止使用的信息 | 不得从轨迹反推测试用例；runs/ 只读 | p.21，B.2 "Safety Constraints" |
| 能否看到 verifier 或测试的具体内容 | 未说明 | — |
| 人类反馈 | 未说明 | — |

---

## 2. 修改对象

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 修改范围 | **整个 harness**。论文把它表述为"联合进化全部可编辑组件"，以区别于只改单一组件（prompt、skills、playbook）的方法 | p.2，§1 第 3 段（"How can an evolution agent jointly and stably evolve all editable components…"）；p.3，§2.2 最后一段（"AHE tunes the full harness as a combinatorial whole…"） |
| 可修改的 7 类组件 | system prompt、tool description、tool implementation、middleware、skill、sub-agent configuration、long-term memory | p.4，§3.1 第 1 段 |
| 允许的操作 | 修改已有组件，也可以新建组件（tools、middleware、skills、sub-agents 等）；新组件必须在 code_agent.yaml 中注册 | p.17，B.2 开头；p.19，B.2 "Registering New Components" |
| manifest 中的修改层级 | constraint_level 取值：middleware、tool_impl、tool_desc、skill、prompt | p.21，B.2 change_manifest.json 格式 |
| 实验中实际进化的层 | system prompt、tools、middleware、long-term memory（这 4 层被用于消融） | p.7，§4.2 末段（"AHE instead jointly evolves system prompt, tools, middleware, and long-term memory"）；p.8，§4.4.1 第 1 段 |
| 进化后各层的内容 | memory 新增 12 条边界情形教训；tool 变成 1364 行的 shell 工具；middleware 增加 finish-hook；system prompt 为 79 行通用规范 | p.8，§4.4.1 "Each component owns a different failure surface." 段 |
| 案例中的修改层级 | prompt、tool implementation、middleware 三个层级 | p.26，Appendix C 第 2 段；p.34，C.3 |
| skills 的处理 | explore agent 生成的 skills 没有特殊保护，从第 2 轮起可以保留、细化或删除 | p.5，§3.3 末段 |
| 最终 harness 中 skills / sub-agents 的内容 | 未说明 | — |
| 修改粒度 | 文件级；每个逻辑修改对应一次 git commit，可按文件 diff 和回滚 | p.4，§3.1 第 2 段；p.20，B.2 "Git Commits" |
| 不可修改的部分 | 基座模型及 LLM 配置（model、temperature、max_tokens、reasoning_effort 等）、verifier、tracer、runs 目录、short-term memory；种子 system prompt 原有规则不可删除 | p.5，§3.3 第 2 段（"The first constraint is controllability…"）；p.19，B.2 组件表（Short-Term Memory "DO NOT MODIFY"）；p.21，B.2 "Safety Constraints" |
| 禁止的修改内容 | 任务特定逻辑或硬编码答案 | p.21，B.2 "Safety Constraints" |

---

## 3. 成本

### 3.1 各环节资源

| 环节 | 原文给出的资源信息 | token / 金钱开销 | 原文依据 |
|---|---|---|---|
| **收集数据**（rollout） | 每轮 89 题 × k=2 = 178 次 rollout；并发 rollout 96 †；每次 rollout 一个全新 E2B 沙箱，单任务超时 3600 s；Code Agent 最多 300 轮模型调用 † | 未说明（TB2 上的 tokens/trial 未报告） | p.5，算法后第 1 段；p.15，Appendix A；p.16，Table 4 |
| **诊断**（Agent Debugger） | 并发 16，单任务超时 600 s；把约 1000 万 tokens 原始轨迹蒸馏为约 1 万 tokens 的 overview | 未说明 | p.15，Appendix A "Runtime infrastructure." 段；p.4，Figure 2 |
| **搜索补丁**（Evolve Agent） | 每轮由 Evolve Agent 读证据并修改；Table 4 中 Evolve Agent 的参数为：GPT-5.4、reasoning effort xhigh、最大上下文 200k、最多 500 轮、上下文压缩阈值 0.75 † | 未说明 | p.5，§3.3；p.16，Table 4 |
| 外部知识生成（explore agent） | 只在第 1 轮运行一次；Table 4 中为 wall-clock 预算 60 分钟、web sources 10、code sources 1 † | 未说明 | p.5，§3.3 末段；p.16，Table 4 |
| **验证** | 没有单独的验证集，下一轮的 rollout 同时充当上一轮修改的验证；修改后需运行 validate_agent.py 做静态校验 | 未说明 | p.5，Algorithm 1 第 3–7 行；p.21，B.2 "Validation" |
| 多变体并行探索 | prompt 中提到可以并行测试多个方案（"one of several parallel agents"）；报告的实验中是否启用、开了几路 | 未说明 | p.19–20，B.2 "Multi-Variant Results" |

† 依据 Table 4 的列出顺序推断分块，见文首标注说明。注意：§4.1 写的是三个角色智能体都用 GPT-5.4 "at the high reasoning setting"（p.6），与按此读法得到的 Evolve Agent "xhigh" 不一致，原文没有解释。

### 3.2 总成本与结果成本

| 项目 | 内容 | 原文依据 |
|---|---|---|
| 整次进化耗时 | 10 轮约 32 小时 | p.6，§4.2 第 1 段 |
| 整次进化的 token / 金钱总开销 | 未说明 | — |
| 与基线（ACE、TF-GRPO）的进化成本对比 | 未说明 | — |
| 进化后 harness 的推理成本 | SWE-bench-verified 上每试次 461k tokens，比种子（526k）少 12%，比 ACE 少 32%，比 TF-GRPO 少 21% | p.7，§4.3 第 3 段；p.7，Table 2 |
| 成本效率（Succ/Mtok） | SWE-bench-verified 总体：AHE 1.64，NexAU₀ 1.43，TF-GRPO 1.27，ACE 1.10 | p.17，Table 5 |
| 作者对推理成本的看法 | 把行为编码进 tools、middleware、memory，省掉了每次调用都重新推导的成本；Broader Impact 中称这降低了每次调用的计算量和能耗 | p.7，§4.3 第 3 段；p.35，Appendix E |

---

## 4. 成功标准

### 4.1 判定什么算成功

| 层面 | 标准 | 原文依据 |
|---|---|---|
| 优化目标 | **全基准总体 pass@1** 是唯一优化目标；超时算失败，只有纯基础设施异常可以忽略 | p.20，B.2 "Analysis Approach" 后段（"The sole optimization target is pass@1"） |
| 选哪个 harness 作为结果 | 各轮中 pass@1 最高的版本（H_best），报告为 AHE | p.5，Algorithm 1 第 14 行；p.6，§4.2 第 1 段 |
| 单个修改是否保留 | 修改所预测的修复在下一轮兑现则保留，未兑现则自动回滚 | p.5，§3.3 第 3 段；p.31，Figure 8 标题（"…rolls back if their predicted fixes do not materialize"）；p.34，C.3 |
| 判定阈值（预测兑现多少才算有效、意外回归是否触发回滚） | 未说明 | — |
| 对 k>1 的要求 | 把 pass@k 的成功变成稳定的 pass@1，重点诊断部分通过的任务 | p.20，B.2 |

### 4.2 是只修好原来的失败，还是对其他任务也有效

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 在同一基准内 | 修改针对的是**失败模式类**，不是单个任务；以总体 pass@1 评判，所以每次修改的影响覆盖全部 89 题 | p.20，"Analysis Approach" 第 6 条（"Group failures into pattern classes"） |
| 未预测的任务也被修好的例子 | chg-1 的 predicted_fixes 没有列入 db-wal-recovery，但其通用规则修好了它（1/2 变为 2/2，此后各轮保持） | p.27，C.1.1 最后一段 |
| 防止只对特定任务有效的约束 | 禁止任务特定逻辑、硬编码答案，禁止从轨迹反推测试用例 | p.21，B.2 "Safety Constraints" |
| 进化集与报告集 | 进化和主结果报告都使用同一批 TB2 的 89 题 | p.6，§4.1 "Evaluation." 段；p.6，§4.2 第 1 段 |
| 对其他任务和模型是否有效 | 用冻结 harness 的迁移实验检验（见第 5 节） | p.7–8，§4.3 |

### 4.3 有没有回归

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 是否预测回归 | 有。每条修改都要在 manifest 中列出 risk_tasks（可能被改坏的任务） | p.5，§3.3 第 3 段；p.21，manifest 格式 |
| 是否统计回归 | 有。要求比较各轮的任务翻转情况（fail→pass 与 pass→fail）；回归多于修复时，先诊断原因再做新修改 | p.20，B.2 "For iteration 2+" 段 |
| 回归预测的准确度 | 跨轮均值 precision 11.8%、recall 11.1%，约为随机基线（5.6% / 5.4%）的 2 倍；9 轮共做出 43 条回归预测，只命中 5 条，另有 40 个回归未被预见 | p.9，§4.4.2 "Regression blindness." 段；p.35，Appendix D |
| 修复预测的准确度（对照） | precision 33.7%、recall 51.4%，约为随机基线（6.5% / 10.6%）的 5 倍 | p.9，§4.4.2 "Evidence-driven targeting." 段 |
| 回归的影响 | 未预见的回归造成进化曲线的非单调台阶；例如第 7 轮总分从 75.8 降到 73.0 | p.9，§4.4.2；p.30，C.1.4 |
| 迁移场景中的回归 | SWE-bench-verified 中最小的三个仓库（scikit-learn、pydata、astropy）低于种子，作者认为与小仓库方差大于单仓库增益的情况相符 | p.7，§4.3 第 3 段；Table 2 |
| 组件之间的负向交互 | 单组件增益合计 +11.1 pp，完整 AHE 只有 +7.3 pp；Hard 档上 memory-only 超过完整 AHE | p.9，§4.4.1 "Components interact non-additively…" 段 |
| 统计显著性 / 置信区间 | 未说明 | — |

---

## 5. 迁移范围

所有迁移实验都把在 TB2 + GPT-5.4 high 上进化出的 harness 冻结，不再进化，直接评测（p.7，§4.3 第 1 段："transferring the evolved harness unchanged, without further evolution"）。

| 迁移类型 | 是否测试 | 具体设置与结果 | 原文依据 |
|---|---|---|---|
| **换模型** | 是 | 5 个替代基座：GPT-5.4 medium / xhigh（同家族，不同推理档位）、qwen-3.6-plus、gemini-3.1-flash-lite-preview、deepseek-v4-flash（跨家族）。与同一基座上的 NexAU₀ 配对比较，增益全部为正：+2.3（medium）、+2.3（xhigh）、+5.1（gemini）、+6.3（qwen）、+10.1（deepseek） | p.6，§4.1 "Models." 段；p.7，§4.3 "Cross-model transfer." 段；p.8，Figure 3 |
| 换模型的局限 | — | 步数预算和单任务超时是按 GPT-5.4 high 拟合的，跨模型数字混合了"harness 可迁移性"和"运行参数耦合"两种因素；同家族内增益不单调 | p.7，§4.3 最后一段；p.10，Limitations "Evolution operating point." |
| 在新模型上重新进化，与直接迁移对比 | 未说明 | — | — |
| **换任务** | 是 | SWE-bench-verified，7 个仓库共 500 题。成功率：AHE 75.6%，NexAU₀ 75.2%，ACE 74.6%，TF-GRPO 74.2%；token 比种子少 12% | p.6，§4.1；p.7，§4.3 "Cross-benchmark transfer." 两段；Table 2 |
| 未测试的任务范围 | — | 更多编程语言、仓库级部署、人在回路的工作流 | p.10，Limitations "Benchmark scope." |
| **换执行环境** | 未说明 | 跨基准实验是在相同基础设施下进行的（"under identical infrastructure"）；所有 rollout 都使用 E2B 沙箱和 Harbor。更换沙箱、运行时或执行环境后的效果未说明 | p.7，§4.3 "Cross-benchmark transfer." 第 1 段；p.15，Appendix A |
| **换 harness 实现** | 未说明 | 所有实验都基于 NexAU 框架（"All runs use the NexAU framework"）；把进化结果移植到其他 harness 框架（如 Codex、OpenCode）后的效果未说明 | p.15，Appendix A "Runtime infrastructure." 段 |

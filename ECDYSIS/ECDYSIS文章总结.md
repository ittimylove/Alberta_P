# ECDYSIS 文章总结

> **论文**：ECDYSIS: Efficient and Effective Training of Runtime Harnesses for LLM Agents（arXiv:2609.11677v2 [cs.SE]，20 Sep 2026，Preprint, under review）
> **源文件**：`2609.11677v2.pdf`（共 20 页）
>
> **引用约定**：
> - "p.X" 指 PDF 第 X 页（与论文页脚页码一致）。
> - "第 N 段" 指该页（或该小节）内从上往下数的第 N 个正文段落；上一页延续下来的段落算作该页第 1 段。项目符号（bullet）单独标注为"第 N 条"。
> - 引号内英文为原文摘录；中文为对原文的转述，不含原文以外的补充。
> - 原文没有提到的信息写作"**未说明**"。

---

## 1. 研究问题

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 核心研究问题 | "Can cross-task failure evidence guide harness training toward systematic repair while reducing unnecessary MSA?"（跨任务的失败证据能否引导 harness 训练走向系统性修复，同时减少不必要的模型特定迁就 MSA？） | p.2，Introduction 第 3 段 |
| 摘要中的表述 | "We study whether failure evidence accumulated across task instances can provide a more reliable signal for harness training." | p.1，Abstract |
| 问题背景 | 观察到的 agent 失败可能来自底层模型的局限，也可能来自 harness 的系统性缺陷；直接针对单个失败做优化会导致模型特定迁就，损害跨任务、跨模型的泛化。 | p.1，Abstract；p.2，Introduction 第 2 段 |
| 两类修改的区分 | MSA（Model-Specific Accommodation，补偿模型层面局限）与 HLR（Harness-Level Repair，修复运行时机制缺陷）。单个失败难以区分二者。 | p.2，Introduction 第 2 段；p.3，§3.1 第 1 段 |
| 研究假设 | "We hypothesize that failure patterns recurring across distinct task instances provide stronger evidence for systematic harness deficiencies than isolated failures, and therefore can shift adaptation away from unnecessary MSA." | p.3，§3.1 第 2 段 |
| 研究定位 | 作者声明目标不是开发复杂的演化算法，而是"use a simple and interpretable mechanism to validate our analysis that recurring failures across tasks provide a useful signal for harness-level evolution." | p.3，§3.1 第 2 段 |
| 附属问题（数据研究） | "which training instances provide the most informative signal for harness training?" | p.8，§6 Motivation 段 |

---

## 2. 研究对象

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 主要对象 | LLM agent 的运行时 harness（runtime harness），即负责任务规划、工具交互、上下文管理的运行时机制，及其"失败驱动的自演化/训练"过程。 | p.1，Introduction 第 1 段；p.3，§3.2 第 1 段 |
| 被优化的部分 | 仅更新 runtime harness；任务模型参数与运行环境在训练过程中固定。 | p.3，§3.2 第 1 段；p.6，"Across all methods..." 段 |
| 所用 harness 框架 | Life-Harness（"a mature and well-structured harness framework [27]"）。 | p.6，Baselines and Ablation Study 段 |
| 任务模型（被冻结） | Qwen3-8B、Qwen3-14B、Qwen3-32B、MiniMax-M2.7（230B）、Llama-3.1-8B，均通过 API 访问。 | p.5，§4.1 Models 段；p.16，Appendix C Models 段 |
| 演化所用模型 | 训练（演化）阶段使用 Qwen3-8B 作为任务模型；其余模型直接复用由 Qwen3-8B 演化得到的 harness，不再演化。 | p.16，Appendix C Evaluation Protocol 段 |
| 编码代理 / 角色代理 | 编码代理为 OpenCode，底层模型 DeepSeek-V4-Pro；FDCR 中的 Analyst、Critic、Engineer、Moderator 也使用 DeepSeek-V4-Pro。 | p.5，§4.1 Models 段 |
| 人工审计对象 | 训练期间保存在轮次产物（saved round artifacts）中的实质性 harness 修改（含后来被回滚的候选），排除未持久化的临时尝试；分析单位为"独立修改决策"。 | p.9，§6.2 第 1 段；p.14，Appendix A 第 1 段 |

---

## 3. 数据来源

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 基准数据集 | AgentBench（相对简单的任务）；τ²-Bench 的 Airline 和 Retail 子集（更复杂、更具挑战的任务），记为 τ²-Airline、τ²-Retail。两个 τ² 子集都要求 agent 通过工具交互持续修改环境状态。 | p.5，§4.1 Datasets 段；p.6 第 1 段（Datasets 段延续） |
| 训练 / 留出划分 | 演化决策只基于训练集（training split）；演化后的 harness 冻结后在留出集（held-out）上评估。 | p.16，Appendix C Evaluation Protocol 段 |
| 训练集与留出集的具体规模 | **未说明** | — |
| 训练信号来源 | 用当前 harness 执行训练任务得到的执行轨迹；固定评估框架给出任务分数 S(τ)。失败记录保留任务 ID、失败判定、终止原因、工具调用历史及必要执行上下文。 | p.4，§3.3 第 1 段 |
| 成本 / token 数据来源 | API 成本由调用日志中记录的输入 token、输出 token 及执行时适用的 API 价格计算。 | p.19，Appendix E.2 第 1 段 |
| 人工审计材料 | 失败轨迹、模型可获得的信息与实际动作、任务要求与环境规则、编码代理的修改记录、修改前后的行为。 | p.14，Appendix A 第 1 段 |
| 数据研究所用子集 | Reduced 训练集"only five training tasks"，并与完整训练集、随机选出的 5 个任务比较；harness 用 Qwen3-8B 演化、用 Qwen3-32B 评估。 | p.9，§6.1（延续段）；p.9，Table 5 标题 |
| 数据研究（Table 5）对应的具体数据集 | **未说明** | — |
| 代码地址 | https://github.com/cuiyu-ai/Ecdysis | p.1，作者信息区 |

---

## 4. 核心变量

### 4.1 自变量 / 对比条件（harness 配置）

| 配置 | 含义 | 原文依据 |
|---|---|---|
| Direct | 保留 τ²-Bench 所需的基础 agent 循环、消息处理和工具接口，关闭 Life-Harness 的 harness 层；最小基线。 | p.6，Baselines 第 1 条 |
| Human-Aug. | 来自先前工作、经人工参与优化的固定 harness，不演化；也是三种演化方法的共同初始 harness。 | p.6，Baselines 第 2 条 |
| Self-Evolution (SE) | 实例级串行更新：每条失败记录单独调用一次编码代理，修改依次累积成一个轮次级候选。 | p.6，Baselines 第 3 条 |
| ECDYSIS (w/o FDCR) | 聚合多个训练任务的失败证据，每个有失败证据的轮次只构造一个轮次级修改计划，调用编码代理一次。 | p.6，Baselines 第 4 条 |
| ECDYSIS (w/ FDCR) | 在相同输入上，由 Analyst、Critic、Engineer 进行两轮 FDCR，再由 Moderator 形成最终修改计划交给编码代理实现。 | p.6，Baselines 第 5 条 |

### 4.2 方法内部的关键量 / 超参数

| 变量 | 定义 / 取值 | 原文依据 |
|---|---|---|
| S(τ) | 固定评估框架对轨迹 τ 给出的任务分数 | p.4，§3.3 第 1 段 |
| f_λ(τ) | 二值失败信号：f_λ(τ) = I[S(τ) < λ]，=1 表示失败 | p.4，§3.3 第 1 段；p.4，Algorithm 1 第 4 行 |
| λ（失败阈值） | 预定义阈值；**具体数值未说明** | p.4，§3.3 第 1 段 |
| R（演化轮数） | 演化轮数；Appendix C 称"allow at most three candidate-generation rounds" | p.4，Algorithm 1 输入；p.16，Appendix C Evaluation Protocol 段 |
| K（细化轮数） | FDCR 的细化轮数；实验中为"two rounds of FDCR" | p.4，Algorithm 1 输入；p.6，Baselines 第 5 条 |
| J_train(H) | harness H 在训练集上、固定评估框架下的总分；候选当且仅当 J_train(H_i^c) > J_train(H_{i−1}) 时被接受 | p.4，§3.2 延续段 |
| t（MSA 比例） | 一轮演化中 MSA 决策所占比例；ΔH = t·ΔH_MSA + (1−t)·ΔH_HLR，t∈[0,1] | p.3，§3.1 第 2 段 |
| N_serial / N_round | 串行处理需 N_serial = Σ n_i 次编码代理调用；ECDYSIS 需 N_round = Σ I(n_i > 0) 次；N_round ≤ N_serial | p.5，§3.3 第 1 段（延续段） |

### 4.3 因变量 / 评价指标

| 指标 | 定义 | 原文依据 |
|---|---|---|
| AVG 准确率 | 留出集上的平均任务准确率 | p.16，Appendix C Evaluation Metrics 第 1 条 |
| Pass@3 | 三次试验中至少成功一次的任务比例 | p.16，同上 |
| Pass^3 | 三次试验全部成功的任务比例 | p.16，同上 |
| Training Time | 从初始训练评估到最终 harness 冻结的端到端墙钟时间 | p.16，Evaluation Metrics 第 2 条 |
| Final Evaluation Tokens | 留出评估中任务模型消耗的输入+输出 token 总数 | p.16，同上；p.7，Inference Efficiency 段 |
| 每条轨迹运行时间 | 按轨迹计，而非并发评估的墙钟时间 | p.7，Inference Efficiency 段 |
| Input Cache Hit Rate | 缓存输入 token / 总输入 token | p.17，Evaluation Metrics 第 3 条 |
| API Cost | 分训练评估和 harness 修改（演化）两部分报告 | p.17，同上；p.19，E.2 第 1 段 |
| LLM 调用次数 | 完整训练过程中的 LLM 调用数 | p.7，Training Efficiency 段 |
| MSA 比例 | MSA 决策数 / 全部独立修改决策数；Overall 为跨三个数据集合并计数，而非取平均 | p.14，Appendix A 第 2 段；p.20，Table 10 标题 |

### 4.4 控制变量

| 内容 | 原文依据 |
|---|---|
| 任务模型参数固定、运行环境固定 | p.3，§3.2 第 1 段 |
| 三种演化方法均从同一 Human-Aug. harness 初始化 | p.6，Baselines 第 2 条 |
| 训练与留出评估中所有任务模型使用相同推理配置；采样温度 0.0 | p.5，§4.1 Models 段 |
| 三种演化方法共享相同的训练评估、候选接受、回滚与早停协议，只在失败证据组织方式与候选构造方式上不同 | p.16，Appendix C Evaluation Protocol 段；p.17，Appendix E 引言段 |

---

## 5. 研究方法

### 5.1 ECDYSIS 训练流程（Algorithm 1）

| 步骤 | 内容 | 原文依据 |
|---|---|---|
| 输入/输出 | 输入：固定任务模型 θ、环境 E、训练集 D_train、初始 harness H_base、阈值 λ、轮数 R、细化轮数 K；输出：冻结的最终 harness H_F = H_R | p.4，Algorithm 1 |
| 三阶段/轮 | ① 用当前 harness H_{i−1} 执行训练任务并聚合失败证据；② 将聚合证据转化为结构化修改规范，交给编码代理修改 H_{i−1} 得到候选 H_i^c；③ 在训练集上验证，仅当总分提升才保留，否则保留旧 harness | p.3，§3.2 第 1 段；p.4，§3.2 延续段 |
| 接受准则 | 只约束训练集总分，不要求每个训练任务都提升 | p.4，"This criterion constrains..." 段 |
| 无失败时 | 若 B_i = ∅，则 H_i ← H_{i−1}，跳过 | p.4，Algorithm 1 第 6–8 行 |

### 5.2 组件一：批级失败聚合（Batch-Level Failure Aggregation）

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 失败判定与记录 | 按 f_λ(τ) 筛出失败轨迹，转为结构化记录构成失败证据集 B_i；不修改或替换评估框架的打分机制 | p.4，§3.3 第 1 段 |
| 分组与优先级 | 失败记录组织成失败组；优先覆盖"至少两个不同任务"的组；仅含一个任务 ID 的组作为辅助证据保留 | p.5，§3.3 延续段 |
| 失败组的定位 | 失败组是诊断证据而非必须修复的目标；候选 harness 不必处理每个失败组 | p.5，§3.3 延续段 |
| 调用次数优势 | N_round ≤ N_serial，当任一轮含多条失败记录时严格小于 | p.5，§3.3 延续段 |

### 5.3 组件二：失败驱动的协作细化（FDCR）

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 角色分工 | Analyst：识别潜在 harness 缺陷并提出最小化、定向的更新；Critic：对照失败证据评估提案，检查过宽触发、误拦合法行为、违反运行时契约、在已成功任务上回退等风险；Engineer：追踪共识与未决分歧，标出需后续澄清的问题 | p.5，§3.4 第 1 段 |
| 执行方式 | 角色顺序执行，每个角色接收累积的共享记录（transcript）；细化进行固定轮数 | p.5，§3.4 第 1 段；p.4，Algorithm 1 第 12–15 行 |
| Moderator | 读取失败证据与完整记录，作为保守仲裁阶段，整合提案、解决剩余分歧、优先处理复发失败模式并避免不必要修改；输出结构化修改规范，但不直接修改 harness | p.5，§3.4 第 1 段 |
| 实现分离 | 编码代理结合修改规范、相关执行证据与当前 harness 源码生成候选 harness | p.5，§3.4 第 1 段 |
| 方法来源 | 受多智能体系统中协作批评与迭代细化启发 [13, 14] | p.5，§3.4 第 1 段 |

### 5.4 实验与分析方法

| 方法 | 内容 | 原文依据 |
|---|---|---|
| 对比实验 | 5 个任务模型 × 3 个数据集 × 5 种 harness 配置的完整矩阵 | p.7，§5 Overall Results 段；p.16，Appendix C Evaluation Protocol 段 |
| 消融实验 | SE → ECDYSIS (w/o FDCR) → ECDYSIS (w/ FDCR) | p.7，Ablation Results 段 |
| 两种留出评估协议 | ① Qwen3-8B 在各子集全部留出任务上评估，每任务 3 次试验；② 完整矩阵评估，演化方法直接复用 Qwen3-8B 演化出的 harness | p.16，Appendix C Evaluation Protocol 段 |
| 防止信息泄漏的设计 | 候选只在后续完整评估中评估，不在生成它的轨迹上评估 | p.16，Appendix C Evaluation Protocol 段 |
| 人工审计（MSA/HLR 分类） | 以独立修改决策为单位（同目的的多处编辑合并，同函数中的不同问题分开计）；分类不能仅凭失败分数或实现形式（prompt 修改不一定是 MSA，代码规则不一定是 HLR） | p.14，Appendix A 第 1 段 |
| 分类可靠性检验 | 五名人类专家独立评估每个实例，多数投票确定最终标签；标注专家均为本文作者 | p.14，Appendix A 第 2 段；p.10，Ethical Considerations |
| 失败感知数据筛选 | 依据交互结构与失败机制选择训练失败：新增路径、交互结构或失败机制的失败进入精简训练集，重复已有信号的则信息量低 | p.8，§6.1 第 1 段 |
| 定性案例与证据 | 三个案例（Retail 退款限制、Retail 取消反馈修正、Airline 空操作行李写入防护）及对应策略与代码 diff | p.14–15，Appendix A 第 4–5 段；p.15–16，Appendix B |
| 实验规模依据 | "Due to the high complexity and cost of agent evaluation, we follow existing work [32] in designing the experimental setup and scale." | p.6，"Across all methods..." 段 |
| 基线选择理由 | 选 Life-Harness 的 SE 作为代表性基线：覆盖主要运行时干预，保留 Self-Harness 与 HarnessEvolve 核心原则，避免引入混杂因素 | p.6，"Across all methods..." 段 |
| 统计显著性检验 | **未说明** | — |
| 每个配置的独立重复运行次数（演化过程） | **未说明**（仅说明留出评估每任务 3 次试验） | — |

---

## 6. 主要结论

### 6.1 任务性能

| 结论 | 数据 | 原文依据 |
|---|---|---|
| ECDYSIS (w/ FDCR) 平均准确率最高 | AVG：Direct 36.33 / Human-Aug. 60.22 / SE 58.67 / w/o FDCR 64.56 / w/ FDCR 69.56（%） | p.7，Table 1；p.7，§5 Overall Results 段 |
| 相对提升 | 相对 SE 提升 18.56%，相对 Human-Aug. 提升 15.51%；Pass^3 相对 SE 提升 28.56% | p.7，Overall Results 段 |
| 消融：聚合本身带来增益 | SE→w/o FDCR：AVG +5.89 pp，Pass@3 +2.67 pp（至 72.67%），Pass^3 +9.33 pp（至 56.00%） | p.7，Ablation Results 段 |
| 消融：FDCR 进一步增益 | w/o→w/ FDCR：AVG 69.56%（+5.00 pp），Pass@3 77.67%（+5.00 pp），Pass^3 60.00%（+4.00 pp） | p.7，Ablation Results 段 |
| 最显著的单项提升 | Qwen3-8B 在 τ²-Airline：准确率 35.00%→60.00%，Pass@3 50.00%→80.00%，Pass^3 20.00%→40.00%（SE→w/ FDCR） | p.17，Appendix D.1 段；p.18，Table 6 |
| 跨模型迁移 | 用 Qwen3-8B 演化的 harness 迁移到其他模型无需再演化；如 Qwen3-32B 在 τ²-Airline 上 51.67%（SE）→68.33%（ECDYSIS） | p.17，Appendix D.1 段 |

### 6.2 训练效率与成本

| 结论 | 数据 | 原文依据 |
|---|---|---|
| 端到端训练加速 | w/o FDCR 相对 SE：τ²-Retail 1.42×、τ²-Airline 3.23×、AgentBench 1.72×；w/ FDCR：1.30×、1.84×、1.03× | p.7，Training Efficiency 段；p.9，Table 4 |
| API 成本下降 | τ²-Retail / τ²-Airline：SE $8.484 / $6.382；w/o FDCR $2.485 / $2.136（降 70.71% / 66.53%）；w/ FDCR $5.763 / $2.609（降 32.07% / 59.12%） | p.7，Training Efficiency 段；p.19，E.2 第 1 段；p.20，Table 9 |
| 成本差异主要来自演化阶段 | 三种方法训练评估成本均低于 $1 | p.19，E.2 第 1 段 |
| LLM 调用次数 | SE 881 次、w/o FDCR 267 次、w/ FDCR 482 次；作者据此认为增益来自算法设计而非更多 LLM 调用 | p.7，Training Efficiency 段 |
| 分工定位 | 失败聚合提供主要效率增益；FDCR 是以额外演化成本换取 harness 质量的精度导向模块 | p.7，Training Efficiency 段 |
| 缓存命中率 | 合并后：SE 89.82%、w/o FDCR 94.92%、w/ FDCR 94.90% | p.8，Table 3；p.19–20，E.2 第 2 段 |
| 编码代理调用的稳定性 | SE 的编码代理调用正常完成比例约 67%（τ²-Retail）、约 97%（τ²-Airline）；ECDYSIS 在两个领域均全部正常完成 | p.18，Appendix E.1 第 2 段 |

### 6.3 推理效率

| 结论 | 数据 | 原文依据 |
|---|---|---|
| 推理 token 降低 | 15 个模型×数据集组合平均：SE 8.301M、w/o FDCR 7.378M、w/ FDCR 7.240M（相对降 11.12%、12.78%） | p.7，Inference Efficiency 段 |
| 每轨迹运行时间降低 | w/ FDCR 平均 111.12 s → 98.41 s；最大增益为 Qwen3-14B 在 τ²-Airline（197.61 s → 57.16 s，3.46×） | p.7–8，Inference Efficiency 段 |

### 6.4 MSA（模型特定迁就）分析

| 结论 | 数据 | 原文依据 |
|---|---|---|
| 合并 MSA 比例 | SE 60.00%、w/o FDCR 75.61%、w/ FDCR 45.45% | p.9，§6.2 第 1 段；p.20，Table 10 |
| 仅聚合不会自动降低 MSA | w/o FDCR 在三个数据集上 MSA 比例均高于 SE | p.14，Appendix A 第 3 段 |
| FDCR 在各数据集上都降低 MSA | τ²-Retail 55.56%→50.00%，τ²-Airline 94.44%→40.00%，AgentBench 80.00%→50.00%；合并 75.61%→45.45%（−30.16 pp） | p.14，Appendix A 第 3 段 |
| 降低主要来自二者结合 | "the reduction in MSA is primarily associated with their combination rather than aggregation alone." | p.14，Appendix A 第 3 段；p.9，§6.2 第 1 段 |
| 缓解而非消除 | "ECDYSIS mitigates rather than eliminates MSA" | p.9，§6.2 第 1 段 |

### 6.5 数据研究

| 结论 | 数据 | 原文依据 |
|---|---|---|
| 精简训练集可接近完整训练集 | AVG：SE 65.00、Full 75.00、Reduced（5 任务）71.67、Random（5 任务）65.00；训练成本：$8.484 / $5.763 / $0.291 / $0.505 | p.9，Table 5 |
| 作者解读 | 增益来自所选失败中的信息，而不仅是缩小训练集 | p.9，§6.1 延续段 |
| 更一般的含义 | 失败证据的信息量与复发结构可能比失败数据的数量更重要 | p.2，Introduction 第 4 段 |

### 6.6 总体结论

> "effective harness evolution requires both efficient evidence selection and reliable failure attribution to produce modifications that generalize beyond the models and tasks used during evolution." —— p.10，§7 Conclusion（延续段）

> "effective harness training depends not only on how failures are repaired, but also on which failures provide reliable evidence for persistent harness changes." —— p.1，Abstract

### 6.7 原文表格中与总体趋势不一致的数值（仅列原文数据）

| 内容 | 原文依据 |
|---|---|
| MiniMax-M2.7 在 τ²-Retail 上，w/o FDCR AVG 为 100.00%，w/ FDCR 为 96.67% | p.18，Table 6 |
| MiniMax-M2.7 在 τ²-Airline 上，w/o FDCR 的 Pass^3 为 80.00%，w/ FDCR 为 70.00% | p.18，Table 6 |
| Qwen3-8B 在 AgentBench 上，SE 与 w/ FDCR 均为 90.00%，w/o FDCR 为 85.00% | p.19，Table 7 |
| τ²-Retail 上 MSA 比例 SE 43.75% → w/ FDCR 50.00%（略升） | p.14，Appendix A 第 2 段 |
| Table 5 中 Reduced 的 Pass^3（45.00）低于 SE（55.00） | p.9，Table 5 |

---

## 7. 适用条件

> 原文**没有单列"适用条件/适用范围"章节**。以下仅列出原文明确陈述的方法前提与实验设定。

| 条件 | 原文依据 |
|---|---|
| 任务模型参数与运行环境在训练期间保持固定（冻结模型下的 harness 演化）；与联合演化模型和 harness 的方法（如 HarnessForge）不同 | p.3，§3.2 第 1 段；p.6，"Across all methods..." 段 |
| 需要一个固定的评估框架为每条轨迹给出任务分数 S(τ)，并据此用阈值 λ 判定失败 | p.4，§3.3 第 1 段 |
| 需要可记录执行轨迹（任务 ID、失败判定、终止原因、工具调用历史、执行上下文） | p.4，§3.3 第 1 段 |
| 编码代理需能访问当前 harness 的源代码以实施修改 | p.5，§3.4 第 1 段 |
| 跨任务证据的前提：失败组需覆盖至少两个不同任务才被优先处理 | p.5，§3.3 延续段 |
| 聚合带来的调用次数节省，仅在某轮含多条失败记录时为严格减少 | p.5，§3.3 延续段 |
| 候选接受只看训练集总分是否严格提升 | p.4，§3.2；p.16，Appendix C |
| 已验证的任务类型：AgentBench（较简单任务）与 τ²-Bench Airline/Retail（需通过工具交互持续修改环境状态的较复杂任务） | p.5–6，§4.1 Datasets 段 |
| 已验证的模型范围：上述 5 个任务模型；演化仅用 Qwen3-8B | p.5，§4.1；p.16，Appendix C |
| 与 HarnessCompass 不同，ECDYSIS 不对演化搜索空间施加硬约束；MSA 是被"缓解"而非"禁止" | p.3，§2 Related Work 第 1 段 |
| 在其他基准、其他 harness 框架或其他编码代理上的适用性 | **未说明** |
| λ、R、K 等超参数的选择原则或敏感性 | **未说明** |

---

## 8. 研究局限

> 原文**没有单列"Limitations/局限"章节**。以下仅列出原文中作者自己明确承认或陈述的不足。

| 局限 | 原文依据 |
|---|---|
| 仅靠复发（recurrence）可能过度放大复发的模型特定失败："recurrence alone may over-amplify recurring model-specific failures" | p.2，Contributions 第 1 条；p.10，§7 Conclusion |
| 仅凭复发不足以可靠区分"复发的模型局限"与"harness 缺陷"："recurrence alone is insufficient for reliably distinguishing recurring model limitations from harness deficiencies" | p.2，Introduction 第 5 段 |
| FDCR 带来额外演化期计算开销（"at the cost of additional evolution-time computation"；"incurs computational overhead"） | p.2，Introduction 第 5 段；p.2，Contributions 第 2 条；p.10，§7 |
| 效率与归因可靠性之间存在权衡（"a clear trade-off between efficient evidence aggregation and reliable failure attribution"） | p.2，Introduction 第 5 段 |
| MSA 被缓解但未被消除 | p.9，§6.2 第 1 段；p.3，§2 |
| 不同数据集上 MSA 比例差异较大，修改倾向依赖任务结构与失败机制；τ²-Retail 上 MSA 比例相对 SE 略有上升 | p.14，Appendix A 第 2 段 |
| 被分类为 MSA 的修改不必然意味着过拟合，也可能有效且可跨任务复用 | p.14，Appendix A MSA 定义条目 |
| 由于 agent 评估复杂度高、成本高，实验设置与规模沿用已有工作 [32] | p.6，"Across all methods..." 段 |
| 运行时间同时反映模型服务延迟与轨迹长度，因此需看完整均值与标准差 | p.8，Inference Efficiency 段（延续） |
| 编码代理调用的稳定性结论限定在"in our experiments" | p.18，Appendix E.1 第 2 段 |
| 人工标注专家均为本文作者，无外部参与者 | p.10，Ethical Considerations |
| 作者对其他潜在局限（如更多基准、更大规模、超参数敏感性、显著性检验）的讨论 | **未说明** |

---

## 附：原文未说明的关键信息汇总

| 信息 | 状态 |
|---|---|
| 失败阈值 λ 的具体取值 | 未说明 |
| 各数据集训练集 / 留出集的任务数量 | 未说明 |
| 数据研究（Table 5）所用的具体数据集 | 未说明 |
| 精简训练集 5 个任务的具体名单 | 未说明 |
| 演化过程的独立重复次数 / 随机种子 | 未说明 |
| 统计显著性检验 | 未说明 |
| 五名专家之间的一致性指标（如 Kappa） | 未说明 |
| 早停（early stopping）的具体规则 | 未说明（仅在 p.17 Appendix E 引言段提及三种方法使用相同的早停协议） |
| 单独的局限性章节 | 未说明（无此章节） |

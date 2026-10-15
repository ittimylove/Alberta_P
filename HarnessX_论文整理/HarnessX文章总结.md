# HarnessX 文章总结

**论文**：*HarnessX: A Composable, Adaptive, and Evolvable Agent Harness Foundry*（Darwin Agent Team，arXiv:2606.14249v3，2026-07-23）
**源文件**：`Chen_2026_HarnessX.pdf`（共 43 页）

**引用约定**
- 页码 = PDF 页脚上印的页码（与 PDF 页序一致）。
- 段落定位：段首有加粗小标题的，直接用小标题标注（如 “§6.1 *Benchmarks* 段”）；没有小标题的，按该小节在该页上的顺序写“第 N 段”。
- 每条后面附一句英文原文短摘录，方便核对。
- 本文只依据论文本身。原文没写的内容标“未说明”。

---

## 1. 研究问题

| # | 内容 | 原文依据 |
|---|---|---|
| 1.1 | 现有 agent harness（提示词、工具、记忆、控制流）**靠手工搭建且是静态的**：换模型、换工具或换任务领域都要定制修改，没有从经验中自动改进的机制。 | p.3，§1 第 2 段：“harnesses are hand-engineered and static … with no mechanism for experience-driven improvement”；p.1 摘要 |
| 1.2 | harness **在架构上彼此纠缠**：提示模板、工具封装、重试策略和记忆写在同一段代码里，改一处会悄悄破坏其他部分，跨领域复用只能复制，不能组合。 | p.3，§1 第 2 段：“harnesses are architecturally entangled” |
| 1.3 | **harness 工程与模型训练彼此脱节**：改进 harness 时收集到的轨迹被丢弃，没有用于模型训练；模型的改进也不会反过来带动 harness 改进。 | p.3，§1 第 2 段：“harness engineering and model training operate independently” |
| 1.4 | 由此提出的核心问题：能否把 harness 当作一等对象，与模型一起组合、自适应调整、演化，并找出这一过程中会出现的失败模式及相应的防御办法？ | p.3，§1 第 3 段：“treating the harness as a first-class object that can be composed, adapted, and evolved alongside the model”；p.5，§2.2 第 4 段（现有工作 “lack a unifying theoretical framework that connects observed failure modes to principled defenses”） |

## 2. 研究对象

| # | 内容 | 原文依据 |
|---|---|---|
| 2.1 | **agent 运行时 harness**，形式化为 H = (M, C)：M 是模型配置，C = (P, S) 是 harness 配置。P 是按 hook 索引的 processor 列表，S 是共享的槽位资源。 | p.5–6，§3.1 第 1–2 段 |
| 2.2 | **harness 的演化过程本身**：把它建模为符号制品上的 MDP（即“operational mirror”，操作镜像），并研究其中的三类病理：reward hacking、灾难性遗忘、探索不足。 | p.7–8，§4.1 定义 1–3；p.8，§4.2 引言段 |
| 2.3 | **harness 与模型的协同演化**：研究对象是 (M_t, H_t) 这一对，其中模型参数可以训练。 | p.12，§5.1 第 1 段 |
| 2.4 | 实验中的被测体：3 个 task agent 家族（Claude Sonnet 4.6、GPT-5.4、Qwen3.5-9B），以及 1 个 meta-agent（Claude Opus 4.6），在 5 个基准上测试。 | p.3，§1 第 4 段；p.15，§6.1 *Models* 段 |

## 3. 数据来源

| # | 内容 | 原文依据 |
|---|---|---|
| 3.1 | **GAIA**：固定 103 题，覆盖 Level 1–3（各级 39/52/12 题），评分方式为 exact match。 | p.15，表 3；p.28，附录 A.1 *GAIA* 段、A.2 |
| 3.2 | **ALFWorld**：valid-unseen 划分的全部 134 个任务，共 6 种任务类型，按目标是否完成评分。 | p.28，附录 A.1 *ALFWorld* 段；A.2 |
| 3.3 | **WebShop**：用固定随机种子抽取 100 个实例，每个实例是一次独立的购物会话，按属性匹配评分。 | p.28，附录 A.1 *WebShop* 段；A.2 |
| 3.4 | **τ³-Bench**：选取 Retail、Airline、Telecom 三个领域，对各领域的完整任务列表评分，按规则遵从度评分。 | p.28，附录 A.1 *τ³-Bench* 段；A.2 |
| 3.5 | **SWE-bench Verified**：抽样 55 题子集，按补丁是否解决问题评分。 | p.28，附录 A.1 *SWE-bench Verified* 段；A.2 |
| 3.6 | **自身生成的执行轨迹**：可观测层记录每次模型调用、工具调用和环境观测，构成 trace store（轨迹库），再由 Digester 压缩。例如，GAIA 单轮约产生 10M token 的原始轨迹。 | p.29，附录 A.5；p.10，§4.3 *Digester* 段 |
| 3.7 | **协同演化实验**：使用 GAIA 纯文本子集（103 题）和 WebShop 子集（100 题）。GAIA 的工具包括 Baidu API 网页搜索、web fetch、bash、文件读取。 | p.18，§6.5 *Experimental setup* 段 |
| 3.8 | 实验时间：2026 年 5 月；CC SDK 版本为 v0.0.25。 | p.15，脚注 1 |
| 3.9 | 数据或代码是否公开：给出了项目主页，并说明附录 prompt 摘自仓库中的对应文件，但**仓库地址和数据集的公开方式未说明**。 | p.1 摘要（项目主页）；p.29，附录 B 引言段（“literal contents of the corresponding files in the repository”） |

## 4. 核心变量

> 原文没有使用“自变量/因变量”这类术语。下表按实验设计归纳，每一项都对应原文中的具体设置或指标。

**自变量（被操纵的条件）**

| 变量 | 取值 | 原文依据 |
|---|---|---|
| harness 是否演化 | Static Harness（固定基线） vs. AEGIS 演化后的 harness | p.15，§6.1 *Baselines* 段 (1) |
| task agent 模型 | Sonnet 4.6 / GPT-5.4 / Qwen3.5-9B | p.15，§6.1 *Models* 段 |
| 基准 | 共 5 个（见 §3） | p.15，表 3 |
| 演化策略 | Global（单一 harness） vs. Ensemble / 变体隔离（最多 K 个变体） | p.17，§6.3 引言段、表 5 |
| evolver 架构 | AEGIS 四阶段 vs. 单 agent 的 CC SDK evolver | p.18，§6.4 引言段、表 6 |
| 优化方式 | 仅演化 harness（模型冻结） vs. 协同演化（AEGIS + cross-harness GRPO） | p.18，§6.5 引言段 |

**因变量 / 观测指标**

| 指标 | 定义 | 原文依据 |
|---|---|---|
| pass@2 成功率（主指标） | 每题跑两次，任意一次成功即算通过；用无偏估计量计算（式 6）；因基础设施故障中断的运行记为失败 | p.15，§6.1 *Metrics* 段；p.29，附录 A.3 |
| Δ、Best round | 演化后峰值减去初始值，以及达到峰值的轮次（Evolved = peak accuracy） | p.16，表 4 标题 |
| Final − Peak | 稳定性指标；负值表示发生灾难性遗忘 | p.17，表 5 标题 |
| Tokens | meta-agent 的 token 消耗 | p.17，表 5；p.22，表 7 |
| hit rate / ship-prediction accuracy | 实际翻转的任务数 / manifest 中预测会翻转的任务数 | p.20，§6.6 *Under-exploration* 段；p.37，附录 D 引言段 |
| 单任务推理 token 消耗 | 演化后每个任务的推理成本变化 | p.22，§7.5 第 3 段 |

**关键超参数**（原文表 8 的版式在提取时有错位，下面按符号与含义逐项对应）

| 参数 | 取值 | 原文依据 |
|---|---|---|
| 最大演化轮数 T | 15 | p.15，§6.1 *Benchmarks* 段 |
| 早停耐心 P | 3 | p.15，§6.1 *Benchmarks* 段 |
| 每轮候选数 K_t | 4 | p.29，表 8 |
| 每个单元的随机种子数 | 3 | p.29，表 8 |
| 噪声阈值（忽略单轮通过数变化） | ±5% | p.29，表 8 |
| H0 | 手工构建的基础 harness | p.29，表 8 |
| 任务并发数 | 10 | p.29，表 8 |
| max-steps | GAIA 20 / WebShop 20 / ALFWorld 15 / τ³ 200 / SWE 200 | p.29，表 8 |
| meta-agent 总 token 预算 | 100M–175M | p.15，§6.1 *Benchmarks* 段 |
| 协同演化的奖励 | GAIA：0.9×正确性 + 0.1×格式；WebShop：原生属性匹配奖励 | p.18–19，§6.5 *Experimental setup* 段 |
| GRPO 训练设置 | 学习率 1e-6、clip ε=0.2、KL 系数 0、每轮 5 步 | 同上 |

## 5. 研究方法

| # | 方法组成 | 要点 | 原文依据 |
|---|---|---|---|
| 5.1 | **类型化组合（Harness Composition）** | processor 协议 `process(event) -> AsyncIterator[Event]`，共 5 种结果：透传、变换、拆分、拦截、中断。processor 挂在 8 个生命周期 hook 上，每个 hook 有允许修改的字段（表 1）。元数据 `_singleton_group`、`_order`、`_after` 约束 processor 之间如何组合；运行循环在每次调用后校验 hook 契约。 | p.5，表 1；p.6，§3.2 第 1–3 段 |
| 5.2 | **九维分类法** | D1 模型选择、D2 上下文组装、D3 记忆、D4 工具、D5 执行环境、D6 评估与奖励、D7 控制与安全、D8 可观测性、D9 训练桥。 | p.6，§3.3 第 1 段 |
| 5.3 | **操作镜像（Operational Mirror）** | 状态 s_t = (H_t, T_t)，动作是类型化的代码级编辑，反馈是轨迹加 verifier 分数，状态转移由确定性接受算子 U 决定。U 执行 seesaw 约束：候选不得让任何已解决的任务退化。 | p.7，表 2；p.8，定义 2–3、*MDP instantiation* 段 |
| 5.4 | **AEGIS 四阶段流水线** | 由同一个 meta-agent 选择性调用 4 个阶段。**Digester** 把轨迹压缩成逐任务摘要。**Planner** 构建 adaptation landscape（调整全景），用来防止探索不足。**Evolver** 生成带 change manifest（变更清单）的类型化编辑，涉及新代码时必须附 smoke test。**Critic** 防范 reward hacking，最多发起一次修订请求。**确定性门控**依次检查：manifest 完整性 → 配置规范化 → 构建/冒烟测试 → seesaw 约束，用来防止灾难性遗忘。只有确定性检查能决定是否上线。 | p.9，算法 1、§4.3 引言段；p.10，*Digester*、*Planner*、*Evolver*、*Critic and gating* 各段；p.11，*Design principle* 段 |
| 5.5 | **变体隔离（Ensemble routing）** | 最多维护 K 个 harness 变体，每个任务路由到在该任务所属簇上历史成功率最高的变体。如果一个编辑改善了部分任务、又让另一部分退化，就分叉出新变体，而不是直接拒绝；seesaw 约束按变体分别生效。 | p.11，§4.5 第 1–2 段 |
| 5.6 | **harness–模型协同演化** | 共享一个 FIFO replay buffer，每轮流程为：rollout → 固定 verifier 打分 → 写入 buffer → AEGIS 更新 harness → 缓存行为策略的 log-prob → GRPO 更新模型。 | p.12，§5.1 步骤 1–7 |
| 5.7 | **Cross-harness GRPO** | 同一任务的所有轨迹归为一组，不论由哪个 harness 或哪个模型版本产生（式 2）；组内计算相对优势（式 3），目标为带 clip 和 KL 项的目标函数（式 4–5）。对齐是任务级而非动作级；每条轨迹都在生成它的那个 H_k 的上下文下重新计算 log-prob。 | p.13，§5.3 式 1–3、*Task-level alignment* 段；p.14，式 4–5 段 |
| 5.8 | **离策略训练** | π_old 在轨迹写入 buffer 时计算一次并缓存。FIFO 容量 C 加上每轮 s 个样本，使模型版本滞后不超过 C/s 轮。GRPO 不额外做 rollout。 | p.14，§5.4 *Behavior policy*、*Bounded off-policy bias*、*Replay reuse* 段 |
| 5.9 | **实验设计** | 主实验：Static 基线 vs. 演化，覆盖 15 个“模型×基准”组合（Global 策略）。消融实验：Global vs. Ensemble（表 5）；AEGIS vs. CC SDK（表 6）；协同演化 vs. 仅 harness（图 5）。另有失败案例分析（图 6），以及附录 D 按基准给出的失败簇和杠杆有效性分析。 | p.14，§6 引言段；p.15–21，§6.2–6.6；p.37–43，附录 D |

## 6. 主要结论

| # | 结论 | 原文依据 |
|---|---|---|
| 6.1 | 15 个组合中有 14 个得到改进，**平均绝对提升 +14.5%，最高 +44.0%**（ALFWorld 上的 Qwen3.5-9B：53.0→97.0）。 | p.15，§6.2 引言段；p.16，表 4；p.1 摘要 |
| 6.2 | **逆缩放**：基线越弱，提升越大。Qwen3.5-9B 在 ALFWorld +44.0、GAIA +17.1、SWE +18.2；τ³-Bench 上 Qwen 基线已接近天花板（93.5%），只提升 +1.1。 | p.15，§6.2 *Inverse scaling* 段；*Overall performance* 段 |
| 6.3 | **跨模型家族泛化**：Opus 4.6 作为 meta-agent，不需要针对模型家族做适配。提升幅度跟随基线水平，与 task agent 是否和 meta-agent 同属一个家族无关。 | p.17，§6.2 *Cross-model generalization* 段；p.22，§7.4 第 1–2 段 |
| 6.4 | **收敛速度取决于失败模式的集中程度**：失败集中在 1–2 类组件时，2–4 轮就达到峰值；GAIA（Sonnet）的失败涉及 4 类组件，需要 11 轮。 | p.17，§6.2 *Convergence rate* 段 |
| 6.5 | 在异质任务集上，**Global 策略会停滞甚至退化**：GAIA GPT-5.4 在 R4 达到峰值 73.8%，之后跌到 49.5%（−24.3，超出 95% 置信区间 ±8.5）。**变体隔离**下终值等于峰值，为 87.4%（+13.6），token 消耗更少（107.8M vs. 143.7M）。 | p.17，表 5、*Failure mechanism of Global* 段、*Why Ensemble* 段；p.18，*Summary* 段 |
| 6.6 | **AEGIS 与单 agent 的 CC SDK evolver 准确率相当**（87.4 vs. 86.4，差距在一个标准误之内），但 AEGIS 少用约 12% 的 token。准确率提升主要来自基础设施（类型化组件、结构化轨迹），而不是 evolver 的内部架构。 | p.18，§6.4 *Accuracy is comparable* 段、*Implication* 段 |
| 6.7 | **协同演化比仅演化 harness 平均再高 +4.7%**（GAIA 37.4→41.7，WebShop 49.0→54.0），优势一直保持到最后一轮，突破了“脚手架天花板”。 | p.19，§6.5 *Co-evolution exceeds* 段、*breaks the scaffolding ceiling* 段 |
| 6.8 | 操作镜像预言的**三种病理在实验中都出现了**：reward hacking（GAIA R10–R12，两轮内被检出并修复）；灾难性遗忘（τ³ Telecom R7 下降 −14.0，到 R9 自愈）；探索不足（ALFWorld R4–R7，ship-prediction 准确率从 80% 降到 0%）。 | p.19–21，§6.6 各案例段及 *Summary* 段；p.20，图 6 |
| 6.9 | **轨迹的丰富程度决定了能安全进行多复杂的演化**：仅凭标量奖励，三种病理都无法识别。结构化轨迹对检测病理是必要条件，但不足以预防病理。 | p.21，§7.2 第 1–2 段 |
| 6.10 | 演化成本可以摊销：GAIA 上单任务推理 token 下降约 25%，约 1,300 次调用后收回 107.8M 的前期成本；ALFWorld 上单任务成本上升约 60%，换来的是准确率提升。 | p.22，§7.5 第 3–4 段 |
| 6.11 | 总结论：agent 的进步不必只依赖模型规模；从执行反馈中组合和演化运行时接口，是一个互补且可操作的杠杆，对能力受限的 agent 尤其有效。 | p.23，§8；p.1 摘要 |

## 7. 适用条件

| # | 条件 | 原文依据 |
|---|---|---|
| 7.1 | agent 的动作空间是**离散的、基于文本的**。 | p.23，§7.7 *Discrete action spaces only* |
| 7.2 | 需要一个**有能力的 meta-agent**，能做多文件代码生成、结构化轨迹分析和多步规划；实验中全部使用 Opus 4.6。 | p.23，§7.7 *Closed-source meta-agent*；p.29，附录 A.4 |
| 7.3 | 需要**固定的 verifier**，这样不同 harness 版本之间的奖励才可比，cross-harness 优势的计算依赖这一点。 | p.12，§5.1 步骤 2 |
| 7.4 | 需要**完整的执行轨迹可观测性**（D8），并且要有前几轮的轨迹可供比较。 | p.21，§7.2 第 2 段（“provided prior-round traces exist for comparison”）；p.10，§4.3 第 2 段 |
| 7.5 | 协同演化要求**同时控制 harness 演化和模型训练**，并能共享一个 replay buffer。 | p.23，§7.7 *Joint control assumption* |
| 7.6 | 演化很快收敛时（如 ALFWorld、SWE-bench），Global 策略就够用；**任务集异质、需要互相冲突的编辑时**，需要用变体隔离。 | p.22，§7.5 第 2 段；p.17–18，§6.3 |
| 7.7 | harness 提升对**较弱的模型**最明显，但存在能力下限：SWE-bench 上 Qwen3.5-9B 各杠杆的 hit rate 只有约 0.05，“evolution cannot compound”。 | p.15，§6.2 *Inverse scaling* 段；p.42，附录 D.5 *Did evolution close the clusters?* 段；p.43，图 12 标题 |
| 7.8 | 部署时，演化得到的 harness 是静态制品，不需要 meta-agent 推理；演化集以外的任务路由到总体成功率最高的变体。 | p.22，§7.5 第 4 段 |
| 7.9 | 增益是**在演化所用的同一任务集上测得的**。 | p.15，§6.1 *Scope* 段 |

## 8. 研究局限

**作者在 §7.7 中明确列出的五条**

| # | 局限 | 原文依据 |
|---|---|---|
| 8.1 | **没有 held-out 评估**：在演化集本身上报告峰值准确率，同时存在选择偏差和过拟合风险；对未见任务的泛化未经检验。 | p.23，§7.7 第 1 条；p.15，§6.1 *Scope* 段 |
| 8.2 | **只测试了离散动作空间**，没有测试连续控制（如机器人）。 | p.23，§7.7 第 2 条 |
| 8.3 | **meta-agent 是闭源模型**：没有测试开源权重模型（如 Qwen3.5-72B、Llama-4-Maverick）担任 meta-agent。 | p.23，§7.7 第 3 条 |
| 8.4 | **联合控制假设**：现实中 harness 和模型往往归属不同团队或组织，共享 buffer 不易实现。 | p.23，§7.7 第 4 条 |
| 8.5 | **基准覆盖有限**：SWE-bench 只用 55 题，τ³-Bench 只有 3 个领域；逆缩放等结论未必能推广。 | p.23，§7.7 第 5 条 |

**正文其他位置提到的局限**

| # | 局限 | 原文依据 |
|---|---|---|
| 8.6 | **逐编辑门控的结构性缺陷**：pass@2 只记录每个任务的二值翻转，低于阈值的耦合会悄悄累积，直到某一轮突然出现明显退化（τ³ Telecom R7）。 | p.17，§6.2 *Domain-level variation* 段；p.23，§7.6 末段 |
| 8.7 | **操作镜像只是设计启发，不是形式化框架**：没有收敛保证，无法预测哪种病理会占主导、何时出现、有多严重；三种病理具有代表性但不完备。 | p.21–22，§7.3 第 1–2 段 |
| 8.8 | 变体隔离的效果**是否能延伸到更长的演化周期**（变体可能过度专门化），或任务之间相互依赖的分布，没有保证。 | p.21，§7.3 第 1 段 |
| 8.9 | **没有评估较弱的 meta-agent** 能否取得相当的提升。 | p.22，§7.4 第 2 段 |
| 8.10 | 更细粒度的策略（Domain-aware clustering、Task-level tournament）只做了小规模试验（30–40 题，最多 8 轮），不足以得出统计结论。 | p.18，§6.3 *Summary* 段 |
| 8.11 | 四阶段分解**在当前规模下没有带来可测的准确率提升**，只带来效率和可审计性。 | p.18，§6.4 *Implication* 段 |
| 8.12 | SWE-bench 题数少（每翻转一题，总分变化约 1.8%），结构性代码编辑的影响面大：GPT-5.4 在峰值之后下降 −12.7。 | p.17，§6.2 *Post-peak degradation* 段 |
| 8.13 | 人在回路审批机制**在自动化实验中没有使用**。 | p.23，§7.6 第 3 条 |
| 8.14 | pass@2 会掩盖低于阈值的成功概率漂移。 | p.15，§6.1 *Metrics* 段 |

---

### 附：原文中需要对照核实的表述
以下几处只是对照原文时注意到的，不代表我对其正误下结论：
- **协同演化的训练批次**：§6.5 写 “set the optimizer batch to the entire replay buffer”，即 GAIA 824 条、WebShop 400 条（p.18）；附录 A.5 写 “batch size 256”（p.29）。两处说法不一致，原文没有说明二者的关系。
- **初始 harness 的来源**：§6.1 说 Static 基线 “constructed from published benchmark-specific prompts and tool definitions”（p.15）；附录 A.4 和表 8 说 round-0 是 “Handcrafted base harness … competent composed harness”（p.29）。原文没有说明二者是否为同一配置。
- **协同演化的 rollout 次数**：§6.5 写 “at two rollouts per task”，但 WebShop 的 buffer 按 100×1×4 计算（p.18）。原文没有说明 WebShop 每题实际跑几次。

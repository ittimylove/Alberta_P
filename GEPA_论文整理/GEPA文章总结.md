# GEPA（Genetic-Pareto）文章总结

- **论文**：GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning
- **作者 / 机构**：Lakshya A Agrawal, Shangyin Tan, Dilara Soylu, Noah Ziems, Rishi Khare, Krista Opsahl-Ong, Arnav Singhvi, Herumb Shandilya, Michael J Ryan, Meng Jiang, Christopher Potts, Koushik Sen, Alexandros G. Dimakis, Ion Stoica, Dan Klein, Matei Zaharia, Omar Khattab；UC Berkeley、Stanford、BespokeLabs.ai、Notre Dame、Databricks、MIT（p.1 标题区）
- **版本 / 发表**：arXiv:2507.19457v2 [cs.CL]，2026-02-14；页眉标注 "Accepted at ICLR 2026 (Oral)"（p.1 左侧页眉与页首）
- **唯一信息来源**：`Agrawal_2025_GEPA.pdf`（共 96 页，PDF 页码与论文印刷页码一致；正文 p.1–15，参考文献 p.15–21，附录 p.22–96）

> **标注说明**
> - 依据统一写作「p.页码，章节，第 N 段（段首原文）」。段落按该页中该章节的正文段落计数，图、表、算法框的标题不计入段落；跨页的段落按它开始的那一页计。
> - 「未说明」表示原文没有给出这项信息。
> - 本文由 pdftotext 抽取的文本整理而成。PDF 中的希腊字母（Φ、Π、Θ、μ 等）在抽取时丢失，下文按公式结构写出，符号含义以原文 p.4 §2 为准。Table 1 预算行和 Table 4 在 layout 模式下错位，已用 raw 模式重新核对；图中只有曲线、没有数值标注的部分不引用具体数字。

---

## 1. 研究问题

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 核心问题 | 每次 rollout 都很昂贵时，怎样从每次 rollout 中提取尽可能多的学习信号，让复杂、模块化的 AI 系统在低数据或受预算约束的条件下有效适配？ | p.4，§2 Problem Statement 末段："The core challenge, then, is: How do we extract maximal learning signal from every expensive rollout to enable effective adaptation of complex, modular AI systems in low-data or budget-constrained settings?" |
| 问题背景 1 | 以 GRPO 为代表的 RLVR 把成功指标当作 rollout 结束时的标量奖励来估计策略梯度，实际中通常需要数万次 rollout 才能拟合新任务，有的工作用到数十万次 | p.2，§1 Introduction 第 1 段（"Large language models (LLMs) have enabled…"） |
| 问题背景 2 | 样本低效会成为瓶颈：很多下游应用调用昂贵的工具、推理预算有限，或者根本无法微调最大、最好的 LLM 的权重 | p.2，§1 第 1 段末句（"This sample inefficiency can quickly become a serious bottleneck…"） |
| 作者的核心主张 | rollout 可以序列化为自然语言轨迹；与从稀疏标量奖励得到的策略梯度相比，在自然语言中反思这些轨迹能更好地利用 LLM 的语言先验 | p.1，Abstract（"We argue that the interpretable nature of language often provides a much richer learning medium…"）；p.2，§1 第 2 段（"We observe that rollouts sampled from even highly sophisticated LLM systems can be serialized into traces of natural language…"） |
| 待检验的命题（论文标题） | 反思式 prompt 进化能否胜过强化学习 | p.1，标题；p.1，Abstract（"To test this, we introduce GEPA…"） |

## 2. 研究对象

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 研究对象 | **复合 AI 系统（compound AI system）**：由一次或多次 LLM 调用组成、可与外部工具调用交错、由任意控制流编排的模块化系统。涵盖 agents、多智能体系统、ReAct、Archon 等脚手架 | p.2–3，§2 第 1 段（"We follow related work in defining a compound AI system as…"） |
| 形式化 | 系统 Φ = (M, C, X, Y)：M 是语言模块集合，C 是控制流逻辑，X、Y 是全局输入 / 输出 schema。每个模块 M_i = (π_i, θ_i, X_i, Y_i)，π_i 是 prompt（含指令和 few-shot 示例），θ_i 是模型权重 | p.4，§2 第 1 段（续段 "chon (Saad-Falcon et al., 2025)…we formalize such a system as…"） |
| 优化变量 | 一般形式允许同时更新 prompts Π 和权重 Θ（为了能与 GRPO 等在不同参数空间工作的方法比较）；**GEPA 只进化 prompts Π，LLM 权重 Θ 保持固定** | p.4，§2 第 2–3 段（"Given Φ, let Π…"；"We adopt this general problem formulation…"）；p.4，§3 第 1 段末句（"Note that GEPA evolves only the set of prompts…whereas the underlying LLM weights…remains fixed."） |
| 具体被优化的系统 | HotpotQA：改造后的 HoVerMultiHop 程序（最后一跳改为回答问题）；IFBench：两阶段系统（先回答，再按约束改写）；AIME-2025、LiveBench-Math：单步 ChainOfThought；HoVer：最多 3 跳检索的 HoverMultiHop 程序（2 个 query writer 模块 + 2 个文档摘要模块）；PUPA：PAPILLON（用户 query 改写器 + 回答改写器，中间调用一次不可信模型） | p.23–25，Appendix E.1 各基准对应段落 |
| 扩展应用中的对象 | 推理时搜索：AMD XDNA2 NPU 内核代码（NPUEval）、NVIDIA V100 的 CUDA 代码（KernelBench）；对抗 prompt 搜索：AIME 上的通用对抗指令 | p.12，§5.1 第 1 段；p.12，§5.2 第 1 段 |

## 3. 数据来源

### 3.1 主实验基准与划分

| 基准 | 任务 | 训练 / 验证 / 测试规模 | 原文依据 |
|---|---|---|---|
| HotpotQA | 多跳问答（原数据集为基于 Wikipedia 的 113K 问答对） | 150 / 300 / 300 | p.23，Appendix E.1 "HotpotQA" 段 |
| IFBench | 精确指令遵循（输出约束）。训练 / 验证集取自 IF-RLVR Train，测试集为 IFBench（58 个新的分布外约束） | 150 / 300 / 294 | p.23，Appendix E.1 "IFBench" 段；p.24，续段（"We split the IF-RLVR Train into our train/val sets…"） |
| AIME-2025 | 数学竞赛。用 2022–2024 年共 90 题对半分为训练集和验证集；AIME-2025 的 30 题每题重复 5 次作为测试集 | 45 / 45 / 30×5（训练与验证按"对半"推算为各 45） | p.24–25，Appendix E.1 "AIME-2025" 段 |
| LiveBench-Math | LiveBench 数学子集，2025-07-30 获取，n=368，以 Python 随机种子 0 打乱后三等分 | 三等分（每份具体条数未说明） | p.25，Appendix E.1 "LiveBench-Math" 段 |
| HoVer | 开放域多跳事实抽取与声明验证，评测检索出全部 gold 文档的能力 | 150 / 300 / 300 | p.25，Appendix E.1 "HoVer" 段 |
| PUPA | 隐私敏感委托（在可信与不可信模型之间处理真实用户查询，尽量少泄漏 PII） | 111 / 111 / 221 | p.25，Appendix E.1 "PUPA" 段 |

- 基准来源：大部分取自 Tan et al. (2025)，并配有现成的复合 AI 系统（p.23，Appendix E.1 第 1 段，"we assemble a diverse suite of benchmarks mostly obtained from Tan et al. (2025)…"）。
- 数据访问规则：优化器可完整访问训练集的文本和标签；验证集只能看分数（如用于早停），不能直接看内容（p.8，§4 第 1 段，"We adopt a standard train/validation/test split…"）。
- GEPA 内部划分：训练集作为 D_feedback（产生学习信号），验证集作为 D_pareto（只用于候选选择）（p.27，Appendix E.4 "GEPA" 段；p.6，Algorithm 1 第 1 行）。

### 3.2 其他数据来源

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 反馈文本 | 由反馈函数 μ_f 从评测过程中提取的文本轨迹（如编译器报错、未满足的评分细则），各基准的反馈内容分别为：剩余待检索文档（HotpotQA、HoVer）、已满足和未满足的约束描述（IFBench）、质量分与 PII 泄漏分的分解（PUPA） | p.6，§3 "Evaluation traces as diagnostic signals" 段；p.23–25，Appendix E.1 各基准段落 |
| 人工反馈（可选） | 有人工评分员时，可把人写的解释加入 D_train，作为辅助 feedback_text | p.7，§3 续段（"In practice, there are domains where human-graders are able to rate…"） |
| NPUEval | 使用 NPUEval 的早期版本；反馈函数根据失败信息（如编译错误）检索技术手册中的相关章节 | p.12，§5.1 第 1–2 段 |
| KernelBench | "representative subset" 中的 35 个任务，覆盖三个难度级别 | p.12，§5.1 "CUDA Kernels" 段 |
| 对抗搜索数据 | 用 AIME 2022–2024 题目作为进化池，在 AIME-2025（30 题）上评估，GPT-5 Mini 每题跑 5 次（共 150 次生成） | p.12–14，§5.2 第 1 段 |
| 代码公开位置 | https://github.com/gepa-ai/gepa | p.1，Abstract 末句 |
| 数据许可 / 数据污染检查 | 未说明 | — |

## 4. 核心变量

### 4.1 自变量（被操纵的因素）

| 变量 | 取值 / 内容 | 原文依据 |
|---|---|---|
| 优化方法 | Baseline（不优化）、GRPO（LoRA；另有全参数微调对照）、MIPROv2、MIPROv2-No-Demos、Trace (OptoPrime)、TextGrad、GEPA、GEPA+Merge | p.8，Table 1、Table 2；p.25–27，Appendix E.4；p.28，Figure 11 标题 |
| 候选选择策略（消融） | SelectBestCandidate（总选当前最优）、BeamSearch (N=4)、GEPA 的 Pareto 采样；保持其余进化流程不变 | p.10，Table 3 标题；p.10，Observation 3 正文第 1 段（"GEPA refines prompts iteratively…"） |
| 是否启用 Merge（系统感知交叉） | GEPA vs. GEPA+Merge | p.11，Observation 5；p.23，Appendix D.1 |
| 基座模型 | Qwen3 8B（开源）、GPT-4.1 Mini（商用）；扩展实验用 GPT-4o、GPT-5 Mini | p.8，§4 第 1 段；p.25，Appendix E.2；p.12，§5.1；p.12，§5.2 |
| 跨模型迁移 | 在 Qwen3-8B 上优化的 prompt 直接拿到 GPT-4.1-Mini 上评估（GEPA-Qwen-Opt） | p.8，Table 2 标题；p.11，Observation 6 |
| rollout 预算 | GEPA 的预算按基准对齐到 MIPROv2 实际用量（差异不超过 10.15%）；GRPO 固定 24,000 次 | p.27，Appendix E.4 "GEPA" 段；p.26，Appendix E.4 "GRPO" 段 |
| 任务 / 基准 | 六个基准（见 3.1） | p.8，§4 第 1 段 |

### 4.2 因变量（结果指标）

| 变量 | 定义 | 原文依据 |
|---|---|---|
| 任务得分 μ | 取值 [0,1] 的指标，如 exact match、F1、通过率等，各基准具体用哪一种未逐一说明 | p.4，§2 第 2 段（"A metric μ…measures the output quality…(for example by calculating, exact match, F1, pass rate, etc.)"） |
| 聚合得分与提升幅度 | 六个基准的 Aggregate 及相对 Baseline 的 Improvement | p.8，Table 1、Table 2 |
| 样本效率 | 达到最优测试表现、或追平 GRPO 最佳验证分所需的 rollout 次数（总 rollout 和仅训练集 rollout） | p.9，Observation 1 第 1–2 段 |
| 泛化差距 | 最终测试集表现与最佳验证表现之差（沿用 Wan et al., 2024） | p.9，Observation 2 第 1 段；p.30，Figure 16 标题 |
| prompt 长度 | 优化后系统的 prompt token 数，作为下游成本代理 | p.11，Observation 4；p.27，Appendix I |
| NPU 向量利用率 | mean vector utilization (%) | p.12，§5.1 "NPU Kernels" 段；p.13，Figure 7 |
| fast_p | 生成的内核比 PyTorch-eager 基线快 p 倍以上的任务比例（p=0.5、1） | p.13，Figure 8 标题 |
| pass@1（对抗搜索） | 对抗 prompt 下 AIME-2025 的 pass@1 | p.14，§5.2 续段（"The learned adversarial instruction reduced pass@1…"） |
| 反思 LM 调用次数 | 优化过程中调用反思 LM 的总次数 | p.96，Appendix N，Table 4 |
| 金钱成本 | 运行实验的美元成本 | p.25，Appendix E.3 |

### 4.3 控制变量（保持不变的条件）

| 变量 | 设定 | 原文依据 |
|---|---|---|
| 模型权重 | GEPA 实验中保持固定 | p.4，§3 第 1 段末句 |
| 推理设置 | 上下文窗口最多 16384 tokens；Qwen3 8B：temperature 0.6、top-p 0.95、top-k 20；GPT-4.1 Mini：temperature 1.0；同一系统内所有模块用同一模型 | p.25，Appendix E.2 |
| 程序结构与初始 prompt | Trace/TextGrad 的程序结构与 DSPy 实现完全相同，使用相同的初始 prompt、相同的数据划分、相同的指标和反馈函数；未优化的 Trace 基线与 DSPy 基线相差不超过 0.5% | p.26，Appendix E.4 "Trace and TextGrad" 段 |
| GEPA 超参数 | minibatch 大小 3；启用 Merge 时最多调用 5 次；GPT-4.1 Mini 和 Qwen3 8B 使用相同超参数 | p.27，Appendix E.4 "GEPA" 段；p.11，Observation 5 第 2 段 |
| 推理时搜索中的随机性 | 用缓存消除温度采样带来的随机性 | p.12，§5.1 第 2 段末句（"We also note that generation stochasticity…is eliminated by operating under a cache…"） |

## 5. 研究方法

### 5.1 GEPA 算法

| 组成 | 内容 | 原文依据 |
|---|---|---|
| 三条设计原则 | 遗传式 prompt 进化；利用自然语言反馈进行反思；基于 Pareto 的候选选择 | p.4，§3 第 1 段（"…motivated by three core principles…"） |
| 输入 | 用简单 prompt 实例化的系统 Φ、训练集 D_train、评测指标 μ、反馈函数 μ_f、总 rollout 预算 B | p.4，§3 第 1 段 |
| 遗传优化循环 | 候选池 P 初始只含基础系统。每轮：(i) 选择有前景的候选；(ii) 通过反思变异或交叉产生变体，在 minibatch 上评估；(iii) 若优于父代，则加入 P 并记录谱系，再在 D_pareto 上评估。预算耗尽后返回 D_pareto 上平均分最高的候选 | p.4，§3 "Genetic Optimization Loop" 段；p.5，续段（"(iii) if it outperforms its parent(s)…"）；p.6，Algorithm 1 |
| 反思式 prompt 变异 | 在 minibatch 上执行所选候选并记录轨迹；调用 μ_f 得到分数和文本反馈；按 round-robin 选择要更新的模块；把（当前 prompt、轨迹、分数、反馈）交给反思 LM，让它把成败归因到 prompt 元素并提出新指令；新程序在 minibatch 上得分提升才加入候选池 | p.5，§3 "Reflective Prompt Mutation" 段及下一段（"Given a candidate to mutate…"）；p.6，续段（"elements and propose revised instructions…"） |
| 隐式信用分配 | LLM 通过反思执行轨迹，把最终结果的责任归到相关模块上，从而对单个模块做有针对性的更新 | p.5，§3 "Reflective Prompt Mutation" 段 |
| 评测轨迹作为诊断信号 | 把奖励 μ 扩展为反馈函数 μ_f，在返回分数的同时返回评测过程中产生的文本（feedback_text），可以是模块级的 | p.6，§3 "Evaluation traces as diagnostic signals" 段；p.7，续段 |
| 元提示（meta prompt） | 给出当前指令、minibatch 的输入 / 输出 / 反馈，要求写新指令：推断任务格式与描述，收录领域特定事实，并纳入可泛化的策略 | p.22–23，Appendix C |
| Pareto 候选选择 | 对每个训练实例记录所有候选中的最高分，形成 Pareto 前沿；保留至少在一个实例上最优的候选，剪除被严格支配的候选；按每个候选"领先"的实例数加权随机采样 | p.7，§3.1 第 2 段（"To address this, GEPA employs a Pareto-based 'illumination' strategy…"）；p.8，续段；p.6，Algorithm 2 |
| 为什么不用贪心 | 总选最优候选容易陷入局部最优，在一个策略上反复尝试改进失败，直到耗尽预算 | p.7，§3.1 第 1 段（"GEPA is a highly modular algorithm…"）；p.10，Figure 6 标题 |
| Merge（系统感知交叉） | 选两个共享祖先、各自优化了不相交 prompt 集合、都是 Pareto 最优、且都优于祖先聚合表现的候选，逐模块挑选更好的版本合成新候选；条件严格，所以合并不常发生 | p.23，Appendix D.1；p.24，Figure 9（Algorithm 3、4） |
| 反思 LM 用的是哪个模型 | 未说明（Table 4 按 GPT-4.1-Mini、Qwen3-8B 两列列出调用次数，但正文没有写明反思 LM 是否就是任务模型本身） | p.96，Appendix N，Table 4 |

### 5.2 实验设计

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 主对比 | 六个基准 × 两个模型，比较 GEPA 与 MIPROv2、Trace、TextGrad、GRPO | p.8，§4 第 1 段；p.9，§4 第 1 段（续） |
| 预算对齐 | 先记录 MIPROv2 在各基准实际用掉的 rollout 数，再把 GEPA 限制在该预算内，差异不超过 10.15%；Trace/TextGrad 使用与 MIPROv2、GEPA 相同的预算 | p.27，Appendix E.4 "GEPA" 段；p.26，Appendix E.4 "Trace and TextGrad" 段 |
| MIPROv2 配置 | auto=heavy（18 个指令候选、18 组 few-shot），rollout 数在 2270（PUPA）到 6926（HoVer）之间 | p.26，Appendix E.4 "MIPROv2" 段 |
| GRPO 配置 | 复合系统用 Ziems et al. (2025) 的实现；group size 12，每步 4 个实例；LoRA rank 16、α=64、dropout 0.05；学习率 1e-5，β=0.01；500 步共 24,000 rollouts，每 20 步验证一次用于早停；1×H100/A100 80GB。单模块用 SkyRL 全参数微调（group size 16，学习率 1e-6，FSDP2，8 GPU）。超参数 [LR, beta, norm clipping] 经人工探索 | p.26，Appendix E.4 "GRPO" 段；p.27，第 1 段；p.9，脚注 1 |
| 消融 | 候选选择策略消融（Table 3）；Merge 有无（Table 1、2） | p.10，Table 3；p.11，Observation 5 |
| 定性分析 | 展示 PUPA 优化轨迹中每步 prompt 的变化；给出所有基准的最优 prompt 并与 MIPROv2 对比 | p.7，Figure 5；p.27，Appendix K；Appendix L（p.41 起） |
| 推理时搜索 | 把待解任务集合同时作为 D_train 和 D_pareto，让 GEPA "过拟合"该任务集并在任务间迁移经验 | p.12，§5.1 第 1 段 |
| 对抗 prompt 搜索 | 反转奖励信号，让优化器添加琐事等信息来最小化 pass@1，同时要求 prompt 不与任务矛盾、仍包含解题所需的全部信息 | p.12，§5.2 第 1 段 |
| 统计显著性检验 / 多次运行的方差 | 未说明 | — |

## 6. 主要结论

| 结论 | 关键数据 | 原文依据 |
|---|---|---|
| **1. 反思式 prompt 进化样本效率高，可以胜过权重空间 RL** | 在 Qwen3 8B 上，GEPA 在 6 个任务中的 5 个超过 GRPO（24,000 rollouts），平均 +6%，最多 +20%（Observation 1 写作 "up to 19%"），rollout 最多少 35 倍；追平 GRPO 最佳验证分只需 243、402、330、1143、1179、306 次 rollout（最高 78 倍样本效率）；只算训练集 rollout，达到最优表现需要 79–737 次 | p.1，Abstract；p.2，§1 第 4 段；p.9，Observation 1 第 1–2 段 |
| 例外 | AIME-2025 上 GRPO（38.00）高于 GEPA（32.00） | p.8，Table 1 及其标题（"…on all benchmarks except AIME"） |
| **2. 只优化指令即可胜过指令 + few-shot 联合优化** | GEPA 在所有设置中都优于 MIPROv2，最大差距为 GPT-4.1 mini +11.1%、Qwen3 8B +10.3%；GPT-4.1 Mini 上聚合提升：GEPA +12.19%、GEPA+Merge +13.33%，MIPROv2 +5.64% | p.9，Observation 2 第 1 段；p.8，Table 2 |
| 泛化差距 | 反思进化出的指令泛化差距更小；作者推测原因是新一代 LLM 更擅长遵循指令和自我反思 | p.9，Observation 2 第 1 段；p.30，Figure 16 标题 |
| **3. Pareto 候选选择明显更好** | Qwen3 8B 上聚合提升：GEPA +12.44%，SelectBestCandidate +6.05%，BeamSearch +5.11% | p.10，Table 3 及其标题；p.10，Observation 3 正文第 1 段 |
| **4. 指令型 prompt 更短、更便宜** | GEPA 的 prompt 最多比 MIPROv2 短 9.2 倍；Appendix I 称约短 33%，Figure 17 标题称约为 MIPROv2 的 33% 以下（两处表述不一致，均为原文说法）；整体上表现越好的优化器产生的 prompt 越短 | p.11，Observation 4 第 2–3 段；p.27，Appendix I；p.30，Figure 17 标题 |
| **5. Merge 可能带来大幅提升，但不稳定** | GEPA+Merge 最多比 GEPA 高 5%，聚合额外 +2%；在 GPT-4.1 Mini 上效果好，在 Qwen3 8B 上导致退化（四个任务中只在一个上受益；例如 IFBench 从 38.61 降到 28.23） | p.11，Observation 5 第 1–2 段；p.8，Table 1 |
| **6. 跨模型泛化** | 在 Qwen3-8B 上优化的 prompt 直接用于 GPT-4.1-Mini，聚合 +9.00%（HotpotQA 最高 +27.67%），超过直接在 GPT-4.1-Mini 上优化的 MIPROv2（+5.64%）、TextGrad（+6.11%）、Trace（+3.27%） | p.11，Observation 6；p.8，Table 2 |
| 推理时搜索（初步） | NPUEval：GPT-4o Sequential10 为 4.25%，+RAG 16.33%，+MIPROv2 19.03%，GEPA 达到均值 30.52%（单个内核最高 70%）；GEPA 产出的单个 prompt 在不用 RAG 的情况下达到 26.85%。KernelBench：GEPA 把 GPT-4o 接近 0% 的 fast_1 提升到 20% 以上 | p.12，§5.1 "NPU Kernels" 段、"CUDA Kernels" 段；p.13，Figure 7、Figure 8 |
| 对抗 prompt 搜索 | 学到的通用对抗指令让 GPT-5 Mini 在 AIME-2025 上的 pass@1 从 76% 降到 10%；人工检查发现模型多以字面占位符 "### <final answer>" 结尾，作者认为下降来自无关信息与严格格式约束的相互作用 | p.14，§5.2 续段第 1–2 段 |
| 成本 | 用 GPT-4.1 mini 跑完 Table 2 全部实验不到 $500；GEPA $86，GEPA+Merge $67，MIPROv2 $76，Trace 和 TextGrad 合计 $172 | p.25，Appendix E.3 |
| 总体结论 | 基于语言的反思可以作为一种可扩展的策略，用于优化复杂的现实 AI 工作流，资源受限时尤其适用；GEPA 作为推理时搜索策略也有潜力 | p.15，§7 Conclusion |

## 7. 适用条件

| 条件 | 内容 | 原文依据 |
|---|---|---|
| 系统形式 | 任何包含一个或多个 LLM prompt 的 AI 系统（复合 AI 系统、agents、工作流） | p.1，Abstract（"Given any AI system containing one or more LLM prompts…"）；p.15，§7 |
| rollout 昂贵 / 预算受限 | 动机场景：工具调用昂贵、推理预算有限、低数据或受预算约束 | p.2，§1 第 1 段；p.4，§2 "Sample-Efficient Optimization" 段；p.2，§1 第 5 段（"…in data- or budget-constrained environments"） |
| 无法或不便改权重 | 权重保持不变，适用于不能微调的大模型或闭源模型 | p.2，§1 第 1 段末句；p.8，Table 2 标题（"GEPA works off-the-shelf on closed-source models as well"） |
| 轨迹可序列化为自然语言 | 需要 rollout 能表示为自然语言轨迹（指令、推理链、工具调用、奖励函数内部信息），并且能被 LLM 理解 | p.2，§1 第 2 段 |
| 有可用的评测指标与反馈函数 | 需要指标 μ；有富文本评测轨迹（编译错误、失败的评分细则等）或模块级反馈时更有利；没有时可以借助人工解释 | p.4，§3 第 1 段；p.6–7，§3 "Evaluation traces as diagnostic signals" 段及续段 |
| 模型具备指令遵循与反思能力 | 作者把"指令优化胜过 few-shot"归因于近期 LLM 在指令遵循和自我反思方面的进步 | p.9，Observation 2 第 1 段；p.30，Figure 16 标题 |
| Merge 的适用时机 | 存在若干独立且表现良好的谱系时，交叉收益最大 | p.11，Observation 5 第 2 段；p.23，Appendix D.1 |
| 推理时搜索 | 把待解任务集同时作为 D_train 和 D_pareto；配合领域特定文本反馈效果更好 | p.12，§5.1 第 1 段、末段 |
| 对模型规模、任务类型的适用边界 | 未说明（实验只覆盖 Qwen3 8B、GPT-4.1 Mini，以及扩展实验中的 GPT-4o、GPT-5 Mini） | — |

## 8. 研究局限

原文没有单独的 "Limitations" 章节，以下各条均来自正文中作者自己的陈述或报告的结果。

### 8.1 作者明确陈述的局限 / 未来工作

| 局限 | 内容 | 原文依据 |
|---|---|---|
| 大部分预算花在验证上 | GEPA 的大部分 rollout 预算用于验证集评估，这部分只服务于候选选择，不产生学习信号；作者提出用更小的验证集或动态选择的验证子集来改进，留作未来工作 | p.9，Observation 1 第 2–3 段（"The majority of GEPA's rollout budget is spent on validation…"；"…both of which we propose as directions for future work."） |
| Merge 的预算分配与调用时机尚待研究 | 两个模型使用相同超参数，对 Qwen3 8B 不是最优选择；变异与交叉之间的预算分配、何时调用 Merge 需要进一步研究，自适应技术留作未来工作 | p.11，Observation 5 标题句与第 2 段 |
| 推理时搜索只是初步结果 | "these are early results and warrant further systematic study"；推广到其他代码生成和领域适配任务留作未来工作 | p.12，§5.1 末段 |
| 归因属于推测 | 指令优化胜过 few-shot、泛化差距更小，作者"attribute"/"hypothesize"为模型能力进步与 GEPA 的设计所致，没有做专门验证 | p.9，Observation 2 第 1 段；p.30，Figure 16 标题 |

### 8.2 原文结果中可见的边界

| 局限 | 内容 | 原文依据 |
|---|---|---|
| 并非全面胜过 GRPO | AIME-2025 上 GEPA（32.00）低于 GRPO（38.00） | p.8，Table 1 及其标题 |
| Merge 可能伤害性能 | Qwen3 8B 上 GEPA+Merge 的聚合（52.40）低于 GEPA（54.85），IFBench 从 38.61 降到 28.23 | p.8，Table 1 |
| 只优化 prompt | GEPA 只进化 prompts，权重固定；与 GRPO 的比较跨越了不同参数空间 | p.4，§2 第 3 段；p.4，§3 第 1 段末句 |
| GRPO 设置 | 主实验的 GRPO 使用 LoRA，并固定 24,000 rollouts；超参数经人工探索。全参数微调的对照只在 2 跳 HoVer 单任务上做过 | p.9，脚注 1；p.26–27，Appendix E.4 "GRPO" 段；p.28，Figure 11 标题 |
| 实验覆盖面 | 主实验只有两个模型（Qwen3 8B、GPT-4.1 Mini）、六个基准；KernelBench 只用了 35 个任务 | p.8，§4 第 1 段；p.12，§5.1 "CUDA Kernels" 段 |

### 8.3 原文未涉及的内容

| 项目 | 状态 |
|---|---|
| 多次随机种子下的方差、置信区间、显著性检验 | 未说明 |
| 反思 LM 的具体模型选择及其对结果的影响 | 未说明 |
| 数据污染 / 泄漏检查 | 未说明 |
| 安全与伦理风险讨论（对抗 prompt 搜索部分只讨论了用于红队和鲁棒性评估的正面用途） | 未说明（正面用途见 p.14，§5.2 续段第 3 段） |

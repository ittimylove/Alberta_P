# Meta-Harness 文章总结

> 论文：*Meta-Harness: End-to-End Optimization of Model Harnesses*（Yoonho Lee, Roshen Nair, Qizheng Zhang, Kangwook Lee, Omar Khattab, Chelsea Finn；arXiv:2603.28052v1，2026-03-30）
>
> 说明：
> - 本总结只依据本文件夹中的 `paper.txt`（与 `paper.pdf` 同源）。页码 = PDF 页码（即 `paper.txt` 中“===== 第 N 页 =====”标记）。
> - 「原文依据」尽量引用英文原句或表格编号；中文为转述。
> - 原文没有提到的信息一律写“未说明”，不做补充。

---

## 1. 研究问题

| 内容 | 原文依据 |
|---|---|
| 核心问题：harness（决定存储、检索、向模型呈现哪些信息的代码）目前主要靠人工设计，**这一 harness 工程过程能否被自动化**。 | p.1–2：“harness engineering remains largely manual … In this paper, we ask whether this process itself can be automated.” |
| 子问题：现有文本优化器为何不适合 harness 工程——它们对反馈压缩过度（无记忆、只看标量分数、或只用短模板/摘要），而 harness 的影响跨越长时程，压缩反馈会丢失把下游失败追溯到早期 harness 决策所需的信息。 | Abstract（p.1）；p.2：“Compressed feedback often removes the information needed to trace downstream failures to earlier harness decisions.” |
| 形式化目标：在固定模型 M 和任务分布 X 下，寻找使期望最终奖励最大的 harness：H* = argmax_H E[r(τ, x)]；多目标时（如准确率与上下文成本）采用 Pareto 支配。 | p.4，“Objective” 段 |
| 消融问题：proposer 接口中哪部分信息最重要（仅分数 / 分数+摘要 / 完整执行轨迹）。 | p.6–7，Table 3 |

## 2. 研究对象

| 内容 | 原文依据 |
|---|---|
| **被优化对象**：任务特定的 harness，即“包裹语言模型、决定模型每一步看到什么上下文的有状态程序”；实验中每个 harness 是单文件 Python 程序，修改提示、检索、记忆和编排逻辑。 | p.4：“A harness is a stateful program that wraps a language model…”；p.5 “Practical implementation” |
| **方法本身**：Meta-Harness，一个在 harness 代码空间中搜索的外循环系统（outer-loop system）。 | Abstract（p.1）；p.4 第 3 节 |
| **三个任务领域**：在线文本分类、检索增强的数学推理、智能体编码（agentic coding）。 | p.5 第 4 节开头 |
| **底层模型（冻结）**：文本分类用 GPT-OSS-120B；数学检索搜索阶段用 GPT-OSS-20B，另在 GPT-5.4-nano、GPT-5.4-mini、Gemini-3.1-Flash-Lite、Gemini-3-Flash 上评估；TerminalBench-2 用 Claude Opus 4.6 与 Claude Haiku 4.5。 | p.5（4.1 节）；p.8（4.2 节）；p.9（Table 7 及 Results） |
| **Proposer**：Claude Code，搭配 Opus-4.6。 | p.5：“the proposer P is Claude Code [4] with Opus-4.6.” |

## 3. 数据来源

| 领域 | 数据 | 原文依据 |
|---|---|---|
| 在线文本分类（搜索/主评估） | LawBench（215 类，由案情描述预测罪名）、Symptom2Disease（22 类）、USPTO-50k（180 类，预测前体反应物）。实验设置沿用 Zhang et al. [59]、Ye et al. [52]。 | p.5–6（4.1 节） |
| 文本分类 OOD 评估 | 9 个搜索阶段未见过的数据集：SciCite、FiNER-139、Amazon Reviews（英文，5 分类）、Financial PhraseBank、GoEmotions、Banking77、AG News、SciTail、TweetEval (Hate)。 | p.7；附录 C.1（p.22–23）；Table 5（p.8） |
| 数学检索语料库 | 来自 8 个开源数据集，共 535,356 道题（正文写作“≥500,000”）：OpenMathReasoning、DeepMath-103K、NuminaMath-1.5、PolyMath、Omni-MATH、FineProofs-SFT、AIME 1983–2024、Putnam-AXIOM。已经过去重和去污染（精确前缀匹配 + 模糊 Jaccard 相似度，阈值 0.8）。 | p.8；附录 C.2（p.23）；Table 10（p.24） |
| 数学搜索集 | 250 道奥赛难度题（OlympiadBench + Omni-MATH hard）。 | p.8 |
| 数学测试集 | 200 道 IMO 级题：IMO-AnswerBench 100 道（分层抽样）、IMO-ProofBench 60 道、ArXivMath 2025 年 12 月 17 道、ArXivMath 2026 年 1 月 23 道。 | p.8；附录 C.3（p.23）；Table 11（p.24） |
| 智能体编码 | TerminalBench-2，共 89 个任务；搜索和最终评估都在这 89 个任务上进行。其他 harness 的结果取自官方排行榜。 | p.9（4.3 节）；Table 7 标题 |
| 定性分析数据 | TerminalBench-2 搜索运行的日志（10 次迭代，Claude Opus 4.6）。 | 附录 A（p.15） |

## 4. 核心变量

> 原文没有使用“自变量/因变量/控制变量”这类术语。下表按原文实际操纵的内容和报告的指标整理，各项均注明出处。

| 类别 | 变量 | 原文依据 |
|---|---|---|
| 被搜索的对象 | harness 代码（提示构建、检索、记忆、编排逻辑） | p.5 “Practical implementation” |
| 对比条件 | 不同的 harness / 优化方法：Zero-shot、Few-shot（N∈{4,8,16,32,all}）、ACE、MCE、Best-of-N、OpenEvolve、TTT-Discover、GEPA；数学领域：No Retriever、Dense Retrieval (k=1/5)、Random Few-shot、BM25；编码领域：Terminus 2、Terminus-KIRA、Claude Code、Goose 等 | p.6；Table 2、4、6、7（p.6–9） |
| 消融条件 | proposer 能获取的信息：Scores Only / Scores + Summary / 完整接口（Scores + Code + Traces） | Table 3（p.7） |
| 结果指标 | 准确率（文本分类）；额外上下文 token 数 Ctx（单位千，越低越好）；pass@1（三次采样平均，数学）；通过率 Pass %（TerminalBench-2）；搜索集上的 Median / Best 准确率；“> ZS”（准确率高于 zero-shot 的运行数） | Table 2、3、4、6、7（p.6–9） |
| 控制条件 | 底层模型 M 冻结；对比各文本优化器时使用相同的 proposer 配置（Opus-4.6，max reasoning）、相同的 harness 评估预算，只按搜索集表现选择候选，测试集留到最终评估。 | p.5：“The base model M varies by domain and is always frozen”；p.6：“For a fair comparison…” |
| 过程统计量 | 每次迭代读取的文件数（中位数 82，范围 69–99）及文件类型占比 | Table 8（p.15） |

注：Ctx 的单位在不同表中写法不同。Table 2 写作“additional input tokens in context (thousands)”（p.6），Table 9 写作“average additional characters in input context (thousands)”（p.20），Figure 3 的横轴为“Additional context (chars)”（p.6）。原文未说明二者如何对应。

## 5. 研究方法

| 内容 | 原文依据 |
|---|---|
| **搜索循环**：(1) 一个编码智能体读取文件系统（含所有历史候选的源代码、执行轨迹和分数），提出新 harness；(2) 在评估任务上评估；(3) 把所有日志（代码、推理轨迹、分数）存入文件系统的新目录，然后重复。 | Figure 2（p.2）；Algorithm 1（p.5） |
| **关键设计**：通过文件系统暴露完整历史，proposer 用 grep、cat 等工具按需检索，而不是把全部内容塞进一个提示。单次评估最多可产生约 10,000,000 tokens 的诊断信息。 | p.2–3；Table 1（p.2，Meta-Harness 为 10.0 MTok/iter） |
| **无父代选择规则**：维护种群 H 和 Pareto 前沿，但 proposer 可以自由查看任何历史 harness；迭代固定次数后，在 Pareto 前沿上做最终测试集评估。proposer 看不到测试集结果。 | p.4 |
| **接口校验**：只有通过 interface validation 的候选才会被评估。 | Algorithm 1 第 11 行（p.5）；附录 D（p.25） |
| **Proposer 引导**：一个最小的领域特定 skill，说明在哪里写新 harness、如何检查历史 harness 和执行轨迹、哪些文件可以或不可以修改。 | p.5 |
| **规模**：典型一次运行约 20 次迭代、评估约 60 个 harness。文本分类：20 次迭代 × 每次 2 个候选 = 40 个候选；数学：40 次迭代，产生 109 个候选；TerminalBench-2 定性分析的运行为 10 次迭代。 | p.5；p.6；p.8；p.15 |
| **初始种群**：文本分类为 zero-shot、few-shot、ACE、MCE；数学为 zero-shot、few-shot、ACE；TerminalBench-2 为 Terminus 2 和 Terminus-KIRA。 | p.6；p.8；p.9 |
| **评估设计**：对比人工设计的 harness 和程序搜索方法；信息接口消融（Table 3）；OOD 数据集泛化（Table 5）；跨模型迁移（Table 6，在 4 个搜索时未见过的模型上评估）；准确率–上下文 Pareto 分析（Figure 3、Table 9）；proposer 行为定性分析（附录 A）。 | p.5–9；附录 A、B |
| **防过拟合检查（TerminalBench-2）**：人工检查，并用正则审计演化出的 harness 中是否泄漏任务特定字符串。 | p.9 |
| 统计显著性检验 | 未说明 |
| 重复次数 / 随机种子数 | 未说明（数学评估为每题 3 次采样取平均，见 p.8；其余实验的重复次数未说明） |

## 6. 主要结论

| 结论 | 原文依据 |
|---|---|
| **在线文本分类**：选出的 harness 平均准确率 48.6%，比 ACE 高 7.7 个百分点、比 MCE 高 8.6 个百分点；上下文仅 11.4K，而 ACE 为 50.8K、MCE 为 28.5K。 | Table 2（p.6）；p.7 |
| **相对文本优化器**：只用 0.1× 的评估次数（4 次评估）就达到 OpenEvolve、TTT-Discover 的最终准确率，最终准确率高出 10 个百分点以上；搜索集 Median 50.0 / Best 56.7，对比 OpenEvolve 39.1/43.3、TTT-Discover 34.1/45.6、GEPA 32.6/40.2、Best-of-N 34.0/44.2。 | p.6；Table 4（p.7）；Figure 4（p.15） |
| **消融**：完整接口 Median 50.0 / Best 56.7；仅分数 34.6/41.3；分数+摘要 34.9/38.7。作者的解读是，访问原始执行轨迹是最关键的组成部分，摘要无法补回缺失的信号，甚至可能有害。 | Table 3 及 p.6–7 |
| **OOD 泛化**：在 9 个未见数据集上平均准确率 73.1%，高于 ACE 的 70.2%；其中 6/9 个数据集最高。另观察到 few-shot 示例超过 32 个后，7/9 个任务性能下降。 | p.7；Table 5（p.8） |
| **数学检索推理**：在 200 道 IMO 级题、5 个模型上，相对无检索平均提升 4.7 个百分点（34.1→38.8），比 BM25 高 1.3 个百分点，且没有出现 dense retrieval 和 random few-shot 在部分模型上的性能倒退。 | Table 6（p.8）；p.8 |
| **TerminalBench-2**：Opus 4.6 上通过率 76.4%，高于 Terminus-KIRA 的 74.7%，在 Opus 4.6 智能体中排第 2（第 1 是 ForgeCode 81.8%，作者表示无法仅凭其公开代码复现该结果）；Haiku 4.5 上为 37.6%，在 Haiku 4.5 智能体中排第 1，比 Goose（35.5%）高 2.1 个百分点。 | Table 7 及 p.9 |
| **发现的 harness 形态**：文本分类得到 Draft-Verification、Label-Primed Query 等一组 Pareto 变体；数学得到四路由 BM25 检索程序；TerminalBench-2 得到“环境快照引导”（environment bootstrapping），在 Terminus-KIRA 基础上约增加 80 行代码，相比 Terminus-KIRA 在 89 个任务中的 7 个上有提升。 | 附录 B.1–B.3（p.18–22）；Table 9（p.20） |
| **Proposer 行为**：每次迭代读取文件数的中位数为 82，源代码占 41%、执行轨迹占 40%；proposer 能识别混杂因素（prompt 修改与结构性修复被捆绑），并转向纯增量式修改。 | Table 8（p.15）；附录 A.2（p.16–17）；p.9 |
| **总体结论**：Meta-Harness 的主要优势不只是在代码空间中搜索，而是在搜索时能**选择性地访问历史诊断经验**。 | p.10：“the main advantage of Meta-Harness is not just search over code, but search with selective access to prior diagnostic experience.” |

## 7. 适用条件

| 内容 | 原文依据 |
|---|---|
| 适用于“语言模型被任务特定 harness 包裹”的场景，作者预期该方法基本与领域无关（largely domain-agnostic）。 | 附录 D（p.24） |
| 需要能力较强的编码智能体作为 proposer。作者认为这一工作流直到 2026 年初编码智能体能力大幅提升后才变得可行；实验只验证了 Claude Code 一种 proposer。 | p.4 脚注 1；p.10 |
| 需要能写出代码、分数和执行轨迹的评估流程，最好使用 JSON 等机器可读格式并分层组织。 | 附录 D（p.25）“Log everything…” |
| 搜索集应对基线 harness 有难度（如果基线已经饱和，可优化的空间就很小），且规模要小到每次运行能做约 50 次完整评估（分类实验为 50–100 个样本，数学检索为 88 题）。 | 附录 D（p.24–25） |
| 需要先写好 skill 文本。作者称 skill 是引导搜索的主要接口，对搜索质量的影响大于迭代次数或种群规模；建议先用 3–5 次迭代的短运行调试 skill。 | 附录 D（p.24） |
| 底层模型冻结，只优化 harness，不更新权重。 | p.3；p.5 |
| 一次搜索运行在墙钟时间上需要“几个小时”。 | p.10：“A search run completes in a few hours of wall-clock time” |
| 所需算力 / API 费用 | 未说明 |

注：附录 D 中的建议，作者明确表示“不是关于方法的科学结论，而是工程经验”（p.24：“The guidelines below are not themselves scientific claims about the method; they are engineering lessons…”）。

## 8. 研究局限

> 原文**没有单独的“Limitations”章节**。以下只列出原文中明确提到的限制或保留意见。

| 局限 | 原文依据 |
|---|---|
| 只用一种很强的编码智能体 proposer（Claude Code）验证了 harness 搜索；效果如何随 proposer 不同而变化，留待未来研究。 | p.10：“our experiments demonstrate that harness search can work with one particularly strong coding-agent proposer (Claude Code); a broader study … remains for future work.” |
| TerminalBench-2 的搜索和最终评估使用同一组 89 个任务，没有单独划分数据。作者称这是标准做法，并用人工检查和正则审计排查过拟合。 | p.9 |
| TerminalBench-2 上得到的 harness 专门针对 TerminalBench-2 场景。 | p.9：“the resulting harness is specialized to the TerminalBench-2 regime” |
| 尚未尝试同时演化 harness 和模型权重，作者将其列为未来方向。 | p.10 |
| 在线文本分类是本文上下文规模最小的设置，作者对更难场景的推断（结构较重的文本优化器在更难场景中劣势可能更大）只是推测。 | p.6：“so if structure-heavy text optimizers already lag here, their limitations may only grow in harder regimes.” |
| 数学测试集的分基准结果在正文中被合并报告；附录 C.3 写道“When included, the table in this section should report each benchmark separately…”，但该节没有给出分基准的结果表。 | p.23 附录 C.3 |
| 数学搜索集规模前后表述不一致：正文为 250 题，附录 D 为 88 题。原文未解释差异。 | p.8；p.25 |
| TerminalBench-2 的基线数值前后不一致：附录 A.2 中 Terminus-KIRA 基线为 64.4%，Table 7 中 Terminus-KIRA（Opus 4.6）为 74.7%。原文未解释二者关系。 | p.16；p.9 |
| 统计显著性、方差 / 置信区间 | 未说明 |
| 计算成本的具体数字（token 费用、GPU 等） | 未说明 |
| 负面社会影响 / 安全风险讨论 | 未说明 |

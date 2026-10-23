# HarnessCompass 文章总结

**论文**：*HarnessCompass: Guiding Automatic Harness Evolution toward Generalizable and Effective Agent Harnesses*
**作者**：Luan Zhang, Ruochen Zhou, Dandan Song 等（北京理工大学、香港城市大学等）
**来源**：arXiv:2608.01918v1 [cs.LG]，2026-08-03；预印本，原文注明"currently under review"（p.1）

> **引用标注说明**
> - 页码为 PDF 页码（共 17 页：正文 p.1–8，附录 p.8–16，参考文献 p.16–17）。
> - "§小节名 第 n 段"按该小节内的自然段顺序计数，跨页段落按起始页标注；正文为双栏排版，段落计数不区分栏。
> - 引号内英文为原文摘录；中文为对原文的转述，不含原文以外的信息。
> - 原文没有提到的内容统一写"未说明"。

---

## 1. 研究问题

**核心问题**：已有的自动 harness 演化方法存在三个局限。怎样设计一个演化框架，让演化出的 harness 既有效，又能推广到未见过的任务和其他模型上？

已有方法的三个局限：

| 局限 | 原文表述（转述） | 依据 |
|---|---|---|
| ① 对搜索任务过拟合 | meta agent 可以针对固定任务集的 verifier 反馈自由修改 harness，容易写入任务专属的捷径，在搜索集上提分，却无法迁移 | p.1，§Introduction 第 3 段（"First, existing methods overfit to the search tasks…"） |
| ② 只依赖轨迹信号 | meta agent 只能从外部观察交互，知道任务在哪里失败，却不知道 agent 为什么觉得 harness 难用，因此容易归因错误 | p.1，§Introduction 第 3 段（"Second, existing methods rely solely on trajectory-derived signals…"） |
| ③ 所有组件联合优化 | prompts、tools、middleware、memory 在同一轮里一起修改，彼此干扰，单独有效的改动合在一起后收益受限 | p.1，§Introduction 第 3 段（"Third, existing methods optimize all harness components jointly…"） |

- 作者认为三者有共同根源：缺少对演化过程的结构化约束（"the lack of structure and discipline governing the evolution process"）。依据：p.2，§Introduction 第 4 段（续 p.1 末段 "These limitations arise at different points in the loop, but…"）。
- 实验部分拆成三个具体问题：**Effectiveness**（是否优于已有方法、需要多少轮迭代）、**Attribution**（三项原则各自是否有贡献、如何权衡）、**Generalization**（能否推广到 held-out 任务和其他基座模型）。依据：p.5，§Experiments 第 1–2 段。

## 2. 研究对象

- **直接对象**：编码 agent（coding agent）的 **harness**，也就是协调模型与环境交互的软件层，包括 prompts、tools、middleware、memory、验证流程等。依据：p.1，§Introduction 第 1 段。
- **harness 的形式化**：harness 是一组可编辑组件，沿用 Lin et al. 2026，分为 **7 种正交组件类型**，归为两类。依据：p.3，§Preliminaries 第 1 段。
  - **结构组件**（structural）：tool implementations、middleware、sub-agent configurations；
  - **引导组件**（guidance）：system prompt、tool descriptions、skills、memory。
- **前提**：演化过程中基座模型 M 固定不变，只改 harness。依据：p.3，§HarnessCompass 第 1 段（"while keeping the base model fixed"）；p.3，§Preliminaries 第 1 段。
- **任务领域**：真实 Python 仓库的 issue 修复（SWE-Bench Verified）。依据：p.5，§Setup "Benchmark"段。

## 3. 数据来源

| 项目 | 内容 | 依据 |
|---|---|---|
| 基准 | **SWE-Bench Verified**：SWE-Bench 中经人工验证的 500 个任务子集，均为 Python 仓库的真实 issue | p.5，§Setup "Benchmark"段 |
| 划分 | 从 500 个任务中**随机抽取 50 个**作为演化集（Sample），演化过程只接触这 50 个；其余 **450 个**作为 held-out，只用于评测 | p.5，§Setup "Benchmark"段 |
| 仓库构成 | 共 12 个仓库，任务数从 django 的 231 个到 pallets/flask 的 1 个，各仓库任务数 n 见表 4 | p.7，§Where the Gains Land Across Repositories 第 1 段；p.15，Table 4 |
| 结果判定 | 由 verifier 给每条轨迹打二值结果 r(τ)∈{0,1}，1 表示任务成功 | p.3，§Preliminaries 第 1 段 |
| 轨迹数据 | 运行时产生的 raw trace，记录完整的消息、工具调用和环境观测序列 | p.3，§Preliminaries 第 1 段 |
| 每任务 rollout 次数 | 正文只写 k；附录写明 **k = 2** | p.3，§Preliminaries 第 1 段；p.14，§Per-Repository Results 第 1 段 |
| 随机抽样的种子 / 具体抽到哪 50 个任务 | **未说明** | — |
| 其他基准（如 Terminal-Bench 等） | **未说明**（实验只用了 SWE-Bench Verified） | — |

## 4. 核心变量

### 4.1 自变量 / 实验操纵

| 变量 | 取值 | 依据 |
|---|---|---|
| Harness 演化方法 | H0（种子）、AHE、HarnessCompass | p.5，§Setup "Baselines"段；Table 1 |
| 逐步消融配置 | H0 → +Generalization Gate → +Proactive Feedback → +R3 Integration（逐项累加） | p.6，§Ablation Study 第 1–2 段；Table 2 |
| 基座模型（跨模型迁移） | GPT-5.4（演化用）；Claude-Sonnet-4.6（冻结 harness 后直接评测） | p.5，§Setup "Models"段；p.6，Table 3 |
| 演化迭代次数 | t = 1…N | p.3，Algorithm 1；p.3，§Preliminaries 第 2 段 |

### 4.2 因变量 / 评价指标

| 指标 | 定义 | 依据 |
|---|---|---|
| **Pass@1** | 每个任务 k 次 rollout 的二值成功率均值 | p.5，§Setup "Metrics"段；p.3，§Preliminaries 第 1 段 |
| Sample / Held-Out / Total | 分别是 50 个演化任务、450 个 held-out 任务、全部 500 个任务上的 Pass@1；Total 按任务等权平均 | p.5，Table 1 标题；§Setup "Metrics"段 |
| **Turns** | 得到最终 harness 所用的演化迭代次数，用来衡量演化效率 | p.5，§Setup "Metrics"段 |
| 分仓库 Pass@1 | 12 个仓库各自的 Pass@1 | p.7，Figure 3；p.15，Table 4 |

### 4.3 控制变量

- 代码 agent、轨迹分析器、反馈 agent、meta agent **共用同一个基座模型 GPT-5.4**，以排除"更强的 meta agent"带来的收益。依据：p.5，§Setup "Models"段。
- 所有 agent 使用 **non-thinking 模式**，以排除扩展推理的影响。依据：p.5，§Setup "Models"段。
- 两种自动方法都从**同一个种子 H0** 开始。依据：p.5，§Main Results 第 1 段。
- 基线"under the same experimental settings"。依据：p.5，§Setup "Baselines"段。

## 5. 研究方法

### 5.1 总体框架

沿用 AHE（Lin et al. 2026）的迭代闭环：**评估当前 harness → 分析轨迹 → 改进 harness**，再在这个闭环上加入三项原则。依据：p.3，§HarnessCompass 第 1 段；完整流程见 p.3，Algorithm 1。

**种子 harness H0**：只有一个 shell 命令工具和一段简短的 system prompt，没有 middleware、skills、sub-agents。这样做是为了让后续 harness 中的每个组件都由演化循环自己引入，贡献可以单独考察。依据：p.4，§Preliminaries 第 3 段（续）；p.8，附录 §Seed Harness H0。

### 5.2 原则一：约束演化（Constrained Evolution / Generalization Gate）

所有候选改动都必须先通过一个全局泛化门（generalization gate），它覆盖全部 7 类组件，提出两条要求。依据：p.4，§Constrained Evolution 第 1 段。

- **内容要求**：拒绝提及具体任务实例、测试函数、被测代码私有符号的改动，也拒绝依据仅在特定任务中出现的 token 触发的代码分支；只接受"可复用的决策准则 + 在新任务上可判断的适用条件"。依据：p.4，§Constrained Evolution 第 2 段。附录给出逐字 prompt（HARD BAN 列表），见 p.8，附录 §Generalization Gate。
- **位置要求**：
  - **Capability edits**（新增可执行功能，如运行测试、安装缺失包）必须写成 middleware、tools 或 sub-agents 里的可执行代码；
  - **Guidance edits**（只提供行为指导）必须放在 system prompt 或 memory 中，不能写成可执行逻辑。
  - 依据：p.4，§Constrained Evolution 第 3–4 段。

### 5.3 原则二：主动反馈（Proactive Feedback）

分三步。依据：p.4，§Proactive Feedback 第 1 段。

1. **引出第一人称反馈**：每轮评分之后，对每个**失败任务**，用生成该轨迹的同一个基座模型 M 发出两个查询。依据：p.4，§Proactive Feedback "Eliciting first-person feedback"段。
   - **Blind report**：给出轨迹但不告诉结果，问 harness 在哪里阻碍了执行、希望 harness 提供什么能力，目的是避免事后偏差；
   - **Hindsight report**：告诉结果，让 agent 把失败归到四类原因之一：harness、agent 自身推理、任务歧义、环境；
   - 然后把两份报告合并为统一的反馈条目，并记录两者是否一致。
2. **对照轨迹做验证（grounding）**：只保留有轨迹支持的反馈条目。对已有组件的抱怨，轨迹必须直接体现相应困难；对新能力的请求，轨迹必须显示出该能力能弥补的具体缺口。没有支持的条目丢弃。依据：p.4，§Proactive Feedback "Grounding against trajectories"段。
3. **聚合与使用**：按确定性流程（不额外调用模型）把条目按涉及的组件分组，按三个因素给每组算置信度：报告间一致程度、受影响的不同任务数、blind 与 hindsight 报告的一致性。分组结果作为**补充证据**交给 meta agent，不替代轨迹证据；由此引发的改动同样需要轨迹支持，下一轮如果没有帮助就回退。依据：p.4，§Proactive Feedback "Aggregation and use"段。

### 5.4 原则三：分组件优化（Component-wise Optimization + R3）

- **两条独立轨道**：每轮并行演化两个 harness 变体，一个只改结构组件，一个只改引导组件，两者修改的组件空间不重叠；评估后 Pass@1 较高的为 winner。依据：p.5，§Component-wise Optimization "Running the tracks independently"段。
- **R3 合并**（以 winner 为基础合入 loser 的有用改动）：
  - **Revision**：逐条检查 loser 的改动，只保留单独有益、且与 winner 不重复不冲突的；丢弃造成退化、任务专属或不确定的改动；
  - **Recombination**：把保留的改动应用到 winner 上，文件级冲突时以 winner 为准；
  - **Refinement**：删除文件级不冲突但功能重叠的冗余改动，保留来自更可靠组件的那一份。
  - 依据：p.5，§Component-wise Optimization "Merging the tracks – R3"段。
- **接受规则**：只有当 winner 的 Pass@1 高于当前 harness 时，才接受本轮合并后的 harness。依据：p.3，Algorithm 1 第 10–14 行。

### 5.5 实验设计

- 主实验：与 H0、AHE 对比。依据：p.5，§Setup "Baselines"段；Table 1。
- 逐步消融：每次加一项原则。依据：p.6，§Ablation Study 第 1 段。
- 跨模型迁移：冻结 GPT-5.4 上演化出的 harness，在 Claude-Sonnet-4.6 上直接评测，不再演化。依据：p.5，§Setup "Models"段；p.7，§Generalization… 第 1 段。
- 演化效率曲线：p.6，Figure 2；p.7，§Evolution Efficiency。
- 分仓库分析：p.7，§Where the Gains Land…；p.14–15，附录 §Per-Repository Results / Table 4。
- 定性案例研究：对比有门控和无门控两次运行中积累的 memory。依据：p.14–15，附录 §Case Study。
- 统计显著性检验、多次独立运行的方差 / 置信区间：**未说明**。
- 演化的总计算成本（token、费用、时间）：**未说明**（只用 Turns 衡量效率）。
- 最大迭代次数 N 的具体设定：**未说明**。
- AHE 的结果是作者复现的还是引自原文：**未说明**（原文只写"under the same experimental settings"，p.5，§Setup "Baselines"段）。

## 6. 主要结论

### 6.1 有效性与效率（Table 1，p.5）

| 方法 | Sample | Held-Out | Total | Turns |
|---|---|---|---|---|
| H0（种子） | 54.0% | 51.6% | 51.8% | 0 |
| AHE | 63.0% | 54.7% | 55.5% | 20 |
| **HarnessCompass** | **66.0%** | **60.4%** | **61.0%** | **5** |

- HarnessCompass 用 5 轮把 Sample Pass@1 从 54.0% 提到 66.0%，比 AHE 高 3 个点，而 AHE 用了 20 轮。依据：p.5，§Main Results 第 1 段；摘要（p.1）。
- 在 held-out 任务上两者差距更大（5.7 点，Sample 上是 3.0 点）。作者把这一点归功于 generalization gate。依据：p.6，§Main Results 第 2 段（"The advantage is more evident on held-out tasks"）。
- 演化曲线：HarnessCompass 上升快、平台更高，用大约 1/4 的迭代次数达到更高性能。作者给出两个解释：门控让改动可迁移，所以不会出现回退；R3 把 loser 轨道的有用改动并回来，让收益逐轮累积。依据：p.7，§Evolution Efficiency 第 2 段；p.6，Figure 2。

### 6.2 消融（Table 2，p.6）

| 配置（累加） | Sample | Held-Out | Total | Turns |
|---|---|---|---|---|
| H0 | 54.0% | 51.6% | 51.8% | 0 |
| + Generalization Gate | 62.0% | 58.4% | 58.8% | 2 |
| + Proactive Feedback | 66.0% | **55.8%** | 56.8% | 12 |
| + R3 Integration | 66.0% | 60.4% | 61.0% | 5 |

- 单独加门控：2 轮内 Sample 达到 62.0%，Held-Out 达到 58.4%。依据：p.6，§Ablation Study 第 2 段。
- 加入主动反馈后 Sample 升到 66.0%，但 **Held-Out 降到 55.8%、Turns 增加到 12**，作者认为反馈驱动的改动可能对演化集过拟合。依据：p.6，§Ablation Study 第 2 段。
- R3 在保持 66.0% 的同时，把 Held-Out 恢复到 60.4%，Turns 降到 5。作者的结论是三项原则作用互补：门控促进迁移，反馈扩大优化信号，R3 保持泛化并提高效率。依据：p.6–7，§Ablation Study 第 2 段。

### 6.3 跨模型迁移（Table 3，p.6）

- 冻结的 harness 在 Claude-Sonnet-4.6 上把 Total 从 70.0% 提到 73.8%（Sample 68.0%→76.0%，Held-Out 70.2%→73.6%）。作者据此认为该方法学到的是可复用的工程改进，而不是针对特定模型或任务的修补。依据：p.7，§Generalization to Held-Out Tasks and Base Models 第 2 段。

### 6.4 分仓库结果（Figure 3，p.7；Table 4，p.15）

- GPT-5.4 下 12 个仓库中 8 个提升、3 个不变、1 个下降（scikit-learn 73.4→68.8，−4.6）。依据：p.14，附录 §Per-Repository Results 第 2 段；p.15，Table 4。
- 提升最大的是 seed harness 最弱、对 harness 最敏感的仓库：sphinx 35.2→58.0、pytest 52.6→68.4、django 59.3→70.1。依据：p.7，§Where the Gains Land… 第 2 段。
- django（231 题）和 sympy（75 题）合计占基准的 61%，对总体 +9.2 点的提升贡献很大。依据：p.14，附录 §Per-Repository Results 第 2 段。

### 6.5 定性案例

- 无门控运行积累的 memory 里大多是引用具体实例（如 django__django-13158）和私有实现细节的"答案式"条目；有门控运行记录的是带适用条件的通用原则（如 "Execution-path ownership rule"）。依据：p.14–15，附录 §Case Study "Two kinds of accumulated memory"。

### 6.6 总体结论

- HarnessCompass 在有效性和泛化上都强于已有方法；作者认为"disciplined evolution is as important as search itself"。依据：p.7–8，§Conclusion。

## 7. 适用条件

原文没有设置专门的"适用条件"章节。以下条件都是原文中明确写出的方法前提或实验设定：

| 条件 | 依据 |
|---|---|
| 基座模型在演化中固定，只优化 harness | p.3，§HarnessCompass 第 1 段；§Preliminaries 第 1 段 |
| harness 可以拆成上述 7 类组件、两大类（结构 / 引导），分组件优化依赖这一划分 | p.3，§Preliminaries 第 1 段；p.5，§Component-wise Optimization "Running the tracks"段 |
| 需要能给每条轨迹打二值结果的 verifier | p.3，§Preliminaries 第 1 段 |
| 需要完整的 raw trace，用于轨迹分析和对反馈做 grounding | p.3，§Preliminaries 第 1 段；p.4，§Proactive Feedback "Grounding"段 |
| 主动反馈只针对失败任务收集，并由生成轨迹的同一基座模型给出 | p.4，§Proactive Feedback "Eliciting first-person feedback"段 |
| 实验设定：所有角色共用 GPT-5.4、non-thinking 模式、从极简 H0 开始 | p.5，§Setup "Models"段；p.4，§Preliminaries 第 3 段 |
| 已验证的任务类型：Python 仓库 issue 修复（SWE-Bench Verified） | p.5，§Setup "Benchmark"段 |
| 跨模型迁移只验证过 GPT-5.4 → Claude-Sonnet-4.6 一个方向 | p.5，§Setup "Models"段；p.6，Table 3 |
| 在 thinking 模式、其他编程语言、非编码类 agent 任务上是否适用 | **未说明** |
| 演化集规模（50 题）改变后效果是否稳定 | **未说明** |

## 8. 研究局限

原文**没有设置独立的 Limitations 章节**。以下是原文中作者自己承认或明确写出的局限：

| 局限 | 原文内容（转述） | 依据 |
|---|---|---|
| 小仓库统计分辨率低 | psf（8 题）、mwaskom（2 题）、pallets（1 题）的 Pass@1 分辨率分别是 6.25、25、50 个百分点，一次试验结果翻转就会造成很大波动；mwaskom 的 +25.0 只相当于 4 次试验中多成功 1 次；作者提醒不要过度解读 | p.14，附录 §Per-Repository Results 第 3 段 |
| 跨模型迁移整体为正但不均匀 | 在 Claude-Sonnet-4.6 上 pytest 下降 10.5 点、sphinx 下降 4.5 点；作者推测部分为 GPT-5.4 演化出的机制对更强的基座模型可能不必要，因此只把整体迁移结果当作主要结论 | p.14，附录 §Per-Repository Results 第 4 段 |
| GPT-5.4 下也有退化 | scikit-learn 从 73.4 降到 68.8（−4.6） | p.15，Table 4 |
| 主动反馈本身可能导致过拟合 | 不配合 R3 时，加入反馈使 Held-Out 从 58.4% 降到 55.8%，Turns 增加到 12 | p.6，§Ablation Study 第 2 段 |
| 案例研究的证据力度有限 | 作者写明 memory 检查本身不能证明泛化，也不意味着每条门控规则都能成功迁移；它只能排除"无门控收益来自答案记忆"这一替代解释 | p.15，附录 §Case Study "Interpretation" 第 2 段 |
| 只依赖结果信号 | 作者在 Future Work 中指出 LLM agent 仍有幻觉和中间行为不可靠的问题，在长程任务中错误会累积；建议未来让 harness 学会识别中间行为的 rubric，而不只依赖结果信号 | p.16，附录 §Future Work |
| 实验覆盖范围 | 只在 SWE-Bench Verified 一个基准、GPT-5.4 一个演化模型上开展演化；作者**没有把这一点写成局限** | p.5，§Setup（事实依据）；作为局限的讨论：**未说明** |
| 统计显著性 / 重复运行 / 计算成本 | **未说明** | — |

### 附：原文中可见的不一致之处（仅列出，不做推断）

- 达到峰值的迭代轮次：Figure 2 标题写 "reaches its peak performance of 66% at iteration 5"（p.6），§Evolution Efficiency 第 2 段写 "reaches its peak at around iteration 6"（p.7）。
- 模型引用：正文使用 Claude-Sonnet-4.6，并引用 "(Anthropic 2025)"（p.5，§Setup "Models"段），但参考文献中该条目是 "Claude Sonnet 4.5"（p.16，References）。
- §Main Results 第 1 段（p.5）中 "The gap further widens on held-out tasks, where HarnessCompass achieves 60.4% Pass@1 versus AHE's 54.7%" 一句与同段前文的表述重复。

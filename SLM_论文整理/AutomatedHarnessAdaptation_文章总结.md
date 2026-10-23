# Automated Harness Adaptation（自动化 Harness 适配）— 文章总结

> **论文**：Better Harnesses, Smaller Models: Building 90% Cheaper Agents via Automated Harness Adaptation
> **作者**：Chenyang Yang, Xinran Zhao, Tongshuang Wu, Christian Kästner（Carnegie Mellon University）
> **来源**：arXiv:2607.08938v1 [cs.SE]，2026-07-09（p.1 页眉）
> **代码**：https://github.com/malusamayo/migration-analysis（p.1 脚注 1）

**说明**
- 本文档只依据 `paper.pdf` 原文整理，不补充原文以外的信息；原文未提及的项写“未说明”。
- 页码为 PDF 页码（共 12 页）。论文为双栏排版，“第 n 段”指该节/小节内的自然段顺序（从节标题后第一段算起）；图表另行标注。
- 方法名：原文没有给方法单独起名，标题和正文用的是 “automated harness adaptation” 和 “harness optimizer”（p.1 标题；p.4 §III 第2段），本文件沿用这两个说法。

---

## 1. 研究问题

**总问题**：对于常规商业任务，能否靠自动适配 harness，让小语言模型（SLM）agent 以远低于前沿 LLM 的成本达到相当的性能？什么时候可行，为什么可行？
- 依据：p.1 §I 第6段（“it is unclear when harness adaptation is sufficient to close the SLM-LLM gap, and which task or model conditions make adaptation more effective. To systematically understand if, when, and how…”）；p.10 §VII 第1段（“a first in-depth empirical analysis of if, when, and how harness adaptations can afford construction of (cost-)effective SLM agents”）。

**四个具体研究问题**（p.6 §IV 开头列表）：

| 编号 | 问题 | 原文依据 |
|---|---|---|
| RQ1 | harness 适配能在多大程度上让 SLM agent 与前沿 LLM agent 相竞争？（比较准确率、成本、延迟） | p.6 §IV，RQ1 条目 |
| RQ2 | 任务多样性如何影响 harness 适配的效果？（是否任务越重复，适配越有效） | p.6 §IV，RQ2 条目 |
| RQ3 | 模型能力如何影响 harness 适配的效果？ | p.6 §IV，RQ3 条目 |
| RQ4 | 哪些 harness 适配策略最常用来提升 SLM 性能？（用 §II 的框架分析优化后的 harness） | p.6 §IV，RQ4 条目 |

---

## 2. 研究对象

| 对象 | 内容 | 原文依据 |
|---|---|---|
| SLM 的定义 | 作者的操作性定义：更便宜、更小、可本地部署的开源权重模型；实验所用模型激活参数 3B–8B、总参数 8B–30B | p.1 脚注 2 |
| 被测 SLM（3 个，来自不同模型家族） | qwen3-coder-30b-a3b、ministral-3-8b、gemma-4-26b-a4b | p.6 §IV-A “Models” 段；p.1 摘要（“three SLM families”） |
| 对照 LLM | gemini-3.1-pro-preview，使用通用 harness | p.6 §IV-A “Models” 段 |
| 任务（7 个常规商业 agent 任务） | attendance-auditing、budget-approval、stock-alert、anomaly-detection、playwright-testing、website-management、code-refactoring | p.7 Table I；p.6 §IV-A “Tasks” 段第1段 |
| 被适配的对象：agent harness | 上下文（系统提示、skills、动态上下文）、工具、hooks、上下文管理、子 agent | p.5 §III-A 列表 |
| 实验单元 | 任务–SLM 组合，共 7×3=21 对 | p.1 摘要（“16 of 21 task-SLM pairs”）；p.6 §IV-A 第1段 |

---

## 3. 数据来源

### 3.1 任务数据集（Table I，p.7）

| 任务 | 样本数 N | 构建方式 | 评估标准 |
|---|---|---|---|
| attendance-auditing | 100 | 基于 TheAgentCompany [50] 编写任务模板，再生成新实例 | 与 ground-truth 比对 |
| budget-approval | 100 | 基于 TheAgentCompany [50] 编写任务模板，再生成新实例 | 与 ground-truth 比对 |
| stock-alert | 100 | 从 LOCA-Bench [51] 中选取，复用其流程生成新实例 | 与 ground-truth 比对 |
| anomaly-detection | 100 | 从 LOCA-Bench [51] 中选取，复用其流程生成新实例 | 与 ground-truth 比对 |
| playwright-testing | 100 | 复用 WebGenBench [52] 的实例，网站 HTML 由前沿 LLM（gemini-3.1-pro）生成 | 运行生成的测试 |
| website-management | 50 | 从 WebArena [22] 的 CMS 子集中采样，只保留前沿 LLM 能解出的实例 | 与 ground-truth 比对 |
| code-refactoring | 100 | 复用 RefactorBench [53] 的实例 | 运行测试，检查重构后代码的 AST |

- 每个任务都是从公开 benchmark 整理出的独立实例集，且有客观评估指标：p.6 §IV-A “Tasks” 第1段。
- 数据划分为 20/20/60 的 train/validation/test：p.6 §IV-A “Tasks” 第3段。

### 3.2 其他数据来源

| 用途 | 来源 | 原文依据 |
|---|---|---|
| 成本计算 | 按记录的 token 用量和官方 API 定价计算。gemini-3.1-pro-preview、gemma-4-26b-a4b 用 Google Vertex AI 定价；qwen3-coder-30b-a3b、ministral3-8b 用 AWS Bedrock 定价 | p.6 §IV-B “RQ1” 段 |
| 模型能力的代理指标 | Artificial Analysis [55] 报告的平均 benchmark 分数（Fig. 6 横轴为 “Artificial Analysis intelligence index”） | p.7 §IV-B “RQ3” 段；p.9 Fig. 6 |
| 任务多样性测量 | LLM agent 的 rollout 轨迹中的工具调用序列 | p.7 §IV-B “RQ2” 第2段 |
| 失败模式框架的来源 | 近期模型卡 [12]–[14] 及其报告的 agentic benchmark [15]–[23] | p.2 §II-A 第2段 |
| 适配策略框架的来源 | 已有文献和流行 harness（如 [27] Claude Code、[28] Codex） | p.3 §II-B 第2段 |

---

## 4. 核心变量

### 4.1 自变量 / 分组变量

| 变量 | 取值 / 操作化 | 原文依据 |
|---|---|---|
| Harness 类型 | 通用 harness 与优化后的 harness | p.6 §IV-B “RQ1” 段 |
| 模型 | 3 个 SLM，外加 1 个前沿 LLM 作对照 | p.6 §IV-A “Models” 段 |
| 任务多样性（跨任务） | 对每个任务收集 LLM rollout，提取工具调用序列，计算序列两两之间的**归一化 Levenshtein 距离均值**，作为工作流多样性的代理 | p.7 §IV-B “RQ2” 第2段 |
| 任务多样性（受控实验） | 在 attendance 和 budget 两个任务上建模板池（每个模板代表一种不同的工作流），通过改变数据集中包含的模板数来控制多样性；Fig. 5 右图横轴为模板数 3 / 8 / 20 | p.7 §IV-B “RQ2” 第3段及续段；p.9 Fig. 5 |
| 模型能力 | Artificial Analysis 平均 benchmark 分数 | p.7 §IV-B “RQ3” 段 |

### 4.2 因变量 / 结果变量

| 变量 | 说明 | 原文依据 |
|---|---|---|
| 测试准确率 | 在留出测试集上的准确率 | p.6 §IV-B “RQ1” 段；p.8 Table II 标题 |
| 单实例平均成本（USD） | 按 token 用量和 API 定价计算 | p.6 §IV-B “RQ1” 段；p.8 Table II 标题 |
| 端到端平均延迟（秒） | — | p.8 Table II 标题 |
| 被处理的失败模式、使用的适配策略（RQ4） | 人工检查优化后的 harness，按 §II 的框架分类 | p.7 §IV-B “RQ4” 段 |

### 4.3 控制条件

| 条件 | 内容 | 原文依据 |
|---|---|---|
| 优化预算 | 每次优化 $20，覆盖 meta-agent 分析、任务 agent rollout 和候选 harness 评估；主实验总预算 $1260 | p.7 §IV-B 第1段（续 p.6）；p.7 脚注 3 |
| 优化重复次数 | 每个任务–模型对运行 3 次优化，保留验证分最高的 harness | p.7 §IV-B 第1段 |
| 评估重复次数 | 每个配置运行 3 次，报告平均值 | p.6 §IV-B “RQ1” 段 |
| Meta-agent 模型 | gemini-3.1-pro-preview | p.6 §IV-B “RQ1” 段末 |

---

## 5. 研究方法

### 5.1 框架构建：失败模式 → 适配策略（§II，Fig. 2）

- **失败模式**按所涉及的模型能力分成 5 类：tool-use、instruction-following、knowledge、long-context、planning/reasoning。依据：p.2 §II-A 第2段；p.3 §II-A a)–e) 各段。
- **适配策略**按所修改的 harness 组件分成三大类、六小类：
  - 上下文（A1 增加上下文，A2 管理上下文）
  - 工具（T1 创建工具，T2 管理工具）
  - agent 循环（L1 代码插桩，L2 编排多 agent）
  - 依据：p.3 Fig. 2；p.3 §II-B 第2段；p.4 §II-B a)–c) 各段。
- 与已有分类的区别：已有工作有 MAST [25]、AgentErrorTaxonomy [26]；本文按模型能力而不是按系统模块来索引失败。依据：p.3 “Comparison with existing failure analysis” 段。
- 框架刻意不绑定模型规模。依据：p.4 §II-C 第1段。

### 5.2 自动化 harness 优化器（§III，Fig. 3）

- **输入 / 输出**：输入为一个 SLM、带训练集和验证集的目标任务、一个初始 harness；输出为优化后的 harness。依据：p.4 §III 第2段末至 p.5 首段。
- **搜索空间**：基于 software-agent-sdk（OpenHands）[48] 的 API 接口，可修改上下文、工具、hooks、上下文管理和子 agent。依据：p.5 §III-A。
- **优化循环**（依据：p.5 §III-B 1)–2)；p.6 首段 3)）：
  1. **采样并评估**：维护一个 harness 池。用 GEPA 风格的遗传搜索 [49] 从 Pareto 前沿采样一个 harness（至少在一个训练实例上最优），在一批训练实例上执行，记录完整轨迹、评估分数和反馈。
  2. **诊断并提出新 harness**：meta-agent 读取四类上下文——任务轨迹、当前 harness 代码、搜索记忆（过去的尝试及其效果）、设计空间文档，然后编辑 harness。
  3. **验证并保存**：先做廉价的 sanity check，不通过则让 meta-agent 修复重试，有最大次数限制。通过后在同一批训练实例上评估；有提升就在验证集上做完整评估，若优于已有候选则加入池中。
- **示例**：budget-approval 任务经过 23 次迭代，得到的 harness 收窄了工具集、把系统提示改写成分步流程、新增 anti_loop_hook.py 防止死循环。依据：p.6 §III-C 第2段；p.5 Fig. 4。

### 5.3 实验设计（§IV）

| RQ | 方法 | 原文依据 |
|---|---|---|
| RQ1 | 比较三种配置：SLM + 通用 harness、SLM + 优化 harness、LLM + 通用 harness；测量测试准确率、成本、延迟 | p.6 §IV-B “RQ1” 段 |
| RQ2 | (a) 跨 7 个任务做多样性与优化后性能的相关分析（结果报告为 Spearman 相关）；(b) 在 attendance、budget 上做多样性受控实验 | p.7 §IV-B “RQ2” 段；p.8 §IV-D “RQ2” 段 |
| RQ3 | 以 Artificial Analysis 分数为能力代理，结合该分数解读性能趋势 | p.7 §IV-B “RQ3” 段 |
| RQ4 | 人工检查每个任务–模型对的优化 harness，按 §II 框架归类其编辑（处理了哪些失败模式、用了哪些策略） | p.7 §IV-B “RQ4” 段 |

- “16/21 对显著提升”中“显著”所用的统计检验方法：**未说明**。

---

## 6. 主要结论

### 6.1 RQ1：成本–性能权衡

- 21 个任务–SLM 对中有 16 个在优化 harness 后性能显著提升，其中 7 对追平了 SLM 与 LLM 的差距。依据：p.8 §IV-D “RQ1” 第1段；p.1 摘要。
- 最好的 SLM（gemma-4-26b-a4b）恢复了 LLM 89%（摘要写 89.7%）的性能，成本为 LLM 的 4%，延迟降低 25%。依据：p.8 §IV-D “RQ1” 第1段；p.1 摘要。
- 一次性离线优化成本为每任务 $20，按观察到的单次运行节省计算，各任务和模型平均运行 13 次后即可收回。依据：p.8 §IV-D “RQ1” 第2段。

**Table II 关键数值**（p.8；准确率 / 单实例成本 / 延迟，取 Avg. 列）：

| 配置 | Avg. 准确率 | Avg. 成本 | Avg. 延迟 |
|---|---|---|---|
| gemini-3.1-pro-preview | 89.7 | $1.735 | 181s |
| gemma-4-26b-a4b | 31.4 | $0.043 | 328s |
| gemma-4-26b-a4b + 优化 harness | 80.2 | $0.071 | 135s |
| qwen3-coder-30b-a3b | 26.9 | $0.085 | 107s |
| qwen3-coder-30b-a3b + 优化 harness | 74.8 | $0.064 | 76s |
| ministral-3-8b | 9.5 | $0.110 | 194s |
| ministral-3-8b + 优化 harness | 25.0 | $0.099 | 172s |

**各任务准确率**（p.8 Table II）：

| 配置 | Attn. | Budget | Stock | Anom. | Playwr. | Web. | Refact. |
|---|---|---|---|---|---|---|---|
| gemini-3.1-pro-preview | 96.5 | 97.3 | 86.7 | 98.9 | 85.7 | 76.7 | 86.1 |
| gemma-4-26b-a4b | 91.7 | 75.0 | 5.0 | 5.0 | 9.9 | 1.1 | 32.2 |
| + 优化 harness | 95.7 | 98.3 | 58.9 | 99.4 | 98.1 | 45.6 | 65.0 |
| qwen3-coder-30b-a3b | 23.7 | 61.0 | 9.4 | 1.1 | 31.4 | 1.1 | 60.6 |
| + 优化 harness | 73.7 | 79.3 | 96.7 | 93.9 | 93.6 | 24.4 | 62.2 |
| ministral-3-8b | 0.2 | 28.8 | 0.0 | 0.0 | 0.4 | 5.6 | 31.7 |
| + 优化 harness | 63.3 | 53.7 | 0.0 | 0.0 | 22.3 | 5.6 | 30.0 |

### 6.2 RQ2：任务多样性

- 7 个任务上，任务多样性与优化后性能呈强、统计显著的负相关（Spearman ρ = −0.96）。依据：p.8–9 §IV-D “RQ2” 第1段；p.9 Fig. 5 左。
- 受控实验中，多样性升高时优化 harness 的性能从 89.1% 降到 68.0%。依据：同上；p.9 Fig. 5 右。
- 引言的说法：从最多样到最不多样的设置，准确率 +21.1%。依据：p.2 §I 贡献列表第3条。
- 定性观察：attendance-auditing 的实例有共同的失败模式（缺知识、输出格式错），容易适配；code-refactoring 的不同仓库、不同需求差异大，难以适配。依据：p.8 §IV-D “RQ2” 第2段。

### 6.3 RQ3：模型能力

- 能力越强的 SLM，优化后性能越好、提升越大（+48.8% 对 +15.5%）。依据：p.8 §IV-D “RQ3” 段；p.2 §I 贡献列表第4条；p.9 Fig. 6。
- 有些核心能力无法转移给 harness：playwright-testing 需要编码能力，stock-alert 需要长上下文能力。依据：p.8 §IV-D “RQ3” 段。

### 6.4 RQ4：适配策略

- 最常被处理的失败模式：instruction-following（81%）、knowledge（81%）、tool-use（62%）、long-context（33%）。依据：p.9 §IV-D “RQ4” 第1段（续 p.8）。
- 最常用的策略：增加上下文（86%）、创建工具（43%）、管理工具（29%）。依据：同上。
- 示例：gemma-4-26b 在 anomaly-detection 上的最佳 harness 新建了 query_mock_bigquery 工具，把 MCP 工具从 40+ 个过滤到 7 个，并在系统提示中写明表命名约定。依据：同上。
- 没有观察到创建子 agent 的成功适配。作者给出两种假设：当前 SLM 管理子 agent 的能力有限，或者是 meta-agent 自身的局限。依据：p.9 §IV-D “RQ4” 第2段。
- 不同 SLM 需要不同的适配策略，且不能直接迁移，所以换 SLM 时需要重新跑优化。例如 ministral-3-8b 在文件编辑上有困难，qwen3-coder-30b-a3b 会输出原始 XML 而不是 JSON 工具调用。依据：p.9 “Different SLMs require different adaptation strategies” 段及下一段。

### 6.5 优化器设计经验（§V-A）

- **经验 1**：优化的瓶颈在诊断质量。传原始 JSON 轨迹比传后处理的 markdown 效果好。把 meta-agent 换成更便宜的 gemini-3-flash-preview 后最终 harness 更差。提供前沿模型的成功轨迹或人工启发式规则，并没有稳定带来改进。依据：p.9 §V-A “Lesson 1” 三段。
- **经验 2**：探索的质量和多样性比数量更重要。保留过往尝试的记忆有帮助；多次独立搜索比把同样预算花在一条长轨迹上更好。依据：p.9–10 §V-A “Lesson 2” 段。

### 6.6 总体结论

- 在给定明确设计空间的前提下，harness 优化器能自动找到有效的适配，显著提升 SLM 性能。这种做法在更重复的任务和基础能力更强的 SLM 上效果最好，而且可以把 harness 的改动对应到失败模式和适配策略来解释其效果。依据：p.10–11 §VII。

---

## 7. 适用条件

| 条件 | 原文依据 |
|---|---|
| 常规商业任务：不需要开放式创造力，只需要在受限环境中可靠执行例行工作流 | p.1 §I 第2段；p.1 摘要 |
| 任务实例之间共享大量难度（工作流稳定、重复性高），因此难度可以从模型转移到 harness | p.1 摘要；p.1 §I 第4段；p.8 §IV-D “RQ2” |
| SLM 本身要有足够的基础能力（如基本的 tool-use 和 instruction-following） | p.1 摘要；p.8 §IV-D “RQ3”；p.10 §VI 第2段 |
| 需要整理好任务实例并有明确的评估标准，供优化器优化 | p.10 §V-B “For agent developers” 第2段；p.6 §IV-A “Tasks” 第1段 |
| meta-agent 最好由前沿 LLM 驱动（更便宜的模型效果更差） | p.10 §V-B “For agent developers” 第2段；p.9 §V-A “Lesson 1” 第2段 |
| 关注成本的场景；离线优化成本平均运行 13 次后收回 | p.10 §V-B “For agent developers” 第1段；p.8 §IV-D “RQ1” 第2段 |
| 每换一个 SLM 都要重新运行优化 | p.9 “Different SLMs…” 段 |
| 选择 SLM 时可参考 benchmark 分数；小型 MoE 模型（如 gemma-4-26b-a4b）是较好的候选 | p.10 §V-B “For agent developers” 第1段 |
| 实验设置为目标清晰、环境干净的任务 | p.7–8 §IV-C “External validity” |

---

## 8. 研究局限

### 8.1 作者明确承认的局限

| 局限 | 原文依据 |
|---|---|
| **内部效度**：agent 运行和优化过程随机性很强，即使做了多次重复也是如此 | p.7 §IV-C |
| **外部效度**：结论基于特定的任务、模型和一种优化器实现，不一定能迁移到其他任务、未来模型或其他优化器实现 | p.7–8 §IV-C |
| 只关注目标清晰、环境干净的任务，不能完全反映专用 agent 的复杂部署场景 | p.8 §IV-C |
| **可复现性**：使用闭源的前沿 LLM API，API 变动或下线后难以精确复现优化过程（作者同时认为结论能推广到其他强能力前沿 LLM） | p.8 §IV-C |
| harness 能吸收大量任务难度，但不能完全替代缺失的核心模型能力 | p.2 §I 贡献列表第4条；p.8 §IV-D “RQ3” |
| 高多样性任务（如 code-refactoring）难以用单一 harness 覆盖 | p.8 §IV-D “RQ2” 第2段 |
| 没有发现成功的子 agent 适配，可能是 SLM 或 meta-agent 的局限 | p.9 §IV-D “RQ4” 第2段 |
| 不同 SLM 的适配不能迁移，不存在通用 harness | p.9 “Different SLMs…” 段 |
| 更复杂的 harness 带来权衡：加上下文会加重长上下文和指令遵循负担；编排 agent 会增加成本和延迟等 | p.4 “Trade-offs of harness adaptations” 段 |
| 作者提出的未来方向（反过来说明当前尚未做到）：更大规模的实验、更细粒度的任务/模型画像、mixture-of-harnesses 路由、应对分布漂移的在线适配 | p.10 §V-B “For researchers” 两段 |

### 8.2 原文未说明的信息

| 项目 | 状态 |
|---|---|
| “16/21 对显著提升”用的统计检验方法和显著性水平 | 未说明 |
| “追平 SLM–LLM 差距”的判定标准 | 未说明 |
| RQ4 中百分比（81%、86% 等）的分母 | 未明确说明 |
| 每次优化运行的迭代次数（除 budget-approval 示例的 23 次外） | 未说明 |
| SLM 的推理超参数（温度等） | 未说明 |
| SLM 的实际本地部署硬件与本地部署成本（成本按云 API 定价计算） | 未说明 |
| Fig. 5、Fig. 6 中各数据点的具体数值 | 未以文字或表格给出 |
| 受控多样性实验在各模板数下的分模型、分任务结果 | 未说明 |
| 人工检查 harness（RQ4）的编码人数及一致性检验 | 未说明 |

### 8.3 原文内部数字表述差异（照录，不做判断）

| 位置 | 表述 |
|---|---|
| p.1 摘要 | “recovering 89.7% of LLM performance at 4% of the cost” |
| p.2 §I 贡献列表第1条 | “recover 89% of LLM-agent performance with 96% cost reduction” |
| p.8 §IV-D “RQ1” 第1段 | “recover 89% of LLM performance at 4% cost, with 25% latency reduction” |
| p.2 Fig. 1 标题；p.6 §III-C | budget-approval 单任务上“8% of the cost” |

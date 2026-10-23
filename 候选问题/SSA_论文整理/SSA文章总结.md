# SSA 文章总结

**论文**：*Dissecting model behavior through agent trajectories*（Gaurav Gupta, Vatshank Chaturvedi, Jun Huan, Anoop Deoras；AWS AI Labs；arXiv:2606.17454v2，2026-06-17）
**方法名**：SSA（Simple Strands Agent）
**源文件**：`Gupta_2026_AgentTrajectories.pdf`（共 106 页：正文 pp.1–14，参考文献 pp.14–18，附录 pp.19–106）
**代码**：https://github.com/strands-labs/benchmark-harnesses（p.1 摘要末）

**引用约定**
- 页码 = PDF 页脚印的页码，与 PDF 页序一致。
- 段落定位：段首有加粗小标题的，直接写小标题（如“§5.1 *One shared harness…* 段”）；没有小标题的，按该小节在该页上的顺序写“第 N 段”；跨页的段落写“p.x–y”。
- 每条附一句英文原文短摘录，方便核对。
- 只依据论文本身。原文没写的写“**未说明**”。
- PDF 抽取后，表 1 和表 3 的“Model card %”列与模型行错位，所以涉及官方数字时只引用正文写明的数值。

---

## 1. 研究问题

| # | 内容 | 原文依据 |
|---|---|---|
| 1.1 | Agent 的表现不只取决于模型，也是一个系统问题：模型的假设与 harness 的行为不一致时，模型能力无法完全转化为 agent 的表现。 | p.1 摘要第 1–3 句：“AI agent performance is not just a modeling problem, it is fundamentally a systems problem.” |
| 1.2 | 把这种不一致形式化为 **intent-execution gap**（意图–执行鸿沟）。它是双向的：一是模型想做的事与 harness 实际执行的事不一致；二是环境中实际发生的事与模型被告知的情况不一致。如果没有对齐，能力更强的模型 M_A 与较弱的 M_B 可能得到相近的准确率，即 acc(H, M_A) ≈ acc(H, M_B)。 | p.2，§1 第 2 段（“As base models improve…”）、第 3 段（“We formalize this misalignment … as the intent-execution gap”） |
| 1.3 | 两个核心问题：(1) harness 是否忠实执行了模型的意图？(2) 模型是否得到了关于 harness 所做之事的准确描述？ | p.3，§1 第 2 段之后的编号列表：“Does the harness faithfully execute what the model intended? / Is the model given an accurate account of what the harness did?” |
| 1.4 | 针对特定问题的调优（定制 prompt、特化 tool schema、手工执行图）很脆弱，换模型后可能退化。作者因此想找出跨模型家族、跨模型代际、跨基准都有效的**不变接口原则**，只在家族之间确实不同的地方做适配。 | p.3，§1 第 2 段（“Problem-specific tunings … But such gains are brittle … identify invariant interface principles”） |
| 1.5 | 第二个问题：pass@1 回答不了“不同模型家族如何解决同一问题”。在前沿模型之间 pass@1 差距较小，作者想看 pass@1 之外的求解行为差异。 | p.3，§1 “We also analyze SSA trajectories…”段；p.1 摘要（“look beyond the pass@1 numbers which tend to be relatively even across frontier models”） |
| 1.6 | 附带问题：公开的 SWE-Bench-Pro 容器会泄露未来的 git 历史，这在多大程度上抬高了测得的 pass@1？ | p.3，Main Contributions 第 4 条；p.12–13，§5.5 *The public SWE-Bench-Pro containers leak future git history* 段 |

## 2. 研究对象

| # | 内容 | 原文依据 |
|---|---|---|
| 2.1 | **模型与 harness 之间的边界**，即 intent-execution gap 本身。 | p.3，§1 第 2 段（“two fundamental questions at the model-harness boundary”） |
| 2.2 | **SSA harness**：组件包括 tasks、prompts、models、tools、environments、hooks；核心是一个精简的“调用模型 → 执行工具 → 追加结果 → 重复”循环，各家族共用同一个 event loop。 | p.3，§2 第 1 段；p.4，图 2 及 *Model interface* / *Agent core and conversation* / *Tools and environment* / *Hooks and instrumentation* 段 |
| 2.3 | **21 个模型，分属 5 个家族**：Claude（Opus 4.6/4.5、Sonnet 4.6/4.5/4.0、Haiku 4.5）、OpenAI（GPT-5.4、GPT-5.3(-Codex)、GPT-5.2、GPT-5.2-codex、GPT-OSS 120B/20B）、Gemini（3.1 Pro、3 Flash）、Grok 4.2(0)、Qwen（Qwen3-Coder Next/480B/30B、Qwen3.6 35B-A3B/27B、Qwen3.5 27B）。 | p.8，§5.1 第 2 段（“a total of 21 models”）；p.9–10，表 1、表 2；p.32，§A.3 第 2 段（按家族列出全部模型） |
| 2.4 | **Agent 轨迹**：约 138k 条，由 (500 + 731 + 89) × 5 次 × 21 个模型得到。 | p.8，§5.1 第 2 段末（“we generate (500 + 731 + 89) × 5 × 21 ≈ 138k high-quality trajectories”）；p.7，§5 第 1 段 |
| 2.5 | **公开 SWE-Bench-Pro 容器中的 git 历史泄露通道**。 | p.12–13，§5.5；p.71，附录 B 引言段 |

## 3. 数据来源

| # | 内容 | 原文依据 |
|---|---|---|
| 3.1 | **SWE-Bench-Verified**（n = 500）：开源仓库加一个待解决的 issue，自带测试；把补丁放到独立的评测容器中重放。 | p.8，§5.1 第 2 段（“SWE-Bench-Verified [17] (n = 500)”） |
| 3.2 | **SWE-Bench-Pro** 公开集（n = 731）：同上；分 Base（官方公开容器）和 Sanitized（清除未来 git 历史）两种环境。 | p.8，§5.1 第 2 段；p.71，§B.1 第 1 段 |
| 3.3 | **Terminal-Bench-2**（n = 89）：涵盖软件工程、机器学习和安全任务，不依附于某个仓库；原地评测，有逐任务的计算和时间限制；开启互联网。 | p.8，§5.1 第 2 段、第 3 段（“Internet access is disabled for SWE-Bench-Verified and SWE-Bench-Pro and enabled for Terminal-Bench-2”） |
| 3.4 | **运行环境**：AWS PCS 集群（c7.48xlarge），最大并发 10。Claude 通过 Amazon Bedrock 调用；OpenAI、Gemini、Grok 用商业 API；开源权重模型用 vLLM 自行托管，部署在内部 SLURM 集群的 EC2 p4.24xlarge 上，前面加 LiteLLM 代理。SWE-Bench-Pro 在 Amazon ECS 上运行。 | p.8，§5.1 第 1 段；p.100，附录 F 第 1 段；p.13，*A few residual evaluation artifacts…* 段 |
| 3.5 | **每次评测产出的两类文件**：`metrics.json`（每个 cycle 的 token、延迟、缓存、工具成功计数）和 `trajectory.json`（完整消息，含推理块、工具调用及结果）。两者不一致时以 trajectory 为准。 | p.20，附录 A 第 1 段 |
| 3.6 | **构造经验解集的数据**：每个 SWE 实例被尝试 105 次（21 模型 × 5 次），每个 resolved 运行提供一个已验证正确的最终状态；没有任何 resolved 运行的实例退回到 gold patch。SBV 中 474/500 个实例有 resolved 参考，平均每个 82 个；SBP 中 562/731 个，平均每个 60 个，其余 169 个用 gold patch。 | p.22，§A.1.3 *The empirical solution subset* 段 |
| 3.7 | **配置与 prompt**：Hydra YAML 配置、Jinja2 prompt 模板、工具定义和 vLLM 启动命令都在附录中列出，并随 SSA 包开源。 | p.8，§5.1 第 1 段末；p.74，附录 C；p.78，附录 D；p.92，附录 E；p.101，附录 F.1 |
| 3.8 | 数据采集时间、轨迹数据集本身是否公开 | **未说明**（原文只说开源“agent logic, tools, prompts, and all model configurations”，p.14，§6 第 2 段） |

## 4. 核心变量

> 原文没有使用“自变量/因变量”这类术语。下表按实验设计归纳，每一项都对应原文中的具体设置或指标。

**自变量 / 实验条件**

| 变量 | 取值 | 原文依据 |
|---|---|---|
| 模型 | 21 个模型，分属 5 个家族（见 2.3） | p.8，§5.1 第 2 段；表 1–3 |
| 基准 | SBV / SBP / TB2 | p.8，§5.1 第 2 段 |
| 按家族的接口适配 | (a) 推理 nudge：Claude 用定量提示（“ideally more than 100 times”），Gemini、Grok 用灵活提示（“use tools as much as possible”）；(b) 工具接口：GPT 用 apply_patch，Grok 用拆分后的原子 edit/view 工具并关闭行号 | p.5，§3.2 第 3 段、§3.3 第 1–2 段；p.8，§5.1 *One shared harness…* 段；p.85，附录 D.3 |
| 命令批量化 | 单条命令 vs. batch 接口（TB2） | p.11–12，§5.4 *Command batching helps some families and hurts others* 段 |
| TB2 资源设置 | Constrained（遵守内存和时间预算） vs. Unconstrained（去掉限制，作为上界） | p.9，“For Terminal-Bench-2 (Table 3)…”段；p.12，*Relaxing the time and memory limits…* 段 |
| SBP 容器 | Base vs. Sanitized；另对 4 个 Qwen 模型加 Git Instructions 条件 | p.71，§B.1；p.71–73，§B.4 |
| gpt-oss 服务端修复 | 修复前 vs. 修复 Harmony 解析后 | p.103，附录 G 第 1 段；p.106，§G.4 |

**因变量 / 观测指标**

| 指标 | 定义 | 原文依据 |
|---|---|---|
| pass@1 / pass@2 | 采用 Chen et al. 的无偏估计；5 次运行，pass@1 取每次运行解决率的均值，附 95% CI | p.8，§5.1 第 2 段；p.32，§A.2 *Definition of pass@k* 段（式 9） |
| Solution distance D(t) | 以 recall 形式衡量当前代码状态到最近的 oracle 正确状态的距离：D=1 表示无重叠，D=0 表示完全复现某个正确修复 | p.6，§4 式 (1)–(5)；p.7，§4 “In practice, S_i is not available…”段；p.23，式 (6) |
| ΔD 回退类指标 | Backtrack fraction（至少回退一次的轨迹占比）、Backtrack-edit fraction（ΔD>0 的编辑占比）、Uphill mass（按幅度加权的回退量） | p.30，§A.1.7（式 8 及三个标量） |
| Arrival-time t@D≤0.1、AUC | 平均 resolved D(t) 曲线首次降到 0.1 以内时的归一化位置；D(t) 曲线下面积 | p.32，§A.3 第 3 段 |
| 行为构成 | 由 LLM judge 按 R1–R8 rubric 打标签：edit/test 比、scratch 与 suite 测试占比、R6 阶段构成（explore/localize/implement/verify）、真实工具错误率（R8）、工具调用分布（R1） | p.9，§5.3 第 1 段；p.20–22，表 4；p.22，表 4 后一段；p.31，表 6 |
| 成本 | 总输出 token（与 pass@1 一起画 Pareto 图） | p.9，“Gemini 3.1 Pro leads pass@2…”段末；p.32，§A.3 第 1 段（图 13） |
| 泄露 | Base − Sanitized 的 pass@1 差；直接泄露率 = E[L_i]（resolved 且上下文中出现 `git show <gold-sha>`） | p.71，§B.1 第 2 段、§B.2 |

**控制条件（各家族相同）**

| 条件 | 内容 | 原文依据 |
|---|---|---|
| 核心循环、反馈语义、指标埋点 | 各家族完全相同 | p.8，§5.1 *One shared harness…* 段末 |
| 推理设置 | 只要模型卡有说明，就用厂商推荐的高推理设置（Claude 4.x 用 adaptive thinking max effort，GPT-5 用 xhigh，Gemini 用 thinking level high） | p.8，§5.1 第 1 段 |
| 默认参数 | max_iterations = 500；retry_on_no_change.max_attempts = 1；单次超时 SWE 为 200s、TB2 为 120s | p.74，附录 C 第 2 段 |
| 不做任务特定调优 | — | p.3，“We show that minimizing this bidirectional gap without task-specific tuning…”段；p.14，§6 第 1 段 |

## 5. 研究方法

| # | 方法 | 原文依据 |
|---|---|---|
| 5.1 | **Harness 设计：反馈充分的工具边界** | |
| | • 结构化执行反馈：每次 shell 调用返回 command、status、exit code、output、error；超时时返回部分输出；长输出截掉中间、保留首尾。 | p.4，§3.1 *Structured execution feedback* 段 |
| | • 更安全的编辑语义：编辑以相邻行为锚点；部分行匹配或重复匹配会被拒绝并给出反馈；编辑成功后返回 unified diff。 | p.4，§3.1 *Safer editing semantics* 段 |
| 5.2 | **推理与工具调用的平衡（reasoning nudge）**：鼓励模型推理到“刚够决定下一步”为止，然后优先通过工具获取证据。没有一条“golden prompt”对所有家族都有效，所以按家族使用定量或灵活的提示。 | p.5，§3.2 第 1–3 段；图 3 |
| 5.3 | **顺应模型的工具偏好**：GPT 用 apply_patch；Grok 用原子工具并关闭行号。 | p.5，§3.3 第 1–2 段 |
| 5.4 | **基准评测协议**：3 个基准 × 21 个模型 × 5 次运行，报告 pass@1（95% CI）和 pass@2，并与官方模型卡数字比较。 | p.8，§5.1 第 2 段；p.8，§5.2 第 1 段 |
| 5.5 | **Solution distance 框架**：在文本级代码空间 X 中定义实例子空间 X_i、oracle O_i、解集 S_i，用 text-diff 投影和 recall-divergence 计算 D(t)。实际计算时用经验参考集近似 S_i：特征为 (文件, ±, 去掉空白的行内容)；Jaccard ≥ 0.9 的近似状态合并；过滤 scratch 文件特征。 | p.6，§4 式 (1)–(5)；p.22，§A.1.3 *The empirical solution subset* 段及其后两段 |
| 5.6 | **Faithful state replay（忠实状态重放）**：解析每次编辑调用，把它应用到内存中的文件内容上，逐步重算 d(t)；统一处理各类编辑工具（表 5）；支持模糊匹配；处理三类回退，但排除成对出现的 git stash / stash pop。 | p.29，§A.1.5 各段；表 5 |
| 5.7 | **Self-anchor**：resolved 运行自身的补丁被放入它的经验解集，使 D(t_end) = 0。作者试过 leave-one-run-out，结果不可用。 | p.30，§A.1.6 第 1–3 段 |
| 5.8 | **LLM judge**：用 Claude Opus 4.6 对每次工具调用分类。轨迹按滑动窗口切分；judge 只能以 enum 形式调用 `submit_classifications` 工具输出，结果在客户端校验。 | p.20，§A.1.1 |
| 5.9 | **交叉验证**：用 judge 给出的 revert 标签（is_revert）独立核对回退排名。 | p.33，§A.3 *Independent corroboration via the judge* 段；p.10，*Backtracking is decoupled…* 段末 |
| 5.10 | **泄露审计**：比较 Base 和 Sanitized（移除 remotes 和 refs，在原 HEAD 处重建单一 main 分支）；用直接泄露率作为下界；对 4 个 Qwen 模型做 Git Instructions 压力测试。 | p.71，§B.1–B.2、§B.4；p.73，Git Instructions 文本框 |
| 5.11 | **日志诊断**：从异常的零缓存命中率入手，定位 gpt-oss 在 Harmony → OpenAI 格式转换时的解析错误。修复方法是把所有错误统一为 HTTP 503 的 HarmonyParseError，由 SSA 退避重试。 | p.103，§G.1 第 1–3 段；p.106，§G.4 |

## 6. 主要结论

| # | 结论 | 原文依据 |
|---|---|---|
| 6.1 | 在一个共享 harness、只加少量家族适配的条件下，所测模型的 pass@1 达到或超过官方模型卡数字：官方数字都落在 SSA 的 95% CI 之内或之下，唯一例外是 SWE-Bench-Pro 上的 GPT-5.2-codex。 | p.8，§5.2 第 1 段、“On SWE-Bench-Pro (Table 1)…”段 |
| 6.2 | SWE-Bench-Pro 上 pass@1 范围为 27.4%（Qwen3-Coder 30B）到 59.9%（GPT-5.4）；pass@2 最高 69.8%（Qwen3-Coder Next）。 | p.8，“On SWE-Bench-Pro (Table 1)…”段 |
| 6.3 | SWE-Bench-Verified 上 pass@1 范围为 51.4% 到 80.8%，Opus 4.6 与 Gemini 3.1 Pro 并列最高（80.80%）；Gemini 3.1 Pro 的 pass@2 最高（84.1%）。 | p.8–9，“On SWE-Bench-Verified (Table 2)…”段 |
| 6.4 | TB2 Constrained 设置下 GPT-5.3（75.05%）与 Opus 4.6（73.93%）领先；Unconstrained 设置下 Opus 4.6 最高（83.14%）。开源模型只跑了 Unconstrained，明显落后（最高为 Qwen3.6 27B 的 56.0%）。 | p.9，“For Terminal-Bench-2 (Table 3)…”段 |
| 6.5 | pass@1 相近的模型，轨迹可以差别很大：Opus 4.6 更早开始测试，并持续投入到最后；两个模型在 SBP 上的编辑占比都更高，测试投入反而更少。 | p.9–10，§5.3 *Trajectories taken by different models can be very different…* 段；p.12，图 5 |
| 6.6 | 不同家族在相同任务上的阶段安排不同：implement 阶段峰值的位置和宽度不同，verify 阶段的长尾不同。 | p.10，*Different model-families execute same tasks with different phase schedules* 段；p.13，图 6 |
| 6.7 | 回退程度与解决率无关：SBV 上 backtrack-edit fraction 从 0.8%（GPT-5.2-codex）到 16.8%（Gemini 3 Flash）；Gemini 3.1 Pro（14.3%）与 GPT-5.4（1.2%）的解决率相差不到一个点。SBP 上全部模型都低于 6%。judge 给出的 revert 标签得到几乎相同的排名。 | p.10，*Backtracking is decoupled from solve rate* 段；p.33，§A.3 第 3 段；p.47，§A.4 Table 8 说明段 |
| 6.8 | 相同 pass@1 的模型在代码状态空间中走的路径不同：Opus 4.6 的 D(t) 骤降，Gemini 3.1 Pro 下降缓慢且较均匀。 | p.11，“Two models can post identical pass@1…”段；p.14，图 7 |
| 6.9 | TB2 的分数混入了基础设施稳定性的影响（推理后端吞吐、延迟方差、单节点并发分摊带宽）；去掉时间和内存限制后 pass@1 提高约 5–10 个点。 | p.11，§5.4 第 1 段；p.12，*Relaxing the time and memory limits…* 段 |
| 6.10 | 命令批量化对 Gemini 和 Grok 有帮助，对 Claude 有害；推理与交互之间的最佳平衡点因家族而异，没有全局最优。 | p.11–12，§5.4 *Command batching…* 段 |
| 6.11 | SWE-Bench-Pro 公开容器泄露未来的 git 历史：Sanitized 后 Opus 4.6 下降 6.87 个点，GPT-5.4 下降 3.32，Sonnet 4.6 下降 3.66；明确指示“查看 git 历史”时，Qwen3.6 27B 提高 17.13%，Qwen3-Coder Next 提高 21.12%。作者建议改用 Sanitized 分数，或明确报告泄露暴露程度。 | p.12–13，§5.5 *The public SWE-Bench-Pro containers leak…* 段；p.72，表 10；p.73，§B.4 末段 |
| 6.12 | gpt-oss-120b 修复 Harmony 解析问题后，SBV pass@1 从 42% 升到 64.64%（模型卡为 62%）。 | p.103，附录 G 第 1 段；p.106，§G.4 *Evaluation after the fix* 段 |
| 6.13 | 总结论：官方报告的表现与复现出的表现之间的差距，很大程度上来自 harness 摩擦，而不是简单的 ReAct 单 agent 设计有根本缺陷。好的 harness 应当 (i) 采用对所有家族都有益的原则（行锚定编辑、编辑后返回 diff、忠实序列化工具输出），(ii) 在家族确实不同的地方顺应其偏好。 | p.14，§6 第 1 段 |
| 6.14 | 模型在这些基准上的分数趋于接近后，pass@1 不再能区分它们，轨迹却仍能区分；理解轨迹行为是做模型–harness 协同设计的前提。 | p.14，§6 第 2 段 |

## 7. 适用条件

| # | 条件 | 原文依据 |
|---|---|---|
| 7.1 | 适配按**模型家族**进行，不针对单个模型优化；允许的修改仅限 prompt 层面的推理 nudge 和工具接口的选择。 | p.8，§5.1 *One shared harness, minimal family adapters* 段 |
| 7.2 | 家族适配不能跨家族照搬：定量 nudge 对 Gemini/Grok 无效（它们会发出空的工具调用来凑数）；批量化对 Claude 不利。 | p.5，§3.2 第 3 段；p.11–12，§5.4 *Command batching…* 段 |
| 7.3 | Solution distance 依赖**文本级代码空间**和 unified-diff 形式的参考补丁。TB2 没有统一 diff 形式的 gold patch，所以不计算 D(t) 和 ΔD。 | p.6，§4 *Prior Work* 段之后一段（“Note that X_i is an instance-level subspace and not a semantic program space”）；p.61，§A.5 第 3 段 |
| 7.4 | D(t) 需要用与参考解相同的 unified diff 算法提取，路径要相对工作区归一化；否则 D 会虚高或一直停在 1。 | p.23，§A.1.3 *Implementation specifics* 段 |
| 7.5 | 经验参考集越稠密越可靠：简单实例上可靠，困难实例或无人解出的实例上要谨慎解读。 | p.31，§A.1.8 第 3 条 |
| 7.6 | 评测基准需要自带测试（作为 oracle）；SWE 类基准要求从最终 diff 中排除 scratch 文件和构建产物。 | p.8，§5.1 第 2 段（“All three ship with pre-existing tests … scratch files and build artifacts must be excluded”）；p.6，式 (3) 前一段 |
| 7.7 | TB2 的结论受基础设施影响（吞吐、并发、带宽），Unconstrained 结果只能作为上界。 | p.11，§5.4 第 1 段；p.12，*Relaxing…* 段 |
| 7.8 | SBP 的比较应使用 Sanitized 容器，或者报告泄露暴露程度。 | p.13，§5.5 续段（“recommend that SWE-Bench-Pro comparisons either use Sanitized scores or explicitly report leakage exposure”）；p.73–74，§B.4 末段 |
| 7.9 | 实证范围：3 个编码/终端类基准、5 个家族的 21 个模型。超出这一范围的效果 | **未说明** |

## 8. 研究局限

**原文明确承认的局限**

| # | 局限 | 原文依据 |
|---|---|---|
| 8.1 | Solution distance 衡量的是**文本重叠，不是语义**：一个正确但写法不同的修复，得分会小于 1。 | p.30，§A.1.8 第 1 条 |
| 8.2 | **经验子集只是对解空间的近似**：一个别人都没找到的独特正确解，可能离经验参考集很远。 | p.30，§A.1.8 第 2 条 |
| 8.3 | **参考集的稀疏程度随难度增加**：难题上只有 gold patch 可作参考，甚至没有参考。 | p.31，§A.1.8 第 3 条 |
| 8.4 | **重放保真度有限**：不执行任意 shell 或 Python（如 `sed -i`、写文件的脚本），这些编辑的中间状态只能近似；终点靠 self-anchor 归因补齐。 | p.31，§A.1.8 第 4 条；p.29，§A.1.5 *Bash-based edit extraction* 段 |
| 8.5 | Leave-one-run-out 打分不可用，所以采用 self-anchor；没有 self-anchor 时，终点距离为正可能只反映参考集稀疏。 | p.30，§A.1.6 第 1–3 段 |
| 8.6 | 选 recall 而不选 Jaccard，放弃了距离的理论性质；precision 类惩罚、编辑距离、语义散度等方案留作未来工作。 | p.23，§A.1.3 *Divergence choices* 续段（“sacrificing the theoretical distance property of Eq.7 … we leave those to future work”） |
| 8.7 | 补丁特征采用简单的“行集合”表示，AST 或语义相似度方法有待探索。 | p.22，§A.1.3 “Next, we represent each patch…”段 |
| 8.8 | 直接泄露率**只是下界**：只能捕捉精确 SHA 的暴露，漏掉 `git log -p`、相邻 commit 等间接泄露。 | p.71，§B.2 第 2 段 |
| 8.9 | SBP 的逐模型行为分析基于 Base 容器的轨迹，而不是 Sanitized 轨迹，因此带有 Base 与 Sanitized 之间的差异。 | p.47，§A.4 “Figures 27–38 show…”段 |
| 8.10 | 评测本身的残余问题限制了可达上限：SBV 中 astropy-8872/8707 等因环境设置问题即使补丁正确也会失败，psf_requests 部分实例依赖不稳定的外部 URL；SBP 中有 3/731 个实例在 AWS 环境下始终失败，所有 SSA 运行都因此损失约 0.41%。 | p.13，*A few residual evaluation artifacts cap the achievable ceiling* 段 |
| 8.11 | TB2 上开源模型只跑了 Unconstrained 设置。 | p.9，“For Terminal-Bench-2…”段；p.11，表 3 标题 |
| 8.12 | SBP 上的小差距要谨慎解读，因为原始比较混入了容器泄露的影响。 | p.8，“On SWE-Bench-Pro (Table 1)…”段末 |

**原文中可核对的不一致或缺口**（只列原文本身能对上的现象，不做推断）

| # | 现象 | 原文依据 |
|---|---|---|
| 8.13 | 配置数量前后不一致：§5.1 写 71 个配置，附录 C 写 63 个（3 个基准 × 21），原文没有解释。 | p.8，§5.1 第 1 段；p.74，附录 C 第 1 段 |
| 8.14 | 正文说 SSA 优于常用的开源 harness，但正文和附录都没有给出对比实验的数据或表格。 | p.7，§5 第 1 段（“ahead of widely-used open-source harnesses”）；p.9，“Taken together…”段 |
| 8.15 | 交叉验证使用的 revert 标签记为 R9，但表 4 的 rubric 只定义了 R1–R8。 | p.33，§A.3 *Independent corroboration* 段；p.20–22，表 4 |
| 8.16 | LLM judge（Claude Opus 4.6）的标注准确性（如与人工标注的一致性） | **未说明** |
| 8.17 | SSA 各项改进（§3.1–3.3）对 pass@1 的单独贡献（消融实验） | **未说明**（原文只报告了 gpt-oss 修复前后的数字，以及定性的 “we found”） |
| 8.18 | 家族适配是怎样搜索或确定的（流程、尝试次数、人工成本） | **未说明**（原文只用 “we found”、“from looking at the logs” 等描述，p.5 §3.2 第 3 段、p.11 §5.4 第 1 段） |
| 8.19 | 金钱成本、总耗时 | **未说明**（只给了输出 token 的 Pareto 图，p.32 图 13 等） |
| 8.20 | 行为差异（阶段构成、回退率等）的显著性检验 | **未说明**（图中只给出 ±1 个运行间标准差的阴影带，p.14 图 7） |

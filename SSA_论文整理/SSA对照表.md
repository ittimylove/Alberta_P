# SSA 对照表

- **论文**：Gupta, Chaturvedi, Huan, Deoras, *Dissecting model behavior through agent trajectories*（arXiv:2606.17454v2，AWS AI Labs）
- **方法名**：SSA（Simple Strands Agent）
- **源文件**：`Gupta_2026_AgentTrajectories.pdf`
- **代码**：https://github.com/strands-labs/benchmark-harnesses（p.1）

> **标注说明**
> - 页码 = PDF 页脚印的页码。依据写作「p.页码，章节，段落（段首加粗小标题或段首原文）」，或写明图、表、附录位置。
> - 「**未说明**」表示原文没有给出这项信息。
> - 与本系列其他对照表的区别：SSA 是作者设计的一个 harness，论文没有描述自动迭代修改 harness 的流程。表中的“适配”指论文所说的“按模型家族的接口适配”（family adapters）。
> - PDF 抽取后，表 1、表 3 的“Model card %”列与模型行错位。凡涉及官方数字的地方，只引用正文中作者明确写出的数值。
> - 原文前后不一致：§5.1 写“71 benchmark configurations”（p.8），附录 C 写“63 benchmark configurations”（3 个基准 × 21，p.74），原文没有解释。

---

## 速览

| 维度 | 一句话回答 |
|---|---|
| 场景与输入 | 用一个共享 harness 加少量按模型家族的接口适配，缩小模型与 harness 之间的 intent-execution gap，不做任务特定调优。作者依据观察到的模型行为和日志信号（如缓存命中率异常）做出适配；是否有固定流程、由什么触发，**未说明** |
| 修改对象 | 按家族允许的修改被刻意限定在两类：提示词层面的推理 nudge，以及工具接口的选择（GPT 用 apply_patch，Grok 用拆分后的原子工具并关闭行号）。核心循环、反馈语义、指标埋点对所有家族相同。此外修复了 gpt-oss 在 serving 侧的 Harmony 解析问题 |
| 成本 | 报告了运行硬件（AWS PCS c7.48xlarge，并发 10；开源模型在 p4.24xlarge 上用 vLLM 托管）、默认配置，并用散点图给出每实例输出 token。设计适配的人工成本、金钱成本、总耗时、LLM judge 的开销均**未说明** |
| 成功标准 | pass@1（5 次运行取均值，附 95% CI）达到或超过官方模型卡数字；另用 solution distance、回退率、阶段构成等轨迹指标刻画求解过程；SWE-Bench-Pro 建议采用 Sanitized 分数 |
| 迁移范围 | 同一 harness 用于 5 个家族的 21 个模型、3 个基准，不做任务特定调优。家族适配不能跨家族照搬：Claude 的定量 nudge 对 Gemini/Grok 无效，命令批量化对 Claude 不利。其他基准类型、其他 harness 实现上的效果**未说明** |

---

## 一、场景与输入

### 1.1 在什么情况下适配

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 动机场景 | 基座模型变强后，harness 与模型是否对齐变得关键：没有对齐时，能力更强的模型 M_A 和较弱的 M_B 可能得到相近的准确率，即 acc(H, M_A) ≈ acc(H, M_B) | p.2，§1 第 2 段（"As base models improve…"） |
| 核心问题定义 | intent-execution gap 是双向的：harness 是否忠实执行了模型的意图；模型是否得到了 harness 实际所做之事的准确描述 | p.2，§1 第 3 段；p.3，§1 两个问题 |
| 反对的做法 | 问题特定的调优（定制提示词、特化工具 schema、手工执行图）很脆弱，换模型后可能退化 | p.3，§1（"Problem-specific tunings…"段） |
| 采取的策略 | 找出跨模型家族、跨模型代际、跨基准都有效的不变接口原则，只在家族之间确实有差异的地方做适配 | p.3，§1（"Problem-specific tunings…"段） |
| 适配粒度 | 按模型家族，不按单个模型单独优化（"not as a separately optimized agent per model"） | p.8，§5.1 *One shared harness, minimal family adapters* 段 |
| 适配是怎么发现的 | 原文用 "we found" 描述提示词 nudge 的效果；TB2 的系统因素来自 "looking at the logs"；gpt-oss 的问题是从异常的零缓存命中率入手查出的 | p.5，§3.2 第 3 段；p.11，§5.4 第 1 段；p.103，附录 G.1 第 1 段 |
| 是否有自动搜索或优化 harness 的流程 | **未说明**（论文没有描述） | — |
| 触发条件（例如新模型发布时重新适配） | **未说明** | — |
| 部署时 / 在线适配 | **未说明** | — |
| 任务特定调优 | 不做（"without task-specific tuning"） | p.3，§1（"We show that minimizing…"段）；p.14，§6 第 1 段 |

### 1.2 能看到哪些轨迹和反馈

**（a）分析与设计层面使用的数据**

| 输入 | 内容 | 原文依据 |
|---|---|---|
| 每次评估的产物 | metrics.json：每个 cycle 的 token、延迟、缓存、工具成功次数等计数器；trajectory.json：逐字记录全部 user / assistant 消息（含 reasoning 块、工具调用、工具结果）。两者冲突时以 trajectory 为准 | p.20，附录 A 第 1 段 |
| 轨迹规模 | (500 + 731 + 89) × 5 × 21 ≈ 138k 条 | p.8，§5.1 第 2 段 |
| 结果反馈 | 三个基准都自带测试。SWE 两个基准把工作区改动序列化为 patch，在独立的评测容器中回放；TB2 原地评测 | p.8，§5.1 第 2 段 |
| 逐调用标注 | 由 Claude Opus 4.6 担任 LLM judge，按 R1–R8 rubric 给每次工具调用分类。轨迹按滑动窗口切分，judge 只能通过 submit_classifications 工具输出枚举值 | p.20，附录 A.1.1；p.20–22，表 4 |
| 轨迹几何 | solution distance D(t)：通过忠实状态回放加经验解集 S̃ᵢ 计算。TB2 没有统一 diff 格式的 gold patch，因此不计算 | p.6–7，§4；p.22，附录 A.1.3；p.29，附录 A.1.5；p.61，附录 A.5 |
| 经验解集的构造 | 每个 SWE 实例被尝试 105 次（21 个模型 × 5 次），每个已解决运行的最终状态都加入解集，无人解决的实例用 gold patch 兜底，按 Jaccard ρ = 0.9 去重。SBV 有 474/500 个实例被已解决运行覆盖，SBP 为 562/731 | p.22，附录 A.1.3 *The empirical solution subset* 段 |
| 系统层信号 | 缓存命中率（用于诊断 gpt-oss 问题）；推理后端吞吐、延迟方差、超时与重试（TB2） | p.103，附录 G.1；p.11，§5.4 第 1 段 |
| 分析结果如何反馈到 harness 修改 | 结论只说“理解轨迹行为使 model-harness co-design 成为可能”；具体如何用于修改 harness，**未说明** | p.14，§6 第 2 段 |

**（b）运行时 agent（模型）能看到的反馈**

| 输入 | 内容 | 原文依据 |
|---|---|---|
| shell 执行反馈 | 每次 shell 调用都返回 command、status、exit code、output、error text。超时时返回已有的部分输出；长输出截掉中间、保留头尾 | p.4，§3.1 *Structured execution feedback* 段 |
| 编辑反馈 | 编辑锚定在相邻行上；部分行匹配和重复匹配会被拒绝，并附上帮助消歧的反馈；编辑成功后返回 unified diff | p.4，§3.1 *Safer editing semantics* 段 |
| 执行环境 | 在 Docker 中隔离。SWE 两个基准禁网，TB2 联网 | p.3，§2 第 1 段；p.8，§5.1 第 3 段 |
| 测试信息 | SWE 提示词告知 agent 测试文件已处理好，无需修改；swe_generic_v3 提醒 agent 还存在看不到的隐藏测试 | p.85，附录 D.3；p.79，附录 D.1 |
| 非预期可见信息 | SWE-Bench-Pro 的公开 Base 容器保留了未来的 git 对象，agent 可以用 git log -p、git show 等命令看到修复；Sanitized 容器会移除这些对象 | p.12，§5.5 第 1 段；p.71，附录 B、B.1 |

---

## 二、修改对象

| 要点 | 内容 | 原文依据 |
|---|---|---|
| harness 组件 | tasks、prompts、models、tools、environments、hooks | p.3，§2 第 1 段 |
| 允许按家族修改的范围 | 刻意限定得很窄：提示词层面的推理 nudge，以及工具接口的选择 | p.8，§5.1 *One shared harness…* 段 |
| 各家族不变的部分 | 核心执行循环、反馈语义、指标埋点 | p.8，§5.1 *One shared harness…* 段 |
| 提示词：推理 nudge | Claude 用定量目标（"make 50+ tool calls"、"ideal tool call count is 100"；Anthropic_v2 写作 "ideally more than 100 times"）。Gemini、Grok 用灵活表述 "You should use tools as much as possible" | p.5，§3.2 第 3 段；p.85，附录 D.3 |
| 提示词：变体 | swe_generic_v1（Python 专用）、v2（语言无关）、v3（强调运行已有测试，提醒存在隐藏测试）、think_v1（在固定节点调用独立的 think 工具）、think_v2（加入推理 / 工具调用 nudge）、swe_generic_v2_gpt（内嵌 apply_patch 说明）、tb2、tb2_gemini、tb2_gpt、tb2_q36（加入重读与最终验证，避免过早退出） | p.78–90，附录 D |
| 工具：GPT | 使用 apply_patch 风格的编辑；剥夺模型偏好的格式会损害表现 | p.5，§3.3 第 1 段；p.92，表 15 |
| 工具：Grok | 把查看和编辑合在一起的单体工具效果差，拆成原子操作（file_read / file_edit / file_write）后更好；关闭行号显示；file_read 用 _infer_line_range 兼容 Grok 给出的各种行号参数名；search 工具的描述里写明“禁止用 bash 调 grep”这条硬规则 | p.5，§3.3 第 2 段；p.99–100，附录 E.3 |
| 工具：TB2 批量接口 | TB2 的 GPT 配置只绑定 batch_shell；Gemini 使用 batch_bash | p.92，附录 E 及表 15 |
| 推理设置 | 凡模型卡有说明的，都采用厂商推荐的高推理设置（Claude 4.x 用 adaptive thinking + max effort，GPT-5 用 xhigh，Gemini 用 thinking level high）；开源模型设置采样参数 | p.8，§5.1 第 1 段；p.74–77，附录 C |
| hooks | 校验模型输出、丢弃格式错误的工具调用、管理上下文窗口、限制循环次数、添加缓存标记、记录轨迹和指标 | p.4，§2 *Hooks and instrumentation* 段 |
| 上下文管理 | conversation manager 保留 tool-use / tool-result 结构，上下文过长时压缩大输出或裁掉旧轮次 | p.4，§2 *Agent core and conversation* 段 |
| serving 侧修复（gpt-oss） | 加固 OpenAI 兼容服务器，把所有已观察到的损坏模式统一成一个信号：HTTP 503 返回的 HarmonyParseError；SSA 把它当作可重试错误并退避。改用修正后的 vLLM 构建后，这类解析错误降为 0 | p.106，附录 G.4 |
| 任务特定逻辑 | 不做 | p.14，§6 第 1 段 |
| 记忆机制 | **未说明** | — |
| 模型权重 | **未说明** | — |

---

## 三、成本

### 3.1 运行评测

| 项目 | 内容 | 原文依据 |
|---|---|---|
| 硬件与并发 | AWS PCS 集群，c7.48xlarge 实例，最大并发 10。Claude 通过 Amazon Bedrock（生产容量）调用；OpenAI、Gemini、Grok 走商业 API | p.8，§5.1 第 1 段 |
| 开源模型托管 | 部署在内部 SLURM 集群（EC2 p4.24xlarge）上的私有 vLLM 端点，前面接 LiteLLM 代理；开启 compile mode，解码速度约为 eager 模式的 2–3 倍；不做量化 | p.100，附录 F；p.102，附录 F.2 |
| 运行量 | 每个模型在每个基准上完整跑 5 次 | p.8，§5.1 第 2 段 |
| 每实例输出 token | 以“输出 token vs pass@1”散点图给出：图 13（SBV）、图 26（SBP）、图 39（TB2）；正文没有给出逐模型的数值表 | p.33，图 13；p.47，附录 A.4；p.61，附录 A.5 |
| 默认配置 | max_iterations = 500；retry_on_no_change.max_attempts = 1；timeout 为 SWE 200 s、TB2 120 s | p.74，附录 C |
| TB2 资源约束 | 基准自带内存、存储、CPU 限制和 agent / verifier 时限。放开这些限制后 pass@1 提高约 5–10 个百分点，收益主要来自少数在限制下反复超时的项目 | p.11–12，§5.4 |
| 金钱成本、GPU / 计算小时、总耗时 | **未说明** | — |

### 3.2 轨迹分析

| 项目 | 内容 | 原文依据 |
|---|---|---|
| LLM judge | Claude Opus 4.6 给每一次工具调用打标签；token 与金钱开销**未说明** | p.20，附录 A.1.1 |
| 忠实状态回放 | 逐个解析编辑调用、在内存中重建文件状态，每步重新计算 d(t)；作者指出单调替代方法“计算便宜”但无法反映回退。回放本身的开销**未说明** | p.29，附录 A.1.5 第 1 段 |

### 3.3 设计适配本身

| 项目 | 内容 | 原文依据 |
|---|---|---|
| 人工投入、尝试过的方案数量、迭代轮数 | **未说明** | — |
| 与其他 harness 的推理成本对比 | **未说明** | — |
| gpt-oss 重试机制带来的额外开销 | **未说明** | — |

---

## 四、成功标准

### 4.1 判定标准

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 主指标 | pass@1（采用 Chen et al. 的无偏估计）：每个模型在完整基准上跑 5 次取均值，并在所有试次上计算 95% CI（SBV 2,500 次，SBP 3,655 次，TB2 445 次）；同时报告 pass@2 | p.8，§5.1 第 2 段；p.31，附录 A.2 *Definition of pass@k* 段 |
| 成功的标准 | 官方模型卡数字落在 SSA 的 95% CI 之内或之下，即“复现或超过” | p.8，§5.2 第 1 段 |
| SWE-Bench-Pro | pass@1 介于 27.4%（Qwen3-Coder 30B）到 59.9%（GPT-5.4）之间。唯一例外是 GPT-5.2-codex，其官方数字高于 SSA 的区间；作者认为 SBP 的原始比较部分受容器泄漏影响 | p.8，§5.2 第 2 段 |
| SWE-Bench-Verified | pass@1 介于 51.4% 到 80.8% 之间，Opus 4.6 与 Gemini 3.1 Pro 并列最高（80.80%）；作者称所有官方数字都落在 SSA 的区间内 | p.8–9，§5.2 第 3 段 |
| Terminal-Bench-2 | Constrained 设定下 GPT-5.3（75.05%）、Opus 4.6（73.93%）领先；Unconstrained 设定下 Opus 4.6 最高（83.14%）；开源模型只跑了 Unconstrained | p.9，§5.2 第 4 段 |
| gpt-oss-120b 修复前后 | SBV 上从 42% 升到 64.64%（模型卡为 62%） | p.103，附录 G 第 1 段；p.106，附录 G.4 |
| 与其他开源 harness 的比较 | 正文称 SSA “ahead of widely-used open-source harnesses”，但没有给出这些 harness 的名称或数值，**未说明** | p.7，§5 第 1 段；p.9，§5.2 末段 |
| 各项改进的单独贡献（消融） | 结构化反馈、安全编辑、推理 nudge、工具偏好对齐各自贡献多少，**未说明**（只报告了 gpt-oss 修复、TB2 放开限制、git 泄漏这几项对比） | — |

### 4.2 过程指标（pass@1 之外）

| 指标 | 内容 | 原文依据 |
|---|---|---|
| solution distance D(t) | 从当前仓库状态到最近的已验证正确解的召回式距离：起点为 1，完整复现修复时为 0 | p.6–7，§4，式 (5)；p.22–23，附录 A.1.3 |
| 回退相关指标 | backtrack-edit fraction、backtrack fraction、uphill mass | p.30，附录 A.1.7 |
| 其他摘要量 | 到达时间 t@D ≤ 0.1、D(t) 曲线下面积（AUC） | p.32，附录 A.3 |
| 行为构成 | 编辑 / 测试占比、R6 阶段构成（explore / localize / implement / verify）、真实工具错误率 | p.9，§5.3 第 1 段；p.31–32，表 6 |
| 主要发现 | 解题率相近的模型，轨迹可以差别很大；回退率与解题率脱钩（SBV 上 Gemini 3.1 Pro 回退率 14.3%，GPT-5.4 为 1.2%，两者 pass@1 相差不到 1 个百分点） | p.10，§5.3 |
| 交叉验证 | 由 judge 得到的 revert 标签给出的排序与忠实回放的结果几乎一致 | p.33，附录 A.3 *Independent corroboration via the judge* 段 |
| 指标局限 | 衡量的是文本重叠，不是语义；经验解集不等于真实解空间；难题上参考解稀疏；回放保真度有限 | p.30–31，附录 A.1.8 |

### 4.3 评测完整性与负面效果

| 要点 | 内容 | 原文依据 |
|---|---|---|
| SWE-Bench-Pro git 泄漏 | 从 Base 换到 Sanitized 后：Opus 4.6 下降 6.87 个点（57.45% → 50.58%），GPT-5.4 下降 3.32，Sonnet 4.6 下降 3.66；其他模型持平或在噪声范围内 | p.12，§5.5；p.72，表 10 |
| 显式 git 指令消融 | 让模型主动查看 git 历史后，相对 Sanitized：Qwen3.6 27B +17.13%，Qwen3-Coder Next +21.12% | p.12–13，§5.5；p.73，附录 B.4 |
| 作者建议 | SWE-Bench-Pro 的比较应使用 Sanitized 分数，或明确报告泄漏暴露程度 | p.13，§5.5；p.74，附录 B.4 末段 |
| 残余评测问题 | SBV 上 astropy-8872、astropy-8707 即使 patch 正确也会失败；部分 psf_requests 实例因外部 URL 无响应而间歇失败；SBP 在 Amazon ECS 上有 3/731 个实例稳定失败，所有 SSA 运行都损失约 0.41% 的上限 | p.13，§5.5 末段 |
| 适配用错家族的负面效果 | Claude 式的定量 nudge 让 Gemini、Grok 为了凑数而发出空工具调用；命令批量化对 Gemini、Grok 有帮助，对 Claude 不利。具体降幅**未说明** | p.5，§3.2 第 3 段；p.11–12，§5.4 *Command batching…* 段 |
| 统计显著性检验 | 只给出 95% CI；显著性检验**未说明** | p.8，§5.1 |

---

## 五、迁移范围

| 换的是什么 | 是否测试 | 具体情况 | 原文依据 |
|---|---|---|---|
| **换模型（同一 harness）** | 是 | 5 个家族（Claude、GPT、Gemini、Grok、Qwen）共 21 个模型，使用同一个 harness 加家族适配器，不为每个模型单独优化 | p.3，§1 贡献 1；p.8，§5.1 *One shared harness…* 段 |
| 同家族跨代 | 部分 | 评测包含同一家族的多代模型（如 Sonnet 4.0 / 4.5 / 4.6，GPT-5.2 / 5.3 / 5.4）；把旧配置直接用于新一代模型的单独实验，**未说明** | p.9–10，表 1、表 2 |
| 家族适配跨家族照搬 | 是（负面） | Claude 的定量 nudge 对 Gemini、Grok 无效；批量命令对不同家族的效果方向相反。作者结论：没有一种工具接口在所有家族上都最优 | p.5，§3.2 第 3 段；p.12，§5.4；p.14，§6 第 1 段 |
| 未评估过的模型家族 | **未说明** | — | — |
| **换任务 / 基准** | 是 | 3 个基准：SWE-Bench-Verified（500 题）、SWE-Bench-Pro（公开集 731 题）、Terminal-Bench-2（89 题，覆盖软件工程、机器学习、安全，不依附于仓库）；提示词按基准区分（swe_generic 系列与 tb2 系列）；不做任务特定调优 | p.8，§5.1 第 2 段；p.78–90，附录 D |
| 其他类型的任务（如非编码、Web 任务） | **未说明** | — | — |
| **换执行环境** | 部分 | 架构图标明环境可以是 Docker 沙箱或本地，实验全部使用 Docker；SBP 运行在 Amazon ECS 上，3 个实例因 AWS 特定的环境假设失败；TB2 结果对基础设施敏感（推理后端吞吐、同一节点上多个项目分摊网络带宽） | p.4，图 2；p.3，§2 第 1 段；p.13，§5.5 末段；p.11，§5.4 第 1 段 |
| **换 serving 栈** | 是（案例） | 同一个 gpt-oss-120b，在存在 Harmony 解析缺陷的 vLLM 上 pass@1 为 42%，修复后为 64.64%；作者把失败定位在模型服务器的 Harmony 处理，而不是模型权重或 SSA 的工具注册表 | p.103，附录 G；p.106，附录 G.4 |
| **换 harness 实现** | **未说明** | 作者主张有一些对所有家族都有用的原则（行锚定编辑、显式的编辑后 diff、忠实的工具输出序列化），但没有在其他 harness 框架中验证 | p.14，§6 第 1 段 |
| solution distance 指标的适用范围 | 部分 | 只用于两个 SWE 基准；TB2 没有统一 diff 格式的 gold patch，无法构造经验解集 | p.61，附录 A.5 |

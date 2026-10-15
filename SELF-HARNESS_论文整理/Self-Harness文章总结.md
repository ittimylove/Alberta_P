# Self-Harness 文章总结

> **论文**：Self-Harness: Harnesses That Improve Themselves
> **作者**：Hangfan Zhang, Shao Zhang, Kangcong Li, Chen Zhang, Yang Chen, Yiqun Zhang, Lei Bai, Shuyue Hu（上海人工智能实验室）
> **版本**：arXiv:2606.09498v3 [cs.CL]，2026-08-20（p.1 页眉）
> **唯一信息来源**：`SELF-HARNESS_论文整理/Zhang_2026_Self-Harness.pdf`（共 31 页）
>
> **标注约定**：`p.X §Y ¶N` = 第 X 页、第 Y 节、该页内该节的第 N 个自然段（跨页段落按其在该页出现的位置计；“续”表示承接上一页的段落）。引文保留英文原句以便核对。原文未提及的内容一律写“未说明”。

---

## 1. 研究问题

**核心问题**：一个参数固定的 LLM 能否在不依赖人类工程师或更强外部智能体的情况下，改进约束其自身行为的 harness（外围执行框架）。

| 要点 | 原文依据 |
|---|---|
| 固定模型能否改进支配其自身智能体行为的 harness | p.14 §5 ¶1：“This paper studied whether a fixed language model can improve the harness that governs its own agent behavior.” |
| 更窄的、受控的问题设定：同一固定模型在当前 harness 下，能否对支配其未来行为的 harness 提出有界的候选修改 | p.4 §2 第4段（“Finally…”）：“whether the same fixed model, operating under the current harness, can propose a bounded candidate change to the harness that governs its own future behavior.” |
| 问题动机①：harness 设计具有模型特异性 | p.1 摘要：“Because different models exhibit distinct behaviors, effective harness design is inherently model-specific.” |
| 问题动机②：人工设计 harness 难以随模型多样化、快速迭代而扩展 | p.1 摘要；p.2 §1 ¶1（续）：“manually redesigning and tuning a model-specific harness for each model becomes increasingly costly and untenable.” |
| 问题动机③：用更强外部智能体改进弱智能体（如 Meta-Harness）的外部指导可能代价高、对前沿模型不可得、或与目标模型的失败模式不匹配 | p.2 §1 ¶2（“In this paper, we explore…”）：“external guidance that may be costly, unavailable for frontier models, or mismatched to the target model's failure modes.” |

---

## 2. 研究对象

| 要点 | 原文依据 |
|---|---|
| **被改进的对象是 harness**，而非模型参数。harness 定义为：支配固定语言模型如何作为智能体部署的非参数化脚手架，包括指令、可用工具、记忆与状态管理机制等 | p.4–5 §3.1 ¶1：“We use harness to denote the non-parametric scaffolding that governs how a fixed language model is deployed as an agent. A harness includes the instructions, the available tools, memory and state-management mechanisms, etc.” |
| harness 的更宽泛描述：prompts、tools、memory、verification rules、permission policies、adapters、runtime mechanisms | p.4 §2 ¶2（“We use harness for this surrounding system layer…”） |
| 形式化：固定模型 M、harness h；任务 x 产生执行轨迹 τ 与输出 y；评估器 𝓔 映射为 pass/fail；M 与 𝓔 固定，只改 h，形成 harness 谱系 h₀, h₁, … | p.5 §3.1 ¶2（“Formally, let M be a fixed language model…”） |
| **基座模型（3 个，来自不同家族）**：MiniMax M2.5、Qwen3.5-35B-A3B、GLM-5 | p.1 摘要；p.9 §4.1 “Models.” 段 |
| **基准（3 个）**：Terminal-Bench-2.0（容器化终端交互）、SWE-bench Verified（仓库级软件修复）、AppWorld（多应用工作流） | p.9 §4 引言段；p.9 §4.1 “Benchmarks.” 段 |
| **初始 harness**：基于 LangChain DeepAgent SDK，刻意保持极简——简短的面向基准的系统提示 + 默认文件系统与 shell 工具 | p.9 §4.1 “Harness.” 段；p.10 Figure 3 及图注 |

---

## 3. 数据来源

| 要点 | 原文依据 |
|---|---|
| **Terminal-Bench-2.0**：共 89 个容器化终端任务；使用固定的 64 例子集，排除依赖不稳定外部网络资源或需要多模态输入的任务 | p.9 §4.1 “Benchmarks.” 段；p.20 附录 A.2 “Terminal-Bench-2.0 configuration.” 段 |
| Terminal-Bench-2.0 的 held-in / held-out 具体划分数量 | **未说明**（原文仅给出 64 例总数） |
| **SWE-bench Verified**：固定 100 例子集，67 例 held-in、33 例 held-out；按仓库比例抽样 | p.9 §4.1 “Benchmarks.” 段；p.20 附录 A.2 “SWE-bench Verified configuration.” 段 |
| **AppWorld**：180 例，held-in / held-out 各 90 例；held-in 为官方 train 划分全部样本，held-out 从官方 test_normal 与 test_challenge 中按比例抽样后固定 | p.9 §4.1 “Benchmarks.” 段；p.20 附录 A.2 “AppWorld configuration.” 段 |
| **执行轨迹数据**：由固定模型在当前 harness 下运行 held-in 任务产生的轨迹（消息、工具调用、验证器结果），作为弱点挖掘的输入 | p.5 §3.1 ¶2；p.5–6 §3.2 ¶1 |
| held-out 划分从不向 proposer 展示，仅用于自动晋升门控 | p.9 §4.1 “Splits and protocol.” 段 |
| **模型推理服务**：MiniMax M2.5 通过 MiniMax 托管 API；GLM-5 通过 OpenRouter；Qwen3.5-35B-A3B 在 4 张 NVIDIA H200 上本地部署（基于 SGLang 镜像 v0.5.12-cu129） | p.20 附录 A.1 |
| **执行环境**：Terminal-Bench-2.0 使用 Harbor，出站带宽上限 2 MB/s，并镜像部分稳定但慢的外部资源；SWE-bench 每实例独立 Docker 沙箱，官方评估器打分；AppWorld 每任务新建上下文，官方评估器打分 | p.20 附录 A.2 各配置段 |

---

## 4. 核心变量

| 变量类型 | 内容 | 原文依据 |
|---|---|---|
| **自变量（被操纵）** | harness 配置（只允许修改 harness 定义文件中声明的可编辑面，如 instruction、tools、verification guidance 等） | p.9 §4.1 “Harness.” 段：“Self-Harness can only change the harness definition file…”；p.10 Figure 3 |
| **因变量（主要指标）** | Pass (%)：通过对应基准官方验证器的任务尝试百分比；每个 harness 候选默认重复 2 次尝试计算，衡量平均单次尝试成功率 | p.11 §4.1 “Metrics.” 段 |
| 因变量分项 | held-in、held-out、overall 三类 Pass | p.11 Table 1 |
| **控制变量（保持固定）** | 模型权重、解码配置、预算、工具集、基准环境、评估器；任务划分在各 harness 变体间固定；每个任务从全新基准环境开始 | p.9 §4.1 “Models.” 段：“the decoding configuration, budget, tool set, benchmark environment, and evaluator are kept unchanged while only the harness is allowed to vary”；p.9–11 “Splits and protocol.” 段；p.20 附录 A.2 “Shared reproducibility settings.” |
| 验收判据中的变量 | Δ_in⁽ʲ⁾ = P_in(hₜ⁽ʲ⁾) − P_in(hₜ)，Δ_ho⁽ʲ⁾ = P_ho(hₜ⁽ʲ⁾) − P_ho(hₜ)，P 为通过任务数 | p.8 §3.4 ¶2 |
| 失败签名变量 | φ(rᵢ) = (cᵢ, qᵢ, mᵢ)：cᵢ 终端验证器层面原因、qᵢ 相关智能体行为的因果状态、mᵢ 抽象智能体机制 | p.7 §3.2 ¶1（“We write this attribution as a failure signature…”） |
| 具体解码参数（温度等）、预算具体数值（Terminal-Bench 的） | **未说明**（仅说明保持不变；SWE-bench 有 3600 秒案例超时和 120 秒工具超时，AppWorld 每任务最多 50 轮，见 p.20 附录 A.2） |

---

## 5. 研究方法

Self-Harness 是一个“提议–评估–接受”的迭代循环（p.14 §5 ¶1），由三个阶段组成（p.2 §1 ¶3；p.5 Figure 2；p.6 Algorithm 1）。

### 5.1 弱点挖掘（Weakness Mining）
- 在第 t 轮，用固定模型 M 与当前 harness hₜ 运行 held-in 任务，得到轨迹记录 rᵢ = (xᵢ, τᵢ, yᵢ, zᵢ)，zᵢ = 𝓔(xᵢ, τᵢ, yᵢ)；由于 M 与 𝓔 固定，跨轮次记录变化可归因于 harness 变化。——p.5–6 §3.2 ¶1
- 只取失败记录，按“验证器锚定的失败签名”聚类；对每条失败记录识别：验证器暴露的终端失败原因、与之相关的智能体行为、该行为在轨迹中的因果状态，以避免把表面症状（如 timeout）与可复用失败机制混淆。——p.6 §3.2 ¶2
- 聚类规则为签名**完全一致**，属于确定性、评估器锚定的聚类，目的不是发现潜在语义相似，而是聚合“可能用同一 harness 干预解决”的失败。——p.7 §3.2 ¶1
- 每个簇生成结构化失败模式（簇大小、代表任务、共同轨迹症状、验证器证据、推断的智能体机制），按支持度与估计可操作性排序；本阶段输出证据包 Bₜ，且 Bₜ **不规定**具体 harness 修改。——p.7 §3.2 ¶2–¶3
- 执行聚类的主体：§1 称“The agent then clusters failed traces”（p.2 §1 ¶3），§3 称由“evaluation system”完成（p.4 §3 引言段；p.6 §3.2 ¶2）；evaluation system 是否就是同一模型——**未说明**。

### 5.2 Harness 提议（Harness Proposal）
- 由**同一固定模型 M（在当前 harness hₜ 下）**以 proposer 角色生成修改，而非外部优化器；输入为有界的提议上下文：当前 harness 的可编辑面、验证器锚定的失败模式、应保留的通过行为记录、以往尝试过的修改摘要。——p.7 §3.3 ¶1
- 并行生成 K 个互不相同的提议包 𝒫ₜ = {(Δⱼ, aⱼ)}ⱼ₌₁ᴷ，Δⱼ 将 hₜ 映射为候选 harness hₜ⁽ʲ⁾ = Δⱼ(hₜ)，aⱼ 为审计记录（目标失败模式、修改面、预期行为效果、回归风险）；候选须实质性不同。——p.7–8 §3.3 ¶2
- 目标失败模式须既有证据支持、又可被可编辑 harness 面解决；任务特定难度、不稳定结果、模型能力上限类的簇被排除。——p.8 §3.3 第2段（“The proposer first selects…”）
- 分支间鼓励多样性，分支内强制最小性：只修改所需的面、保留无关行为、避免大幅重写控制架构。——p.8 §3.3 第3段（“Diversity is encouraged…”）
- K 与总轮数 R 的具体取值——**未说明**。

### 5.3 提议验证（Proposal Validation）
- 每个候选作为新 harness 变体，用同一评估器在 held-in 与 held-out 上同时评估当前 harness 与候选 harness；held-in 衡量是否解决了动机证据，held-out 作为对 proposer 不可见行为的回归测试。——p.8 §3.4 ¶1
- **接受规则**：Δ_in ≥ 0、Δ_ho ≥ 0 且 max(Δ_in, Δ_ho) > 0；仅以一个划分换另一个划分的提议即使总通过数上升也被拒绝。——p.8 §3.4 ¶2
- 评估具有随机性时重复评估并对聚合通过数应用同一规则；同轮多个兼容候选通过则合并入下一版 harness；被拒候选记录但不改变活动 harness；未修改任何可编辑面或在得到有效评估前执行失败的提议也被拒绝；记录变更面、分划结果、重复次数、提议摘要与决策，使谱系可审计。——p.8 §3.4 ¶2
- 合并多个候选时如何处理冲突（MergeAccepted 的具体实现）——**未说明**（p.6 Algorithm 1 第 21 行仅写“accepted edits are merged”）。

### 5.4 实验设计
- 组内（within-model）比较：同一模型的初始 harness vs. Self-Harness 最终 harness。——p.9 §4.1 “Models.” 段；p.11 Table 1
- 定性分析：演化轨迹图、保留的代码级修改、修改前后的代表性轨迹对比。——p.11–14 §4.3；附录 B（p.20–31）
- 与其他方法（如 Meta-Harness、人工设计 harness、其他自改进方法）的定量对比——**未说明**（原文结果表仅含初始 harness 与 Self-Harness 两列，p.11 Table 1）。
- 消融实验（如去掉聚类、去掉 held-out 门控等）——**未说明**。

---

## 6. 主要结论

### 6.1 定量结论（p.11 Table 1；p.11 §4.2 ¶1）

全部 9 个模型–基准组合中，最终 harness 在 held-in 与 held-out 上均提升 Pass。

| 基准 | 模型 | 初始 (in / ho / overall) | Self-Harness (in / ho / overall) |
|---|---|---|---|
| Terminal-Bench-2.0 | MiniMax M2.5 | 43.0 / 40.5 / 42.2 | 50.0 / 61.9 / 53.9 |
| | Qwen3.5-35B-A3B | 15.1 / 23.8 / 18.0 | 36.0 / 38.1 / 36.7 |
| | GLM-5 | 47.7 / 42.9 / 46.1 | 57.0 / 57.1 / 57.0 |
| SWE-bench Verified | MiniMax M2.5 | 51.5 / 34.8 / 46.0 | 58.2 / 40.9 / 52.5 |
| | Qwen3.5-35B-A3B | 20.1 / 18.2 / 19.5 | 42.5 / 39.4 / 41.5 |
| | GLM-5 | 53.7 / 48.5 / 52.0 | 58.2 / 50.0 / 55.5 |
| AppWorld | MiniMax M2.5 | 51.7 / 45.6 / 48.6 | 62.8 / 55.0 / 58.9 |
| | Qwen3.5-35B-A3B | 25.0 / 20.0 / 22.5 | 60.0 / 44.4 / 52.2 |
| | GLM-5 | 47.8 / 41.1 / 44.4 | 92.2 / 77.8 / 85.0 |

> 注：PDF 文本层中 Table 1 排版错位，上表按原文 p.3 §1 ¶5（续）给出的 Terminal-Bench 数字、p.12 §4.3 及 p.21–22 附录 B.2/B.3 给出的 overall 数字交叉核对后重排。

- 最大相对增益 132%（Qwen3.5-35B-A3B @ AppWorld），最大绝对增益 40.6 个百分点（GLM-5 @ AppWorld，44.4% → 85.0%）。——p.11 §4.2 ¶1
- SWE-bench Verified 与 AppWorld 的 overall 增益分别为 3.5–22.0 和 10.3–40.6 个百分点。——p.3 §1 ¶5（续）
- 在 4 个组合中 held-out 相对增益超过 held-in：Terminal-Bench 上的 MiniMax M2.5 与 GLM-5，SWE-bench 上的 MiniMax M2.5 与 Qwen3.5-35B-A3B。——p.11 §4.2 ¶1
- 增益不局限于用作提议证据的 held-in 失败；三个后端在 held-out 上都提升，且没有被晋升的 harness 让任一划分退化。——p.11 §4.2 ¶2

### 6.2 定性结论
- 修改不是简单加长 prompt 或加通用指令，而是针对各模型反复出现的问题的定向修改。——p.3 §1 ¶6
- 各基准保留的修改针对的瓶颈：Terminal-Bench——产物处理、失败恢复、运行时控制；SWE-bench——补丁验证与可执行验证；AppWorld——完整状态检索与正确的完成语义。具体实现仍是模型特异的。——p.11–12 §4.3 ¶1
- 模型特异的修改示例：MiniMax M2.5 提前创建输出产物、谨慎处理结构化工具输出、限制工具消息总数以打断停滞循环（p.12 §4.3 ¶2；p.13 §4.3 “Trace-Level Analysis” 第2段）；Qwen3.5 依赖预检、打破循环、命令重试纪律、工具错误后以产物为中心的恢复（p.12 §4.3 ¶3；p.13 第3段）；GLM-5 让环境修改跨 shell 会话持久化、从探索转向实现与测试（p.12 §4.3 ¶4；p.13–14 第4段）。
- Self-Harness 也能引入更宏观的结构机制，如基于 subagent 的分解、middleware 创建。——p.3 §1 ¶6
- SWE-bench 上三个模型都强化验证但结构不同（MiniMax 分离空 diff 检测与定向测试；Qwen 委托 patch-verifier subagent；GLM 在验证指令中嵌入依赖修复）；AppWorld 上各模型收敛到完整状态检索与正确完成语义，但通过模型特异机制实现。——p.22 附录 B.3 末段
- 作者总结的主要教训：harness 改进应被视为一次**经验性状态转移**；一个有用的修改必须说明其意在改变的行为、修改的面、动机证据以及支持晋升的评估结果。——p.14–15 §5 ¶2
- 即便初始 harness 很稀疏，在提议受执行证据约束、并经回归测试验证时，也能支持有效的自我改进。——p.15 §5 ¶3

---

## 7. 适用条件

以下条件均为原文明确陈述的设定或前提：

| 条件 | 原文依据 |
|---|---|
| 模型权重与评估器在整个循环中固定，只修改外围 harness | p.5 Figure 2 图注：“Throughout the loop, the model weights and evaluator remain fixed; only the surrounding harness is modified.” |
| 需要有可验证结果（pass/fail）的执行轨迹 | p.2 §1 ¶3：“producing execution traces with verifiable outcomes”；p.5 §3.1 ¶2 |
| 需要事先固定的 held-in / held-out 划分，held-out 对 proposer 不可见 | p.9 §4.1 “Splits and protocol.” 段 |
| harness 需暴露**声明好的可编辑面**（配置点），修改仅限于这些面 | p.9 §4.1 “Harness.” 段；p.14 §5 ¶1：“proposes bounded edits to declared harness surfaces” |
| 失败须能被可编辑的 harness 面所解决；任务难度、不稳定结果、模型能力上限类失败不在可处理范围 | p.8 §3.3 第2段（“The proposer first selects…”） |
| 修改为有界、最小化修改，不替换整体控制架构 | p.4 §3 引言段；p.8 §3.3 第3段（“Diversity is encouraged…”） |
| 依赖验证器结果与轨迹记录的质量 | p.15 §5 ¶4 |
| 实验中的任务集已排除依赖不稳定外部资源与多模态输入的任务（初始 harness 不支持多模态） | p.9 §4.1 “Benchmarks.” 段；p.20 附录 A.2 |
| 所需的最小模型能力、计算/API 成本、迭代轮数上限 | **未说明** |

---

## 8. 研究局限

### 8.1 原文明确承认的局限（p.15 §5 ¶4）
1. 研究的是**固定基准上的有界 harness 修改**，而非开放式自我改进（“not open-ended self-improvement”）。
2. 被接受的修改**仍可能反映基准特定的失败模式**（“Accepted edits may still reflect benchmark-specific failure patterns”）。
3. 协议**依赖验证器结果和轨迹记录的质量**。
4. 对高风险的 harness 修改，**仅靠通过率不退化作为接受门控是不够的**，需要更强的接受门控。

### 8.2 原文提及的其他相关限制
- Terminal-Bench-2.0 只使用 64/89 个任务，排除了多模态与依赖不稳定外部资源的任务（作者陈述的理由是降低评估噪声）。——p.9 §4.1 “Benchmarks.” 段
- 未来工作方向：在更广泛环境中应用 self-harness 式的自我改进。——p.15 §5 ¶5

### 8.3 原文未说明的信息（仅列出缺失项，不作推断）
- 与 Meta-Harness、人工设计 harness 等方法的直接定量对比：**未说明**
- 消融实验：**未说明**
- 计算成本 / token 开销 / 运行时间：**未说明**
- 提议宽度 K、迭代轮数 R 的取值：**未说明**
- Terminal-Bench-2.0 的 held-in / held-out 划分规模：**未说明**
- 统计显著性检验或方差/置信区间：**未说明**（仅说明每个候选重复 2 次尝试，p.11 §4.1 “Metrics.” 段；p.20 附录 A.2）
- 弱点挖掘中执行失败归因与聚类的“evaluation system”是否由同一模型担任：**未说明**

### 8.4 原文内部表述需核对之处（仅指出文本差异）
- p.12 §4.3 ¶1 写“Red nodes and grey slots denote rejected branches and failing trials”，而 p.12 Figure 4 图注写“gray crosses denote rejected candidates, and red crosses mark explored branch endpoints”，两处对图例颜色含义的描述不一致。

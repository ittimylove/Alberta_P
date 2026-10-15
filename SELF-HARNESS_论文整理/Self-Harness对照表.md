# Self-Harness 对照表

> **论文**：Self-Harness: Harnesses That Improve Themselves（Zhang et al., arXiv:2606.09498v3, 2026）
> **唯一信息来源**：`SELF-HARNESS_论文整理/Zhang_2026_Self-Harness.pdf`
> **标注约定**：`p.X §Y ¶N` = 第 X 页、第 Y 节、该页该节第 N 段。原文未提及的信息写“未说明”。

---

## 总览

| 核查维度 | Self-Harness 的回答（一句话） |
|---|---|
| 场景与输入 | 固定模型 + 固定评估器 + 有可验证结果的基准任务；proposer 只看到 held-in 失败的**结构化失败模式**，不看原始日志，也看不到 held-out |
| 修改对象 | 只改 harness 定义文件中**事先声明的可编辑面**（prompt 类指令、subagent、skill、middleware、运行时控制策略等）；不改模型权重，不重写整体控制架构 |
| 成本 | 仅给出评估协议参数（每候选 2 次重复、超时、轮数上限、硬件/API）；token、时间、费用、K、T 均**未说明** |
| 成功标准 | held-in 与 held-out 的通过数**均不下降且至少一个上升**；以聚合通过数判定 |
| 迁移范围 | 在**同一模型 × 同一基准**内，从 held-in 推广到 held-out 任务；跨模型、跨基准、跨 harness 实现的迁移**未说明** |

---

## 1. 场景与输入

| 子问题 | 原文内容 | 依据 |
|---|---|---|
| 适用场景 | 固定语言模型以智能体方式运行；模型 M 与评估器 𝓔 固定，只把 harness 作为改进对象 | p.5 §3.1 ¶2；p.5 Figure 2 图注 |
| 任务前提 | 任务运行需产生“具有可验证结果的执行轨迹” | p.2 §1 ¶3：“producing execution traces with verifiable outcomes” |
| 实验场景 | 容器化终端交互（Terminal-Bench-2.0）、仓库级软件修复（SWE-bench Verified）、多应用工作流（AppWorld） | p.9 §4 引言段；p.9 §4.1 “Benchmarks.” 段 |
| 排除的场景 | 依赖不稳定外部网络资源的任务、需要多模态输入的任务（初始 harness 不支持） | p.9 §4.1 “Benchmarks.” 段；p.20 附录 A.2 |
| 轨迹包含什么 | 消息、工具调用、验证器结果；每条记录为 rᵢ = (xᵢ, τᵢ, yᵢ, zᵢ)，即任务、轨迹、输出、评估结果 | p.5 §3.1 ¶2；p.6 §3.2 |
| 反馈形式 | 评估器给出的行为结果，如 pass/fail；实验中为各基准**官方验证器** | p.5 §3.1 ¶2；p.11 §4.1 “Metrics.” 段 |
| 弱点挖掘看到什么 | 仅 held-in 上的**失败**记录，按失败签名 φ(rᵢ) = (cᵢ, qᵢ, mᵢ) 精确聚类 | p.6 §3.2 ¶2；p.7 §3.2 ¶1；p.6 Algorithm 1 第 3 行注释 |
| proposer 看到什么 | ① 当前 harness 的可编辑面；② 验证器锚定的失败模式；③ 应保留的通过行为记录；④ 以往尝试过的修改摘要。看到的是结构化的跨案例证据，**而非原始执行日志** | p.7 §3.3 ¶1 |
| proposer 看不到什么 | held-out 划分从不展示给 proposer，只用于自动晋升门控 | p.9 §4.1 “Splits and protocol.” 段 |
| 谁来诊断 | §1 称“The agent then clusters failed traces”；§3 称由 “evaluation system” 完成。evaluation system 是否为同一模型：**未说明** | p.2 §1 ¶3；p.4 §3 引言段；p.6 §3.2 ¶2 |
| 谁来提议 | 同一固定模型 M，在当前 harness hₜ 下以 proposer 角色提议 | p.7 §3.3 ¶1；p.6 Algorithm 1 第 4 行 |

---

## 2. 修改对象

| 子问题 | 原文内容 | 依据 |
|---|---|---|
| 修改范围 | 只能修改配置 DeepAgent 实例化与控制方式的 **harness 定义文件**；可编辑面对应该文件中**声明的配置点**，如 instruction、tools、verification guidance 等 | p.9 §4.1 “Harness.” 段 |
| 声明的可编辑接口 | system prompt、memory sources、subagents、skills、bootstrap / execution / verification / failure-recovery 指令、runtime control policy | p.10 Figure 3 |
| 不改什么 | 模型参数；解码配置、预算、工具集、基准环境、评估器保持不变 | p.5 §3.1 ¶1；p.9 §4.1 “Models.” 段；p.5 Figure 2 图注 |
| 修改粒度 | 有界、最小化修改：只改解决所选机制所需的面，保留无关行为，避免大幅重写控制架构 | p.4 §3 引言段；p.8 §3.3 第 3 段 |
| 是否改整个 harness | 否，“without replacing the overall control architecture” | p.4 §3 引言段 |
| **prompt / 指令** | 是。例：MiniMax 的 bootstrap 指令改为尽早创建产物；GLM 的验证阶段约束；AppWorld 上 Qwen 的两条 prompt-level guard | p.13 §4.3 Trace-Level 第 2、4 段；p.22 附录 B.3 |
| **skill** | 出现过。例：Qwen @ Terminal-Bench 的 “dependency-verifier skill” 候选 | p.12 Figure 4(a) 标注 |
| **subagent** | 是。例：Qwen @ Terminal-Bench 的 “artifact-ensure subagent” 候选；SWE-bench 上 MiniMax 的 diff-inspection / test-verifier subagent、Qwen 的 patch-verifier subagent；AppWorld 上 MiniMax 的 state-auditor subagent | p.12 Figure 4(a)；p.21–22 附录 B.2、B.3 |
| **控制逻辑 / 运行时机制** | 是。例：MiniMax 启用运行时策略并限制工具消息总数；Qwen 的工具错误触发 middleware；论文称可引入 subagent 分解、middleware 创建等结构机制 | p.13 §4.3 Trace-Level 第 2、3 段；p.3 §1 ¶6 |
| 无效修改 | 不修改任何可编辑面的提议会被拒绝 | p.8 §3.4 ¶2 |

---

## 3. 成本

| 环节 | 原文内容 | 依据 |
|---|---|---|
| **收集数据** | 每个候选评估做 2 次重复尝试；每轮评估当前 harness 和每个候选，均在 held-in 与 held-out 上评估 | p.20 附录 A.2 “Shared reproducibility settings.”；p.8 §3.4 ¶1；p.6 Algorithm 1 第 2、8 行 |
| | 单任务预算：SWE-bench 每例 3,600 秒超时、每次工具调用 120 秒超时；AppWorld 每任务最多 50 轮；Terminal-Bench 的单任务预算 | p.20 附录 A.2；Terminal-Bench 部分：**未说明** |
| | 推理资源：Qwen3.5-35B-A3B 本地部署于 4 张 NVIDIA H200；MiniMax M2.5 用 MiniMax 托管 API；GLM-5 用 OpenRouter | p.20 附录 A.1 |
| | 数据收集的 token / 时间 / 费用 | **未说明** |
| **诊断（弱点挖掘）** | 方法：失败归因 + 签名精确聚类 + 按支持度与可操作性排序 | p.6–7 §3.2 |
| | 诊断的 token / 时间 / 费用 | **未说明** |
| **搜索补丁（提议）** | 每轮并行生成 K 个提议，共 T 轮 | p.6 Algorithm 1 |
| | K、T 的取值 | **未说明** |
| | 各演化图横轴“Candidate evaluations”的刻度范围：Qwen@TB 0–20、MiniMax@TB 0–15、GLM@TB 0–18、MiniMax@SWE 0–9、Qwen@SWE 0–25、GLM@SWE 0–10、MiniMax@AppWorld 0–13、Qwen@AppWorld 0–22、GLM@AppWorld 0–13 | p.12 Figure 4；p.23–30 Figure 10、11、13、15、17、19、21 |
| | 提议环节的 token / 时间 / 费用 | **未说明** |
| **验证** | 候选在 held-in 与 held-out 上各评估（每候选 2 次重复）；评估随机时重复评估并对聚合通过数应用规则 | p.8 §3.4 ¶2；p.20 附录 A.2 |
| | 验证的 token / 时间 / 费用 | **未说明** |
| 总成本、与其他方法的成本对比 | **未说明** | — |

---

## 4. 成功标准

| 子问题 | 原文内容 | 依据 |
|---|---|---|
| 验收规则 | Δin⁽ʲ⁾ ≥ 0、Δho⁽ʲ⁾ ≥ 0 且 max(Δin⁽ʲ⁾, Δho⁽ʲ⁾) > 0；P 为通过任务数 | p.8 §3.4 ¶2；p.6 Algorithm 1 第 9–11 行 |
| 两个划分的角色 | held-in：衡量提议是否解决了**促成它的证据**；held-out：作为对 proposer **不可见行为**的回归测试 | p.8 §3.4 ¶1 |
| 是否修好原失败 | 以 held-in 通过数变化衡量；另以代表性轨迹对比展示具体失败被修复（如 Qwen 在 extract-elf 任务上恢复了被删除的产物） | p.8 §3.4 ¶1；p.13 §4.3 Trace-Level 第 3 段；p.15、21 Figure 7–9 |
| | 每个失败簇是否被修复（逐簇修复率） | **未说明** |
| 是否对其他任务有效 | 是，以 held-out 衡量：9 个组合 held-out 均提升；4 个组合 held-out 相对增益超过 held-in | p.11 §4.2 ¶1–¶2；p.11 Table 1 |
| 有没有回归 | 规则拒绝任一划分下降的候选；以“只是用一个划分换另一个划分”方式提高总数的提议也被拒绝；“no promoted harness degrades either split” | p.8 §3.4 ¶2；p.11 §4.2 ¶2 |
| | 回归判定粒度为**聚合通过数**；逐任务回归（原先通过的某个任务变为失败）的情况 | **未说明** |
| 其他拒绝条件 | 未修改任何可编辑面，或在得到有效评估结果前执行失败的提议被拒绝 | p.8 §3.4 ¶2 |
| 作者承认的局限 | 对高风险 harness 修改，仅靠通过率不退化作为接受门控不够 | p.15 §5 ¶4 |

---

## 5. 迁移范围

| 迁移类型 | 原文内容 | 依据 |
|---|---|---|
| **换任务（同一基准内）** | 有。held-out 任务的轨迹从未作为 evaluation system 的输入，所有组合在 held-out 上均提升；作者称能发现迁移到“traces are hidden from the proposer”的任务上的修改 | p.3 §1 ¶5（续）；p.11 §4.2 ¶2 |
| **换模型** | 3 个模型**各自独立**运行 Self-Harness，得到各自的 harness；作者强调不同模型得到不同修改。把一个模型演化出的 harness 用到另一个模型上 | 独立运行：p.9 §4.1 “Models.” 段、p.3 §1 ¶6；跨模型迁移：**未说明** |
| **换基准 / 任务类型** | 3 个基准**各自独立**运行；初始 harness 为 benchmark-specific。把一个基准上演化出的 harness 用到另一个基准上 | 独立运行：p.9 §4 引言段；跨基准迁移：**未说明** |
| **换执行环境** | 每个基准使用固定的执行环境（Harbor、每实例 Docker 沙箱、AppWorld 上下文）。在不同执行环境之间迁移 | 环境设置：p.20 附录 A.2；跨环境迁移：**未说明** |
| **换 harness 实现** | 所有实验的初始 harness 均基于 DeepAgent SDK。在其他 harness 框架上的效果 | DeepAgent：p.9 §4.1 “Harness.” 段；其他框架：**未说明** |
| 作者对迁移的保留意见 | 被接受的修改仍可能反映基准特定的失败模式；研究的是固定基准下的有界修改；更广泛环境的应用列为未来工作 | p.15 §5 ¶4、¶5 |

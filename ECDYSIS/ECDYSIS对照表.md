# ECDYSIS 相关工作核查对照表

> 来源：arXiv:2609.11677v2《ECDYSIS: Efficient and Effective Training of Runtime Harnesses for LLM Agents》
> 内容仅依据论文原文（含附录）。原文未提及的写"未说明"。标注"据 Table X 计算"的数字是用原文表格数据直接算出的，不是原文给出的结论。

## 总表

| 核查维度 | 要回答的问题 | ECDYSIS（据原文） | 出处 |
|---|---|---|---|
| **场景与输入** | 在什么情况下适配？ | 任务模型参数和运行时环境固定，只优化 runtime harness；在训练任务集上做多轮进化（最多 3 轮候选生成），之后冻结 harness 再到 held-out 数据上评测。 | §3.2；附录 C |
| | 可以看到哪些轨迹和反馈？ | 每条训练轨迹由固定的评测框架打出任务分数 S(τ)，S(τ) < λ 记为失败。失败轨迹被转成结构化记录，包含：任务 ID、失败判定、终止原因、工具调用历史、必要的执行上下文。记录再按失败模式分组，优先使用覆盖 ≥2 个不同任务的组，单任务组作为辅助证据保留。coding agent 拿到的是：修改规格、相关执行证据、当前 harness 源码。 | §3.3；§3.4 |
| | 阈值 λ 的取值 | 未说明 | — |
| | 训练集 / held-out 集的划分与规模 | 未说明（仅提到"training split"和"held-out data"；数据研究中精简训练集为 5 个训练任务） | 附录 C；§6.1 |
| **修改对象** | 修改 prompt、skill、控制逻辑，还是整个 Harness？ | 修改对象是 harness 本身：coding agent（OpenCode + DeepSeek-V4-Pro）读取当前 harness 源码并直接修改，产出候选 harness。附录 A 的人工审计提到，一个修改决策可能跨多个函数、注册点或辅助 skill。附录 B 的实例包括：修改工具调用的校验逻辑（退款限制）、修改返回给模型的反馈文本（取消订单提示）、新增执行规则类（拦截无效行李写操作）。修改规格中会确定修改层级、触发条件、修改范围和安全约束。 | §3.2；§3.3；§3.4；附录 A、B；§4.1 |
| | 是否修改模型参数？ | 否，训练全程模型参数固定。 | §3.2；§4.1 |
| **成本** | 收集数据（跑训练任务） | 训练评测的 API 成本三种方法均低于 $1。ECDYSIS (w/ FDCR)：τ²-Airline $0.870，τ²-Retail $0.506，AgentBench $0.320。 | 附录 E.2；Table 9 |
| | 诊断（失败分析 / FDCR） | 未单独说明。FDCR 的成本和 coding agent 的成本合并计入"Evolution"一项。 | 附录 E.2 |
| | 搜索补丁（coding agent 修改） | 未单独说明（同上，合并计入"Evolution"）。"Evolution"一项的成本（ECDYSIS w/ FDCR）：Airline $1.739，Retail $5.257，AgentBench $1.513。每轮只调用一次 coding agent，调用次数 N_round = Σ I(nᵢ>0)。 | 附录 E.2；Table 9；§3.3 |
| | 验证 | 未单独说明。每轮用完整训练集重新评测候选 harness，其成本应包含在训练评测中，但原文没有拆分。 | §3.2；附录 C |
| | 总体成本 | 总 API 成本（w/ FDCR）：Airline $2.609，Retail $5.763，AgentBench $1.833；对比 SE 分别为 $6.382、$8.484、$2.030。整个训练过程的 LLM 调用次数：SE 881，w/o FDCR 267，w/ FDCR 482。输入缓存命中率：SE 89.82%，w/o FDCR 94.92%，w/ FDCR 94.90%。在 τ²-Retail 上，w/ FDCR 的输入 token 多于 SE，但总 API 成本更低。 | 附录 E.2；Table 3、8、9；§5 |
| | 时间 | 端到端训练时间（w/ FDCR）：Airline 4,403.0 s，Retail 1,405.9 s，AgentBench 1,890.5 s；相对 SE 加速 1.84×、1.30×、1.03×。没有按"收集 / 诊断 / 修改 / 验证"拆分。 | Table 4；附录 E.3 |
| | 精简数据的成本 | 用失败感知筛选出的 5 个训练任务训练，成本 $0.291（完整训练集 $5.763，随机 5 个任务 $0.505）。 | Table 5；§6.1 |
| **成功标准** | 训练时的接受标准 | 候选 harness 当且仅当训练集总体分数严格提升（J_train(Hᶜᵢ) > J_train(Hᵢ₋₁)）时被接受，否则回滚。原文明确说明，这一标准不要求每个训练任务都提升。 | §3.2；附录 C |
| | 是否检查修好了原来的失败？ | 未说明。原文没有报告触发修改的那些失败是否被修复；失败组也被说明为"诊断证据而非必须修复的目标"，候选 harness 不需要处理每个失败组。 | §3.3 |
| | 对其他任务是否有效？ | 在 held-out 任务上评测，指标为平均准确率、Pass@3、Pass^3。ECDYSIS (w/ FDCR) 15 个组合的平均值：AVG 69.56%，Pass@3 77.67%，Pass^3 60.00%（SE 为 58.67 / 70.00 / 46.67；Human-Aug. 为 60.22 / 70.67 / 49.33）。另报告推理 token 与每条轨迹耗时。 | Table 1、2；附录 C |
| | 有没有回归？ | 未系统说明。接受标准只看训练集总分，个别训练任务可以退步；原文没有报告逐任务的回归情况。FDCR 中 Critic 的职责之一是审查"在之前成功任务上的回归"风险，但没有给出回归的量化结果。 | §3.2；§3.4 |
| | 修改质量的额外评估 | 人工审计每个独立修改决策，分为 MSA（迁就模型）或 HLR（修复 harness 机制），由 5 名人类专家独立判断、多数投票（伦理声明说明标注者均为论文作者）。合并后 MSA 比例：SE 60.00%，w/o FDCR 75.61%，w/ FDCR 45.45%。 | §6.2；附录 A；Table 10；Ethical Considerations |
| **迁移范围** | 换模型 | 是。三种进化方法的 harness 都只用 Qwen3-8B 训练，直接复用到 Qwen3-14B、Qwen3-32B、MiniMax-M2.7、Llama-3.1-8B，不再进化。数据研究中，用 Qwen3-8B 进化、在 Qwen3-32B 上评测。 | 附录 C；Table 5 |
| | 换任务 | 在同一数据集的 held-out 任务上评测。是否把一个数据集上训练的 harness 迁移到另一个数据集：未说明。 | 附录 C |
| | 换执行环境 | 未说明（未报告跨环境迁移实验；三个数据集各自报告训练时间和成本）。 | — |
| | 换 Harness 实现 | 未说明。实验只基于 Life-Harness 一种 harness 框架，所有进化方法都从同一个 Human-Aug. harness 初始化。 | §4.1 |

## 补充：实验配置要点（据原文）

| 项目 | 内容 | 出处 |
|---|---|---|
| 任务模型 | Qwen3-8B、Qwen3-14B、Qwen3-32B、MiniMax-M2.7（230B）、Llama-3.1-8B，均通过 API 调用 | §4.1 |
| 训练用任务模型 | 仅 Qwen3-8B | 附录 C |
| 采样温度 | 0.0 | §4.1 |
| coding agent | OpenCode，底层模型 DeepSeek-V4-Pro | §4.1 |
| FDCR 角色模型 | Analyst、Critic、Engineer、Moderator 均为 DeepSeek-V4-Pro | §4.1 |
| FDCR 轮数 | 2 轮 | §4.1 |
| 进化轮数 | 最多 3 轮候选生成 | 附录 C |
| 数据集 | τ²-Airline、τ²-Retail（τ²-Bench 子集）、AgentBench | §4.1 |
| 每个 held-out 任务的运行次数 | 3 次 | 附录 C |
| 基线 | Direct、Human-Aug.、Self-Evolution (SE) | §4.1 |

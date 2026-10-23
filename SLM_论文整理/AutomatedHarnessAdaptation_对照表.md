# Automated Harness Adaptation（自动化 Harness 适配）— 相关工作核查对照表

> **论文**：Better Harnesses, Smaller Models: Building 90% Cheaper Agents via Automated Harness Adaptation（Yang, Zhao, Wu, Kästner；arXiv:2607.08938v1）
>
> **说明**
> - 本表只依据 `paper.pdf` 原文填写。原文没有提到的写“未说明”，不自行补充。
> - 页码为 PDF 页码。“第 n 段”指该节/小节内的自然段顺序。
> - 论文没有给方法单独起名，文中称作 automated harness adaptation 或 harness optimizer（p.1 标题；p.4 §III 第2段）。

---

## 一、总表（速览）

| 核查维度 | 本文的做法（一句话） | 主要依据 |
|---|---|---|
| 场景与输入 | 部署前离线进行，每个任务–SLM 对单独优化；meta-agent 能看到带结果信号的完整轨迹、评估分数与反馈、当前 harness 代码、搜索记忆和设计空间文档 | p.4–5 §III；p.5 §III-B 1)–2)；p.6 首段 |
| 修改对象 | 整个 harness 的 Python 实现，可改上下文（system prompt、skills、动态上下文）、工具、hooks、上下文管理和子 agent | p.5 §III-A；p.5 §III-B 2) |
| 成本 | 每次优化总预算 $20（包含分析、rollout 和候选评估），主实验共 $1260；各阶段分别花多少：未说明 | p.7 §IV-B 第1段、脚注 3 |
| 成功标准 | 按分数判定：同批训练实例上有提升 → 验证集上优于已有候选 → 最终在留出测试集上评估；不检验跨任务效果；没有专门的回归分析 | p.6 首段 3)；p.6 §IV-A；p.7 §IV-B 第1段 |
| 迁移范围 | 换的是**模型**（为前沿 LLM 设计的 harness → 适配给 SLM）；任务、环境和 harness 框架保持不变；优化出的 harness 不能在 SLM 之间直接迁移 | p.1 摘要；p.9 “Different SLMs…” 段 |

---

## 二、逐项核查

### 1. 场景与输入

**它在什么情况下适配？**

| 问题 | 原文内容 | 依据 |
|---|---|---|
| 触发场景 | SLM 被直接换进为前沿 LLM 设计的 harness 后表现不佳，这时为它适配 harness | p.1 摘要；p.2 Fig. 1 标题 |
| 适配时机 | 部署前离线进行：“one-time offline optimization cost per task”；优化好的 harness 在之后的 agent 运行中复用 | p.8 §IV-D “RQ1” 第2段 |
| 适配粒度 | 每个任务–SLM 对单独搜索专用 harness | p.6 §IV-A 第1段 |
| 何时需要重新适配 | 每次迁移到新的 SLM 都要重跑优化 | p.9 “Different SLMs…” 段 |
| 在线 / 运行时适配 | 本文没有做；只作为未来方向提出（应对任务分布漂移的在线监控与适配） | p.10 §V-B “For researchers” 第2段 |
| 优化器输入 | 一个 SLM、带训练集和验证集的目标任务、一个初始 agent harness | p.4 §III 第2段末 – p.5 首段 |
| 数据要求 | 整理好的任务实例，有明确评估标准；划分为 20/20/60 的 train/validation/test | p.10 §V-B “For agent developers” 第2段；p.6 §IV-A “Tasks” 第3段 |

**它能看到哪些轨迹和反馈？**

| 信息 | 原文内容 | 依据 |
|---|---|---|
| 任务轨迹 | 最新一批训练实例的任务轨迹，带结果信号标注；记录完整轨迹，包括工具调用、中间观察和输出 | p.5 §III-B 1)、2) |
| 评估反馈 | 保存评估分数和反馈 | p.5 §III-B 1) |
| 当前 harness | 正在编辑的 harness 的完整 Python 实现 | p.5–6 §III-B 2) |
| 搜索记忆 | 过去提案及其观察到的效果的摘要，避免反复尝试同样无效的修改 | p.6 §III-B 2) |
| 设计空间文档 | 记录可用 harness 组件的 API 参考 | p.6 §III-B 2) |
| 轨迹格式 | 传原始 JSON 比传后处理的 markdown 效果更好 | p.9 §V-A “Lesson 1” 第1段 |
| 前沿模型的成功轨迹 | 试过提供，但没有稳定带来改进（不属于最终方案的默认输入） | p.9 §V-A “Lesson 1” 第3段 |
| 人工编写的“失败模式 → 适配策略”启发式 | 试过提供，同样没有稳定带来改进 | p.9 §V-A “Lesson 1” 第3段 |
| 验证集 / 测试集轨迹是否给 meta-agent 看 | meta-agent 用训练集收集轨迹、提出更新；验证集用于评估和选择；测试集留出做最终评估。验证集的轨迹是否也提供给 meta-agent：未说明 | p.6 §IV-A “Tasks” 第3段 |

### 2. 修改对象

**结论：修改整个 harness（以代码形式），不只是 prompt。**

| 可修改组件 | 原文内容 | 依据 |
|---|---|---|
| Prompt / 上下文 | system prompt（任务说明、示例）、skills、按需追加的动态上下文 | p.5 §III-A |
| Skill | 包含在上下文组件中；meta-agent 可以插入可复用的 skill | p.5 §III-A；p.5 §III-B 2) |
| 工具 | 基础工具（文件编辑器、bash、浏览器）、通过类定义的自定义工具、自定义脚本；也可过滤工具集（例如 `filter_tools_regex`） | p.5 §III-A；p.5 Fig. 4 |
| 控制逻辑 | hooks：由工具调用触发的自定义脚本（例如 anti_loop_hook.py） | p.5 §III-A；p.5 Fig. 4 |
| 上下文管理 | 外部文件系统、上下文压缩器 | p.5 §III-A |
| 子 agent / 编排 | 有独立上下文的自定义 agent | p.5 §III-A |
| 编辑载体 | harness 的完整 Python 实现，基于 software-agent-sdk 的 API 接口 | p.5 §III-A；p.5–6 §III-B 2) |
| 模型权重 | 搜索空间里只有上述 harness 组件，原文没有描述对模型权重的修改；是否涉及模型训练：未说明 | p.5 §III-A |

**实际被采用的修改**（RQ4，人工检查优化后的 harness）：

| 修改类型 | 采用比例 / 情况 | 依据 |
|---|---|---|
| 增加上下文 | 86% | p.9 §IV-D “RQ4” 第1段 |
| 创建工具 | 43% | 同上 |
| 管理工具 | 29% | 同上 |
| hooks（强制约束） | 有实例：budget-approval 的防循环 hook；gemma 在 playwright 任务上用 hook 强制 pytest 运行成功；总体比例：未说明 | p.5 Fig. 4；p.9 “For example, for the playwright-testing task…” 段 |
| 子 agent | 没有观察到成功的子 agent 适配 | p.9 §IV-D “RQ4” 第2段 |

### 3. 成本

| 成本项 | 原文内容 | 依据 |
|---|---|---|
| 每次优化总预算 | $20，包含 meta-agent 分析、任务 agent rollout 和候选 harness 评估；选这个数是因为早期试点中在此预算下观察到优化饱和 | p.7 §IV-B 第1段；p.7 脚注 3 |
| 主实验总预算 | $1260 | p.7 脚注 3 |
| 重复次数 | 每个任务–模型对跑 3 次优化，保留验证分最高的那次 | p.7 §IV-B 第1段 |
| 预算的概念分解 | B = Σₜ (bₜ^analyze + bₜ^evaluate)：预算可花在分析失败或评估候选修复上，并分多轮迭代 | p.9 §V-A 第2段 |
| **收集数据**的成本 | 未说明（只说明了数据的整理方式和规模：每任务 N=100，website-management 为 50） | p.7 Table I |
| **诊断**（meta-agent 分析）的成本 | 单独金额：未说明。meta-agent 用 gemini-3.1-pro-preview | p.6 §IV-B “RQ1” 段末 |
| **搜索补丁**的成本 | 单独金额：未说明。迭代次数只给出了 budget-approval 一个例子（23 次） | p.6 §III-C 第2段 |
| **验证**的成本 | 单独金额：未说明。原文只说 sanity check “cheap”（廉价），且只有在同批训练实例上有提升时才跑完整验证集评估 | p.6 首段 3) |
| 降低成本的尝试 | 把 meta-agent 换成更便宜的 gemini-3-flash-preview，迭代次数增加了，但最终 harness 更差 | p.9 §V-A “Lesson 1” 第2段 |
| 成本回收 | 平均运行 13 次后即可收回优化成本 | p.8 §IV-D “RQ1” 第2段 |
| 部署后推理成本 | 见 Table II，例如 gemma-4-26b-a4b + 优化 harness 的单实例平均成本为 $0.071，gemini-3.1-pro-preview 为 $1.735 | p.8 Table II |
| 计费方式 | 按 token 用量和官方 API 定价（Vertex AI / AWS Bedrock） | p.6 §IV-B “RQ1” 段 |
| 时间（墙钟时间）成本 | 未说明 | — |
| 人力成本（整理数据、人工检查等） | 未说明 | — |

### 4. 成功标准

| 问题 | 原文内容 | 依据 |
|---|---|---|
| 搜索过程中如何接受一个候选 | ① 通过 sanity check；② 在同一批训练实例上有提升；③ 在验证集上做完整评估，优于已有候选才加入 harness 池 | p.6 首段 3) |
| 采样依据 | 从 Pareto 前沿采样，即被采样的 harness 至少在一个训练实例上是最优的 | p.5 §III-B 1) |
| 最终选择 | 3 次优化中验证分最高的 harness | p.7 §IV-B 第1段 |
| 最终评估 | 在留出的测试集（60%）上测准确率、成本和延迟，每个配置跑 3 次取平均 | p.6 §IV-A 第1段、“Tasks” 第3段；p.6 §IV-B “RQ1” 段 |
| **是否修好了原来的失败** | meta-agent 根据失败轨迹进行诊断和修改，但接受标准是分数是否提升。是否逐条核对原有失败已被修复：未说明 | p.5 §III-B 2)；p.6 首段 3) |
| **对其他任务是否有效** | 没有检验。harness 按任务–SLM 对单独优化；“泛化”只指同一任务内未见过的测试实例 | p.6 §IV-A 第1段 |
| 对其他 SLM 是否有效 | 不能直接迁移（见第 5 项） | p.9 “Different SLMs…” 段 |
| 衡量“追平差距”的标准 | 原文报告“7 对追平 SLM–LLM 差距”，判定阈值：未说明 | p.8 §IV-D “RQ1” 第1段 |
| “显著提升”的统计检验 | 未说明 | p.8 §IV-D “RQ1” 第1段 |
| **有没有回归** | 原文没有专门讨论回归。验证集上“优于已有候选才入池”是在聚合分数层面防止退步；实例级的回归：未说明 | p.6 首段 3) |
| Table II 中优化后没有提升或略有下降的情况（照录数值，原文未讨论） | ministral-3-8b：Stock 0.0→0.0，Anom. 0.0→0.0，Web. 5.6→5.6，Refact. 31.7→30.0 | p.8 Table II |

### 5. 迁移范围

| 迁移维度 | 原文内容 | 依据 |
|---|---|---|
| **换模型** | 是核心场景：从前沿 LLM 换成 SLM，把为 LLM 设计的通用 harness 适配给 SLM | p.1 摘要；p.2 Fig. 1 标题 |
| 优化后的 harness 能否迁移到其他 SLM | 不能直接迁移：每个模型的失败模式不同，需要专门适配，因此换 SLM 时要重跑优化 | p.9 “Different SLMs…” 段 |
| 框架是否绑定模型规模 | 失败模式 → 适配策略的框架刻意不绑定模型规模，适用于一般的语言模型 agent | p.4 §II-C 第1段 |
| **换任务** | 不做跨任务迁移，每个任务单独优化一个 harness。对高多样性任务，作者把 mixture-of-harnesses（多个专用 harness 加路由）列为未来方向 | p.6 §IV-A 第1段；p.10 §V-B “For researchers” 第2段 |
| **换执行环境** | 没有做。外部效度部分说明只覆盖目标清晰、环境干净的任务 | p.8 §IV-C |
| **换 Harness 实现** | 没有做。所有 harness 都基于 software-agent-sdk 实现；外部效度部分说明结论可能不适用于其他优化器实现 | p.6–7 §IV-B 第1段；p.5 §III-A；p.8 §IV-C |
| 任务分布漂移 | 本文没有做，作为未来方向（在线监控、维护多个 harness 版本、更新路由） | p.10 §V-B “For researchers” 第2段 |
| meta-agent 模型更换 | 只比较了 gemini-3.1-pro-preview 和 gemini-3-flash-preview，后者效果更差；作者认为结论能推广到其他强能力前沿 LLM（属作者推断，未经实验验证） | p.9 §V-A “Lesson 1” 第2段；p.8 §IV-C “Reproducibility” |
| 与 Life-Harness [62] 的区分 | Life-Harness 演化的是能跨 benchmark 环境和模型 backbone 迁移的 harness；本文把 harness 适配当作一种成本–性能策略来研究 | p.10 §VI 最后一段 |

---

## 三、附：原文提及的相关工作（只按本文的描述填写）

本文 §VI 对相关工作的描述很简略，下表只填本文明确写到的内容，其余写“未说明”。

| 相关工作 | 场景与输入 | 修改对象 | 成本 | 成功标准 | 迁移范围 | 依据 |
|---|---|---|---|---|---|---|
| Lee et al. [39] | 小模型面对不熟悉的工具 schema | 工具 schema | 未说明 | 未说明 | 依赖对特定 SLM 行为的观察，可能不能很好地推广到能力更强的新模型 | p.10 §VI 第3段 |
| Srivastava et al. [57]（EffGen） | 缓解 SLM 局限（如长 prompt 下的指令遵循） | 一整套人工设计的启发式 | 未说明 | 未说明 | 同上 | p.10 §VI 第3段 |
| ADAS [58] 及扩展 [59]–[61] | 多为预定义工作流的 agent 系统，任务不需要与环境交互 | 用代码定义的 agent；模块化组件（规划、记忆）[59]；多 agent 拓扑 [60][61] | 未说明 | 通常通过增加模型计算量（如 test-time scaling）来提升性能 | 未说明 | p.10 §VI 第4段 |
| 自我改进 agent [10]（Darwin Gödel Machine）、[47]（Live-SWE-agent） | 连前沿模型都觉得难的任务（如 TerminalBench） | agent 修改自己的代码，常见做法是新建工具 | 未说明 | 提升给定任务上的性能 | 未说明 | p.10 §VI 第5段 |
| Meta-Harness, Lee et al. [11] | 同上 | 用 meta agent 自动设计 agent harness | 未说明 | 提升给定任务上的性能 | 未说明 | p.10 §VI 第5段 |
| Life-Harness [62]（并行工作） | 冻结模型 | agent harness | 未说明 | 未说明 | 跨 benchmark 环境和模型 backbone 迁移 | p.10 §VI 第5段 |

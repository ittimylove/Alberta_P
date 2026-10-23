# HarnessCompass 对照表（相关工作核查维度）

**论文**：*HarnessCompass: Guiding Automatic Harness Evolution toward Generalizable and Effective Agent Harnesses*（arXiv:2608.01918v1）

> **说明**
> - 只依据论文原文作答。原文没有提到的内容写"**未说明**"。
> - 页码为 PDF 页码。"§小节 第 n 段"按小节内自然段计数。附录 prompt 按"行号"引用，行号是原文 prompt 框中的编号。
> - AHE 一列只记录本文对 AHE 的描述，不引用 AHE 原论文。

---

## 一、总览表

| 核查维度 | HarnessCompass | AHE（本文中的描述） |
|---|---|---|
| **场景与输入** | 场景：编码 agent 在可执行环境中解决 Python 仓库 issue，基座模型固定，有二值 verifier。输入：50 个演化任务的完整 raw trace 和 verifier 结果、轨迹分析证据，以及只针对失败任务收集、经轨迹验证的第一人称反馈 | 通过 agent–环境交互演化 harness，只依赖从轨迹得到的信号；meta agent 只能从外部观察交互（p.1，§Introduction 第 2–3 段；p.3，§Related Works 第 2 段；p.4，§Proactive Feedback 第 1 段） |
| **修改对象** | 整个 harness：7 类组件（tool 实现、middleware、sub-agent；system prompt、tool 描述、skills、memory），但拆成结构 / 引导两条轨道分别修改再合并。不改基座模型、LLM 配置、tracer、verifier 和基础设施 | 整个 harness，所有组件联合优化（p.1，§Introduction 第 3 段；p.3，§Related Works 第 2 段） |
| **成本** | 只报告演化轮数（Turns = 5）。数据收集、诊断、补丁搜索、验证各自消耗的 token、时间和费用都**未说明** | Turns = 20（p.5，Table 1）；其他成本**未说明** |
| **成功标准** | 演化内部：winner 变体在 50 个演化任务上的 Pass@1 高于当前 harness 才接受。外部评测：450 个 held-out 任务、全部 500 题、跨模型 Pass@1。回归：有回滚机制，也报告了分仓库的退化 | Pass@1：Sample 63.0% / Held-Out 54.7% / Total 55.5%（p.5，Table 1）；其内部的接受标准**未说明** |
| **迁移范围** | **任务**：50 个演化任务 → 450 个同基准 held-out 任务。**模型**：GPT-5.4 → Claude-Sonnet-4.6（harness 冻结）。**执行环境**：未改变 / 未说明。**Harness 实现**：未改变（冻结后直接使用）。**其他基准**：未说明 | 本文引用 Wang et al. 2026，称这类方法在搜索集与评测集不相交时泛化较差（p.1，§Introduction 第 3 段；p.3，§Related Works 第 2 段）；本文只做了 held-out 任务对比（p.5，Table 1），AHE 的跨模型迁移**未说明** |

---

## 二、逐项详述

### 1. 场景与输入

**1.1 适用场景：在什么情况下适配？**

| 条件 | 原文依据 |
|---|---|
| 对象是编码 agent 的 harness，基座模型 M 在演化中固定 | p.3，§HarnessCompass 第 1 段；§Preliminaries 第 1 段 |
| 任务：SWE-Bench Verified 中真实 Python 仓库的 issue | p.5，§Setup "Benchmark"段 |
| 需要能给每条轨迹打二值结果 r(τ)∈{0,1} 的 verifier | p.3，§Preliminaries 第 1 段 |
| 种子 harness 设定为非交互场景（"You solve software tasks in a non-interactive setting"），只有一个 shell 工具 | p.8，附录 §Seed Harness H0，seed prompt 第 1–3 行 |
| 所有角色（代码 agent、轨迹分析器、反馈 agent、meta agent）共用 GPT-5.4，non-thinking 模式 | p.5，§Setup "Models"段 |
| 适配的触发时机：每轮 verifier 评分之后（反馈只对失败任务收集） | p.4，§Proactive Feedback "Eliciting first-person feedback"段 |
| 线上部署 / 运行时自适应等其他场景 | **未说明** |

**1.2 meta agent 可以看到的轨迹和反馈**

| 输入 | 内容 | 范围 | 原文依据 |
|---|---|---|---|
| Raw trace | 完整的消息、工具调用、环境观测序列 | 50 个演化任务，每任务 k 次 rollout（附录写 k = 2） | p.3，§Preliminaries 第 1 段；p.14，附录 §Per-Repository Results 第 1 段 |
| Verifier 结果 | 二值结果；hindsight 报告中还附带 verifier 的输出 | 同上 | p.3，§Preliminaries 第 1 段；p.10，附录 Step 2 说明（"The verifier verdict and its output are appended to this prompt"） |
| 轨迹提炼证据 E | 由 Distill(T) 生成；附录中称为 "Agent Debugger analysis"，被当作比第一人称反馈更客观的主要来源 | 同上 | p.3，Algorithm 1 第 4 行；p.11，附录 "Aggregated first-person evidence" prompt 第 4–6 行 |
| 第一人称反馈 F | 四步流程：blind 报告 → hindsight 报告（含失败归因：harness / agent 推理 / 任务歧义 / 环境）→ 合并 → 对照轨迹验证（grounding）。前三步由同一个 M 完成，第四步由分析器模型作为外部检查者完成 | **只针对失败任务** | p.4，§Proactive Feedback 第 2–4 段；p.9，附录 §Proactive Feedback 第 1 段 |
| 反馈聚合结果 | 按组件分组，置信度由报告一致性、受影响任务数、blind / hindsight 一致性决定；只作为补充证据，并按当前变体的轨道过滤 | 同上 | p.4，§Proactive Feedback "Aggregation and use"段；p.11，附录 "Aggregation and use"段 |
| 上一轮的 Change Manifest | 记录每条改动的 failure_pattern、predicted_fixes、risk_tasks 等，下一轮据此把分数变化归因到单条改动并决定是否回滚 | 每轮 | p.13–14，附录 §Change Manifest |
| R3 阶段的输入 | loser 的 manifest 和 diff，以及各变体的评估证据（哪些任务翻转为成功、哪些退化） | 每轮 | p.13，附录 R3 integrator prompt 第 17–24 行 |
| 450 个 held-out 任务 | 演化过程**不接触**，只用于评测 | — | p.5，§Setup "Benchmark"段 |

> 附注：附录开头说会给出 "the trajectory-analysis prompts inherited from the standard evaluate–analyze–improve loop"（p.8，附录 §Prompts 第 1 段），但在附录正文中没有找到这部分 prompt。轨迹分析的具体做法**未说明**。

### 2. 修改对象

**结论：修改整个 harness，但按轨道分开修改、每轮再合并；不改模型。**

| 对象 | 能否修改 | 归属轨道 / 约束 | 原文依据 |
|---|---|---|---|
| system prompt | ✅ | 引导轨道；只能写带适用条件的跨任务原则 | p.3，§Preliminaries 第 1 段；p.12，附录 Guidance variant constraint |
| skills（SKILL.md 和脚本） | ✅ | 引导轨道 | 同上；p.8，附录 Generalization Gate 第 10 行 |
| tool descriptions | ✅ | 引导轨道 | 同上 |
| 长期记忆（LongTermMEMORY） | ✅ | 引导轨道；写入前要合并或删除互相矛盾的单任务规则 | p.12，附录 Guidance variant constraint 第 6–7 行 |
| tool 实现 | ✅ | 结构轨道；必须是确定性机制，并返回 agent 自己拿不到的可观测结果 | p.12，附录 Structural variant constraint 第 9–13 行 |
| middleware（控制逻辑） | ✅ | 结构轨道；同上 | 同上 |
| sub-agents（含其 prompt.md / agent.yaml） | ✅ | 结构轨道 | p.8，附录 Generalization Gate 第 13–15 行 |
| code_agent.yaml | 受门控约束（列在门控覆盖范围内） | 归入哪条轨道**未说明** | p.8，附录 Generalization Gate 第 16 行 |
| 短期记忆（ShortTermMEMORY） | ❌ 不能直接编辑，也不能通过 prompt / skill 规则间接写入任务专属内容 | — | p.8，附录 Generalization Gate 第 17–19 行 |
| LLM 配置、tracer、verifier、基础设施 | ❌ | — | p.13，附录 R3 integrator 第 62 行 |
| 基座模型权重 | ❌（演化中固定） | — | p.3，§Preliminaries 第 1 段 |

**额外约束：**
- **位置要求**：新增能力必须写成代码（middleware / tools / sub-agents）；纯行为指导必须放在 prompt 或 memory 里，不能伪装成代码。依据：p.4，§Constrained Evolution 第 3–4 段。
- **内容要求**：禁止出现任务 ID、测试名、被测代码私有符号，以及按特定任务关键词分支的代码。依据：p.4，§Constrained Evolution 第 2 段；p.8，附录 Generalization Gate 第 21–30 行。
- **改动类型**：manifest 中每条改动标为 new / improvement / rollback。依据：p.14，附录 Change manifest schema 第 6 行。

### 3. 成本

| 环节 | 原文给出的信息 | 具体资源消耗（token / 时间 / 费用 / 调用次数） |
|---|---|---|
| **收集数据** | 每轮在 50 个演化任务上 rollout，每任务 k 次（k = 2）。按 Algorithm 1，每轮包括对当前 harness 的 1 次 Rollout（第 3 行）和对两个轨道变体各 1 次 Rollout（第 8 行） | **未说明** |
| **诊断** | 轨迹提炼 Distill(T)（Algorithm 1 第 4 行）；第一人称反馈对每个失败任务调用 4 次模型（3 次用 M，1 次用分析器）（p.9，附录 §Proactive Feedback 第 1 段）；聚合步骤为确定性流程，"without any additional model calls"（p.4，§Proactive Feedback "Aggregation and use"段） | **未说明** |
| **搜索补丁** | 每轮两个 meta agent 并行，各生成一个变体（p.12，附录 Track constraints 段）；之后由 R3 integrator 合并（p.12–13） | **未说明** |
| **验证** | 两个变体分别在演化集上 rollout，选出 winner（Algorithm 1 第 8、10 行）；winner 的 Pass@1 高于当前 harness 才接受（第 12–13 行）。原文把每一轮称为一次对演化后 harness 的评估（"each turn entails evaluation of the evolved harness"，p.5，§Setup "Metrics"段） | **未说明** |
| **总体** | 唯一量化的成本指标是 **Turns**：HarnessCompass 5 轮，AHE 20 轮（p.5，Table 1）；消融中各配置为 2 / 12 / 5 轮（p.6，Table 2）。作者称 HarnessCompass 用大约 1/4 的 AHE 迭代次数（p.7，§Evolution Efficiency 第 2 段） | 单轮成本、HarnessCompass 与 AHE 每轮成本是否相同：**未说明** |
| 最大迭代次数 N | — | **未说明** |

> 附注：Algorithm 1 中，接受与否看的是 winner 变体的 Pass@1（第 12 行），R3 合并后的 harness Ht 在当轮没有单独的 Rollout 步骤（第 10–14 行）。这是对算法文本的直接描述；原文没有讨论合并后 harness 的验证成本。

### 4. 成功标准

**4.1 演化内部的接受标准**
- winner 变体在演化集上的 Pass@1 严格高于当前 harness，才接受本轮 harness。依据：p.3，Algorithm 1 第 12–14 行。
- R3 的目标是合并结果 "≥ winner, never worse"。依据：p.13，附录 R3 integrator 第 11–12 行。
- 原文没有说明这一点是否经过单独评估来确认（见 §3 附注）。

**4.2 是修好了原来的失败，还是对其他任务也有效？**

| 层面 | 原文情况 | 依据 |
|---|---|---|
| 修好原来的失败 | manifest 中有 `predicted_fixes`（预期修好的任务）字段；但论文**没有报告**原失败任务的修复率 | p.14，附录 Change manifest schema 第 10 行；修复率：**未说明** |
| 对其他任务有效 | 主要评测标准：450 个 held-out 任务上 Pass@1 从 51.6% 升到 60.4%（AHE 为 54.7%） | p.5，Table 1；p.6，§Main Results 第 2 段 |
| 对其他模型有效 | Claude-Sonnet-4.6 上 Total 从 70.0% 升到 73.8% | p.6，Table 3；p.7，§Generalization… 第 2 段 |
| 改动本身是否可迁移 | 门控要求每条改动都带有在新任务上可判断的适用条件；附录 litmus test："Would this still help a task from a library I have never seen?" | p.4，§Constrained Evolution 第 2 段；p.8，附录 Generalization Gate 第 32–39 行 |

**4.3 有没有回归？**

| 回归相关机制 / 结果 | 内容 | 依据 |
|---|---|---|
| 逐条改动回滚 | manifest 用于把分数变化归因到单条改动并决定回滚；由反馈驱动的改动如果下一轮没有帮助就回退 | p.13，附录 §Change Manifest 段；p.4，§Proactive Feedback "Aggregation and use"段 |
| 风险任务标注 | manifest 中有 `risk_tasks` 字段 | p.14，附录 Change manifest schema 第 11 行 |
| R3 过滤回归 | Revision 步骤丢弃看起来会造成退化的 loser 改动，拿不准时也丢弃；依据是各变体中哪些任务翻转、哪些退化 | p.5，§Component-wise Optimization "Merging the tracks"段；p.13，附录 R3 integrator 第 23–36 行 |
| 演化集上的回归 | 消融中加入主动反馈后（无 R3），Held-Out 从 58.4% 降到 55.8% | p.6，Table 2；§Ablation Study 第 2 段 |
| 分仓库回归（GPT-5.4） | scikit-learn 73.4 → 68.8（−4.6） | p.15，Table 4 |
| 分仓库回归（Claude-Sonnet-4.6） | pytest 84.2 → 73.7（−10.5）；sphinx 65.9 → 61.4（−4.5） | p.15，Table 4；p.14，附录 §Per-Repository Results 第 4 段 |
| 逐任务回归统计（原先成功、改动后失败的任务数） | — | **未说明** |

### 5. 迁移范围

| 迁移维度 | 是否验证 | 具体设置与结果 | 依据 |
|---|---|---|---|
| **换任务** | ✅ 同一基准内 | 在 50 个随机抽取的任务上演化，在其余 450 个任务上评测：Held-Out 51.6% → 60.4% | p.5，§Setup "Benchmark"段；Table 1 |
| **换任务：跨仓库** | ✅ 分仓库分析 | 12 个仓库中 8 个提升、3 个不变、1 个下降（GPT-5.4） | p.14，附录 §Per-Repository Results 第 2 段；p.15，Table 4 |
| **换任务：跨基准 / 跨领域** | ❌ | — | **未说明** |
| **换模型** | ✅ | GPT-5.4 上演化的 harness 冻结后，在 Claude-Sonnet-4.6（non-thinking）上评测，不再演化：Total 70.0% → 73.8%；作者说明迁移"positive but uneven" | p.5，§Setup "Models"段；p.6，Table 3；p.14，附录 §Per-Repository Results 第 4 段 |
| **换模型：thinking 模式** | ❌ | 所有实验都用 non-thinking 模式 | p.5，§Setup "Models"段；thinking 模式下的结果：**未说明** |
| **换执行环境** | ❌ | — | **未说明** |
| **换 Harness 实现** | ❌ | 跨模型实验中 harness 被冻结、原样使用；迁移到其他 agent 框架 / harness 实现上：**未说明** | p.7，§Generalization… 第 1 段 |
| **换演化用的模型（例如用 Claude 来演化）** | ❌ | — | **未说明** |

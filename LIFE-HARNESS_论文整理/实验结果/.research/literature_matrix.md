# 文献矩阵：模型升级后的 harness 适配

> 生成：2026-10-19，按 `literature-triage-matrix`（ai-research-skills v0.5.4）流程。
> 视角：服务于你的主线——**模型升级后，旧 harness 在哪里失配、怎么定位、怎么修**。
>
> **证据等级说明**
> - `笔记`：来自本文件夹里你的精读笔记（每条都带 PDF 页码），本次没有逐条回到 PDF 复核。
> - `PDF✓`：本次用 pdftotext 回到 PDF 原文核对过原话（2026-10-19）。
> - 没有写进来的数字一律不算本次核实过。

## 1. 主矩阵（8 篇）

| 引用 | 研究问题 | 方法 | 数据 / 基准 | 主要结论 | 证据等级 | 最重要的局限 | 与主线相关度 | 用作 |
|---|---|---|---|---|---|---|---|---|
| **MH** Lee 2026 (2603.28052) | harness 工程能否自动化？优化器该看到多少反馈？ | Claude Code + Opus 4.6 读取文件系统里的全部历史代码、分数、原始轨迹，搜索单文件 harness | 文本分类、IMO 级数学检索、TB2（89 题） | 完整原始轨迹 > 分数 + 摘要 > 只有分数（Table 3）；TB2 Opus 76.4%、Haiku 37.6% | 笔记 | 只验证了一种强 proposer；TB2 搜索集与测试集相同；没有方差 | 中：提供"反馈不压缩"的基线立场 | 相关工作（优化器 / 反馈） |
| **AHE** Lin 2026 (2604.25850) | 能否联合、稳定地演化 coding harness 的全部组件？ | 7 类组件以文件暴露（NexAU）；Agent Debugger 蒸馏轨迹；change manifest 做可证伪预测 + 回滚 | TB2 全部 89 题；迁移到 SWE-bench Verified 500 题；5 个替代基座 | TB2 69.7→77.0%；增益来自 tools / middleware / memory，system prompt 单独换入 −2.3 pp；**单组件增益之和 +11.1 pp，完整版只有 +7.3 pp**；Hard 档 memory-only 63.3% 高于完整版 53.3% | **PDF✓**（非加性段、操作点 Limitations、回归盲区） | 迁移数字混杂了"harness 可移植性"和"操作点耦合"（步数 / 超时按 GPT-5.4 high 拟合）；只跑了一次演化；只做了"单独加入"的消融 | **高**：7 层划分的来源；非加性 + 操作点混杂正是你要拆的东西 | 核心对话对象；分层透镜 |
| **LIFE** Xu 2026 (2605.22166) | 能否把失败结构变成运行时接口，并跨模型迁移？ | 393 条失败轨迹 → 四类失败；Contract / Skill / Action / Trajectory 四层；Codex 提议更新 | τ / τ²-bench、AgentBench 共 7 个环境；源模型 Qwen3-4B，迁移到另外 17 个 | 126 个设置中 116 个提升；作者据此认为 harness 抓住的是"环境侧结构，与模型无关" | 笔记 | 只针对确定性、规则约束的环境；没有方差；演化成本未说明 | 中：给出"接口可迁移"的强主张，和"模型特异"主张对立 | 对立证据（可迁移一方） |
| **GEPA** Agrawal 2025 (2507.19457) | 反思式 prompt 进化能否胜过 RL？ | 反思变异 + Pareto 候选选择，只改 prompt | 6 个基准 × Qwen3-8B / GPT-4.1 Mini | 6 个里 5 个胜过 GRPO；Qwen 上优化的 prompt 迁移到 GPT-4.1 Mini 聚合 +9.0% | 笔记 | 只改 prompt；两个模型；没有方差 | 低–中：prompt 层跨模型迁移的早期证据 | 背景 / 基线 |
| **Self-Harness** Zhang 2026 (2606.09498) | 固定模型能否不靠更强外部 agent 改进自己的 harness？ | 弱点挖掘（按失败签名聚类）→ 同一模型提议最小改动 → held-in / held-out 双门控 | TB2（64 题）、SWE-bench Verified 100 题、AppWorld 180 题；MiniMax M2.5 / Qwen3.5-35B-A3B / GLM-5 | 9 个组合 held-in 和 held-out 都提升；保留的瓶颈类别跨模型相近，具体实现模型特异；摘要原话 "effective harness design is inherently model-specific" | **PDF✓**（摘要） | 没有和 MH / 人工 harness 的定量对比；没有消融；K、R、成本未说明 | **高**：同一模型当 proposer；"模型特异"一方的主证据 | 对立证据（模型特异一方）；proposer 选择 |
| **HarnessFix** Chen 2026 (2606.06324) | 能否从失败轨迹定位 harness 缺陷并做限定范围的修复？ | HTIR（步骤 + 数据流 + 控制流 + 实现锚点）→ 按 ETCLOVG 七层归因 → 修复算子 → 回归感知验收 | GAIA / SWE-bench Verified / AppWorld / TB2 Verified，各自划分 train / val / test；5 个 task model | 相对 H0 平均 +11.1；比 Meta-Harness 高 2.6–5.0 且 token 更少；**用 GPT-5 mini 修好的 harness 迁移到另外 4 个模型 +5.5~+9.5，小于源模型的 +18.4** | **PDF✓**（RQ4 迁移段） | 迁移只在 GAIA 上测过；关键细节放在网站上；没有专门的局限章节 | **高**：层级归因 + 跨模型迁移打折的直接证据 | 方法参照（轨迹→层归因）；"迁移打折"证据 |
| **HarnessX** Darwin Agent Team 2026 (2606.14249) | 能否把 harness 当一等对象组合、演化，并找出演化的病理？ | 类型化 processor + 8 个 hook；AEGIS 四阶段（Digester / Planner / Evolver / Critic）+ seesaw 门控；变体隔离；与 GRPO 协同演化 | GAIA、ALFWorld、WebShop、τ³、SWE-bench Verified 55 题；3 个 task agent，meta-agent 固定为 Opus 4.6 | 15 个组合里 14 个提升，平均 +14.5；逆缩放（基线越弱增益越大）；**原话 "we do not evaluate whether a weaker meta-agent can achieve comparable gains"** | **PDF✓**（§7.4） | 没有 held-out；meta-agent 闭源且固定；SWE 只用 55 题 | 中–高：你的候选问题"旧模型当 meta-agent"就出自这里 | 空白来源；病理分类 |
| **SSA** Gupta 2026 (2606.17454) | harness 是否忠实执行了模型意图？pass@1 之外模型行为有何不同？ | 一个共享 harness + 少量按家族的适配；solution distance D(t)；LLM judge 给工具调用打标签；SBP 泄露审计 | SWE-bench Verified 500、SWE-bench Pro 731、TB2 89；21 个模型、5 个家族、约 138k 条轨迹 | 共享 harness 下 pass@1 达到或超过模型卡数字；按家族的适配不能跨家族照搬；gpt-oss 修好解析后 42%→64.64%；**原话 "optimizations that overfit one model's quirks can degrade on the next"** | **PDF✓**（§1） | judge 准确性未验证；各项改进没有消融；适配是怎么找到的没有说明 | **高**：区分"接口失败"和"能力 / 风格差异"的信号来源 | 诊断信号（R8 工具错误、D(t)）；对立证据 |

## 2. 横向读法：三个与主线直接相关的分歧

1. **harness 到底可不可以跨模型迁移？**
   - 可迁移一方：LIFE（17 个模型中绝大多数提升）、AHE（5 个基座全部为正）、HarnessFix（4 个目标模型全部为正）、GEPA（prompt 跨模型 +9%）。
   - 模型特异一方：Self-Harness（摘要原话 "inherently model-specific"）、SSA（"can degrade on the next"，适配不能跨家族照搬）。
   - **两边都没有拆到"层"**：AHE 承认迁移数字混杂了操作点（PDF✓）；HarnessFix 只报告整体迁移打折（+18.4 → +5.5~+9.5）。没有一篇说明**哪一层**可以迁移、**哪一层**需要重调。
2. **组件之间是否可以简单相加？** 只有 AHE 给出了非加性证据（PDF✓），并且把"面向交互的演化"留作未来工作。另外 7 篇都没有测交互项。
3. **"模型升级"这个场景本身**：8 篇里没有一篇做过同一家族前后两代的"旧 harness + 新模型"实验。AHE、Self-Harness、SSA 都把它当作研究动机，但没有直接测量。

## 3. 本次运行记录

```
[literature-triage-matrix]
  Wrote: .research/literature_matrix.md（新建）
  Papers in matrix: 8（8 new this run）
  Source sections read: 5 篇的关键原话回到 PDF 核对（AHE p.8–10、HarnessX §7.4、SSA §1、HarnessFix §V-D、Self-Harness 摘要）
  Claims checked / total load-bearing: 7 / 7 已核对（主线依赖的 7 条原话）；其余单元格为笔记级
  Completion: partial；只核对了主线依赖的论断，其他数字仍是笔记级证据
  Suggested next: 看 topic_dossier.md 里的候选题评估
```

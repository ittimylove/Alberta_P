# 候选问题 v3：模型升级带来的增量，能否跨 harness 迁移（harness hotfixing）

> **标注约定**：【已核实】= 已对照原文；【待核实】= 仅来自搜索摘要；【假设】= 本问题要检验的推测。
> 前两版见 [候选问题.md](候选问题.md)、[候选问题_v2.md](候选问题_v2.md)。

---

## 问题陈述

> 在 harness A 上，分别用旧模型 M_old 和新模型 M_new 运行，得到轨迹 T_old、T_new。能否从 (T_old, T_new) 中提取出**模型增量 Δ(M_old→M_new)**，用它去升级**一个没跑过的 harness B**，并且在 B 上只需要很少的 rollout？

核心假设【假设】：Δ = **与 harness 无关的部分**（可以从 A 迁移过来）+ **与 harness 有关的部分**（需要在 B 上用少量 rollout 修正）。

## 场景

- 新模型发布后，一个组织往往同时维护多个 harness（不同产品线、不同 benchmark）。现在的做法是每个 harness 都在新模型上重新演化一遍，成本随 harness 数量线性增长。
- 理想情况：只在一个"参考 harness"A 上看清新模型和旧模型的行为差异，再把差异快速"热修"到其余 harness 上。

## 缺口

| 已有工作 | 迁移的是什么 | 和本问题的差距 |
|---|---|---|
| Life-Harness、HarnessCompass、Harness-R1、Meta-Harness | 同一个 harness 跨模型复用 | 固定 harness、换模型；不利用 T_new，也不换 harness |
| PromptBridge（2512.01420）【已核实】 | 在模型对之间学一个 prompt 映射，用于没见过的任务 | 对象是单个 prompt，不是 harness；跨的是任务，不是 harness |
| Fine-tuning Transfer（2503.20110）【已核实】 | 把权重差值移植到新版本底座 | 在权重空间、方向对偶：固定改动、换底座 |
| Migration to Calibration（2609.35149）【已核实】 | 用源模型产物诊断目标模型 | 只是方法提案，没有实验，也没有跨 harness |

**空白**：没有工作把"模型差值"当作可以跨 harness 迁移的对象来提取和检验。

## 后果（如果成立）

- **成本**：每个 harness 升级到新模型的代价，从"完整演化一遍"降到"k 次 rollout 校正"。
- **方法论**：harness 演化的产出从"一个改好的 harness"变成"一个可复用的模型差值描述"。这和 ECDYSIS 区分 MSA / HLR 的思路相接：Δ 主要应该体现在 MSA 那一侧。【推断】
- **评估**：它给出一个新的评估维度：**适配样本效率**，即在 B 上需要多少次 rollout，才能达到从头演化的效果。

## 最强反例

1. **Δ 只有在特定 harness 机制下才会显现**：HarnessDev（2609.01437）观察到，换了执行模型后，重复查询率从 10.1% 升到 88.2%，并认为能力只有在 prompt、工具协议、预算、停止规则都兼容时才会迁移【待核实】。如果 B 没有 A 的预算或停止机制，在 A 上观察到的 Δ，在 B 上可能完全不出现，或者以另一种形式出现。
2. **交互项不小**：2609.17394 中纯相加模型的 R² 为 0.989，但 9 个交互检验里有 6 个在 Holm 校正后仍显著，原文说 "the observed scaffold ordering thus depends on the model"【已核实】。
3. **同一个模型在不同 harness 下的轨迹不同**：LIDAR（2609.28559）原文说 "the same model may follow different trajectories under different hosts"，并且分类器是每个 harness 单独建库，没有测跨 harness 迁移【已核实】。
4. **跨 harness 训练不能泛化**：Harness-Zero 转述的一项研究发现，用多个 harness 的反馈训练，没有提升在 held-out harness 上的效果【待核实，二手转述】。

**要回应的核心**：如果 γ（交互项）占主导，Δ 就不可迁移，"热修"退化成"在 B 上从头演化"。

## 最小验证

**设定**：2 个 harness（A、B）× 2 个同族模型（M_old→M_new，例如同一系列的相邻两代），同一个 benchmark（例如 τ-bench 的一个域，或 SWE-bench Verified 的一个子集）。

**第 0 步：预检交互项**：在 2×2 网格上计算 DiD = [P(B,new) − P(B,old)] − [P(A,new) − P(A,old)]。如果 |DiD| 远小于 Δ 本身，说明有可迁移的空间；否则直接转向研究 γ 的结构。

**第 1 步：对比条件**（B 上的 rollout 预算 k ∈ {0, 少量, 中等, 完整}）：

| 条件 | 用到 T_old / T_new? | 作用 |
|---|---|---|
| C0：B 不改，直接换成 M_new | 否 | 零样本下限 |
| C1：在 B 上用 M_new 从头演化 | 否 | 同预算对照；完整预算时是上限 |
| C2：把 A 上的修改原样移植到 B | 间接 | 检验"直接搬补丁"是否足够 |
| C3：从 (T_old, T_new) 提取与 harness 无关的 Δ，再在 B 上用 k 次 rollout 落地 | 是 | **本方法** |

**主要指标**：性能随 B 上 rollout 次数 k 变化的曲线；达到 C1 完整预算 90% 效果时所需的 k。

**否证标准**：如果在小 k 下 C3 不能稳定优于同预算的 C1 和 C2，那么"模型增量可以跨 harness 迁移"这个假设不成立。

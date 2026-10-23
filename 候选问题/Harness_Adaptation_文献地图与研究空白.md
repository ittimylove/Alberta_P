# Harness Adaptation 文献地图与研究空白

截至 2026-10-20

## 说明

Harness adaptation 指：基础模型 + harness，由另一个 proposer 模型读取执行轨迹来修改 harness。本文梳理了约 60 篇相关工作，几乎都是 2026 年 3–10 月的 arXiv 预印本，并对候选空白逐条做了反向检索。

- 标「已读」的论文：打开看过摘要或正文。
- 其余论文：只看了检索摘要，arXiv 编号和数字引用前应再核对。
- 检索截止 2026-10-20；该方向每周都有新论文，立项前应重新检索。

## 文献地图

前 4 个方向是已知的改进线，后 5 个是检索中发现的其他活跃方向。

| 方向 | 代表论文 | 关键发现 |
| --- | --- | --- |
| 0. 起点与综述 | [Meta-Harness 2603.28052](https://arxiv.org/abs/2603.28052)；[文献清单](https://github.com/leezythu/Awesome-Harness-Self-Improvement)；综述 [2606.20683](https://arxiv.org/pdf/2606.20683)、[2605.29682](https://arxiv.org/html/2605.29682v1) | proposer 用 Claude Code + Opus 4.6，通过文件系统读取全部历史代码、轨迹和分数；前作包括 ADAS、AFlow、DGM、Hyperagents、GEPA |
| 1. 小模型当 proposer | [Harness Updating ≠ Harness Benefit 2605.30621](https://arxiv.org/abs/2605.30621)（已读）、[Harness-R1 2608.02276](https://arxiv.org/abs/2608.02276v1)、[HarnessTTT 2610.17370](https://arxiv.org/html/2610.17370)、[HarnessOpt-Bench 2608.06301](https://arxiv.org/html/2608.06301)（已读）、[Better Harnesses, Smaller Models 2607.08938](https://arxiv.org/abs/2607.08938) | Qwen3.5-9B 写的更新与 Opus 4.6 相当；中等能力模型从更新中获益最多；换 proposer 的影响（0.142）大于换 harness（0.079） |
| 2. 怎么读轨迹 | Meta-Harness（全读）；[HarnessFix 2606.06324](https://arxiv.org/abs/2606.06324v2)、[AutoSaddler 2608.23041](https://arxiv.org/html/2608.23041v1)（结构化）；[HarnessEvolve 2609.00829](https://arxiv.org/html/2609.00829)、[ModularRSI 2609.14857](https://arxiv.org/pdf/2609.14857)、[CHASE 2609.18366](https://arxiv.org/pdf/2609.18366)、[DarwinX 2608.07545](https://arxiv.org/html/2608.07545v1)（成功/失败对比）；[RHO 2606.05922](https://arxiv.org/html/2606.05922v1)（无标注） | 对比式信号优于仅用失败轨迹；RHO 用自偏好选 harness 更新，无需验证集 |
| 3. 模型侧 vs harness/环境侧 | [Model or Harness? 2607.28802](https://arxiv.org/abs/2607.28802)、[Ecdysis 2609.11677](https://arxiv.org/pdf/2609.11677)、[Harness Evolution Hits a Ceiling 2610.11655](https://arxiv.org/html/2610.11655v1)（已读）、[2607.15684](https://arxiv.org/html/2607.15684v1)、[2608.13867](https://arxiv.org/pdf/2608.13867) | 41 种失败模式分类法；流程类失败交给 harness，内容类失败交给权重训练 |
| 4. 冗余组件与组件归因 | [SHarP 2610.04178](https://arxiv.org/abs/2610.04178v1)（已读）、[HarnessCompass 2608.01918](https://arxiv.org/html/2608.01918)、[AHE 2604.25850](https://arxiv.org/pdf/2604.25850)、[Where Does Value Live 2609.02889](https://arxiv.org/abs/2609.02889) | 多数 harness 高度冗余；组件作用非可加；增益集中在个别位置 |
| 5. 评测可信度与过拟合 | [Harness-Delta Attribution 2610.22466](https://arxiv.org/html/2610.22466)（已读）、[Rethinking Evaluation 2607.12227](https://arxiv.org/abs/2607.12227)、[Gated QD 2607.13683](https://arxiv.org/html/2607.13683v1)、[HarnessDev 2609.01437](https://arxiv.org/pdf/2609.01437)、[Evo-Bench 2608.09096](https://arxiv.org/pdf/2608.09096) | 搜索集增益平均约 54% 来自过拟合；在 Terminal-Bench 2.1 上未胜过简单的测试时扩展 |
| 6. harness 与权重协同演化 | [SIA 2605.27276](https://arxiv.org/html/2605.27276v2)、[Co-Harness 2607.22688](https://arxiv.org/html/2607.22688v1)、[HarnessX 2606.14249](https://arxiv.org/abs/2606.14249)、[EvoHarness-RL 2608.05446](https://arxiv.org/html/2608.05446v1)、[Evo² 2610.13025](https://arxiv.org/html/2610.13025v1)、[Harness-Zero 2609.24974](https://arxiv.org/html/2609.24974v1)（已读） | 只演化 harness 会遇到上限；Harness-Zero 把 harness 蒸馏进权重 |
| 7. 多目标、安全、持续演化、新场景 | 多目标 [MoMHa](https://arxiv.org/html/2609.30967v1)、[HarnessPro](https://arxiv.org/html/2610.26770)；安全 [Auditing Harness Tampering](https://arxiv.org/pdf/2609.00069)、[HackProbe](https://arxiv.org/abs/2609.04665)、[SafeEvolve](https://arxiv.org/pdf/2609.02786)；持续 [HCL 2608.19013](https://arxiv.org/html/2608.19013v1)、[Continual Harness](https://arxiv.org/abs/2605.09998)、[Adaptive Auto-Harness](https://arxiv.org/html/2606.01770v1)；多智能体 [HarnessNet](https://arxiv.org/abs/2610.15221)、[MILO](https://arxiv.org/abs/2609.38349)；GUI [GUI-HARVEST](https://arxiv.org/abs/2610.00948)；具身 [EMHO](https://arxiv.org/abs/2610.08432) | 各方向均已有专门方法 |
| 8. 跨模型迁移 | [StarHarness](https://arxiv.org/pdf/2608.24804)、[Living-Harness](https://arxiv.org/pdf/2607.26598)、[Evo-Harness 2608.15071](https://arxiv.org/html/2608.15071)（已读）、[One Recipe, Many Harnesses 2608.10178](https://arxiv.org/html/2608.10178)（已读） | 冻结 harness 直接迁到其他模型多数有效，但都是整体迁移，不做组件级分析 |

## 研究空白

三个空白经反向检索确认：A 证据最硬，B、C 实验成本低，可作为 A 的辅助分析或单独成文。

### A. 同家族模型升级后，演化 harness 的哪些组件仍有用、如何重新适配

未找到有人做。

- [DarwinX](https://arxiv.org/pdf/2608.07545) 作者写明未测：换基座后 harness 能保留多少，以及从已有存档热启动与从头搜索各需多少代。
- [Harness-Zero](https://arxiv.org/html/2609.24974v1)（已读）只说 harness 假设会「过时」，没有实验。
- [Evo-Harness](https://arxiv.org/html/2608.15071)（已读）在 Opus 4.5/4.6/4.7 上都报告了增益，但不分析哪些组件过时。
- [HDA](https://arxiv.org/html/2610.22466)（已读）实验覆盖 Qwen3/3.5/3.6，但不按代际分析。
- [SHarP](https://arxiv.org/abs/2610.04178v1) 做剪枝，摘要中无跨模型实验。
- 已有跨模型工作只测整个冻结 harness 的迁移增益，不做组件级分析，也不比较热启动、剪枝、重新演化三种策略。

### B. proposer 与目标模型同家族是否带来偏差

未找到有人做。

- [Harmony Harness](https://arxiv.org/html/2610.15150v1) 只换了不同家族的 judge，没换 proposer。
- [HarnessOpt-Bench](https://arxiv.org/html/2608.06301)（已读）明确未分析 proposer 与目标的家族关系。
- [Self-HPO](https://arxiv.org/html/2610.22791) 只比较了「自己当 proposer」和「更强的外部 proposer」。
- LLM-as-judge 领域已证实同家族偏好（[2508.06709](https://arxiv.org/pdf/2508.06709)），但尚未在 harness 优化中验证。

### C. proposer 诊断准确率缺少有标准答案的评测

已有工作有明显缺口。

- HarnessFix 和 HarnessOpt-Bench 都只看最终涨分，不检查 proposer 找到的原因是否正确；HarnessOpt-Bench 还发现读轨迹动作占比与增益负相关。
- [AgentBug-Smith](https://arxiv.org/html/2609.37864v1) 是修复基准，不检验定位。
- Who&When 一类基准针对多智能体失败归因（[DoVer](https://arxiv.org/pdf/2512.06749)、[FALAT](https://www.alphaxiv.org/abs/2606.00765)），step 级准确率约 20–46%，且不是 harness 优化场景。
- 可做方向：向 harness 注入已知缺陷作为标准答案，同时评测诊断准确率，以及「诊断正确」与「修改后确实涨分」的关系。

## 部分覆盖与已饱和方向

下列方向不建议作为主贡献，部分覆盖的两条可作细分切入点。

| 方向 | 状态 | 依据 |
| --- | --- | --- |
| rollout 与 proposal 的算力分配 | 部分覆盖 | [预算分配陷阱 2609.02889](https://arxiv.org/abs/2609.02889)、[反馈算力扩展律 2605.29682](https://arxiv.org/html/2605.29682v1)；尚无明确的最优分配曲线 |
| 工具 API 漂移下的在线适配 | 部分覆盖 | [EvoHarnessBench 2609.04280](https://arxiv.org/html/2609.04280)、[Life-Harness 2605.22166](https://arxiv.org/html/2605.22166v1) |
| GUI / 多智能体 / 具身场景 | 已饱和 | GUI-HARVEST、HarnessNet、MILO、EMHO |
| 成功/失败轨迹对比 | 已饱和 | HarnessEvolve、ModularRSI、CHASE、DarwinX |
| 多目标优化 | 已饱和 | MoMHa、HarnessPro |
| 持续演化与遗忘 | 已饱和 | HCL、Continual Harness、Adaptive Auto-Harness |
| harness 与权重协同演化 | 已饱和 | SIA、Co-Harness、HarnessX、Evo² |
| 小 proposer 也有效 | 已饱和 | 2605.30621、Harness-R1、HarnessTTT |

总体判断：空白 A 的证据最硬，因为 DarwinX 作者自己写明了未测；B 与 C 可以在 A 的实验框架内顺带完成。

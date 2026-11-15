# 世界模型训练目标核对（2026-11-09）

## 原论文实际做法

| 方法 | 训练时的监督目标 | 推演时如何前进 | 对 AMHS 的适用范围 |
|---|---|---|---|
| DeepFleet RC | 真实观察和历史动作条件下，对下一动作做行为克隆负对数似然；训练采用 teacher forcing | 逐机器人异步预测动作，并更新其周围状态 | 适合训练真实状态下的局部 move/wait；原文没有监督一条长期路线 |
| DeepFleet RF | 对每辆机器人下一动作做交叉熵，梯度跨机器人和时间累积 | 异步生成下一动作 | 适合共享的下一步行为头 |
| DeepFleet GF | 机器人身份、move/wait、移动方向做交叉熵；朝向、载货和目标方位等属性做余弦损失 | 每步预测动作，用固定地图动力学更新位置，再处理占位冲突，重复滚动 | 图拓扑和动态占用值得参考；其“方向”是局部移动，不等于我们的 route commit |
| DreamerV3 | 观察重建、奖励、继续标志的预测损失，加 prior/posterior 的 dynamics KL 与 representation KL；在真实序列上训练 RSSM | 从当前 posterior 出发，在 latent 中用动作和 prior 想象短窗口 | 我们需要对应的状态预测和 latent 一致性；没有奖励/策略任务时不能照抄奖励和 actor 损失 |
| PlaNet | 观察与奖励似然、posterior/prior KL，另以多步 latent overshooting 使多个距离的 prior 对齐对应 posterior | latent 中按候选动作序列推演，MPC 每次只执行第一步 | 可用多步 latent 一致性约束长期误差，目标 posterior 只用于训练 |
| TD-MPC | H 步 latent 递推，对目标编码器的下一状态做一致性损失，并训练奖励与 TD value；梯度沿 latent 轨迹反传 | 短视窗 latent MPC，终端 value 估计视窗外回报 | 多步一致性有启发；目前没有定义回报/策略，不应加入 TD value 目标 |

原文：[DeepFleet v3 §5.2、§6.2、§8.2–8.3](https://arxiv.org/html/2508.08574v3)、[DreamerV3 §2](https://arxiv.org/pdf/2301.04104)、[PlaNet 原作者项目页](https://planetrl.github.io/)、[TD-MPC 论文 §4](https://proceedings.mlr.press/v162/hansen22a/hansen22a.pdf)。用户提供的 DeepFleet v1 PDF 的 RC/RF/GF 目标也逐项核过，与这里引用的章节目标一致。

## 当前实现与这些目标的实际差别

1. `train_semimarkov_event_rssm.py` 的 `_loss_and_metrics` 在一个完整秒之后汇总距离、运动、速度、mode、资源关系的误差。真实 route commit 是外部输入，因此内部 route 分类损失为零。这不是 DeepFleet 的逐次下一动作监督。
2. `EventGraphRSSM` 已有 `prior`、`posterior` 和 `kl_loss`，但训练窗口没有调用 `posterior` 或把 `kl_loss` 加进优化目标。`warmup_history` 只在起点之前无梯度地滤波。故当前 RSSM 的 latent 没有 Dreamer/PlaNet 式的预测一致性训练。
3. `LearnedLedgerRollout.step` 每个微步默认 `truncate_belief=True`，并把类别与轨道投影写回 Python ledger。下一步虽读取自己的预测状态，跨步状态误差无法沿 ledger 反传；这也不是 TD-MPC 的多步 latent BPTT。
4. 轨道状态由当前 RailPath、弧长 `x` 和持久 `routing` 决定。当前测试中的 RailPath 错误可能来自运动相位、route 输入对齐、边界跨越或内部改路缺失。单看 RailPath 正确率不能把这些情况统称为“轨道选择错误”。

## 对下一轮目标的约束

先从真实当前状态和真实已提交 route 出发，按**实际模型更新间隔**监督下一步可观测结果：move/wait、沿轨道位移或速度、跨段是否发生、mode 与资源关系变更。跨段的可选后继必须由已提交 route 和有向拓扑限定；若原始数据存在未作为输入记录的内部改路，应单独计数、补数据或单独建模，不能默认为模型每帧重新规划全局路线。

同时以对应真实下一状态的**停止梯度编码**训练 prior/posterior 一致性，并保留短的自由 latent rollout 一致性实验；这部分是根据 Dreamer/PlaNet/TD-MPC 对 AMHS 的设计推论，须用消融与自由 rollout 测试验证，不是论文直接规定的 AMHS 损失。测试分开报告一步准确度、跨段事件、计划耗尽、真实路线可用性与多步位置误差。

目前的 3-seed 小实验只能证明运动精确率和投影失败率发生变化，尚不能证明新架构长期更准确。对 seed 4 三个起点的 10 秒自由 rollout：原 v1 RailPath 正确率 0.531、位置 MAE 3.314 m、运动精确率 0.462、每帧投影失败 24.992；v2 加运动对齐监督并训练 6 轮后分别为 0.442、3.112 m、0.717、0.200。两次训练轮数与损失不同，不能把差异归因于单一改动。

## 下一帧目标的小实验结果

`deepfleet_hybrid_v1/train_next_step.py` 从 v17 真实帧及当帧已知输入训练下一帧的运动、距离、速度变化、mode、资源关系，并用真实下一帧构造 posterior 计算停止梯度的双向 KL。它在训练时使用 3 个 route seed，验证和测试分别使用独立 seed。运动距离与速度变化以 30 Hz 间隔换算为每秒速率；自由测试仍是 30 Hz 逐帧闭环。此实现是对论文目标的 AMHS 适配，**没有**奖励、价值或多步 latent overshooting。

| seed 4 三起点自由测试 | 1 s RailPath | 10 s RailPath | 10 s 位置 MAE | 10 s 预测/真实速度 | 10 s 运动精确率 | 10 s 投影失败/帧 |
|---|---:|---:|---:|---:|---:|---:|
| v2 整秒闭环训练，6 轮 | 0.860 | 0.442 | 3.112 m | 0.594 / 0.636 m/s | 0.717 | 0.200 |
| v2 下一帧 + KL 训练，5 轮 | 0.882 | 0.483 | 3.512 m | 0.427 / 0.636 m/s | 0.759 | 1.161 |

下一帧实验的验证损失在第 4 轮最低（0.501），latent KL 从第 1 轮约 0.128 降到第 4 轮约 0.036；但 10 秒速度和位置都退步。由于 posterior 没有独立的观察重建目标，KL 下降也可能是 posterior 变得不依赖观察，不能视为学到了足够的系统状态。该结果支持增加**多步可观测状态一致性**和真实观察编码的防塌缩目标，并用同一损失/同一训练量做消融后再扩大数据。当前一步目标不能替代自由 rollout 评价。

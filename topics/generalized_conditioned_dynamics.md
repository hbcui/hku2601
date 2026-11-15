# 面向泛化的硬约束条件动力学

2026-11-15 实现。当前为新的可训练实验分支，未替换既有检查点，也未宣称通过长时域或跨地图验收。

## 执行契约

新增 `conditioned_world_model_generalized_v1`（指数时间记忆）与 `conditioned_world_model_generalized_mamba_v1`（官方 Mamba2 时间记忆）。二者共用 `learned_hard_only_v1` 执行语义。

保留有向拓扑、提交路线、资源容量与关闭状态、控制区保护及制动距离、车距、速度/加速度界限、任务归属、服务停止与账本一致性。候选执行前仍重新检查，事件仍原子提交，投影后的状态仍反馈给下一步。

新版本不启用 native motion prior、强制 Merge 唤醒、队首独占筛选、固定任务分配策略及额外恢复控制器。下游资源预约不再必须等待上游已授权；安全撤回授权仍受控制区与制动距离约束，但不再必须把授权交给更近的车。这些策略选择交给学习事件率。旧模型保持旧语义。新 profile 拒绝混入旧启发式模块或诊断开关，避免静默改变被评估模型。

## 学习与泛化

- 路线与任务端点使用共享节点属性网络，输入为平移/尺度归一化坐标、入度、出度、站点存在性、Bay 存在性。节点 ID 只查表，不对应可训练向量。测试覆盖重新命名节点后的输出一致性；评估可绑定新地图属性。这并不证明跨地图泛化：现有完整图特征和执行器仍要求 AMHS 数据契约，归一化及局部结构表达能力需要跨拓扑实验验证。
- 运动头直接输出界限内加速度。训练和推演使用同一输出，不再训练时选择真实 phase 专家、推演时选择预测专家。phase 是辅助监督，不强制停止或启动。
- 所有共享图与时间参数参与事实序列训练。必须重新生成 matching-profile 候选缓存，不能复用旧策略筛选后的标签。分片合并检查 profile 和节点描述一致性。
- 已验证仿真器分支训练支持直接加速度；原生运动先验不参与该版本的损失。现有分支训练仍剔除不支持的外部输入边界，并在不连续片段重置学习记忆，因此不能声称完整端到端长时域训练。需要用当前检查点和当前代码重新采集、资格检查，再训练。
- Mamba 仍采用 0.25 秒物理 tick；现有事实缓存可能跨越 tick，观察端点重复仍是近似。精确 tick 对齐及长期记忆消融是后续验收项。

## 运行入口

在项目根目录运行。先用小数据分片验证接口；完整训练需使用默认 50/5/5 划分，省略 `--allow-shard` 和三个种子选项。

```bash
PYTHONPATH=src .venv/bin/python -m conditioned_world_model_v4.data \
  --execution-profile learned_hard_only_v1 --allow-shard \
  --train-seeds 0 --validation-seeds 50 --test-seeds 55 \
  --anchors 9000 --frames 30 --output runs/generalized_hard_only_v1/smoke_cache

PYTHONPATH=src .venv/bin/python -m conditioned_world_model_v4.train \
  --architecture conditioned_world_model_generalized_v1 \
  --cache runs/generalized_hard_only_v1/smoke_cache/cache.pt \
  --output runs/generalized_hard_only_v1/smoke_model \
  --hidden 32 --unroll 8 --updates 25 --device cpu

PYTHONPATH=src .venv/bin/python -m conditioned_world_model_v4.evaluate_ensemble \
  --checkpoint runs/generalized_hard_only_v1/smoke_model/latest.pt \
  --data data/processed/native_route_plans_seed0_routeseed55_60m_v17 \
  --start-frame 9000 --frames 1800 --event-seeds 0 1 2 3 4 \
  --output runs/generalized_hard_only_v1/ensemble_60s
```

Mamba 训练选择 `conditioned_world_model_generalized_mamba_v1` 和 `--device cuda`。评估不能启用旧运动先验、惯性覆盖或强制唤醒参数。ensemble 保留失败样本和失败率；成功样本指标明确为以完成为条件的统计，不挑选 best-of-K。

## 验收边界

接口实验使用每个划分一个种子、一个起点、1 秒训练数据；不能据此宣称完成正式训练。独立评估需同时覆盖多起点、事件随机种子、交通负载、路线与地图变化，并报告安全违规、任务完成、等待与恢复、资源状态及整段轨迹指标。路线反事实排序必须另用仿真器分支真值验证；记录路线条件下的预测不能替代该实验。

实现：[通用动力学](../../src/conditioned_world_model_v4/generalized.py)、[训练](../../src/conditioned_world_model_v4/train.py)、[分支训练](../../src/conditioned_world_model_v4/train_executed_joint.py)、[多采样评估](../../src/conditioned_world_model_v4/evaluate_ensemble.py)。

## 本次已执行验证

[验证记录](../../runs/generalized_hard_only_v1/verification.json)：125 项模块测试及 16 项安全投影测试通过。Graph 小样本训练 25 次更新，Mamba 小样本训练 5 次更新；两者均验证到共享编码、时间记忆与加速度头梯度。两个事件采样种子的 1 秒自由推演完成；模型访问状态采集完成一个探针，但尚未完成新版本的合格教师分支训练。

Graph 小样本检查点还完成一次 seed 55/frame 9000、事件随机种子 2 的 60 秒闭环，无状态重置或未来 posterior 校正。60 秒位置 MAE 为 19.649 m，运动召回 0.0759，预测均速 0.0676 m/s（真实 0.7626 m/s）。这表明接口可以持续运行，但小样本模型仍停滞，明确不晋升。不能用此次 1 秒数据预训练实验推断充分训练后的架构效果。

分支损失额外覆盖停止状态的梯度问题：负加速度经过速度非负截断后，速度/距离误差可能没有梯度；对具有有效 ordinary 标签的区间保留加速度监督，让实际观察到的重启提供学习信号。没有加入强制重启规则。

## 论文核对后的训练改进：确定性多步预测记忆

新增架构 `conditioned_world_model_predictive_v1`，实现于 [predictive.py](../../src/conditioned_world_model_v4/predictive.py)，目标在 [predictive_objective.py](../../src/conditioned_world_model_v4/predictive_objective.py)。这不是在旧记忆上直接添加 KL，也不宣称复现 Dreamer、PlaNet 或 TD-MPC2。

### 先纠正训练概念

真实状态上的 teacher forcing 是有效的序列概率训练方法，不能因为使用它就断言无法长推演。DeepFleet RC 明确采用这种方法；DreamerV3 也在真实序列上训练观察模型和 prior/posterior。模型访问状态的仿真器分支监督是可用的补充，并非所有世界模型都必须先实现的前提。

参考：[DeepFleet §5.2、§8](https://arxiv.org/html/2508.08574v3)、[DreamerV3 agent](https://github.com/danijar/dreamerv3/blob/main/dreamerv3/agent.py) 与 [RSSM](https://github.com/danijar/dreamerv3/blob/main/dreamerv3/rssm.py)、[PlaNet 多步先验目标](https://planetrl.github.io/)、[TD-MPC2 latent rollout](https://github.com/nicklashansen/tdmpc2/blob/main/tdmpc2/tdmpc2.py)。上述工作的想象步数不是秒，也不等于长时域完整轨迹精度保证。

### 新版本训练和执行的定义

1. **显式状态与局部似然保留。** 当前执行图仍输入事件和运动头；资源候选合法性、原子事件提交和硬投影仍由原执行器完成。连续运动只在有效、可辨识区间监督。没有添加强制启动或吞吐量奖励。
2. **新增真正递推的预测记忆。** 从观测起点编码初始化，每完成 0.25 秒物理时间运行一次共享 GRU 转移。外部 route commit 和实际任务到达在揭示时更新条件。任务到达目前使用共享数量脉冲，具体端点仍由显式任务图和条件头读取；这仍是表达能力的限制。
3. **监督段不回填中间真值。** 先验递推不读取中间的真实位置、队列、资源或任务特征。真实历史上下文可以用于起点滤波；自由段以后，真实图只形成固定教师的目标。推演也使用同一个先验。当前显式预测图仍供局部解码器读取，因此这不是完全隐式的 latent-only 模型。
4. **目标明确限制为 OHT 表征。** 教师是预训练检查点的状态编码器，冻结参数；监督未来 OHT 的编码，梯度穿过多次先验转移。采用平方误差，学习条件特征均值。它不是完整账本的概率似然，也不把一次随机预测事件分支强行回归到原分支的物理轨迹。任务实体的出生、消失不会通过池化泄露给 OHT 先验。
5. **真实出生与已有实体分开处理。** 已揭示的新实体可以从自身当前特征初始化；已有实体的预测记忆不能通过执行状态反馈重新编码。真实观测到来时才调用显式滤波函数。事件和运动头仍能立即看到自己的执行图。
6. **物理时间窗口和课程。** `--unroll-seconds`、`--context-seconds` 替代仅凭事件段数理解跨度。可选择 `--curriculum-seconds 1 2 5 10`，并保持历史上下文与梯度；若原始连续片段不足 10 秒，训练入口拒绝把短数据冒充该跨度。窗口上下文重叠，但监督标签不重复、不丢首尾。

目标为原有事件/运动/phase 监督加固定教师 OHT 表征预测误差。当前确定性先验不是随机状态分布，因此没有 posterior、ELBO 或 latent KL；随机性主要来自事件过程。该训练没有穿过完整 Python 账本的离散分支求导。

### 运行与消融入口

以下使用已有 5 秒扩展试验缓存，属于实验，不是完整 50/5/5 训练：

```bash
PYTHONPATH=src .venv/bin/python -m conditioned_world_model_v4.train \
  --architecture conditioned_world_model_predictive_v1 \
  --init runs/generalized_hard_only_v2/factual_model/latest.pt \
  --cache runs/generalized_hard_only_v2/expanded_cache/cache.pt \
  --output runs/predictive_hard_only_v1/new_curriculum \
  --hidden 64 --curriculum-seconds 1 2 5 --context-seconds 2 \
  --updates 600 --validation-every 100 --device cpu
```

`--predictive-coef 0` 可关闭多步辅助误差，保留相同预测记忆架构；`--selection-loss local` 可用相同局部验证目标选择该消融与完整模型，避免比较不同损失尺度。多步系数和时域应由验证实验确定，不是物理限制。完整训练需要更长片段、更多种子和交通状态，再独立验证，不能由上述接口实验替代。

### 本轮已完成的配对训练

- 两个模型均从同一个 600-update 事实预训练实验所选检查点开始，hidden=64、同一缓存、相同学习率、每次监督约 5 秒，CPU 各更新 60 次。
- 对照保留 generalized 指数记忆；候选使用新预测记忆及多步表征误差，因此它是整体方法比较，不是只改变一个损失项的消融。两者验证总损失定义不同，不能直接比较数值大小。
- 缓存训练种子 0/1/2、两个起点，每段 5 秒，总计 30 秒独立训练轨迹；验证 seed 50。该规模只适合方法验证，不能说明跨地图泛化。
- 1→2→5 秒课程另执行 3-update 接口测试。它不参与模型质量比较。
- 中断或被实现变更作废的开发运行均有单独状态记录，不能计入完成结果。代码测试覆盖先验不读取中间真值、仅多步目标的梯度、物理时间分段一致性、目标教师冻结和检查点恢复。

长闭环配对报告将分别保存验证 seed 50/frame 18000 和测试 seed 57/frame 9000 的 10/30/60 秒结果。测试使用事件随机种子 0/1；这两个采样共享一个测试交通起点，不能作为两个独立交通场景做统计晋升。

### 残差先验：避免未训练转移破坏已知状态

绝对先验的验证诊断未通过：1/2/5 秒 OHT 表征 MSE 分别为 0.496/0.586/0.660，均高于固定起点表征的 0.129/0.248/0.376。该版本保留作失败对照，不将“多步损失下降”解释为有效预测能力。

新增 `conditioned_world_model_predictive_residual_v1`，见 [predictive_residual.py](../../src/conditioned_world_model_v4/predictive_residual.py)。其物理时间先验为 `h_next = h + dt * learned_delta(h, context)`，增量输出层及外部条件增量初始化为零。因此初始化时，未获学习证据的转移严格保持已有表征；之后增量及外部条件反应由监督学习。它不控制任何车辆必须启动、等待或获得资源，也不限制学得的表征必须停留在 GRU 的绝对输出范围。

该版本从同一预训练检查点开始，CPU 更新 60 次，按相同局部验证损失选择第 60 次权重。冻结教师未参与优化。训练报告见 [residual_model/report.json](../../runs/predictive_hard_only_v1/residual_model/report.json)。在两个验证片段中，1/2/5 秒表征 MSE 为 0.103/0.173/0.283，相对保持基线分别改善约 21%/30%/25%；这是短期表征预测证据，不是 60 秒物理推演质量证明。详见 [表征诊断](../../runs/predictive_hard_only_v1/residual_feature_validation.json)。

运行新版本时，将上方命令的架构改为 `conditioned_world_model_predictive_residual_v1`。旧绝对先验和新残差先验使用不同架构名，不能静默互换检查点。每个 epoch 与最终检查点现在均包含独立恢复所需的架构、特征维数、节点描述和执行配置。

残差版本额外使用未参与本轮预训练、验证选择和先前绝对先验试验的 route seed 58/frame 18000 做 60 秒配对测试。这里只用一个事件采样，仍不足以证明跨负载、跨地图或统计上的普遍改善。不同架构的更新频率和参数量也不同；配对比较属于方法诊断，不是严格等算力的单因素消融。

### 完成的长闭环结果与决定

本轮共完成 9 次完整 60 秒执行。所有运行均无物理状态重置、无未来 posterior 校正；报告中的内部事件拒绝为零。这是已编码执行路径的运行证据，不是所有原生规则的形式化安全证明。141 项模块测试、16 项安全投影测试通过；epoch 检查点实际独立恢复成功，残差版本冻结教师与来源编码器逐位一致。

**当前候选未晋升，默认模型未改变。** 完整汇总：[verification.json](../../runs/predictive_hard_only_v1/verification.json)。

| route seed / 起点 | 模型 | 60 秒位置 MAE | 运动召回率 | Merge 持有召回率 |
| --- | --- | ---: | ---: | ---: |
| 验证 50 / 18000 | 同预算事实训练对照 | 10.989 m | 0.581 | 0.229 |
| 验证 50 / 18000 | 残差预测记忆 | 13.779 m | 0.318 | 0.183 |
| 新测试 58 / 18000 | 同预算事实训练对照 | 11.845 m | 0.521 | 0.246 |
| 新测试 58 / 18000 | 残差预测记忆 | 14.844 m | 0.314 | 0.149 |

来源：[验证配对](../../runs/predictive_hard_only_v1/residual_validation_comparison.json)、[新测试配对](../../runs/predictive_hard_only_v1/residual_test_comparison.json)。此前绝对先验在 seed 57 两个事件采样上的 60 秒均值也没有通过：位置 MAE 为 12.676 m，对照 12.556 m；召回为 0.558，对照 0.562，见[绝对先验测试](../../runs/predictive_hard_only_v1/absolute_test_comparison.json)。

残差版本短期特征预测改善，但两个 60 秒起点的位置与运动指标都退步，不能据此声称学到了更好的长期环境动力学。新测试的任务启动审计中，残差模型匹配到 1/100 个真实新启动车辆—任务对，对照为 0/100；该指标受身份和时序共同影响，说明任务演化仍然严重失配，不能把长期误差单独归因于记忆网络或硬投影。

本轮交付的是正确区分事实概率监督、多步表征递推和硬执行验收的训练/验证机制，以及明确失败的候选实验。下一步需要更长、更独立的数据和任务/资源概率验证，而不是继续根据这个测试起点修规则或把特征误差改善当成物理质量提升。`--predictive-coef 0` 的同架构消融、共享局部选模目标和 1→2→5→更长物理时间课程已有入口；尚未完成完整数据量上的实验，不能将其写成已验证结论。

# 模型与实验进度

更新日期：2026-11-15。以下按现有源码说明和结果文件整理；运行仍可能继续变化。不同实验的数据、锚点、训练预算和执行规则不一定相同，不能仅凭版本号或单个指标排序。

## 当前研究线

当前边界已明确为 **routing 唯一 action，任务与车辆绑定属于外部条件**。实现入口为 `conditioned_world_model_executed_routing_v1`，保留执行图记忆，移除自主派发头和派发损失，见[当前条件协议](routing_only_external_conditions.md)。旧 `conditioned_world_model_executed_graph_v1` 的自主派发属于历史实验；其任务匹配失败不是给定绑定条件下的动力学结论。

当前关键改进为[资源—运动联合训练](joint_trajectory_training.md)：硬执行器中采样自身轨迹，回放其概率并对神经转移网络自动微分，替代先前参数差分诊断。4 次新增联合更新后，在同协议的两个场景、共 8 条成对 60 秒轨迹中，平均 energy score 降低约 22.8%、位置误差降低约 21.8%、Merge 召回提高，但运动召回下降。用户已明确要求扩大到完整 50／5／5 划分，采用冻结当前版本的 tmux 后台任务，见[配置和运行记录](formal_joint_50_5_5.md)；长期性能尚未验收、默认模型未晋升。旧 factual-only 自动扩训仍禁用。

新增 `conditioned_world_model_predictive_residual_v1`：观测自由的确定性预测记忆、固定教师多步 OHT 表征目标和物理时间课程已实现。短期表征验证超过保持基线，但验证 seed 50 和新测试 seed 58 的 60 秒位置与运动指标均退步，候选未晋升。完整结果见[训练设计与验证记录](generalized_conditioned_dynamics.md)。

新增[面向泛化的条件动力学](generalized_conditioned_dynamics.md)：共享结构条件编码、直接学习加速度与无额外启发式接管的执行 profile 已实现并通过接口验证。仅完成小样本训练，60 秒诊断仍停滞，未晋升；见[验证记录](../../runs/generalized_hard_only_v1/verification.json)。

| 分支 | 作用与状态 | 证据 |
| --- | --- | --- |
| 条件世界模型 V4 | 条件事件动力学和硬执行器；另有 Graph → 官方 Mamba2 的实验分支。V4 状态记录为已实现、训练和评估，但未晋升，默认模型未改变。 | [V4 README](../../src/conditioned_world_model_v4/README.md)、[状态](../../runs/conditioned_world_model_v4/status.json) |
| 继承式联合候选 V2/V3 | 更近期的训练与闭环候选，已有单次 60 秒报告；它们是研究结果，尚需独立锚点及一致预算的比较。 | [V2 比较](../../runs/inherited_joint_v2/comparison.json)、[V3 比较](../../runs/inherited_joint_v3/comparison.json) |
| 硬约束 GF V6 | 合法资源动作、事件监督和安全运动投影；20-seed 候选的 60 秒运动召回仍偏低。 | [V6 README](../../src/deepfleet_gf_constrained_v6/README.md) |
| 条件混合模型 V1 | 资源事件率、运动和恢复头的实验原型，训练与事件时间近似均有明确限制。 | [混合模型 README](../../src/conditioned_hybrid_wm_v1/README.md) |
| 早期 GF boundary-v3 | 在早期筛选中通过既定架构选择；这是当时的结论，不是对后续分支的统一排名。 | [架构筛选](../../src/deepfleet_final_selection_v1/README.md) |

V4 的 Graph → Mamba2 分支以实体身份对齐时间记忆，使用固定物理时间 tick，并通过硬执行器反馈状态。官方 Mamba2 分支需要 CUDA；其架构和实验范围见 [V4 README](../../src/conditioned_world_model_v4/README.md)。V4 状态文件指出，模型访问状态的通用仿真器恢复、相位持续时间建模和完整 60 秒训练仍未完成，也没有统计上支持优于保留的 V2 的改进。

## 最近记录的 60 秒候选结果

下表是 route seed 55、frame 9000 的单锚点 60 秒自由闭环结果；三个报告使用相同数据和起点。基线采用 `liveness_service_stop_only_v1` 检查点，并启用 native motion prior 和 bounded Merge wakeup。结果不能视为跨种子的最终结论。

| 候选 | 位置 MAE | 运动召回率 | 路径准确率 | Merge 持有召回率 |
| --- | ---: | ---: | ---: | ---: |
| 共同基线 | 5.889 m | 0.759 | 0.387 | 0.344 |
| inherited joint V2 | 6.475 m | 0.662 | 0.353 | 0.328 |
| inherited joint V3 | 6.597 m | 0.710 | 0.390 | 0.281 |

来源：[基线报告](../../runs/inherited_joint_v2/baseline/report.json)、[V2 报告](../../runs/inherited_joint_v2/candidate/report.json)、[V3 报告](../../runs/inherited_joint_v3/candidate/report.json)。V3 相对 V2 的运动召回和路径准确率较高，但相对共同基线，位置误差、运动召回和 Merge 持有召回均未改善；这些结果不足以单独确定晋升。

`boundary_tasks_v1` 的[训练日志](../../runs/boundary_tasks_v1_training.log)记录了锚点缺少事件队列检查点导致的异常；其[探针报告](../../runs/boundary_tasks_v1/probe/report.json)记录了一个接受的序列及 189 个正事件。这些是数据和训练路径的诊断，尚不是候选模型验收结果。

## 评估与后续工作

1. 固定训练、验证、测试种子及锚点，记录输入条件、模型权重、执行器配置和预算；报告 10/30/60 秒自由闭环的逐锚点结果及多种子汇总。
2. 同时检查位置、速度、路径、运动 precision/recall、任务生命周期、资源持有与 grant，以及 route 耗尽、投影原因和紧急制动；单一位置或运动指标不能代表系统质量。
3. 区分真实未来输入与模型生成状态。分支偏离后，只有可靠恢复的仿真器或有依据的标签才能提供反事实监督；设计文档中的方案不等于已有实现。
4. 扩展任务事件顺序、资源竞争和模型访问状态的可审计标签，并以独立测试确定候选是否改善；安全执行检查仅覆盖已经编码的规则，并非完整安全证明。

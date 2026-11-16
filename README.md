# AMHS 世界模型文档

更新：2026-11-16。本目录是全 Fab 多车搬运系统的设计、训练和实验说明入口。
当前主线是 `conditioned_world_model_executed_routing_v1`：任务到达与任务—车辆
绑定是外部条件，路线是唯一动作，模型在硬执行器内预测后续动力学。

> 当前候选尚未晋升为默认模型。50／5／5 正式训练及其 60 秒验证/测试仍未全部完成；
> 设计文档说明的是模型边界与实现，不等于性能或泛化已获验证。

## 建议阅读路径

### 1. 先理解当前模型

1. [四层条件世界模型设计](topics/four_layer_conditioned_world_model.md) —— 外部条件、路线动作、执行图记忆、硬执行器如何闭环协作。
2. [Routing 唯一动作与外部条件注入](topics/routing_only_external_conditions.md) —— 为什么任务绑定不是模型动作，以及时间与因果边界。
3. [硬执行器中的资源—运动联合训练](topics/joint_trajectory_training.md) —— 自身执行轨迹上的概率重放、energy score 和当前验证证据。
4. [50／5／5 联合训练运行协议](topics/formal_joint_50_5_5.md) —— 固定数据切分、训练课程、审计与恢复方式。

### 2. 再查看当前实现与状态

| 入口 | 内容 |
| --- | --- |
| [模型与实验进度](topics/model_status.md) | 各模型分支、已完成实验、限制与后续验证 |
| [V4 实现与实验记录](../src/conditioned_world_model_v4/README.md) | 当前源码模块、检查点兼容性和实验入口 |
| [当前运行目录](../runs/executed_routing_v1/) | 固定配置、训练进度、验证/测试报告与审计文件 |
| [项目 README](../README.md) | 全项目入口与环境说明 |

## 当前模型设计

| 主题 | 文档 | 说明 |
| --- | --- | --- |
| 四层架构 | [四层条件世界模型设计](topics/four_layer_conditioned_world_model.md) | 当前模型的数据流、职责分层、时间语义和使用边界 |
| 条件契约 | [Routing 唯一动作与外部条件注入](topics/routing_only_external_conditions.md) | 输入协议、任务绑定语义与反事实解释范围 |
| 泛化结构 | [面向泛化的硬约束条件动力学](topics/generalized_conditioned_dynamics.md) | 结构节点描述、直接加速度与硬执行约束 |
| 联合训练 | [硬执行器中的资源—运动联合训练](topics/joint_trajectory_training.md) | 随机执行、概率回放、多粒子目标和既有诊断 |
| 正式实验 | [50／5／5 联合训练运行协议](topics/formal_joint_50_5_5.md) | 训练/验证/测试隔离、课程与可恢复运行 |

## 数据、地图与评估背景

| 主题 | 文档 | 说明 |
| --- | --- | --- |
| 全 Fab 地图 | [全 Fab Graph-Floor 映射](topics/deepfleet_full_fab_mapping.md) | 原生拓扑、OHT、资源映射与采样审计 |
| 论文与实现审计 | [DeepFleet Graph-Floor 审计](topics/deepfleet_graph_floor_audit.md) | DeepFleet 论文与早期实现的差异 |
| 小规模基准 | [Directed-grid MAPD pilot](topics/toy_pilot.md) | 早期 toy 场景及适用边界 |
| V6 约束实验 | [V6 硬约束实验](../src/deepfleet_gf_constrained_v6/README.md) | 资源事件、安全投影与长时域停滞评估 |

## 历史设计与实验

下列文档保留重要的失败分析、术语和决策背景，但不应覆盖当前条件世界模型的
定义或被引用为当前性能结论。

| 文档 | 历史定位 |
| --- | --- |
| [执行图记忆与派发语义修正](topics/executed_graph_and_dispatch_contract.md) | 历史自主派发实验；已由 routing-only 外部条件契约取代 |
| [训练目标审计](topics/world_model_training_objectives_audit.md) | 早期目标、实验与局限 |
| [任务完成目标审计](topics/task_completion_objective_audit.md) | 生命周期概率与稀有标签问题 |
| [硬约束训练重构](topics/constrained_world_model_training_redesign.md) | 历史失败诊断与重构方案 |
| [长时域约束架构 V7](topics/long_horizon_constrained_architecture_v7.md) | 当时提出的架构建议与验收门槛 |
| [随机动力学训练](topics/stochastic_dynamics_training_design.md) | 概率建模与验证方案的背景材料 |
| [面向决策的短视域想象](topics/decision_centric_imagination.md) | 控制与重规划的研究方案 |
| [历史架构筛选](../src/deepfleet_final_selection_v1/README.md) | 早期 boundary-v3 筛选；不能推广为当前模型结论 |

## 工程资料

- [开发约定](topics/development.md)
- [迁移记录](topics/migration.md)
- [论文文件](papers/)

`topics/` 保存叙述性设计与审计文档；模型源码、运行日志和数值结果仍以各自目录
中的文件为准。引用实验结果时，请同时给出检查点、数据切分、预测时长和随机粒子
数，避免将历史协议与当前协议混用。

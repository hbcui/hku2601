# AMHS 世界模型文档

这里汇总全 Fab 多车搬运系统的地图、世界模型、训练与评估文档。更新日期：2026-11-15。代码和实验是变化中的研究工作；下列历史审计和设计稿保留撰写时的结论，不代表当前版本已实现或通过验收。

## 当前进度

- [模型与实验进度](topics/model_status.md)：当前模型分支、可追溯的实验结果、限制和后续验证。
- [V4 实现与实验记录](../src/conditioned_world_model_v4/README.md)：条件事件动力学、Graph → Mamba2、执行器及闭环实验。V4 状态文件仍标记为未晋升；实验候选不等于默认模型。
- [V6 硬约束实验](../src/deepfleet_gf_constrained_v6/README.md)：资源事件与安全投影，以及长时域停滞的评估。
- [历史架构筛选](../src/deepfleet_final_selection_v1/README.md)：早期 boundary-v3 筛选，不能直接推广为当前所有模型的最终结论。

## 主题索引

| 主题 | 文档 | 定位 |
| --- | --- | --- |
| 地图与数据 | [全 Fab Graph-Floor 映射](topics/deepfleet_full_fab_mapping.md) | 原生拓扑、OHT 和资源映射及采样审计 |
| 地图与数据 | [Directed-grid MAPD pilot](topics/toy_pilot.md) | 早期小规模基准及其边界 |
| 架构审计 | [DeepFleet Graph-Floor 审计](topics/deepfleet_graph_floor_audit.md) | 论文与早期实现的差异 |
| 训练 | [训练目标审计](topics/world_model_training_objectives_audit.md) | 历史目标、实验和局限 |
| 训练 | [任务完成目标审计](topics/task_completion_objective_audit.md) | 生命周期概率与稀有标签问题 |
| 设计 | [硬约束训练重构](topics/constrained_world_model_training_redesign.md) | 失败诊断及训练方案 |
| 设计 | [长时域约束架构 V7](topics/long_horizon_constrained_architecture_v7.md) | 当时的架构建议与验收门槛 |
| 设计 | [随机动力学训练](topics/stochastic_dynamics_training_design.md) | 概率建模与验证方案 |
| 设计 | [面向决策的短视域想象](topics/decision_centric_imagination.md) | 控制与重规划的研究方案 |
| 工程 | [开发约定](topics/development.md) · [迁移记录](topics/migration.md) | 模块边界与历史迁移 |

`topics/` 按主题保存文档；旧的 `parts/` 路径保留为兼容链接。实验报告和模型源码仍以各自目录为准。全项目入口见 [项目 README](../README.md)。

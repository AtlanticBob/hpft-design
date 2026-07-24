# HPFT — design repo

设计文档：HPFT 系统的权威说明、历史决策记录、平台踩坑手册。不含代码、
不含实验数据——那两块在兄弟仓库里，见下。

## 先读什么

**`docs/design.md`** ——唯一权威的设计文档（2026-07-23 按现行实现重写），
从头讲清楚系统现在的样子：解决什么问题、每个部件为什么这样设计、性质与
实测定数。历史演进（方案 A/C/D 的取舍）不在这篇里，按需去
`rd_fairness_design_c.md`、`rd_fairness_design_e.md` 和 `docs/archive/`。
（旧的统一文档 `design_and_implementation.md` 已删除，由 `design.md` 取代。）

## 文件

- `design.md` —— 主设计文档（见上）；`design_en.md` —— 其英文版
  （内容一致，审阅与修改以英文版进行）。
- `planned_extensions.md` —— 三个计划中组件（多属主组合实证、对抗性
  租户加固、在线政策重编程）的详细设计与实验方案。
- `response_law_mimd_analysis.md` —— MIMD 响应律的理论分析、实验计划、
  参数调优讨论；配套 `design.md` §4/§8。
- `rd_fairness_design_c.md`、`rd_fairness_design_e.md` —— 方案 C/E 的原始
  设计说明，均已被主文档取代，留作决策历史存档（各自文首标注了取代关系）。
- `ops_notes.md` —— 平台缺陷病历与实验卫生规则，每条问题按"症状—根因—
  修复—教训"记录，动 lab 前必读。
- `cc_mode_switching.md` —— `tools/cc_mode.sh`（在 `hpft-implementation`
  仓库）的配套教程，DCQCN↔PCC+HPFT、GBN↔SR 切换的固件行为细节。
- `evaluation_index.md` —— 论文评估章节的实验数据总索引，把每节素材指向
  `hpft-paper` 仓库 `results/` 下的具体目录。
- `archive/` —— 4 个归档文件，合并了 17 个已被主文档或后续设计吸收的
  历史文档：`superseded-designs.md`（被放弃的设计方向：方案 A、
  receiver-driven、TCP shaper v2、T3.2 DPU TCP 卸载）、
  `v2-shaper-phase-reports.md`（早期 PCC 可行性调研时间线）、
  `tuning-and-diagnostics-log.md`（调参与诊断记录）、
  `handoff-history.md`（历史交接文档）。

## 仓库关系

本仓库和 [`hpft-implementation`](../hpft-implementation)、
[`hpft-paper`](../hpft-paper) 是同一个 git 历史用 `git filter-repo`
切出来的三个仓库，各自保留对应路径的完整提交历史。约定放在同一父目录下
作兄弟目录。代码/lab 配置/工程验证数据在 `hpft-implementation`；论文草稿
和评估实验数据在 `hpft-paper`。本仓库目前只在本地，没有建远程仓库。

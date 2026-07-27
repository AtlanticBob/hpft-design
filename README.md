# HPFT — design repo

设计文档：HPFT 系统的权威说明、理论分析、平台踩坑手册。不含代码、不含
实验数据——那两块在兄弟仓库里，见下。

## 先读什么

**`docs/design.md`**（英文版 `docs/design_en.md`）——唯一权威的设计
文档，从头讲清楚系统的样子：解决什么问题、每个部件为什么这样设计、
性质与实测定数。**注意版本落差**：文档描述的是 v3.1（跟踪-审计律），
lab 现在跑的是它已充分实测的前身（MIMD 律）；两者只差发送端律与遥测
格式，差异在 `design.md` 文首与 §4.2、§8.3 就地标注。

## 现行文件（4 篇，都是当前有效的）

- `design.md` / `design_en.md` —— 主设计文档，中英双版内容一致
  （审阅与修改以英文版进行）。
- `design_theory.md` —— 响应环的流体模型理论分析（教程式：通用方法 →
  HPFT 逐步分析与实测对账 → 参数整定规则）。设计参数的取值理由在此。
- `ops_notes.md` —— 平台缺陷病历与实验卫生规则，每条按"症状—根因—
  修复—教训"记录，**动 lab 前必读**。
- `cc_mode_switching.md` —— `tools/cc_mode.sh`（在 `hpft-implementation`
  仓库）的配套教程，DCQCN↔PCC+HPFT、GBN↔SR 切换的固件行为细节。

## 辅助文件

- `planned_extensions.md` —— 三个已立项、未开工的扩展（多属主组合实证、
  对抗性租户加固、在线政策重编程）的设计与实验方案。
- `evaluation_index.md` —— 论文素材索引，把评估章节指向 `hpft-paper`
  仓库 `results/` 下的目录。**内含的数字是旧配置期的**，可信度分代见
  `hpft-implementation/results/EXPERIMENT_STATUS.md`。

## archive/ —— 已被取代，不要当作现行设计读

历史决策与被放弃方向的存档。写论文考据、追溯"某个决定当初为什么这么
定"时才需要；**其中的机制描述、参数值、实验结论都已过时，实现时以
`design.md` 为准**。

- `rd_fairness_design_e.md` —— 方案 E 定型时的原始设计 + Q1–Q27 决策
  日志（决策出处仍以它为准，机制叙述已被 `design.md` 取代）。
- `rd_fairness_design_c.md` —— 方案 C 存档；其加权律被 `design.md` §9
  的核心网扩展引用。
- `response_law_mimd_analysis.md` —— AIMD→MIMD 定案期的理论分析与三律
  对照（其 §3 提出的"比例/EMA 松弛"方向即后来的 v3.1 跟踪律）。
- `superseded-designs.md` —— 被放弃的设计方向（方案 A、receiver-driven、
  TCP shaper v2、T3.2 DPU TCP 卸载）。
- `v2-shaper-phase-reports.md` —— 早期 PCC 可行性调研时间线。
- `tuning-and-diagnostics-log.md` —— 调参与诊断记录。
- `handoff-history.md` —— 历史交接文档。

## 仓库关系

本仓库和 [`hpft-implementation`](../hpft-implementation)、
[`hpft-paper`](../hpft-paper) 是同一个 git 历史用 `git filter-repo`
切出来的三个仓库，各自保留对应路径的完整提交历史。约定放在同一父目录下
作兄弟目录。代码/lab 配置/工程验证数据在 `hpft-implementation`；论文草稿
和评估实验数据在 `hpft-paper`。本仓库目前只在本地，没有建远程仓库。

# hpft-design

HPFT 的设计文档。实现在 `hpft-implementation`，论文素材在 `hpft-paper`。

新接手一个任务时：先看 `hpft-implementation/README.md` 知道 lab 长什么样、
怎么跑实验；要动 lab 再读 `docs/ops_notes.md`；要理解系统为什么这样设计
才读 `docs/design.md`。不必通读。

## 读什么

- **`docs/design.md`** —— 系统唯一权威的设计说明，描述的即现行实现。
  §1–2 问题与全貌，§3–5 按控制环顺序讲三个子系统，§6 参数，§7 实现
  概况，§8 性质与实测定数，§9 边界与扩展。
- **`docs/design_en.md`** —— 与中文版逐节对应。**审阅与修改以英文版
  进行**；改任何一版都要同步另一版。
- `docs/design_theory.md` —— 响应环的流体模型分析：均衡、收敛、稳定性
  的推导与实测对账，以及参数整定规则。§6 每个取值的理由在这里。
- `docs/ops_notes.md` —— **动 lab 前必读**：操作规程与平台硬约束，按
  "开跑之前 / 改东西的时候 / 平台约束 / 症状速查"组织。只写现在还成立的
  规则，事故经过在 git 里。
- `docs/cc_mode_switching.md` —— CC 模式（DCQCN / PCC、GBN / SR）怎么切。
- `docs/planned_extensions.md` —— 三个已立项未开工的扩展。

## 纪律

- `design.md` 是权威。实现与文档不一致时，先默认是实现的缺口。
- 中英两版必须同步。

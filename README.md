# hpft-design

HPFT 的设计文档。实现在 `hpft-implementation`，论文素材在 `hpft-paper`。

新接手一个任务时：先看 `hpft-implementation/README.md` 知道 lab 长什么样、怎么跑实验；要动 lab 再读 `docs/ops_notes.md`；要理解系统为什么这样设计才读 `docs/design.md`。不必通读。

## 读什么

- **`docs/design.md`** —— 论文的设计章节。§1 模型、目标与现有机制为什么不够；§2 设计总览；§3–§6 四个部件（造信号 / 算份额 / 跟踪 / 在租户 CC 之下执行）；§7 性质与边界；§8 实现（一段）。**只写设计，不写工程细节**——lab 长什么样、怎么跑实验在 `hpft-implementation/README.md`，动 lab 的规则在 `ops_notes.md`。
- **`docs/design_en.md`** —— 与中文版**逐节逐段对应**（32 个标题一一对齐）。**审阅与修改以英文版进行**；改任何一版都要同步另一版。
- `docs/design_theory.md` —— 响应环的流体模型分析，以及**整定流程**：均衡、收敛、稳定性的推导，再反解成一套顺序步骤，回答「换一个拓扑该怎么把参数算出来」（第三部分）。design.md §3 的 $V$ 与 §5.3 的 $k$、$\zeta$ 从哪来，看这里。
- `docs/ops_notes.md` —— **动 lab 前必读**：操作规程与平台硬约束，按"开跑之前 / 改东西的时候 / 平台约束 / 症状速查"组织。只写现在还成立的规则，事故经过在 git 里。
- `docs/cc_mode_switching.md` —— CC 模式（DCQCN / PCC、GBN / SR）怎么切。
- `docs/planned_extensions.md` —— 三个已立项未开工的扩展。

## 纪律

- `design.md` 是权威。实现与文档不一致时，先默认是实现的缺口。
- 中英两版必须同步，节编号一一对应。
- **不要硬折行**：一个段落写成一行，交给渲染器软折。硬折行让 prose 的 diff 变成整段重排，看不出改了哪句。

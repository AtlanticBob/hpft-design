# hpft-design

HPFT 的设计文档。实现在 `hpft-implementation`，论文素材在 `hpft-paper`。

新接手一个任务时：先看 `hpft-implementation/README.md` 知道 lab 长什么样、怎么跑实验；要动 lab 再读 `docs/ops_notes.md`；要理解系统为什么这样设计读 `docs/design_v4.md`，要理解它为什么稳、保证了什么、换规模怎么调参读 `docs/design_theory_v4.md`。不必通读。

## 读什么

- **`docs/design_v4.md`** —— **现行设计，以它为准**。只写设计本身：§1 原则、§2 名词与符号、§3 结构、§4 接收端、§5 发送端、§6 执行面、§7 参数、§8 边界与开放问题、§9 实现位置与已知差异、§10 环路滞后附录。§1–§6 换平台不变；§7 是参数及其定法。当前验证状态在 `hpft-implementation/validation/STATUS.md`。
- **`docs/design_theory_v4.md`** —— **配套的控制理论**。把系统写成一个可分析的对象（两时间尺度、快环的闭合与开环两个区间），据此给出三条关键性质、一条尺度不变性命题，以及从中直接读出来的调参配方：两个测量加两个无量纲选择，其余是代数。设计文档不重复这里的推导。
- `docs/design.md` —— **早一版（v2/v3）的设计章节，已被 `design_v4.md` 取代**。保留的唯一理由是实现里还有 24 处代码注释按节号引用它（`rx_agent.py`、`tx_agent_e.py`、`hw_maxrate.py`、`flow_postflight.py`）；要删它得先把那些注释改指 v4 的节号。**两者不一致时一律以 v4 为准。**
- `docs/ops_notes.md` —— **动 lab 前必读**：操作规程与平台硬约束，按"开跑之前 / 改东西的时候 / 平台约束 / 症状速查"组织。只写现在还成立的规则，事故经过在 git 里。
- `docs/cc_mode_switching.md` —— CC 模式（DCQCN / PCC、GBN / SR）怎么切。
- `docs/planned_extensions.md` —— 三个已立项未开工的扩展。

## 纪律

- `design_v4.md` 是权威。实现与文档不一致时，先默认是实现的缺口。
- **不要硬折行**：一个段落写成一行，交给渲染器软折。硬折行让 prose 的 diff 变成整段重排，看不出改了哪句。

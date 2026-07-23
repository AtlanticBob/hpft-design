# 归档：交接文档历史（2026-07-09 → 07-11）

> 两份阶段性交接文档，任务均已完成、问题均已解决，内容被后续实现和
> `docs/design_and_implementation.md` 吸收。保留原因：记录了实现期的
> 任务边界约定和一个后来被根治的 bug 的完整取证过程。原始文件：
> `HANDOFF_IMPL_E.md`、`HANDOFF_PERIOD.md`。

## HANDOFF_IMPL_E：方案 E 实现启动交接（2026-07-09）

任务：实现方案 E（`rd_fairness_design_e.md`，EuroSys 投稿核心系统），
按 M0→M5 里程碑推进。硬性规则：不改设计（发现问题记录问用户，不自行
改）；不动 `devlink port function rate`（会 wedge vport，唯一恢复是
OVS del-port+add-port）；TCP shaper 统一叫"host fq+edt"不叫"opt3"；
不删 dpu2 上的三条分类 OpenFlow 规则。预授权：lab 扰动性操作（重启
服务/改链路速率/长实验）无需逐次询问。

已验证的关键操作（当时的现状快照，部分已过时——比如遥测通道后来的
vport 直读方案与此处描述的 OVS 分类规则方案不同）：逐流集合计数依赖
dpu2 underlay-p1 上的三条 OVS 规则（运行时状态，重启即失，新 rx_agent
启动时必须自装）；incast 制造用 `ethtool -s p1 speed 100000`；PCC
构建流程 `meson setup -Denable_pcc=true && ninja -C build`；RP 内窥
`echo "0xdeb <idx>" > /tmp/rp_fifo`。

**已知 bug（记录于此，后续已根治）**：`tx_agent2` 长跑约 4 天后卡死，
mailbox 只更新 pair 3，新 QP 卡在 min-level 地板。2026-07-09 取证发现
其实是两个独立缺陷叠加：① 接收端配置表里 vf1 的 IP 是陈年错值，导致
vf1 的限速值永远查不到、被静默跳过；② 旧代理用 representor 字节计数器
测速率（TCP+RDMA 混在一起），执行面却只有 PCC 能压 RDMA，于是 TCP
一超速控制器就拼命压 RDMA、压穿到最低档。取证过程和教训（测量必须与
执行面同粒度、配置必须由 registry 生成、静默跳过是缺陷的藏身处）已
并入 `docs/ops_notes.md` 家族一。

## HANDOFF_PERIOD：控制周期 1ms 化交接（2026-07-11，内容已被完整取代）

这份文档最初记录的"核心矛盾"（一套参数不能同时适配 RDMA/TCP 两条执行
路径、欠利用 66-84%）在同一天下午被大幅推翻——真凶是 MD 剂量未按周期
缩放（50 倍过量）叠加 RP 状态劣化污染测量，两者都已修复，M1b 在 1ms
下反超 50ms 基线约 10%。完整的诊断过程和最终结论已经并入
`docs/archive/tuning-and-diagnostics-log.md` 的"控制周期 1ms 化"两节，
本文档本身的组件化优化路线（组件一测量平滑、组件二按路径分别标定、
组件三参数重整定、组件四 TCP 重传、组件五合并写节奏）在那次下午的
突破后大部分不再适用——只有"组件一"（rate_window_s 加到 20-30ms）
被实际采纳，其余组件的问题被证明是同一个 MD 剂量 bug 的不同表象，
不需要分别处理。

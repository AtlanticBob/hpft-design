# 归档：hpft-shaper v2 阶段报告（2026-07-03 → 07-05）

> 本文档归档了 DOCA PCC 可行性调研到 v2（RDMA+TCP 双协议 shaper）收官
> 的完整时间线。内容已被 `docs/design_and_implementation.md` 取代——
> 这里的机制、参数、架构描述均为历史状态，不代表当前系统。保留原因：
> 完整记录了"为什么这样设计"的调研证据链，论文 lessons-learned 部分
> 可能需要取材。原始文件：`s0_inventory.md`、`s1_blocker_pcc_fw_mismatch.md`、
> `phase1_pcc_probe_plan.md`、`phase1_report.md`、`phase2_report.md`、
> `v2_report.md`。

## S0 环境盘点（2026-07-03）

固件 32.47.2682，`USER_PROGRAMMABLE_CC=1` 已开启，DOCA 2.9.3008。DPU
Arm 侧 RDMA 设备：`mlx5_0`=p0、`mlx5_1`=p1（PCC 目标设备），host 侧
VF(dpu1vf0..3) 挂在 host PF1、物理口 p1。PCC 参考应用位于
`/opt/mellanox/doca/applications/pcc/`。基线：全 case 196 Gb/s 线速，
延迟 2.40µs。回滚 SOP：`doca_pcc` 进程退出即恢复固件 DCQCN，无持久状态；
异常时最坏手段 `mlxfwreset`（需 maintenance window）。

## S1 阻塞：固件/DOCA 版本错配（2026-07-03）

`doca_pcc` 启动失败，syndrome 0x6d0fd5（`flexio_create_prm_outbox` 失败）。
排查证据链：`USER_PROGRAMMABLE_CC` 确认开启、无权限限制、无进程冲突、
FlexIO 通用 RPC 样例本身能跑——唯独 PCC 专用的 outbox 创建接口不兼容。
根因：固件 32.47.2682 比 DOCA 2.9 用户态新了约 4 个版本，PCC 专用接口
不兼容。决定性证据：DPU 上 2026-05-28 的前人实验（同一固件/DOCA 组合）
曾经验证 PCC 完全可用，说明 5/28 之后有人升级了固件、破坏了 DOCA 2.9
的 PCC。**结论：刷新 DPU BFB 到 DOCA 3.3/3.4 bundle，让固件与用户态
回到同代**——这是唯一干净路径，其余方案（容器、混版本 apt 升级、固件
降级）都有明显缺陷。

## Phase 1 探测计划要点（2026-07-03）

五个探测问题：Q1(go/no-go) PCC 能否控制 host VF 流量；Q2 流标识（能否
区分 src/dst）；Q3 控制路径延迟/频率/精度；Q4 多 QP/多进程共享 cap；
Q0(贯穿) 安全回滚。风险表标出的最大不确定性是"5/28 曾观察到 VF 流量
似乎不受 PCC 控制"，这也是 Phase 1 报告要重点回答的问题。

## Phase 1 报告：**GO**（2026-07-03，刷机到 DOCA 3.4.0112/fw 32.49.1014 后）

- **Q1 VF 覆盖 = GO**：固定 10G 算法把 PF 钳到 9.73G、VF 钳到 9.18G，
  host 侧零改动。5/28 的"VF 不受控"是旧固件栈（PCC 整体损坏）的另一
  表象，新栈不存在这个问题。
- **Q2 流标识**：`flowtag`=源 function 稳定标识，`flow_qpn`=发送端本地
  QPN；dst 不在事件中，需 Arm/host agent 带外维护 `{(src_vnic,qpn)→dst_vnic}`
  映射。
- **Q3 控制路径**：mailbox 同步 RPC ≈13.2ms（单通道 75.3Hz 封顶，与
  handler 内容无关）；cap 精度全部 -1.7%~-3.2%；10Hz 方波跟踪干净。
- **Q4 共享 cap**：cap/N 均分下 1/2/4 进程×1/4/16 QP 聚合误差<0.3%；
  16 槽注册表超订时按比例超发——Phase 2 换成 pair 级令牌桶。

Phase 2 设计要点（从探测直接导出）：生产算法 = 参考 DCQCN +
`rate=min(cc_rate, pair_budget)`；pair 键 = DPA 内 flowtag→src ×
host agent 带外 qpn→dst；controller 用单 mailbox 消息打包 N 个
`{pair_id,cap}` 批量更新以达 100Hz 等效。

**2026-07-04 用户指令修正**：dst_vnic 近似为 dst_ip（一 vNIC 一 IP）；
控制路径换机制——mailbox 延迟太高，改用接收端驱动（PCC RTT
request/response 通道，NP 在响应负载里携带 cap，µs 级、零 host 参与）。

## Phase 2 报告：五条设计目标全部达成（2026-07-04）

统一粒度 {src_vnic,dst_vnic}、租户零改动、发送端 pacing 非丢包、多
QP/进程/flow 共享 cap、活跃流高频改速——全部 ✓，1024 QP 聚合误差
+0.3%，四 pair 并发误差≤1.7%，cap 变更传播 394ms。

关键工程发现：flowtag 是流级 hash（含 dst），同 src VF 到不同 dst 的
flowtag 不同，algo_ctxt 按 flowtag 共享（4 QP→1 ctx），CC 状态必须自建
于全局 pair 表；R 测量必须是发送端 wire 或接收端 per-dst 口径（接收端
goodput 在丢包下与 wire 脱钩）；高流数下积分控制需要慢速小步长（1024QP
+0.3%）；min-level 地板防止启动突发压穿 level 导致 QP 死亡（消息超 RC
重传窗口）；DPA API 上下文白名单极窄（packet handler 里调 trace 会
core dump）。

遗留（按优先级）：真实 CNP 自动触发验证（后续在 `b_dcqcn_cnp_status.md`
追踪，已于 2026-07-13/14 随 ECN/DCQCN 工作解决）；多 src 到同一 dst 的
per-(src,dst) R 测量；控制面进程生产级 systemd 托管；device 代码结构性
重构；TCP shaper v2。

## v2 总收官报告：RDMA 完全达成，TCP 透明性未闭合（2026-07-05）

RDMA 侧（DOCA PCC）五条设计目标全部达成，见上。TCP 侧（TC-BPF EDT
+ sch_fq，opt3 v3 定稿）机制四条达成（cap 精度、pacing 非丢包、共享
cap、高频改速 6-7ms/140Hz），**唯独"租户透明"未达成**——enforcement 在
host VF netdev egress，租户内核可见。opt3 v3 相比之前两版
（opt2：小包旁路漏掉大包 latency 流；opt3 v2：pair 债务超 horizon 丢包、
4 流竞争下塌陷 8G→1.1G）是定稿版本：聚合限速用共享 per-pair EDT 债务
延迟（不丢包），latency 流按 inter-packet gap>40µs 旁路。

统一控制面 `tools/hpft-unified-controller`：TCP 走 in-process 直写 BPF
map（28-38µs），RDMA 走 PCC receiver-cap 文件。

**唯一未闭合缺口：TCP 卸载到 DPU 的透明性**（详细排查过程见
`docs/archive/superseded-designs.md` 的 T3.2 章节）——当前 OVS-offload+
switchdev+fw 32.49 下没有既透明又保持 sender pacing 语义的现成 DPU 挂载
点，真正卸载只剩 DOCA/DPDK 例外路径（独立大工程，后来尝试后确认此路
不通，见上述归档）。近期生产可用形态：RDMA PCC(透明) + TCP host opt3
(pacing，机制完备待透明化) + 统一 controller。

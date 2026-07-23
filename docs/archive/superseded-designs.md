# 归档：被放弃的设计方向

> 四个不同阶段被放弃/替换的设计方案，按时间顺序排列。全部已被
> `docs/design_and_implementation.md` 取代或标记为负结论。保留原因：
> 论文 related-work / design-alternatives 讨论可能需要引用"为什么不这样
> 做"。原始文件：`rd_fairness_design.md`、`p2_receiver_driven_design.md`、
> `tcp_shaper_v2.md`、`t32_dpu_tcp_design.md`。

## 方案 A：Receiver-Driven 跨类公平控制环（2026-07-09，已归档）

EuroSys 投稿最早的设计骨架，经过 16 项决定的 grill 评审，但**实现从未
开始**——被方案 E（现 `design_and_implementation.md` 的前身）取代。核心
思路：接收端 DPU 判别"下行是否饱和"，饱和则直接算 water-filling 份额
下发 per-sender Limit 租约；未饱和则看逐路径拥塞证据（CE 比例/RTT），
用 Probe（γ 折扣短租约）试探后迭代收敛，非拥塞则计数器衰减、租约到期
斜坡释放。Advice 协议 `{Op, tenant, class, dst, rate, τ}`，`Op∈{Probe,
Limit, Release}`，租约 soft-state 续租、到期斜坡回升消除极限环。

被取代的原因：方案 A 的"侦探臂"（Probe/计数器机制）是为"接收端看不见
竞争者全集"的场景设计的，但 scope 对齐后这个场景不在论文范围内——
方案 E 走的是虚拟队列差分标记这条更简单、构造性免疫崩塌的路线（见
主设计文档 §2/§7.1）。方案 A 里仍然存活的部分（层级 water-filling、
per-sender 拆分、瓶颈属主原则、min 不变式）都被 E 继承，只是实现方式
完全不同（虚拟队列标记 vs 显式 Advice 租约协议）。文献锚点保留供参考：
EyeQ (NSDI'13)、Gatekeeper、PicNIC (SIGCOMM'19)、ElasticSwitch
(SIGCOMM'13)、FairCloud (SIGCOMM'12)、BwE (SIGCOMM'15)、Seawall
(NSDI'11)、Justitia (NSDI'22)、Swift (SIGCOMM'20)、IRN (SIGCOMM'18)、
DCQCN/DCTCP、HPCC (SIGCOMM'19)、Annulus (SIGCOMM'20)。

## P2-3：接收端驱动的 cap 与 RX 测量回传（2026-07-04，已被水填充+VQ 取代）

早期方案：RP 周期性对活跃流置 `rtt_req=1`，固件发 CCMAD RTT 探测包到
接收端，接收端 NP 上下文解析 IP 头得到 `{src_ip,dst_ip}` pair 身份，把
`{cap, rx_rate}` 写进响应负载带回发送端 RP。延迟预算：agent→NP 表
(mailbox~13ms) → 下一个 RTT 响应(探测周期~1ms) → RP 生效(µs)，总约
15ms，且可绕开发送端 mailbox 的 75Hz 上限。

被取代原因：这套"cap 通过 RTT 探测包回传"的机制被后来的方案 E 替换为
虚拟队列标记 + 独立遥测通道（DPU 间带内路径，稳态往返 0.1ms，见主设计
文档 §6.4）——不再依赖 CCMAD RTT 通道传递策略信息，机制更简单、不占用
RTT 探测本身的语义。

## TCP shaper v2 设计（2026-07-04，机制被"host fq+edt"吸收）

设计目标与 RDMA v2 相同：租户透明、发送端 pacing（非丢包）、粒度
`{src_vnic,dst_vnic}`、共享 cap、活跃流高频改速。核心判断：RDMA 有 DOCA
PCC 这个 VF 边界以下的硬件抓手，**TCP 没有等价物**——TCP 的拥塞/发送
状态在租户 kernel 里，NIC 上没有可编程的"TCP 发送调度器"。因此列出两条
路：① host kernel TC-BPF EDT（本阶段验证的路线，对 VM-passthrough 租户
透明，机制已验证：8G cap 精确、动态改速跟随、fail-open 正确）；② DPU
用户态数据面 EDT（设计但未实现，需要 OVS-DOCA/DPDK，真正租户透明）。

本文档验证的 host TC-BPF EDT 机制后来演进为 opt3（见
`v2-shaper-phase-reports.md`），最终成为现在系统里的"host fq+edt"
执行器（`tcp/bpf-opt3/`）——机制本身被继承，本文档作为历史设计记录。

## T3.2：TCP shaping 卸载到 DPU 用户态（2026-07-05/06，已暂停/负结论）

目标：把 TCP enforcement 从 host 内核移到 DPU，达到 RDMA/PCC 那样的完全
租户透明。**执行摘要（原文 19K 字的详细排查记录已精简，完整过程见 git
历史）**：

逐一排除了简单路线——representor ingress mirred/ifb（OVS flower 占用
ingress）、uplink egress tbf/fq+EDT（switchdev 下硬件搬运绕过内核
qdisc，实测挂 tbf 3gbit 无效）、devlink function rate（fw 32.49 损坏，
会 wedge vport）、DOCA Flow meter（只有 policer 没有 delay-shaper）。
唯一候选：**DOCA Flow RSS 例外路径 + DPDK 软件 EDT pacing**——只把目标
VF 的 TCP steer 到 Arm DPDK 队列做软件 pacing，其余流量不受影响。

实现路径本身被验证可行到 D1c（自定义 DOCA Flow app、CPU RX/TX 转发、
与 OVS/RDMA 共存均已用工作代码证明），但撞上一堵**架构墙**：要在共存
模式（`fdb_def_rule_en=1`，保留 OVS 默认转发）下工作，root pipe 只能
挂在浅层 parser_meta 类型寄存器上，深层包头字段（IP 地址、端口、源
vport）在这个挂载点完全不可用（决定性验证：deep-match 在 root 和子
pipe 均 0 命中）——这意味着**没法在这个模式下区分流量方向**
（host→net vs net→host），而方向判别是"只 punt 发送方向、不干扰接收
方向"的必要条件。唯一出路是 `fdb_def_rule_en=0`（DOCA 完全接管 FDB），
但这意味着要自己重写 OVS 的全部转发逻辑（RoCE/ARP/其它 VF/net→host），
风险覆盖整机所有 VF 网络，等价于一次大重写。

**结论：暂停。** host opt3 已是完整可用的 TCP shaper（性能全部达标，
唯独不透明）；DPU 侧透明化的基础设施摩擦已被证明很大，值不值得继续
投入是待用户裁决的问题，本项目现阶段维持 host fq+edt 现状。

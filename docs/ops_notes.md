# 操作规程与平台硬约束

动 lab 之前读这一份。**它只写现在还成立的规则**——每一条背后都有一次
"每一层都报健康、数据却是错的"的事故，但事故经过不在这里，在 git 里。

按需要它的时机组织：开跑之前 → 改东西的时候 → 平台的硬约束 → 症状速查。
系统本身怎么工作看 `design.md`，lab 长什么样和怎么跑一场实验看
`hpft-implementation/README.md`。

## 开跑之前

**`deploy_check.sh` 必须通过。** 四台 DPU 与 host 侧载荷和 repo 一致、
启用的 agent 健康无 traceback、流量生成器同版本。不过就不要跑——一个崩掉的
发送端 agent 会让 RDMA 完全不受限，而其余每一层都报正常。

**每场实验重启一次 RP。** 执行面会跨实验失去限速能力：预算照收、`0xdeb`
回读的 level 照样正确，就是不作用到线上。同一家族还有第二种形态——设备侧把
新预算搁置约 12 秒才一步应用（3 遍中 1 遍复现），线上钉在旧 level 上，而 rx
的裁定与 tx 的 pace 三层证据都正确。`roles.sh set` / `incast8_regression.sh` /
`cc_mode.sh pcc` 都已内建；手工起流之前自己补一次。涉及在线改预算的实验，
对 >10s 的收敛长尾先怀疑陈旧应用形态，别记成控制环缺陷。

**多对并发 RDMA 的聚合需求必须从第一个包起就低于瓶颈容量。** 8 条无限速
RDMA 每条 ~27G、合计 216G 灌进 100G 下行，RC 重传风暴会打死其中几个 QP，
而**RC 的错误完成是终态**——QP 不会自己回来，哪几条死由启动顺序决定，看起来
像"某些流对坏了"。做法：给 perftest 加 `--rate_limit`（硬件档位，只接受
2.5/5/10 这类离散值），或者让被测系统在流量起来之前就把限速下发下去。
**不要指望控制环"一两个周期内收敛"兜住启动瞬态**：那两个周期里 QP 已经死了。
HPFT 的新流集合以 $R=Tree_f$ 乐观起步，$N$ 条同时起步聚合是 $N\times Tree_f$，
所以多对 RDMA 的 runner 要么错开启动、要么加 `--rate_limit`。

**不要一步大幅降速。** 单对 RDMA 在 cap 20G→1G 的**一步 20 倍**政策降幅下，
3 次里有 1 次线上归零且不再恢复。危险的是**降幅本身**而非终点速率——稳态低速率
不危险（cap 3G/2G/1G 各跑 25s，实现率 87–91%，无死亡），PCC 水位控制也不是元凶
（线上归零时控制器一直在往上救）。控制律自身不会产生这种阶跃（目标以 1/k=50ms
的时间常数移动），只有**人为的大幅政策变更**才会。做这类实验分两三步降，或者
接受偶发 QP 死亡并在分析时识别出来（症状：线上恒为 0 而预算与水位正常）。

**preflight 一遍。** `flow_preflight.sh "<对>" <发送端> <接收端>` 逐对验活，
每个发送端各跑一次。空载 ping 只判连通性，**不作为开跑门槛**——冷启动首包与
变速暂态的 RTT 和满载稳态无关。

**暂停浸泡流量的 cron**（`soak_traffic` 那一行），否则每 15 分钟一轮的背景
流量会污染测量；实验后恢复。

**一次只跑一场。** 两场重叠时后一场的角色重启会打断前一场，而前一场仍会写出
一整套看起来完整的产物。`incast8_regression.sh` 用 `/tmp/hpft_run.lock` 互斥。

## 改东西的时候

**`config/lab-registry.json` 的 repo 副本是唯一真值。** 一切修改先改 repo 再
下发；实验里临时 ssh 直改必须在收尾同步回 repo。曾因一次反向覆盖丢掉 6G 政策，
表现为"cap 不生效"的假故障。

**改 cap 必须两端一起改。** 发送端树 $Tree_f=\min(MaxRate_{src},\text{需求})$，
只抬接收端的 MaxRate 不生效——流被发送端的 MaxRate 锁死。

**"杀 tx_agent" 不等于"解除限速"。** RP 在设备内存里保留每个 flowtag 上一次的
预算，tx 停了它继续按旧预算 pacing。真要解除：先停 tx，再
`echo "0xb47c0001 <flowtag> 0 0" > rp_fifo`（budget=0 删除 pair），`0xdeb`
确认 bud=0。**对偶事实**：重启 RP 会清空全部流对预算，所以凡依赖"停 tx 冻结
预算"的编排，必须先让 tx 对全部目标流集合推过预算再停。

**PCC 设备码改完必须 `meson setup --reconfigure build` 再 `ninja`**（dpacc 在
configure 步），**每台 DPU 都要**。只改 host 侧（`pcc/host/pcc.c`，仓库副本名
`pcc_host.c`，改完 scp 成 `pcc.c`）时 `ninja` 即可。DPU 上的活树是
`~/bzx/doca34-apps`。

**重新 apply TCP 执行面之前先 `rm -rf /sys/fs/bpf/hpft_tcp_edt`**：pin 还在时
`bpftool prog load` 失败，而 `tcp-shaper-apply` 只把失败记进结果、tc 仍指着旧
程序，表现是"部署完了却没生效"。

**改 CC / 重传 / ECN 开关的 runner 一律 `trap EXIT` 复原。** 这类状态泄漏到
后续批次后，症状极易误诊——例如"关 CC"泄漏后 NP 装聋（np_cnp=0）、配额被穿透，
看起来像 CE 在 decap 路径丢失。跨批次第一场实验前把 `dcqcn.sh status` 的
enable 位列入核对清单。

**不要在实验运行期间改它的脚本。** bash 边读边执行，改文件会让运行中的实例读到
写了一半的内容。

**`pkill -f` 与目标进程的启动永远分开两条 ssh。** `pkill -f <pattern>` 按整条
命令行匹配，写在同一条 ssh 命令里时会匹配到承载它自己的那个 shell（里面就有目标
程序的完整调用文本）并自杀。表现为 ssh 返回 255，或者 server 从未起来而实验静默
连续失败。`[x]` 字符类技巧只在同一命令行不含目标真实调用文本时才有效。

**VF 重建之后跟一次 `cross_pair_net.sh apply`（两端）。** arp_ignore /
rp_filter 是 per-netdev 状态，随 VF 一起消失；缺失时四个 VF 抢答彼此的 ARP，
数据面塌缩到单 VF，rx 归因忠实报出全部流集落在 vf0。

## 平台硬约束

**VF MTU 必须是 1500。** MTU=8192 时 RoCE path MTU 协商成 4096，4KB 数据帧过不了
VxLAN encap 路径：QP 建连成功、数据 WQE 全部 CQE flush error、字节不出 VF vport，
而同路径 ICMP/TCP 正常。遇到"建连成功但零吞吐 + CQE error"先查 MTU。

**`options:tos=inherit` 是硬性配置。** OVS VxLAN 默认 tos=0，不复制内层 DSCP，
`--tclass` 流量会全落交换机 TC0，类分队列无法成立。ECN 折回在两种设置下都完好
（RFC 6040 由硬件 decap 执行）。

**mlx5 流计数器缓存 1 秒。** TC 卸载流的硬件字节计数器由驱动每秒批量轮询一次
（5.15 内核硬编码，无旋钮），所以毫秒级控制周期不能直接用 megaflow 计数器测速率。
解法是混合测量：vport 计数器（任意频率都新鲜）供每 VF 总量，megaflow 供类构成比。
同源的残余：发送方停发后它的陈旧字节还能占住份额约 1 秒——这由发送端的活性回传
（1ms 新鲜的本端 vport TX 计数，UDP 9713）作为**门控**消除，比例仍由接收端自己测。

**p1 的软件 netdev 计数器在硬件卸载下不动**（rx_bytes 恒 0），外层字节一律读
`ethtool -S p1` 的 `rx_bytes_phy`。

**VxLAN 封装税实测 4.99%**（65536B 消息、MTU 1500）。账本与 pace 全链是内层口径
且自洽，外层税只在 root 容量层面由 headroom 吸收——**不要给 TCP shaper 单独加
外层补偿**，那会重造 TCP↔RDMA 不对称（RDMA 硬件调速计不了外层）。

**多 VF 并发 TCP 一律 `%dev` 强绑定**（`-B 10.1.N.x%dpu1vfN`）。只绑源 IP 时
TCP 会间歇性走错 VF egress，接收端 per-VF 计数器错记，类拆分把某 VF 的 TCP 记成
0，遥测"断流"触发 fail-open，TCP 冲到线速。

**fw reset 杀死 DEVX 上下文但不杀进程。** `vport_meter` 之类持有设备上下文的常驻
进程会变成僵尸：进程还在、systemd 显示 active、共享内存永远停更——下游测不到速率，
预算全部 fail-open 成 MaxRate，限速名存实亡，而流量图看起来还算合理。**判活不能看
进程或 unit 状态，也不能看 mmap 文件 mtime**（mmap 写不更新 mtime），要跑一条流看
rx_agent jsonl 的 `r` 字段是否非空。规则：任何 fw reset 之后，所有持有设备上下文
的常驻进程一律主动重启。

**需要 SSH 的常驻进程必须带登录用户环境。** `qpn_resolver` 这类要发起 SSH 的组件
放进 systemd 裸环境会因为没有 `HOME`/`.ssh` 直接退出。只做本地 ethtool/UDP/FIFO
的进程无所谓。反复重启还会撞 TIME_WAIT 导致 bind 失败——`SO_REUSEADDR` 加重启前
`systemctl reset-failed`。

**Arm 重启 / fw reset 清掉的易失状态**由 `reboot_recover.sh` 统一恢复：VF、TCP
EDT 与其辅助 map、p1 的速率与 MTU 与 underlay/遥测 IP、ARP 全互联、PFC。hugepages
已用 `/etc/sysctl.d/99-hpft-hugepages.conf` 持久化（DPDK 组件需要 ≥4GB）。
ROCE_ACCL 寄存器（SR 开关）是易失的，fw reset 后归零，`cc_mode.sh` 会记住并复原。

**lossy 前提。** 两端 p1 的 PFC 全关、802.3x 全局 pause 关闭，全路径不依赖交换机
无损，丢包由 RC 重传兜底。

**带内遥测走数据路径。** 两端 PF1 侧的 SF（`enp3s0f1s0`）挂 underlay、配 10.1.9.x，
遥测与被控数据同走 p1 与交换机（稳态往返 0.1ms），不依赖专用管理网。p1 改线速时
链路闪断约 10 秒，遥测随之中断，fail-open 按设计接管。`ip link set <SF> down/up`
会抹掉其上的 IP，OVS 重启会抹掉 rx_agent 的分类 OpenFlow 规则——两者都由
`rx_agent` 与恢复链幂等重装。

**devlink 限速可用**（fw 32.49.1014 实测：带载设置、翻转、解除共 14 次全部干净，
10G 档实测 9.78G）。它是 VM 级 MaxRate 的硬件兜底。

## perftest 的坑

- `-D` 两端必须一致，不一致报 "Failed to negotiate parameters"。
- `-D` 模式下结果表的 iterations 列不可靠（约为真实消息数的一半），总量用
  BW average × 时长算。
- `--rate_limit` 是硬件档位制，传非档位值直接报错退出——这个失败很安静。
- **交叉对必须走非 CM 路径**（`-x <gid>`，不能用 `-R`）：用 rdma_cm 时内核按目的
  子网先选路，源 VF 是拥有那个子网的那个，`-d` 说了不算，流量实际是直连对。

## PCC 执行面

**限速与拥塞控制是两件事。** `level` 是控制律的执行（budget/N）；`cc_rate` 是租户
自己的 CC。执行面**只读 cc_rate、不改它**：把它的下降累积成一个相对政策份额的偏离
$d$，自己把 $d$ 收回 1，下发 $rate=d\cdot level$。被自己的整形卡住时（实发 ≈ 已
下发）那次降速不计入。取小值的旧组合 `min(cc_rate, level)` 作为对照臂保留
（`0xcca 0`）。

**`level = budget/N` 里的 N 来自 `qpn_resolver`。** 它不在位时 RP 对不上 CNP 与
流对（命中率 ~0.3%），而且多 QP 流对的 N=1，**每个 QP 都拿整份预算**——n 个 QP 的
流会跑到 n 倍份额，而账本、律、邮箱三层读数全部正确。它是发送端必备组件。

**cc_rate 三选一**，mailbox `0xccd <0|1|2>` 切换（切换会重置各流对 CC 状态）：

- **0 = AIMD**：CNP 触发固定比例乘性减、每 epoch 固定步长加性增。**它不是 DCQCN**
  ——没有 α、没有目标速率、没有三阶段恢复，文档与论文里必须叫它 AIMD。
- **2 = 软件 DCQCN**：`Rt=Rc`、`Rc=Rc(1-α/2)`、`α+=g(1-α)`，恢复定时器驱动
  快恢复 → 加性增 → 超增。三个旋钮经 `0xcce <值> <0=AI|1=HAI|2=恢复定时器us>` 调。
  定时器在 epoch（1ms）边界检查，实际周期是 `max(epoch, 设定值)`，可调但不细于
  epoch。**新分配的流对必须置 `Rt=MAX、α=1`**，否则 `Rc=(Rt+Rc)/2` 会每拍折半
  直至归零。
- **1 = ZTR-RTTCC**：厂商 RTT 基算法。阈值必须用本 fabric 实测值——设备时钟下
  base RTT min=3006ns、26G 负载无拥塞抖动带 3.3–5.7µs，故 `ZTR_BASE_RTT=7000`、
  `ZTR_MAX_DELAY=70000`（厂商模板默认的 13µs/150µs 是别人 fabric 的数）。

**RTT 通路成立的三个条件**，缺一则 `rtt_req` 石沉大海、`rtt_events` 恒 0 而 CNP
一切正常（极易误诊）：

1. `USER_PROGRAMMABLE_CC=1` 且 **`PCC_INT_EN=0`**（`PCC_INT_*` 是交换机 INT 遥测
   那套开关，开了反而封死 CCMAD 应答）。
2. RP 启动**不带 `--remote-sw-handler`**——带上它等于声明"应答由对端 NP 软件处理"，
   对端没有 NP 进程时探针被静默丢弃；不带时由对端网卡固件硬件应答。
3. 有正常数据流在跑（探针从流对的事件路径发起）。

验证：`0xdec` 的 w9=rtt_req_n 与 w10=rtt_n 应同步增长（实测应答率 99.98%）。
overlay 封装不妨碍 CCMAD，固件流量走端口层，**tcpdump 抓不到不代表没发**。

**已知的收敛长尾：RDMA 执行面的上升爬坡。** 发送端 pace 在 30ms 内就到位，
线上速率却按约 12.5%/步 往上爬——RP 把任何预算变更都当作"上限改变"并重置整定窗口。
测同 dst 的加入/退出收敛时，归属侧已降到 0.24–0.49s，**剩余的约 2–2.8s 全在这段
爬坡**，不要记成控制环或账本的问题。根治要改 DPA 设备码的水位赋值逻辑，未做。

**诊断邮箱**：`0xdeb <pair>` 回读 {flowtag, budget, level, remote_rx, r_units,
cc_rate, epochs, cap, α, Rt, stage}；`0xdec <pair>` 回读 {cnp_any, cnp_hits,
qp_count, cc_rate, dbg_hits, flowtag, rtt_last, rtt_min, rtt_events, rtt_req_n,
rtt_n}。**tx agent 在跑时查询不可靠**——它每 13ms 往同一个 FIFO 写批量预算，查询
会被交错吞掉；要读干净状态就先停 tx，或看它自己的日志。

## 症状速查

| 症状 | 先查 |
|---|---|
| 授权对、线上不对（预算照收、level 照读） | RP 陈旧 → 重启 RP |
| 收敛长尾 >10s，三层读数都正确 | 预算陈旧应用形态 |
| n 个 QP 的流跑到 n 倍份额 | `qpn_resolver` 是否在位 |
| RoCE 建连成功但零吞吐 + CQE error | VF MTU 是不是 1500 |
| 某几对 RDMA 报 Completion with error 且永不恢复 | 启动瞬态聚合过载，加 `--rate_limit` |
| 跨主机数据面完全不通而 cc/agents 全对 | `ovs-vsctl list-ports ovsbr-p1` 有无残留；p1 在不在桥上 |
| TCP 不受限而律/shim/map 全对 | EDT 辅助 map 是否为空（`bpftool map dump pinned .../hpft_pair_state`） |
| 限速名存实亡而流量图貌似合理 | `vport_meter` 僵尸（跑一条流看 rx jsonl 的 `r`） |
| 类分队列不成立、流量全在 TC0 | `tos=inherit` |
| 交叉对实验的归属全落在直连对上 | 用了 `-R`，改 `-x <gid>` |
| 某个 VF 抢答别人的 ARP、流集全落 vf0 | `cross_pair_net.sh apply` 没做 |

## Jakiro DHTB（对照环境）

1. **冷启动概率性不转发**：同一 conf 同一序列，有的实例正常、有的把 TCP/RoCE 全
   黑洞而 ICMP 照过（ICMP 不在它的分类器里）。**ping 判活无效**，唯一可信的就绪门
   是起 server 后打一条真 TCP 流过树看吞吐。
2. 因此**不要每 run 重启 DHTB**——重启一次就重掷一次硬币。现行做法是单实例战役：
   起一次、TCP 探测验证转发后连跑全部 run。
3. 探针复用 iperf3 server 会踩"单 test 服务"陷阱（被 timeout 掐死的客户端让 server
   卡住，其后探测全零）——每次探测前 kill 重起。
4. Arm 重启后 hugepages 需 ≥4GB（2048×2MB），否则先死在 EAL、补 2GB 仍死在 mbuf 池。

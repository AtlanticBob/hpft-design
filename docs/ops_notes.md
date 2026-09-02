# 操作规程与平台硬约束

动 lab 之前读这一份。**它只写现在还成立的规则**——每一条背后都有一次
"每一层都报健康、数据却是错的"的事故，但事故经过不在这里，在 git 里。

按需要它的时机组织：开跑之前 → 改东西的时候 → 平台的硬约束 → 症状速查。
系统本身怎么工作看 `design_v4.md`（为什么稳、换规模怎么调参看 `design_theory_v4.md`），lab 长什么样和怎么跑一场实验看
`hpft-implementation/README.md`。

## 开跑之前

**`deploy_check.sh` 必须通过。** 四台 DPU 与 host 侧载荷和 repo 一致、
启用的 agent 健康无 traceback、流量生成器同版本。不过就不要跑——一个崩掉的
发送端 agent 会让 RDMA 完全不受限，而其余每一层都报正常。

**每场实验重启一次 RP。** 执行面会跨实验失去限速能力：预算照收、`0xdeb`
回读的 level 照样正确，就是不作用到线上。同一家族还有第二种形态——设备侧把
新预算搁置约 12 秒才一步应用（3 遍中 1 遍复现），线上钉在旧 level 上，而 rx
的裁定与 tx 的 pace 三层证据都正确。`roles.sh set` / `validation/run/run.sh` /
`cc_mode.sh pcc` 都已内建；手工起流之前自己补一次。涉及在线改预算的实验，
对 >10s 的收敛长尾先怀疑陈旧应用形态，别记成控制环缺陷。

**多对并发 RDMA 的聚合需求必须从第一个包起就低于瓶颈容量。** 8 条无限速
RDMA 每条 ~27G、合计 216G 灌进 100G 下行，RC 重传风暴会打死其中几个 QP，
而**RC 的错误完成是终态**——QP 不会自己回来，哪几条死由启动顺序决定，看起来
像"某些流对坏了"。做法：给 perftest 加 `--rate_limit`（硬件档位，只接受
2.5/5/10 这类离散值），或者让被测系统在流量起来之前就把限速下发下去。
**不要指望控制环"一两个周期内收敛"兜住启动瞬态**：那两个周期里 QP 已经死了。
设计 v4 起，新流集合的**围栏**从地板起步（不再是 $Tree_f$），但真正决定启动
瞬态的是**执行面给未知流的额度**：RDMA 执行面在第一份预算到达之前按每 QP
`rdma_unknown_rate_bps`（现 5 G）放行，所以一个 4 QP 的流集合起步就是 20 G，
$N$ 条同时起步聚合是 $N$ 倍。多对 RDMA 的 runner 要么错开启动、要么加
`--rate_limit`。

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
一整套看起来完整的产物。`validation/run/run.sh` 用 `/tmp/hpft_run.lock` 互斥。

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
10G 档实测 9.78G）。它是 VM 级 MaxRate 的硬件兜底。 **入参是 bit/s，JSON 读回是
byte/s**，差 8 倍——两边单位不同这件事没有任何报错，`hw_maxrate_test.py` 专门守它。
**它只有 `tx_share`/`tx_max`，没有任何接收方向的参数**，所以接收端的下行上限做不成
devlink，只能是 OVS meter（见下）。

**接收端的 OVS meter 会周期性地把新流发现拖慢五倍。** 每 VF 50 G 的下行上限用的是
OVS drop meter，而这套 OVS（`doca-openvswitch 3.4.0040`，内核数据面 + `hw-offload`）
在卸载带 police 动作的流时会持续报 `tc|ERR|Failed to parse police action options`，
有流量时约每秒 19–20 条。三对三对照：meter 在时接收端接纳新流集合要 1111/1101/229 ms，
撤掉 meter 后收敛到 257/153/268 ms，police 错误 60/60/60 → 0/0/0。它不是让接纳整体
变慢，而是**让它偶发地卡住四到五倍**——同一配置下 153 ms 到 1169 ms 的巨大散布就是
这么来的。已试且无效的两条：显式调 meter 的 burst（`3063225600b` → `93132000b`，
错误数不变），换 OVS 版本（仓库里没有别的版本）。撤 meter 不是方案（那个上限有用途）。
真要根治得把下行限速挪出 DPU 的 OVS。**注意 `dpif_flow_put_error` 约 43% 的安装失败
率不是 meter 造成的**（撤掉后仍在），那是另一个没查的问题。

**RP 邮箱有快慢两态。** `doca_pcc_mailbox_send` 的耗时在 **13.3 ms 与 21.9 ms** 之间
切换，每台 DPU 约三分之一时间在快态，各台周期不同且相位不同，跨 doca_pcc 重启存活；
切换时机器的负载、中断率、上下文切换、温度都没有阶跃，原因未查明。已验证**对稳态没有
可测影响**，所以不必为它排队等窗口；`validation/run/run.sh` 把每次运行各发送端的实测
值记进 `results/<tag>/mailbox_mode.txt`，比较收敛时按模式分组即可。

## 造隐形瓶颈与量丢包（2026-09-02 探路）

要验证置信度就得造一个"接收端账本管不到的瓶颈"。现有星型拓扑里唯一会拥塞的链路就是接收端口，而账本建模的正是它，所以自然的隐形瓶颈不存在，只能人工造。把接收 VF 的 OVS 丢包 meter 调低、注册表不改，是最省事的一种。探路量到的四件事：

- **接收端 vport 计数器读的是 meter 丢包之后的量。** meter 压到 8 G 时 vport 读 8.10 G，meter 50 G 时读 19.36 G。也就是说账本只看得见活下来的那部分，看不见被丢掉的，它会把这条流集合当成出借方。
- **OVS meter 的 band 丢包字节计数在 hw-offload 下恒为 0**，哪怕限速明显生效。别用 `meter-stats` 量丢包，用 NAK 计数或速率差。
- **健康路径上没有连续的背景丢包。** `packet_seq_err` 只在建链那一秒爆一次（约 1300），之后二十秒严格为 0；隐形瓶颈打开后是每秒 8.5 万到 9.3 万，持续整段。所以"最近 100 ms 有没有丢包"这个证据是干净的，不会被背景噪声长期顶开。
- **崩溃是真的。** 19.4 G 灌进 8 G 的口子，应用 goodput 掉到 4.9–5.2 G——**比这个口子本身能过的还少**，而接收端账本全程报队列 0、入场额 25 G。这正是置信度要治的病，二十秒就能复现。

**不要拿没接好的 VF 做实验。** 在 vf5 上试的时候，接收端算出了 e = 25 G，但发送端 agent 的 Ehat = 0、围栏停在 2 G 地板、预算 0，RDMA 执行面里根本没有这个 pair 的条目，于是四条 QP 走"未知流"兜底额度（每 QP 5 G）跑到 19.36 G——**实际速率是围栏的 9 倍，围栏一点没生效**。原因还没查（可能只是这个 VF 对没进管理集合），但结论是实验必须用 vf0–vf3 这种在场景里跑熟的 VF，并且走 `validation/run/run.sh` 的完整装配，别手工起流。

## 执行面的 CC 项与丢包

**四个 CC 项现在都会因丢包降速**（`0xccd`：0 = AIMD、1 = ZTR、2 = DCQCN RP 状态机、3 = Swift；启动路径里没人写 `0xccd`，缺省是编译期默认值 **2**）。ZTR 和 Swift 在自己的 RTT 事件里处理；AIMD 与 DCQCN 走 2026-09-02 加的 **slow restart**：NACK 事件里乘性压低 `Rc`、恢复阶梯清零、重置速率定时器，**不碰 alpha**（alpha 估计的是标记强度，丢包不是标记）。

这么组织是照着硬件来的：固件的丢包反应在 ROCE_ACCL 寄存器的 `roce_slow_restart_en`（本 lab 四台都是 1），与 DCQCN 并列；`ecn/roce_rp/` 下十七个 DCQCN 旋钮无一与丢包有关。所以丢包反应加在状态机旁边，不加在里面。

两个参数：`0xcce <fxp16> 8` 是切幅（默认 32768 = 0.5，取自固件的 `rpg_min_dec_fac = 50`），`0xcce <us> 9` 是两次切之间的最小间隔（默认 300 µs，与 `rpg_time_reset` 同量级）。**节流不能去掉**：隐形瓶颈下 NACK 能到每秒 1.9 万次，每次都切会把速率瞬间钉在地板。切幅设 65536 就是关掉。诊断计数 `dq_n_loss` 在 `0xded` 回读的第 9 个字，`rp_probe.py` 吐成 `sr_cuts`。

**这两个参数还没有对着固件标定过**，初值是从固件旋钮借的。要做到软件像固件，得把 lab 切到 UPCC=0 跑同一个隐形瓶颈场景量固件的 goodput / NACK 率 / 恢复形状，再回来调。切 UPCC 要 `mlxfwreset`，而 fw reset 会清掉 ROCE_ACCL 的 SR 位，回来要 `cc_mode.sh sr` 重设。

## perftest 的坑

- **2026-09-02 起打流器统一是 `~/hyperfront/perftest-enhanced`**，四台同一份二进制。
  它是原封的 perftest 6.28 加三样东西：每 QP 带宽时序（`--report-per-qp`）、大 QP 数
  下的并行建链（`--setup-threads`）、准点起步（`--start_at`）。`perftest-26015` 已停用，
  只作已跑完的 motivation 包的历史记录。
- **`--rate_limit` 一定要确认软件限速真的接管了。** 在 RoCE 上硬件限速必然被 QP 拒绝
  （PCC 执行面占着 QP 的速率），perftest 于是退回软件限速并在 stderr 上打
  "providing SW rate limit"。**看不到这行就是没限住**——限速值会被完全忽略，流照着
  线速发。
- `-D` 两端必须一致，不一致报 "Failed to negotiate parameters"。
- `-D` 模式下结果表的 iterations 列不可靠（约为真实消息数的一半），总量用
  BW average × 时长算。
- `--rate_limit` 是硬件档位制，传非档位值直接报错退出——这个失败很安静。
- **交叉对必须走非 CM 路径**（`-x <gid>`，不能用 `-R`）：用 rdma_cm 时内核按目的
  子网先选路，源 VF 是拥有那个子网的那个，`-d` 说了不算，流量实际是直连对。

## iperf3 的坑

- **`--start-at <unix time>` 存在，但不在 `--help` 里。** 这是本地补丁（源码在
  `~/hyperfront/iperf320`，四台已装 `/usr/local/lib/libiperf.so.0`），只加了选项与
  解析、没加帮助文本。**别用 `--help` 判断它在不在**——查
  `strings /usr/local/lib/libiperf.so.0 | grep start-at`。没有它，TCP 的首字节比名义
  时刻晚 0.5–1.1 s（建链加参数交换），而收敛是从名义时刻起算的，这一秒会被记在系统
  头上。选项解析在**库**里，所以只换 `iperf3` 二进制不换 `libiperf` 是无效的。
- 多 VF 并发一律 `%dev` 强绑定，见上面"平台硬约束"那条。

## PCC 执行面

**限速与拥塞控制是两件事。** `level` 是控制律的执行（budget/N）；`cc_rate` 是租户
自己的 CC。执行面**只读 cc_rate、不改它**。设计 v4 起下发的是**限幅**：
$rate=\operatorname{clip}(cc,\ (1-T)\cdot level,\ level)$——上界恒等于 level（与
置信度无关），下界随执行面自有的置信度 $T$ 放开。落在区间内时 $cc$ 一字不改，
所以行为正常的 CC 看不见我们。设备端由 `0xb47d` 批量格式启用（每条目多带一个
置信度字），走 `trust_mode` 分支。

**置信度靠丢包维持、停了就自动退**（design v4 §6.2/6.3，2026-09-02 起）：上升
三条件（队列为零、`cc_rate` 低于 level 的 $1-\delta$（$\delta$ = 15%，与注册表 `delta_demand` 同值）、最近 100 ms 有 NACK）成立的每个
1 ms epoch 涨 $(1-T)\cdot$步长，不成立的每个 epoch 按 $T\cdot\max(q/D,\ \text{过期步长})$
降。两个步长都是 fxp16/epoch 的邮箱旋钮：`0xcce <步长> 7` 是上升（默认 66 =
$\tau_r$ 1 s；0 = 置信度冻结在零，V8 臂 B），`0xcce <步长> 10` 是过期（默认 13 =
$\tau_d$ 5 s；0 = 关掉过期项，即 2026-09-02 之前只有队列一条下降通路的旧规则，
V9 臂 B）。过期项必须在的原因：瓶颈消失时网络不给任何新信号，而置信度高的流又
永远不会多拿（上界恒为 level），没有过期项置信度就只有上去的路、没有下来的路——
实测旧规则下瓶颈撤掉后 $T$ 十几分钟才消退（时间常数约 278 s），这期间下界保护
一直是关着的。TCP 执行面（`hpft_tcp_edt_kern.c`）是同一套规则的编译期常量版：
`HPFT_TRUST_STEP` 66、`HPFT_TRUST_DECAY` 13、`HPFT_TRUST_BELOW_FXP16` 55706、空闲
超过 `HPFT_TRUST_IDLE_NS` 5 s 从零起（design v4 §6.4）；改它要重编 .o 并按
"EDT 重装"步骤在四台 host 上重装。

两条对照臂仍在设备码里：`0xcca 1` 是旧的观测耦合 $rate=d\cdot level$，`0xcca 0`
是基线 `min(cc_rate, level)`。**注意设备码里 `trust_mode` 字段旁的注释写着
`rate = T*cc + (1-T)*level`，那是加权平均，不是现在的实现**——加权平均会向上漏
（置信度 12% 就能漏出 24 G，实测），设计 v4 明确否掉了它。

**`level = budget/N` 里的 N 来自 `qpn_resolver`。** 它不在位时 RP 对不上 CNP 与
流对（命中率 ~0.3%），而且多 QP 流对的 N=1，**每个 QP 都拿整份预算**——n 个 QP 的
流会跑到 n 倍份额，而账本、律、邮箱三层读数全部正确。它是发送端必备组件。

**cc_rate 四选一**，mailbox `0xccd <0|1|2|3>` 切换（切换会重置各流对 CC 状态）：

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
- **3 = Swift**：时延基，阈值同样要本 fabric 校准（base 6µs / fs_range 20µs，
  校准见 `results/swift_20260827/summary.md`）。它是 V6"换一种 CC"场景的被测臂；
  motivation 用的是独立的 `pcc_swift_stock`，不是这一项。

**RTT 通路成立的三个条件**，缺一则 `rtt_req` 石沉大海、`rtt_events` 恒 0 而 CNP
一切正常（极易误诊）：

1. `USER_PROGRAMMABLE_CC=1` 且 **`PCC_INT_EN=0`**（`PCC_INT_*` 是交换机 INT 遥测
   那套开关，开了反而封死 CCMAD 应答）。
2. RP 启动**不带 `--remote-sw-handler`**——带上它等于声明"应答由对端 NP 软件处理"，
   对端没有 NP 进程时探针被静默丢弃；不带时由对端网卡固件硬件应答。
3. 有正常数据流在跑（探针从流对的事件路径发起）。

验证：`0xdec` 的 w9=rtt_req_n 与 w10=rtt_n 应同步增长（实测应答率 99.98%）。
overlay 封装不妨碍 CCMAD，固件流量走端口层，**tcpdump 抓不到不代表没发**。

**执行面的上升爬坡已经没有了**（2026-08-31 起）。设备码曾用积分搜索水位，把每次
预算变更都当作"上限改变"重置整定窗口，线上按约 12.5%/步 爬升，给加入/退出收敛
额外加 2–2.8 秒。现在 `level` 是**一步赋值** `budget/N`，$N$ 从事件流里数。受控
阶跃实测：命令写进 FIFO 到接收端线速改变 **6 ms**（TCP 侧同口径 8 ms），比
`doca_pcc_mailbox_send` 这个阻塞调用自己返回还早——**那 13.5 ms 是主机侧阻塞的
时长，不在控制路径上**，它只限制刷新频率。看到收敛长尾不要再往这里记。

**$N$ 必须单向（涨得快、跌得慢）。** `level=budget/N` 里 $N$ 估低会让每条 QP 分得
更多，流对因此**越过围栏**——这是设计承诺不会发生的事。被限速越狠的 QP 抬事件越
稀，正在发送的 QP 也会偶尔掉出活跃窗口：实测冻结预算、十条 QP 满发时，$N$ 读到
10 占 92.4%、读 9 占 5.2%、读 8 占 2.4%，对应超发 11% 与 25%，开环下就把线速抖出
9%。所以 $N$ 一见到 QP 立刻上调、只在低读数连续站住 64 个 epoch 后才下调。

**诊断邮箱**：`0xdeb <pair>` 回读 {flowtag, budget, level, remote_rx, r_units,
cc_rate, epochs, cap, α, Rt, stage}；`0xdec <pair>` 回读 {cnp_any, cnp_hits,
qp_count, cc_rate, dbg_hits, flowtag, rtt_last, rtt_min, rtt_events, rtt_req_n,
rtt_n}。**tx agent 在跑时查询不可靠**——它每 13ms 往同一个 FIFO 写批量预算，查询
会被交错吞掉；要读干净状态就先停 tx，或看它自己的日志。

## 症状速查

| 症状 | 先查 |
|---|---|
| 授权对、线上不对（预算照收、level 照读） | RP 陈旧 → 重启 RP |
| 收敛长尾 >10s，三层读数都正确 | 预算陈旧应用形态（RP 是否该重启）；**不要再怀疑执行面爬坡，那条路已经是一步赋值** |
| n 个 QP 的流跑到 n 倍份额 | `qpn_resolver` 是否在位 |
| RoCE 建连成功但零吞吐 + CQE error | VF MTU 是不是 1500 |
| 某几对 RDMA 报 Completion with error 且永不恢复 | 启动瞬态聚合过载，加 `--rate_limit` |
| 跨主机数据面完全不通而 cc/agents 全对 | `ovs-vsctl list-ports ovsbr-p1` 有无残留；p1 在不在桥上 |
| TCP 不受限而律/shim/map 全对 | EDT 辅助 map 是否为空（`bpftool map dump pinned .../hpft_pair_state`） |
| 限速名存实亡而流量图貌似合理 | `vport_meter` 僵尸（跑一条流看 rx jsonl 的 `r`） |
| 类分队列不成立、流量全在 TC0 | `tos=inherit` |
| 交叉对实验的归属全落在直连对上 | 用了 `-R`，改 `-x <gid>` |
| 某个 VF 抢答别人的 ARP、流集全落 vf0 | `cross_pair_net.sh apply` 没做 |
| 加入事件里 TCP 首字节晚约 1 s、接收端那段时间零字节 | iperf3 没带 `--start-at`（用 `strings` 查库，别查 `--help`） |
| 新流接纳延迟同配置下从 150 ms 跳到 1100 ms | 接收端 OVS meter 的 police 卸载失败（查 vswitchd 日志的 `police action`） |
| 一条流对跑到超过自己份额，而账本与预算都正确 | 设备端 `N` 读低了（`0xded` 读回 level，`预算/level` 就是 N） |
| RDMA 归因抖动远大于 TCP，且控制环停掉也还在 | 同上，先在冻结预算下量 level 稳不稳 |

## Jakiro DHTB（对照环境）

1. **冷启动概率性不转发**：同一 conf 同一序列，有的实例正常、有的把 TCP/RoCE 全
   黑洞而 ICMP 照过（ICMP 不在它的分类器里）。**ping 判活无效**，唯一可信的就绪门
   是起 server 后打一条真 TCP 流过树看吞吐。
2. 因此**不要每 run 重启 DHTB**——重启一次就重掷一次硬币。现行做法是单实例战役：
   起一次、TCP 探测验证转发后连跑全部 run。
3. 探针复用 iperf3 server 会踩"单 test 服务"陷阱（被 timeout 掐死的客户端让 server
   卡住，其后探测全零）——每次探测前 kill 重起。
4. Arm 重启后 hugepages 需 ≥4GB（2048×2MB），否则先死在 EAL、补 2GB 仍死在 mbuf 池。

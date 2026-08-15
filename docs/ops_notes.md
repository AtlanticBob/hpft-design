# 平台缺陷病历与实验卫生规则（2026-07-10 整理）

这份文档收录两天实现期里踩过的全部平台级问题：每个问题按"症状—根因—
修复—教训"的结构写成一段，供后续实验者和论文的 lessons 小节取材。

## 三个"卡死"家族

**家族一：旧 tx_agent2 的两个缺陷（2026-07-09 取证，已随旧链路退役）。**
交接文档记载旧系统"运行约 4 天后卡死、重启即愈、根因未查"。M0 期间它
在混合流量下当场复现，strace 取证发现其实是两个独立缺陷叠加。其一，
接收端配置表里 vf1 的 IP 是一个陈年错值（10.1.0.4，应为 10.1.1.2），
导致 vf1 的限速值永远查不到、被静默跳过——vf1 的 RDMA 从未被限过速
（实测单流跑到 185G）。所谓"只更新 pair 3"其实是把批量消息头里的
"3 个条目"误读成了编号。其二，旧代理用 representor 的字节计数器测速率，
而这个计数器把 TCP 和 RDMA 加在一起；执行面却只有 PCC 能压 RDMA。于是
同一 VF 上 TCP 一旦超过 RDMA 的限速值，控制器就拼命压 RDMA，直到把它
压穿到最低档（0.044G）。教训：测量必须与执行面同粒度；配置必须由
registry 生成而不是手工维护；静默跳过是缺陷的藏身处。

**家族二：RP 的 FIFO 读端会被 EOF 永久杀死（2026-07-09/10 两次咬人）。**
PCC 的速率服务进程从一个命名管道读取预算指令。rp_service.sh 用一个
`sleep 1200`（20 分钟）当保活写端；旧 tx_agent2 长期持有写端时没有问题，
但它一停役，20 分钟后管道上再没有任何写端，读端读到 EOF 之后就永远
不再读了——此后所有预算写入都进入死管道，限速面静默失效。第一次咬人
是在旧链路退役时（M1b 的 4G 阶跃不生效），第二次是参数扫描批次反复
重启发送端代理时。修复有两道：保活写端改为 `sleep infinity`，且新
发送端代理启动即打开写端并永久持有。教训：命名管道的"最后一个写端
关闭 = 读端 EOF"语义对常驻服务是隐形地雷。

**家族三：RP 内部状态会随实验翻腾而劣化（2026-07-10 发现，根因未明）。**
在几十轮实验（代理反复重启、预算高频改写）之后，新建的 RDMA QP 会
出现"压到地板后每周期只爬 7% 左右、反复被压碎"的病态，单流均值跌到
0.6-4.1G。重启两端代理无效，重启 RP 进程立刻痊愈（恢复 5.94G、带内率
92-93%，且与控制律的新旧特性无关）。根因需要进入 PCC device 代码排查
（per-flowtag/QP 的累积状态），device 代码目前是禁区，待用户授权。
**临时对策：每个实验批次之间重启一次 RP**（`bash /tmp/rp_service.sh
start`）。这也提醒我们：当天下午在劣化窗口内测得的数据（部分 solo 与
双类 A 扫描）已标注污染并重测。

## pkill 的自匹配陷阱（两次事故）

`pkill -f <pattern>` 按整条命令行做正则匹配。若 pkill 与它要杀的目标
写在同一条 ssh 远程命令里，pattern 会匹配到承载它自己的那个 shell 的
命令行（里面就有目标程序的完整调用文本），于是把自己杀死。第一次表现
为 ssh 返回 255（远端 shell 被杀），第二次让浸泡流量的 RDMA 轮连续
9 轮静默失败（server 从未启动成功）。规则：**pkill 与目标进程的启动
永远分开两条 ssh**；pattern 里的 `[x]` 字符类技巧只在同一命令行不含
目标真实调用文本时才有效。

## perftest（ib_write_bw）的三个坑

第一，duration 模式的 `-D` 参数两端必须一致，不一致时报
"Failed to negotiate parameters"（分阶段实验要为每个阶段起匹配的
server）。第二，结果表里的 iterations 列在 -D 模式下不可靠（约为真实
消息数的一半），计算总量要用 BW average × 时长。第三，`--rate_limit`
是硬件档位制，只接受 2.5/5/10/... 这样的离散值，传 2 会直接报错退出
——这个失败模式很安静，容易造成"以为流量在跑其实没跑"。

## 实验卫生规则

第一，`config/lab-registry.json`（repo 内）是政策与参数的唯一真值：
一切修改先改 repo 再 scp 下发；实验中临时的 ssh 直改必须在实验尾同步
回 repo（我们曾因一次反向覆盖丢掉 6G 政策，出现"cap 不生效"的假故障）。
第二，跑受控实验前必须暂停浸泡流量的 cron（把 soak_traffic 一行注释
掉），否则每 15 分钟一轮的背景流量会污染测量，实验后记得恢复。第三，
批次之间重启 RP（见家族三）。第四，长跑日志在 DPU 的 /tmp（tmpfs，
内存盘）上，看门狗已带 300MB 自动截断。

## 平台事实（影响设计的硬约束）

**mlx5 流计数器缓存 1 秒。** TC 卸载流的硬件字节计数器由驱动每秒批量
轮询一次（5.15 内核硬编码，无调节旋钮），所以 50ms 控制周期不能直接
用 megaflow 计数器测速率——20 个周期里 19 个读到零增量，1 个读到整秒
的堆积。解法是混合测量：representor vport 计数器（任意频率都新鲜）供
每 VF 总量，megaflow 计数器供类构成比（2 秒滑窗；megaflow 的 key 出现
是即时的，单流集合冷启动可立刻 100% 归因）。

**devlink 限速改判可用。** 2026-07-05 曾测得 `devlink port function
rate` 会把 vport 打瘫（该结论一度成为禁碰项）；2026-07-10 经用户授权
在 fw 32.49.1014 上复测：带载设置、翻转、解除共 14 次操作全部干净，
限速精确（10G 档实测线上 9.78G）。现在它是两层模型（§3.6）第一层的
硬件执行兜底。生产依赖前建议再做长时间浸泡。

**带内遥测网。** 两台 DPU 的 PF1 侧各有一个现成的 SF（enp3s0f1s0），
把它们的 representor 挂上 underlay-p1、配上 10.1.9.1/2，遥测便与被控
数据同走 p1/交换机路径（稳态往返 0.1ms），不依赖任何生产环境不存在的
专用管理网。注意 p1 改线速时链路会闪断约 10 秒，遥测随之短暂中断，
fail-open 会按设计接管。

**lossy 配置。** 两端 p1 的 PFC 全关（原本即关）；dpu2 侧 802.3x 全局
pause 原为开启（等于把最后一跳无损化），2026-07-09 已关闭。全路径无
交换机 ECN 依赖，丢包由 RC 重传兜底——与设计的 lossy RDMA 前提一致。

## 压测战役追加病历（2026-07-11 晚）

**p1 变速的暂态延迟病。** 反复 ethtool 变速（200↔100↔25G）后，全
fabric 路径（VF 与 SF 皆然）出现空载 RTT 尖峰 10-200ms（正常 0.1ms），
两端 OVS 重启与定速重训均无效，约 1 小时后自行消退。期间带内遥测尖峰
超过 fail-open 阈值造成间歇性控制面失守。规则：变速类实验尽量减少翻转
次数，相邻实验之间用 `ping -c 100 -i 0.05` 验证 fabric 干净再开跑。

**⚠ 别把 megaflow 冷启动税误判成这个病（2026-07-24，白等 7.5 小时的教训）。**
间歇探测（或拓扑刚建好）时，OVS 的 megaflow 表项空闲被逐出，每一轮的
**首包**走 DPU 慢路径 20-100ms，热路径立刻回落 0.04ms——这不是变速病，
不用等，一预热就没。当时判净标准误取了"含冷启动的 ping max"，于是永远
达不到、白等自愈。**鉴别方法（真病 vs 假李鬼）：**

| | 真·变速病 | 假·megaflow 冷启动税 |
|---|---|---|
| 触发 | 反复变速后 | 任何间歇探测、拓扑刚建好 |
| 表现 | **连续** ping 也持续尖峰 | 只有**首包**慢，热路径立刻 0.04ms |
| 自愈 | ~1 小时 | 不用等，一预热就没 |
| 判据 | `ping -c 100 -i 0.05` 的 **avg** 仍高 | avg 干净，只有 max 高 |

**规则订正：判 fabric 干净一律看 `ping -c 100 -i 0.05` 的 avg（≈0.04ms
为净），不看 max。** avg 也高才是真病该等，只有 max 高就是冷启动、直接
开跑。（原规则只说"用 ping 验证"没说看哪个统计量，正是这个歧义导致了
误判。）

**SF bounce 会掉 IP；OVS 重启会掉分类规则。** `ip link set enp3s0f1s0
down/up` 抹掉其上的 10.1.9.x 地址（带内遥测双端全断）；OVS 重启抹掉
rx_agent 的三条分类 OpenFlow 规则。恢复：重加 IP + 重启 rx_agent
（幂等重装）。实验前 preflight 建议加两条：SF 互 ping、tx journal 无
持续 fail_open。

**"预算永不下降"型执行面失守的辨认。** 症状：律面 mode=md 且 R 在降，
0xdeb 却显示 bud 停在高位不动、ep 冻结、线速失控。根因族：tx 侧预算
下发链的语义 bug（曾出现 slew=1.0 边界把下降路径短路）。速查：手写一
条 0xb47c 批量到 rp_fifo 后再查 0xdeb——若手写也"无效"，先怀疑 tx
每 13ms 的覆盖写，而不是 RP。

## cap 变更的两个陷阱（2026-07-12，压测战役追加）

**改 cap 必须两端一起改。** 发送端树 Tree_f=min(MaxRate_src, 需求)，
所以只抬接收端 MaxRate 不生效——流被发送端 MaxRate 锁死。曾因只抬
sgpu02/vf0 到 50G、忘了 sgpu01/vf0（仍 20G），把流限在 20G 却误判成
"18.3G 硬件天花板"。改 cap 的脚本必须同时改 {src, dst} 两侧的 vnic。

**"杀 tx_agent" 不等于"无限速"。** RP（doca_pcc）在 device 内存里保留
每个 flowtag 上一次的预算，tx 停了它继续按旧预算 pacing。要真正取消某
流的限速：先停 tx，再删预算 `echo "0xb47c0001 <flowtag> 0 0" > rp_fifo`
（budget=0 删除 pair），0xdeb 确认 bud=0，此后流跑裸 DCQCN（单 QP vf0
实测 185G）。要恢复：重启 tx 即自动重下预算。

## 常驻进程的运行环境坑（2026-07-04，来自 C1 统一 controller 交付）

**需要 SSH 的进程必须在登录用户环境跑。** `qpn_resolver` 这类需要发起
SSH 连接的组件放进 systemd 裸环境会失败——没有 `HOME`/`.ssh`，ssh 直接
报错退出。只做 ethtool+UDP+FIFO 这类本地操作的进程（tx_agent2/rx_agent
一类）可以正常 systemd 托管。

**转瞬重启的 bind 冲突。** 常驻进程若短时间内被反复重启（调参批次），
会撞上端口 TIME_WAIT 导致 bind 失败；修法是 `SO_REUSEADDR` + 重启前
`systemctl reset-failed`。

**当前最稳的托管方式是 keepalive wrapper（while 循环裸跑），不是
systemd unit**——生产化时应该统一改成带正确 `Environment=` 的 systemd
unit，但直到那一步做完之前，遇到需要 SSH 的常驻进程报"莫名其妙连不上"，
先检查是不是登录环境缺失，而不是先怀疑网络。

## 多 VF 并发 TCP 必须用 %dev 强绑定（2026-07-12，大规模 L2）

持续 4 对 RDMA+TCP 满载下，iperf3 用 `-B 10.1.N.1`（仅绑源 IP）会让
TCP 间歇性走错 VF egress（源地址路由 + 跨对 rp_filter=2/arp_ignore=1
的交互），接收端 per-VF netdev 计数器错记 → rx 类拆分把某 VF 的 TCP
误记成 0（nfs 掉到 4）→ 该 TCP 遥测"断流"触发 fail-open → TCP 失控
冲到线速。修复：`-B 10.1.N.1%dpu1vfN`（SO_BINDTODEVICE，iperf3 3.7+）
把流钉死在指定 VF。规则：任何多 VF 并发 TCP 实验一律 %dev 强绑定。
（这也提示一个真实加固点：混合流量下 TCP 从非零突然归零应回退 megaflow
归因而非直接 fail-open——记为设计待议。）

## 三套实验环境的快速切换（lab_env.sh，2026-07-24）

lab 现在有三套互斥的实验环境，用 `hpft-implementation/tools/lab_env.sh`
（在 sgpu01 上跑）一条命令切换，别再手工拼 cc_mode.sh + 拓扑：

- **hpft**：PCC+HPFT 生产栈（UPCC=1、doca_pcc、rx/tx agent、pace-shim、
  直连拓扑、带内遥测）。评估实验用这套。
- **plain**：固件原生 DCQCN（UPCC=0）、无 HPFT、无 Jakiro、直连拓扑。
  **motivation 实验（1-1/1-2）的基线就是这套。**
- **jakiro**：固件 DCQCN + VxLAN overlay（ovsbr-p1 + vxlan100）+ 接收端
  decap 点的 Jakiro DHTB。motiv 1.3 的对比环境。

`lab_env.sh status` 只读，先看当前在哪套再切。

三套环境差在**四个轴**，不只是 CC 模式：UPCC/CC、HPFT agents 死活、
数据面拓扑（VF representor 挂 underlay-p1 还是 ovsbr-p1）、Jakiro 死活、
VF IP 方案（直连 vf_i=10.1.i.x vs Jakiro 单 overlay IP 源 10.1.0.{11..14}
→ 目的 10.1.0.2）、p1（100G/underlay vs 200G/直挂 underlay IP）。

**两个必须知道的坑**（lab_env.sh 已内建处理，手工切时照做）：

1. **fw reset 不清 OVS。** 从 jakiro 切回 hpft/plain 时，光做 fw reset
   （cc_mode.sh 内含）**不会**删掉 ovsbr-p1——OVS 配置跨 reset 持久，
   孤儿 ovsbr-p1 会把 VF representor 一直扣在自己身上、不在 underlay-p1
   上，直连数据面就一直不通。必须显式 `del-br ovsbr-p1` + 把 representor
   加回 underlay-p1（lab_env.sh 的 teardown_overlay 干这个，是
   build_overlay 的逆操作）。**症状：cc/agents 看着全对，但跨主机数据面
   完全不通**——先查 `ovs-vsctl list-ports ovsbr-p1` 有没有残留。
2. **reboot_recover.sh 不管 representor 摆放。** 它只补遥测 IP + VF +
   EDT，假定 OVS DB 里直连配置已在。所以 VxLAN 拆除这步 lab_env.sh 必须
   自己做，不能指望 cc_mode.sh pcc 顺带修好。

## 多对并发 RDMA 必须从一开始就限速（2026-07-27）

**症状**：8 对 RDMA 并发时，其中固定的 2 对报 `Completion with error at
client` 并且**永不恢复**，另外 6 对正常。看起来像"某些流对坏了"，极易
误判成 fabric 故障或 flowtag 问题。

**它不是**。同样那两对：单独跑 27 G 正常，两两并发正常，ICMP 全通，
RP 里根本没有它们的预算条目。真正的原因是**启动瞬态的聚合过载**——
8 条无限速 RDMA 每条能跑 ~27 G，合计 216 G 灌进 100 G 下行，RC 重传
风暴打死其中几个 QP，而 RC 的错误完成是**终态**，QP 不会自己回来。
哪几条死是由启动顺序决定的，所以看起来"确定性"。

**判别**：给每条流加硬件限速使聚合低于链路容量再跑一遍。实测 8 对
各 `--rate_limit=10 --rate_units=g`（合计 80 G < 100 G）后，**8 对全部
正常**（9.70–10.39 G），包括原先必死的两对。

**规则**：任何多对并发 RDMA 实验，**聚合需求必须从第一个包起就低于
瓶颈容量**。两种做法——用 `--rate_limit` 给每条流设硬件档位，或者让
被测系统在流量起来之前就已经把限速下发下去。**不要依赖控制环"1–2 个
周期内收敛"来兜住启动瞬态**：那两个周期里 QP 已经死了，之后收敛得再
准也救不回来。

**对 HPFT 实验的具体影响**：新流集合以 $R = Tree_f$ 乐观起步（design.md
§4.2），$N$ 条流对同时起步时聚合是 $N \times Tree_f$，可以远超瓶颈。
设计里"瞬态超额由租户 CC 的即时反应兜底"这句话对 TCP 成立，**对 RDMA
不成立**——RC 的错误完成不可逆。所以多对 RDMA 场景的 runner 要么错开
启动，要么给 perftest 加 `--rate_limit`。

**同一家族的第二种触发：瞬时大倍率降速。** 单对 RDMA 在 cap 20 G→1 G
的**一步 20 倍**政策降幅下，3 次里有 1 次线上归零且不再恢复。对照实验
排除了两个嫌疑：稳态低速率**不危险**（cap 3 G/2 G/1 G 各跑 25 s，线上
2.72/1.78/0.87 G，实现率 87–91%，无死亡）；PCC 水位控制也**不是元凶**
（线上归零时 `err = bud − 0 > 0`，水位每周期反而上抬 12.5%，控制器一直
在救）。所以危险的是**降幅本身**，不是终点速率。

这修正了一条旧记载："RP 在 ≲1 G 预算下卡死"——真正的条件不是预算低，
而是**一步降得太狠**。控制律自身不会产生这种阶跃（目标以 1/k = 50 ms
的时间常数移动，每 13 ms 一次的预算刷新之间变化只有几个百分点），只有
**人为的大幅政策变更**才会。做这类实验时分两三步降，或接受偶发的 QP
死亡并在分析时识别出来（症状：线上恒为 0 而预算/水位正常）。

## overlay 常驻时代（2026-07-29，P0 spike 定案）

**lab 数据面自 2026-07-29 起常驻泛化 VxLAN overlay**（用户裁决，实证
`hpft-implementation/results/p0_overlay_20260729/summary.md`）：两台 DPU 的
p1 直配 underlay 172.16.1.x + 遥测 10.1.9.x（第二地址）+ MTU 9000，
ovsbr-p1 挂 4 个 representor + vxlan100，VF IP 用直连方案原样，p1 保持
100G。hpft/plain/jakiro 三环境共用这层数据面，只切 CC/agents/DHTB
（`lab_env.sh` 的 ensure_overlay 负责结构，`reboot_recover.sh` 负责易失
态）。rx_agent 默认桥已改为 ovsbr-p1。registry headroom=0.08（3% 队列
预警 + ~5% 封装税）；**V 与 headroom 解耦**（真值规则 V=4ζ²γê*/k，
现值 576 Mbit 经 v_seconds=0.072 拨入，判别实验 vtune_20260729）。
随之而来的硬事实：

- **`options:tos=inherit` 是硬性配置**：OVS VxLAN 默认 tos=0 不复制内层
  DSCP，`--tclass` 流量全落交换机 TC0（实测 TC0 +98.0M 帧/TC3 +0），
  with-TC 类分队列无法成立；设 inherit 后 TC3 +65.1M/TC0 +30。ECN 折回
  在两种设置下都完好（RFC 6040 由硬件 decap 执行，np_ecn_marked 过载下
  +20k/+13.6k）。
- **p1 的软件 netdev 计数器在硬件卸载下不动**（rx_bytes 恒 0），外层
  字节一律读 `ethtool -S p1` 的 `rx_bytes_phy`。
- **RP（doca_pcc）重启会清空设备内存里的全部流对预算**——与"杀 tx 不等于
  解除限速"互为对偶：杀 tx 预算冻结，重启 RP 预算归零。凡依赖"停 tx
  冻结预算"的编排，必须先让 tx 对全部目标流集合推过预算再停。
- 拆建 overlay 后每条新路径首包付 megaflow 冷启动税（20–100ms），判净
  仍按本文上方的真/假鉴别表。
- 封装税实测 4.99%（65536B 消息、MTU 1500，内层 host IB 计数 vs p1
  phy）；账本/pace 全链内层口径自洽，外层税只在 root 容量层面由
  headroom 吸收，**不要给 TCP shaper 单独加外层补偿**（会重造
  TCP↔RDMA 不对称——RDMA 硬件调速计不了外层）。

**家族三新形态：预算陈旧应用（2026-07-30，eval 批 1 D1 实验定位）**。
劣化不止 fail-open 一种表现：tx 已推送新预算（遥测 R=Tree=pace 全部
正确）、rx 裁定无辜（u/e/s 正确）的情况下，PCC 设备侧把新预算搁置
~12 秒后才一步应用（线上钉在旧 level 值）。3 遍中 1 遍复现。每实验前
"RP 重启 + 执行守卫探针"（eval_lib.sh 的 rp_guard）能拦截 fail-open
形态，**测不出陈旧应用形态**——涉及预算在线变更的实验（动态类）对
>10s 的收敛长尾先怀疑此病，用"rx u 正确 + tx pace 正确 + 线上钉旧值"
三层证据链确诊，不要误记为控制环缺陷。

**Jakiro DHTB 的 hugepages 前提（2026-07-30）**：dpu2 Arm 重启（fw reset
随行）会清掉 hugepages 预留（非持久），DHTB 先死在 EAL（"Cannot get
hugepage information"），补 2GB 仍死在 mbuf 池（"Cannot allocate mbuf
pool"）——**需要 ≥4GB（2048×2MB）**。`echo 2048 > /sys/kernel/mm/
hugepages/hugepages-2048kB/nr_hugepages` 后重启 DHTB 即愈。

**实验卫生新条目：CC 开关状态必须 trap 复原（2026-07-30，烧掉批 4 首轮的学费）**。
native-None 臂用双端 host PF 的 `roce_rp/roce_np enable` 全 prio 置 0 实现
"关 CC"，runner 未设退出复原 → 状态泄漏到后续 Jakiro 批次：DHTB 打了上亿
CE 而 NP 装聋（np_cnp=0），30G 配额被 87G 穿透，8 个运行作废。症状极易误诊
为"CE 在 decap 路径丢失"。**规则：凡改 CC/重传/ECN 开关的 runner 一律
trap EXIT 复原；跨批次首个实验前把 `dcqcn.sh status` 的 enable 位列入
核对清单。**

## TCP 执行面：债务赦免与债务上限（2026-07-30，eval 修复）

**症状**：BBR 租户稳定跑到 pace 的 1.22 倍（类间 56:38 vs 政策 46:46），
而分配全程正确（u/pace 都是 11.5G），reno/cubic 无此现象。

**根因（两层，缺一不可）**：①pace-shim 每次下发预算都 `make_generation()`
换一个新 generation，BPF 侧把"换代"理解为重配置、将该流对的整形债务
`state->next_ns` 直接重置为 `now`——于是每 ~100ms 债务被赦免一次；
②深债流对的时间戳被盖到很远的未来，fq 先囤后成串放出，形成瞬时线速
突发。loss 基 CC 因丢包自限看不出来，BBR 这类自带 pacing、不轻易退让的
CC 就把赦免额度全额兑现。

**修法**：generation 改为 per-pair 稳定值（仅首装或速率跳变 >25% 时换代，
`_gen_for()`）；BPF 侧换代时只做**有界赦免**（把债务截到 `now+50ms`），
且债务超过 50ms 上限的报文**直接丢弃**（`TC_ACT_SHOT`）而不是盖更远的
时间戳——既给 rate-based CC 真实丢包信号，又把 fq 排队时延封顶。

**顺带治好的问题**：incast 角点（TCP 流数 ≥31、RDMA 租户仅 1–2 QP 时
RDMA 被 CE 风暴钉死）**随之消失**——1:31 点四租户从 8.2/3.0/3.0/29.5
变成 21.9/22.5/22.3/22.7，交换机 CE 标记归零。突发源头就是上面第②层。
反向验证：把 shim 的 burst 收紧到 32KB 反而让受害者掉回 3.1G、CE 98 万，
**故不引入 burst 旋钮**，保持 256KB 默认。

## 归属滞后守卫（2026-07-30，eval 修复）

**症状**：同 dst 新发送方加入后 ~2s 内，其字节被记到在位者头上（曾观测
单拍 46.69G 的物理不可能读数），在位者账本被打满、吃 25% 冤枉折扣，
D3 加入方向收敛 3.5–4.0s。

**修法**（`rx_agent.HybridRates`）：把"划分出错免疫"从执行面内环延伸到
账本充入路径——**部分可见的新成员**（年龄 < mix 窗且已有字节）出现时，
该 dst 的池子改按上一拍授予额分摊，不信尚不可信的 megaflow 比例。
**只对新成员生效**：退出者字节为零，由既有 `unmeasured` 分支覆盖；早期
把退出也纳入守卫会让幸存者迟迟爬不进被释放的份额。

**效果与残余**：加入方向 3.5–4.0s → **0.31/0.43s**（10×）。退出方向
仍 ~2.8s，机制不同——幸存者的 r 受 2s megaflow 窗口内退出者的陈旧字节
稀释，属测量链延迟，非律或账本问题；根治需要更新鲜的 per-sender 数据源
（发送端自报或 RP per-flowtag 计数），列为后续。

## 发送端活性回传：归属的新鲜信号（2026-07-30）

接收端把一个 dst 的精确池子分给各发送方时，比例来自 megaflow 字节计数，
而该计数被硬件缓存约 1 秒。**发送方停发后，它的陈旧字节还能占住份额**，
幸存者的实测速率被稀释、迟迟爬不进被释放的容量。接收端没有任何计数器
能在 1 秒内看出这件事，但**发送方自己的 vport TX 计数是 1ms 新鲜的**。

做法：发送端 DPU 也跑 `hpft-vport-meter`（同一个二进制，读本端 PF），
`tx_agent_e` 每拍把 per-(src vnic, class) 的实发速率经 UDP 9713 发给
接收端；`rx_agent` 只把它当**门控**用（"这个发送方还在发吗"），比例仍由
接收端自己测。发送方停发 → 当拍从归属里消失 → 幸存者立刻拿到全部池子。
回传缺失或过期（>0.3s）时行为与之前完全一致，基线臂不受影响。
遥测增量约 1.7 Mbit/s。**键名坑**：payload 的 key 已是 `<vnic_id>|<class>`
而 vnic_id 自带 host（`sgpu01/vf0`），接收端不要再拼一次 host。

**效果与剩余**：同 dst 加入方向 3.7s → **0.24–0.49s**；退出方向的归属
部分被彻底消除（B 退出当拍就从 r 里消失、目标 u 立即跳到新份额），但
**总收敛仍约 2–2.8s，剩余全在 RDMA 执行面的上升爬坡**：发送端 pace 在
30ms 内就到位，线上速率却按约 12.5%/步 爬。根因是 RP 把任何预算变更都
当作"上限改变"并重置整定窗口，因此 tx 用 3% 迟滞保持预算准静态、设备
只能靠自身积分往上爬。试过"大幅上调时连推几拍"促使设备走直接赋值分支，
无效（且重复推送反而重置整定窗口），已撤回。**根治需要改 DPA 设备码的
水位赋值逻辑，未做。**

## RP 预算新鲜度探针（2026-07-30）

`eval_lib.sh` 的 `rp_guard` 只验"执行面还在压着"，测不出"压的是不是
新预算"——正是造成 12–24 秒收敛长尾的那个形态。新增 `rp_fresh`：跑中把
dst 配额 20G→8G，检查线上是否跟随（<9.5G 判过）。实测通过（7.76G）。
涉及在线改预算的实验批次建议在 `rp_guard` 之后加跑一次。

## PCC 里的拥塞控制项：三选一，以及 RTT 通路的现状（2026-08-03）

HyperFront 的限速（`level = budget/N`）与其下的拥塞控制是两件事，执行面
永远是 `rate = min(cc_rate, level)`。cc_rate 现有三种实现，运行时用
mailbox `0xccd <0|1|2>` 切换（切换会重置各流对的 CC 状态）：

- **0 = AIMD**：CNP 触发固定比例乘性减 `cc_rate -= (cc_rate>>6)/nqp`，
  每 epoch 固定步长加性增 `+= MAX>>8`。**它不是 DCQCN**——没有 α、没有
  目标速率、没有三阶段恢复。文档与论文里必须叫它 AIMD。
- **2 = 软件 DCQCN**：RP 状态机的忠实实现——CNP 时 `Rt=Rc`、
  `Rc=Rc(1−α/2)`、`α+=g(1−α)`；α 定时器按 g 衰减；恢复定时器驱动
  快恢复（F 步，`Rc=(Rt+Rc)/2`）→ 加性增（`Rt+=AI`）→ 超增（`Rt+=HAI`）。
  三个旋钮经 mailbox `0xcce <值> <0=AI|1=HAI|2=恢复定时器us>` 可调，
  对应固件的 rpg_ai_rate / rpg_hai_rate / rpg_time_reset。
  **量化事实**：定时器在 epoch（1ms）边界检查，故实际周期是
  `max(epoch, 设定值)`——设 300µs 与 3000µs 实测每 10 秒推进 12310 与
  3712 个阶段（≈1ms 与 ≈2.7ms 周期），单调可调但不细于 epoch。
  **初始化是必须的**：新分配的流对若不置 `Rt=MAX、α=1`，`Rc=(Rt+Rc)/2`
  会把速率每拍折半直至归零（实测流量跑成 0 字节）。
- **1 = ZTR-RTTCC**：厂商 RTT 基算法（NACK/CNP/RTT-超基准三档乘性减、
  否则加性增）。阈值必须用本 fabric 实测值：设备时钟（1ns 粒度）下
  base RTT min=3006ns、26G 负载无拥塞抖动带 3.3–5.7µs，故
  `ZTR_BASE_RTT=7000`、`ZTR_MAX_DELAY=70000`（厂商模板默认 13µs/150µs
  是别人 fabric 的数，在这里 4 倍偏高）。

**RTT 通路成立的三个条件**（缺一则发送端 `rtt_req` 石沉大海、
`rtt_events` 恒 0，而 CNP 一切正常——极易误诊）：
1. `USER_PROGRAMMABLE_CC=1` 且 **`PCC_INT_EN=0`**（doca_pcc.h:72 成文；
   `PCC_INT_*` 是交换机 INT 遥测那套的开关，开了反而封死 CCMAD 应答）。
   当前双端另配了 `PCC_NP_HANDLE_CORE_UTIL=3`/`PCC_HANDLE_CORE_UTIL=3`
   （工作配置的一部分，单独必要性未拆开验证）。
2. RP 启动**不带 `--remote-sw-handler`**。带上它=声明"应答由对端 NP
   软件进程处理"，对端没有 NP 进程时探针被静默丢弃；不带时由**对端网卡
   固件硬件应答**，无需对端跑任何 PCC 进程。这是 rp_service.sh 里的一行
   之差，曾让 RTT 断了整个战役周期。
3. 正常数据流在跑（探针从流对的事件路径发起）。
验证手感：`0xdec` 的 w9=rtt_req_n 与 w10=rtt_n 应同步增长（实测应答率
99.98%）；overlay 封装不妨碍 CCMAD（固件流量走端口层，tcpdump 对其
不可见——抓不到包不代表没发）。

**构建与部署提醒**：设备码改完必须 `meson setup --reconfigure build`
再 `ninja`（dpacc 在 configure 步）；host 侧编译的是
`pcc/host/pcc.c`（仓库副本名为 `pcc_host.c`，改完要 scp 成 `pcc.c`），
只改 host 时 `ninja` 即可。DPU 上的活树是 `~/bzx/doca34-apps`。诊断探针：
`0xdeb <pair>` 回读 {flowtag,budget,level,remote_rx,r_units,cc_rate,
epochs,cap,α,Rt,stage}，`0xdec <pair>` 回读 {cnp_any,cnp_hits,qp_count,
cc_rate,dbg_hits,flowtag,rtt_last,rtt_min,rtt_events,rtt_req_n,rtt_n}。

## fw reset 后的僵死病：vport_meter 与一切 DEVX 常驻进程（2026-08-03）

`vport_meter` 持有 DEVX 上下文常驻采样。**fw reset 杀死上下文但不杀进程**：
进程还在、systemd 显示 active、共享内存却永远停更——下游 rx_agent 测不到
任何速率，political budget 全部 fail-open 成 MaxRate，限速名存实亡，而一切
流量图看起来"还行"（devlink 顶+对称结构给出貌似合理的份额）。极难察觉。

- 判活不能看进程/unit 状态，也不能看 mmap 文件 mtime（mmap 写不更新
  mtime）——要跑一条流看 rx_agent jsonl 的 `r` 字段是否非空。
- 恢复链（reboot_recover.sh）现已无条件 `systemctl restart
  hpft-vport-meter`；实验 runner 的 rp_guard 失败路径也会踢它。
- 同族风险：任何 fw reset 后，所有持有设备上下文的常驻进程
  （vport_meter、doca_pcc、依赖 devx/mlx5dv 的自定义采样器）一律主动重启，
  不要相信"进程还活着"。

同日另一个恢复链缺口：`hpft-qpn-resolver`（host 侧,喂 {qpn->pair} 给 RP,
决定 level=budget/N 的 N）不在任何恢复链里,fw reset 后长期缺位 → 多 QP
流对 N=1,level 放大 4 倍。已加入 cc_mode.sh pcc 路径与 status 行。

## VF MTU 是 RoCE 的生死开关（2026-08-13/14，一整天的学费）

**症状**：RoCE QP 握手成功、数据 WQE 全部 CQE flush error、字节不出
VF vport；同路径 ICMP/TCP 完全正常；双向皆断。极具迷惑性——像 eswitch/
对端/交换机故障，实际与它们全部无关。

**根因**：VF MTU=8192 时 RoCE path MTU 协商为 4096，4KB 数据帧死在
VXLAN encap 路径（1500 下 path MTU 1024，一切正常）。握手包小，照常
通过，所以"能建连不能传数据"。**vf_setup.sh 曾显式设 8192**——用它
重建 VF 而不跟 cc_mode post_recover（强制 1500）就中招。已改源头
（两台 host 的 vf_setup.sh 均为 mtu 1500）。

**判据**：遇到"RoCE 建连成功但零吞吐+CQE error"，先查
`ip link show <vf> | grep mtu`，再做任何重启。当天为此白做了两台 Arm
重启、双端 synced fw reset、两台 host 重启、两侧 mlx5_ib 重载。

## Arm 重启的连锁僵尸（同日，第二课）

Arm OS 重启清掉 hugepages（sysctl 非持久）→ OVS-DOCA/DPDK 组件
（vswitchd doca-init、jakiro_dhtb）拉不起或行为退化。已在两台 DPU 写
`/etc/sysctl.d/99-hpft-hugepages.conf`（nr_hugepages=2048）持久化。
同族已知项：/tmp 清空（vpm_sample.py、rp_service.sh 要重新 scp）、
ROCE_ACCL 寄存器归零、p1 速率回 200G、ethtool pause 复位。

## jakiro DHTB 的两条部署纪律（同日，第三课）

1. **冷启动概率性不转发**：同一 conf 同一序列，有的实例正常、有的把
   TCP/RoCE 全黑洞而 ICMP 照过（ICMP 不在它的分类器里）。ping 判活
   无效；唯一可信的就绪门 = 起 server 后打一条真 TCP 流过树看吞吐。
2. **每 run 重启 DHTB 的纪律与 1 冲突**——重启一次就重掷一次硬币。
   现行做法：**单实例战役**——起一次、TCP 探测验证转发后连跑全部
   run（令牌桶秒级回稳，跨 run 状态影响可忽略）。
3. 顺带：探针/gate 复用 iperf3 server 会踩"单 test 服务"陷阱（timeout
   掐死的客户端让 server 卡住，其后探测全零）——每次探测前 kill 重起。

## CC 兜底项的真实咬合史与 CNP 命中率（2026-08-15 凌晨定案）

执行面 rate = min(cc_rate, level) 里的 cc_rate 兜底，其真实行为由
**CNP 能否命中流对表**决定，而这一点在战役中翻转过一次：

- **8-03 之前（全部 17 个有效 eval 格的时代）**：CNP 反向包的 flowtag
  与数据向不同、QP 哈希表无人喂（qpn_resolver 缺位）——CNP 命中率
  实测 0.3%（cnp_hits=55 vs cnp_any=18522）。**兜底项形同虚设，
  rate ≡ level，即纯对数跟踪律**。"cc_rate 弱 DCQCN 兜底形同虚设"
  旧结论的机制根源就是命中率，不是算法温和。
- **8-14 修好 qpn_resolver 之后**：CNP 命中率 ~100%，兜底火力放大
  300 倍。叠加两个环境因子——接收端 min_time_between_cnps 在 VF
  重建后变成 4µs（CNP 率 29 万/s）、恢复侧（AIMD 的 AI/DCQCN 的
  定时器）都由 TX 事件驱动——形成**自锁陷阱**：cc 一旦被打穿到地板，
  流量归零→TX 事件消失→恢复停摆→永锁（实测 epoch 率 43/s
  vs 名义 1000/s）。AIMD/软件 DCQCN 全数被打穿。
- **与已验格同语义的补格方案**：设备码给 CNP MD 加了 freeze 门
  （与 AI 侧对称），`0xccc 1048576 0` = cc 钉 MAX = rate 纯 level。
  expM 补格用它；expD（要扫活的 CC）在当前全命中环境下不存在
  "咬而不崩"的中间态，档位扫描需重新设计，留设计侧裁决。
- 遥留问题（设计侧）：恢复路径是否应改为时钟驱动；兜底剂量如何
  随命中率标定。

同日多轮重启后的其他既定病：ARP 纪律（arp_ignore）随 VF 重建丢失
导致数据面塌缩到单 VF（详见 8-15 归因调试）；vf_setup 之后必须跟
cross_pair_net apply。

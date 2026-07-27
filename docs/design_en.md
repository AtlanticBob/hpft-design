# HPFT: Virtual-Queue Fairness at the DPU Edge — Design

**Version 3.1 (2026-07-24).** This document is the sole authoritative
design description of the system. v3.1 replaces the sender-side law
with the "track-and-audit" form (§3.4, §4.2, prompted by the
fluid-model analysis in `design_theory.md`); the implementation still
runs its thoroughly measured predecessor (the MIMD law, 1 ms period,
direct vport measurement) — every component other than the law and the
telemetry format matches this document, and the migration plus
revalidation are pending. Historical decision logs, abandoned
directions, and the older designs and law analyses this document
supersedes all live in `archive/`; none of it is maintained alongside
this document — **the implementation follows this document**. Operational procedures are out of scope — read
`ops_notes.md` before touching the lab and `cc_mode_switching.md` for
CC-mode switching. A Chinese edition is kept at `design.md`.

**How to read.** §1–2 state the problem and give the big picture. §3–5
walk the three subsystems in control-loop order; each section first
says what problem the component solves, then gives the mechanism, and
closes with why it is designed that way. §6 covers parameters, §7 the
implementation at a glance, §8 properties and measured results, §9
boundaries and extensions.

## 0 Terminology

| Term | Definition |
|---|---|
| Tenant / VM | In this document tenant = instance = virtual machine; one granularity |
| Flow-set $f$ | The traffic identified by the triple (src VM, dst VM, class), class ∈ {TCP, RDMA}. The smallest unit the system measures, allocates to, marks, and paces |
| Policy | The operator's and tenants' allocation intent: tenant weight $W_i$, tenant bandwidth cap $MaxRate_i$, class weights, per-sender weights (§3.2) |
| Owner | The party that owns a bottleneck resource. The sender owns its uplink; the receiver owns its downlink and each dst VM's policy share |
| Virtual queue (VQ) | A queue that exists only in a ledger: the time integral of (arrival rate − entitled rate), with no real packets queued (precedent: HULL's phantom queues) |
| Entitled rate $e_f$ | The bandwidth the owner's policy rules that $f$ deserves right now, computed by hierarchical water-filling |
| Fair-share ceiling $\hat e_f$ | $f$'s share ceiling: what water-filling would give $f$ if its own demand were infinite while siblings keep their actual demands. Always $\hat e_f \ge e_f$ |
| Mark $s_f$ | The ledger-audited "degree to which $f$ has persistently exceeded its entitled rate", in $[0,1]$; never sent directly — it enters the target as a discount (§3.4) |
| Target rate $u_f$ | The owner's ruling compressed into one number, $u_f = \hat e_f(1-\gamma s_f)$, broadcast in telemetry; what the sender tracks (§3.4) |
| Tracking law | The fixed rule by which a sender low-pass-tracks the rate cap $R_f$ onto the target $u_f$ in log coordinates (§4.2); also called the response law |
| pace | Limiting a flow-set's rate by spacing packet departures; delay only, never drop |
| Tree share $Tree_f$ | The share the sender's local tree gives $f$, i.e. the product of sender-side uplink policy (§4.3) |
| Work-conserving | Capacity never idles while unmet demand exists; "one class idle, the other may borrow" is this property |
| Headroom | Deliberately allocating slightly below physical capacity (3%) so virtual queues alarm before physical ones |
| Fail-open | On control-channel failure the system degrades to "no rate limiting" rather than wedging in a limited state |
| Control period $T$ | The beat of one measure→allocate→mark→respond loop, currently 1 ms |

## 1 Problem

### 1.1 Goal: two-level fairness

In a public cloud, tenant VMs generate both TCP and RDMA (RoCEv2,
lossy, no PFC) traffic that shares the network. The operator must
provide, simultaneously:

- **Inter-tenant hard isolation.** Each tenant has a purchased
  bandwidth cap $MaxRate_i$; tenants competing at the same bottleneck
  split it by operator-configured weights.
- **Intra-tenant inter-class soft isolation.** TCP and RDMA share a
  tenant's allocation by tenant-configured class weights; when one
  class is idle the other may borrow its share, and returns it when
  demand comes back.

### 1.2 Why existing mechanisms cannot deliver it

Two obstacles stand together; either alone would be fatal.

**First, the collision of heterogeneous congestion controls carries no
policy semantics.** TCP (Cubic, reacting to loss at RTT timescales) and
RDMA (DCQCN, reacting to ECN at millisecond timescales) listen to
different signals. On a shared bottleneck queue, the switch's ECN
marks land only on RoCE packets, while TCP — running without ECN —
parks the queue above the marking band: the one class that obeys the
signal is starved by its own compliance. Measured (128-flow incast,
total flow count held constant, only the class ratio varied): as soon
as TCP has more than a single flow, the RDMA class in aggregate is
crushed below 2% of the 100G bottleneck, almost independently of the
flow ratio; switching the retransmission scheme (GBN↔SR) changes
nothing, and sweeping ECN thresholds over 16× or recovery parameters
over 30× cannot flip the outcome. Who gets how much is a by-product of
two algorithms colliding, and no parameter tuning on either side
aligns it with the operator's weight intent.

Separating the two classes into different switch TC queues (DWRR class
weights) cures exactly half. In the TC arm of the same experiment the
class-level 50:50 holds precisely — but at the tenant level, the
tenant that has the TCP class to itself takes 2× the equal-weight
share while the three tenants crowded into the RDMA class get 0.65×
each: a 3× structural unfairness among four equal-weight tenants,
present unchanged at every ratio point. This is not a scheduler defect
but an expressiveness ceiling: a TC has one knob — the class weight —
and §1.1's two-level intent (one share per tenant, classes sharing
within each tenant) cannot be written in that vocabulary; nor does the
TC count (typically eight) scale with the tenant count.

**Second, policy bottlenecks generate no physical congestion signal.**
When a VM is allowed 20G by policy on an idle 100G link, traffic beyond
20G encounters no queue, no ECN, no loss — there is nothing for any
congestion control to react to. Contention for policy shares is
invisible to the transport layer.

The constraints are part of the problem: **tenant transparency** (no
changes to tenant stacks or applications) and **zero data-plane
changes** (no switch modifications, no new packet formats — the
public-cloud reality).

### 1.3 Scope of the guarantee

HPFT's fairness guarantee covers **edge bottlenecks** only — those that
have an owner, where the owner can see all competing traffic:

| Bottleneck | Physical/policy | Owner | Mechanism |
|---|---|---|---|
| Sender uplink | physical | sender | sender tree (§4.3) |
| dst VM policy share | policy | receiver | virtual-scheduler marking (§3.2–3.3) |
| Receiver downlink (incast lives here: the only place an N-to-1 mismatch materializes is the last-hop switch port toward this host) | physical | receiver | scheduler root-layer marking (§3.2) |
| Core network | physical | operator | **not guaranteed**; extension in §9.1 |
| Intra-host (PCIe/memory) | physical | — | not guaranteed (PicNIC territory) |

**Guarantee**: on the three classes of edge bottleneck, the
steady-state allocation converges to the hierarchical weighted max-min
shares defined by policy. **Non-goals**: core-network fairness; loss
elimination; path load balancing; sub-period (< $T$) burst precision —
instantaneous bursts are absorbed by the tenants' own congestion
control (DCQCN's CNP reaction is sub-millisecond and always runs
underneath the pace, §5.1).

## 2 Approach

### 2.1 Core idea

HPFT in one sentence: **compute the shares, send down the targets,
and let every sender run one tracking rule.**

**Shares are computed where all the competition is visible.** The
design covers edge bottlenecks only (§1.3), and they share one
property: every competing flow passes through the same DPU — whatever
a host sends passes through its sender DPU, whatever lands on a host
passes through its receiver DPU. With every competitor in view,
nothing needs to be probed or inferred: each period, that DPU plugs
the policy (weights, caps) into water-filling and computes outright
what every flow-set currently deserves.

**The missing congestion signal is manufactured by bookkeeping.** A
policy bottleneck has no queue and drops nothing, but it can be
accounted for: each flow-set gets a virtual queue that integrates
(arrival rate − deserved share) over time; a flow-set persistently
above its share accumulates debt and gets marked. In front of the
marks, "a 20G policy allowance oversubscribed" and "a 100G physical
port oversubscribed by incast" are the same event — a ledger overload.
This removes the first obstacle of §1.2 (policy bottlenecks emit no
signal).

**Every sender runs one and the same weightless rule against the
target** (the tracking law, §4): the receiver compresses its policy
ruling and ledger audit into one target rate, and the sender's only
job is to track its rate smoothly onto it. The rule contains
no weights and no class distinction — the weights were already spent
when the shares were computed. TCP and RDMA are thus shaped by the
same hand (each one's own congestion control keeps running underneath
the pace, §5.1), and "who gets how much" at a shared bottleneck stops
being a by-product of two colliding congestion controls. This removes
the second obstacle of §1.2.

Multiple bottlenecks need no extra machinery: every bottleneck along a
flow's path keeps its own ledger and expresses its own constraint —
the receiver's arrives as a target, the sender's is its locally computed
tree share — and the enforced rate is the minimum of what the
bottlenecks ruled, so the tightest one binds automatically. At no
point does the system compare weights across hosts (each side
normalizes its own; they were never comparable).

The structural property the rest of this document leans on also falls
out here: **fairness lives in the target, not in the rule.** The
deserved share is computed explicitly at the receiver and, together
with the audit discount, compressed into one target broadcast in
telemetry, so the sender never has to search for the fair point — it
only tracks the target. This single fact dictates the choice of
tracking rule (§4.1).

### 2.2 The control loop

```
sender host          sender DPU                            receiver DPU
┌─────────┐   ┌────────────────────────────┐   wire   ┌───────────────────────────┐
│ tenant VM│──▶│ ④ pace_f = min(R_f, Tree_f) │─────────▶│ ① measure per-flow-set r_f │
│ (TCP/    │   │    R_f ← track u_f     §4.2 │          │ ② allocate: water-filling  │
│  RDMA CC)│   │    Tree_f ← local tree §4.3 │          │        → e_f, ê_f     §3.2 │
└─────────┘   └──────────────▲─────────────┘          │ ③ audit: VQ integral → s_f │
      ▲          telemetry    │                        │   target u_f = ê_f(1−γs_f) │
      └── TCP pace ──(in-band)┴──── {f: u_f, r_f} per period ────────┘         §3.4
```

One round (period $T$ = 1 ms): the receiver DPU measures each
flow-set's arrival rate $r_f$ (①) → three-layer water-filling yields
the entitled rate $e_f$ and the fair-share ceiling $\hat e_f$ (②) →
virtual queues integrate the excess, audit out $s_f$, and the ruling
is compressed into the target $u_f$ (③) → telemetry returns to the
senders → each sender moves its rate cap $R_f$ one tracking step
toward $u_f$ and enforces the pace (④) → next period's $r_f$ reflects
the effect. Total loop delay is roughly 1–2 periods; the tracker's
time constant covers exactly this staleness (§4.2).

### 2.3 Why the flow-set is the unit of control

(src VM, dst VM, class) is the finest unit that carries policy
semantics: tenant weights act on VMs, class weights on classes,
per-sender weights on sources — anything finer (per connection, per QP)
carries no policy meaning. It also has two engineering properties.
State is O(VM pairs × 2), independent of how many connections tenants
open. And shares are ruled per flow-set, so opening more flows **earns
nothing** — placing the control unit here makes the system immune to
flow-count gaming by construction (a flow-set with 1024 QPs receives
the same share as one with a single QP; the executor is responsible for
converging the aggregate inside the set, §5.2).

Conversely, any unit coarser than the flow-set — controlling per VM
pair only — forfeits inter-class fairness. A pacing actuator can only
constrain the aggregate at its boundary; it cannot arbitrate the
composition inside. If TCP and RDMA of the same pair share one rate
cap, how the two classes split that cap is still decided by two
congestion controls colliding at the actuator's queue — if the
actuator drops, TCP is favored (RDMA ends in Go-Back-N retransmit
storms); if it marks ECN, TCP is favored (with ECN off, only RDMA
slows down); if it backpressures, RDMA is favored (hardware-issued
traffic never feels it). Under every discipline one side is deaf: the
collision of §1.2 has merely moved from the switch into the actuator,
not been resolved. In one sentence: **fairness granularity equals
enforcement granularity** — for inter-class fairness, measurement,
allocation, and enforcement must all land on (pair, class).

### 2.4 Relation to prior work

The control skeleton of this design has a clear lineage: edge
arbitration, receiver-side measurement with sender-side enforcement,
explicit feedback, and headroom all trace back to the EyeQ line of
edge bandwidth-guarantee systems (NSDI'13); manufacturing signals with
virtual queues descends from HULL's phantom queues and AVQ's integral
marking. The skeleton is mature, and this design adopts it openly.

What is new are the three parts of that skeleton that had to be
rebuilt for a world of two congestion-controlled traffic classes and
hardware enforcement (each argued in place in the body; this section
only collects them). First, the fairness unit must be refined from
VM/pair to (pair, class), or the inter-class collision merely moves
from the switch into the actuator (§2.3). Second, EyeQ-style
allocation sidesteps demand estimation by dividing capacity among
active endpoints in proportion to their guarantees — sufficient for
minimum guarantees, but the policy target of weighted max-min with
borrowing and overselling requires the *degree* of demand to enter
the allocation, making demand estimation unavoidable: §3.2's demand
estimation, the $e_f/\hat e_f$ dual outputs, and §3.3's
drain-by-ceiling are all its products. Third, the EyeQ-style control
loop assumes a lag-free actuator and passively obedient traffic (a
software token bucket takes effect as written; TCP is clamped
directly by qdisc backpressure), whereas tenant-transparent RDMA
pacing is, in any realization, an inner control loop with its own
measure→apply lag, running above two live tenant CCs — the integral audit (§3.3),
quasi-static setpoints (§5.2), and a delay-matched first-order tracker
(§4.2) are all consequences of the plant having dynamics of its own.

One sentence each for the remaining neighbors: VCC/AC-DC interpose a
uniform congestion control for TCP at the vSwitch, covering neither
RDMA nor policy hierarchy; EQDS solves coexistence by credit-based
admission with a data-plane replacement — this design is its
rate-based dual: zero data-plane change, class-granular policy.

## 3 Receiver: compiling policy into targets

The receiver DPU owns two bottlenecks — the dst VMs' policy shares and
the receiver downlink. Each period it does three things — measure,
allocate, audit — and publishes its ruling as one target rate.

### 3.1 Measurement: per-flow-set arrival rates

The design puts three requirements on measurement, each traceable to
controller correctness:

1. **Granularity must match the actuators.** Enforcement is per class
   (RDMA through hardware pacing, TCP through host pacing), so counting
   must be per class. If measurement lumps two classes together while
   only one can be actuated, the controller dumps the entire error onto
   the controllable class — until it is crushed.
2. **Bytes and timestamps must come from one source and one clock.**
   Rate is a differential quantity; once byte counts and time come from
   two clock domains (or the sampling cadence is perturbed by other
   work), the division amplifies into tens of percent of rate noise —
   enough to drown the marking signal.
3. **Freshness is tiered by use.** Class-level totals feed the VQ
   integrals and must be fresh every period; the per-sender composition
   within a class only refines the third allocation layer and may lag
   by seconds.

Measurement is therefore a tiered hybrid chain. **Per-VF class-level
rates** are read directly from the NIC's hardware vport counters — RoCE
and kernel-path (TCP etc.) bytes land in two separate hardware buckets,
giving the class split natively, fresh at any sampling rate, sampled by
a DPU-local process that stamps everything with this DPU's own
monotonic clock. **Per-sender composition within a class** is refined
by sliding-window shares of flow-table (megaflow) byte counters. The
chain degrades safely stage by stage: when any source goes stale the
next one takes over, so the sampling helper can die at any time without
taking the system down. Measured accuracy: byte-exact reconciliation
against host-side counters; 0.36% steady-state error under an 8-flow
incast (§8.3).

### 3.2 Allocation: three-layer water-filling

**A worked example used throughout this section.** Receiver downlink
100G, headroom 3% ⇒ ledger capacity $C' = 97$G. Two dst tenants A and
B, equal weights, each with $MaxRate$ = 60G. Tenant A runs both TCP
and RDMA with class weights 1:1; tenant B runs RDMA only. Each class
has a single sender (layer 3 degenerates; covered separately at the
end).

#### 3.2.1 Why demand must be estimated

The definition of weighted max-min already contains "how much do you
want": pour capacity to each party by weight, **give whoever wants
less than their share exactly what they want, and re-divide what they
left over among those still unsatisfied**, repeating until done. The
bold clause *is* borrowing.

Example: 97G, four equal-weight tenants. If all want a lot, each gets
24.25G; if one of them wants only 5G, the weighted max-min answer is
5G for it and $(97-5)/3 = 30.7$G for each of the other three. To
produce that answer, the allocator **must** know "this one only wants
5G". An allocator that ignores demand entirely (dividing by weight
among the active — what minimum-guarantee systems do) always answers
24.25G each: 19.25G of capacity sits idle while three tenants starve.
In one sentence: **borrowing means handing the share someone cannot
use to someone who can, and "cannot use" is a statement about demand
— there is no way around estimating it.**

The difficulty is that demand is not directly observable: all we can
measure is the arrival rate $r_f$, and $r_f$ is capped by the very
pace the system imposed on that flow — a suppressed flow *looks* like
it has little demand, which is circular. The base estimate is
therefore

$$D_f = r_f\,(1+\delta), \qquad \delta = 15\%$$

i.e. every flow is offered a trial raise of 15% over what it currently
uses. If it can use the raise, next period's estimate rises another
15%, doubling in 5 periods ($1.15^5 \approx 2.0$, i.e. 5 ms at a 1 ms
period) — the climb is geometric, not slow. If it cannot, the estimate
stays put and the surplus is borrowed by a sibling. $\delta$ must stay
small: allocation is zero-sum, and the 15% handed to a flow that
cannot use it this period is taken from what a sibling could actually
have used.

#### 3.2.2 The ratchet bias, and the backlog test

$r_f$ is noisy: one TCP loss recovery, one RDMA burst gap, or plain
measurement jitter drops $r_f$ for a tick. And $D_f = r_f(1+\delta)$
follows a drop **immediately** while it can only climb back 15% per
step — a ratchet that falls fast and recovers slowly. At the class
layer, the sibling class cashes that asymmetry in at once:

> Tenant A's 48.5G is split 1:1 between TCP and RDMA, both saturated
> at 24.25G. On one tick TCP, recovering from a loss, only manages
> 18G:
> - TCP's class demand that tick is $18 \times 1.15 = 20.7$G <
>   24.25G ⇒ water-filling rules "TCP wants less than its share",
>   grants it 20.7G, and the remaining 3.55G goes to RDMA;
> - RDMA is backlogged and fills its 27.8G instantly;
> - next tick TCP wants to climb back, but its estimate restarts from
>   $18 \times 1.15$ and needs 2–3 ticks to reach 24.25G — and one
>   more dip in between restarts the climb.

Averaged over time, the noisier class sits systematically below its
policy weight (a class ratio of 0.75 was measured): **not because it
wanted less, but because the demand estimate misread "did not manage
it this instant" as "does not want it".**

The fix is a **backlog test** on the demand estimate. First compute a
neutral reference $\hat s_f$ — the amount this tree would give $f$ by
weight if *all* flow-sets were greedy (every demand set to infinity),
i.e. $f$'s **pure weighted share**. With every demand infinite the
only binding constraints left are the VMs' $MaxRate$, so $\hat s_f$
**depends solely on policy and the current set of active flow-sets,
never on any measured rate** — the yardstick itself is steady, moving
only when membership or policy changes, which is exactly what lets it
serve as a neutral reference. The test:

$$r_f \ge \theta_b\,\hat s_f\ (\theta_b = 0.5)\ \Rightarrow\ f\ \text{is backlogged},\ D_f \leftarrow C'$$

A backlogged flow's demand is set to the root capacity $C'$ — a
**finite** value larger than any node's share, enough to disable the
demand cap at every layer (equivalent to infinity) while remaining a
concrete number, because the $\hat e_f$ derived from it must be
broadcast to the sender as a rate (§3.4). Such a flow claims its full
weighted share and is no longer capped by its own momentary rate; a
flow below the threshold keeps
$D_f = r_f(1+\delta)$. Back to the example: when TCP drops to 18G,
$18 \ge 0.5 \times 24.25$ holds, so it is still judged backlogged and
still gets 24.25G — the sibling cannot take it, and jitter is no
longer misread as yielding. A genuinely idle flow (say 2G, far below
half its share) still reports $r_f(1+\delta)$, so its unused share is
lent out as before and borrowing is intact.

Two honest notes. First, the hard threshold produces neither chatter
nor bistability here: being judged backlogged raises the share, a
raised share raises $r_f$, and the positive feedback latches the flow
in the backlogged state; conversely a backlogged flow misjudged as
idle climbs past the threshold in about 5 ticks on the 15%/period
ladder — a short escape path. Second, the cost is real: a flow using
almost exactly half its share occupies the whole allowance and no
longer lends the unused part. $\theta_b$ is precisely the "ratio
fidelity ↔ utilization" knob, 0.5 being the measured compromise;
$\theta_b \to 0$ degenerates into a share floor (most accurate ratios,
borrowing dead), $\theta_b \to 1$ into no test at all (liveliest
borrowing, driftiest ratios).

#### 3.2.3 Four quantities: what each is, and who consumes it

| Quantity | How it is obtained | Who consumes it |
|---|---|---|
| $D_f$ demand estimate | $r_f(1+\delta)$; set above capacity if backlogged | the **input** to allocation |
| $\hat s_f$ pure weighted share | one allocation pass with **every** demand set to infinity | **only** the yardstick of the backlog test (§3.2.2) |
| $e_f$ entitled rate | the output of the allocation run on $\{D_f\}$ | ledger charge reference: only $r_f > e_f$ accrues debt (§3.3) |
| $\hat e_f$ fair-share ceiling | one pass with **$f$'s own** demand set to infinity, siblings keeping $D$ | ledger drain reference (§3.3) + anchor of the target $u_f$ (§3.4) |

Note the difference between $\hat s_f$ and $\hat e_f$: the former is
the share when *everyone* is greedy — a neutral yardstick; the latter
is the share when *only this flow* is greedy and everyone else carries
on — this flow's ceiling. They coincide only when all flows are
backlogged.

#### 3.2.4 How the hierarchical allocation is computed

The tree is walked **twice**: **bottom-up to aggregate demand** (leaf
$D_f$ summed layer by layer, with the VM layer then clipped by the
purchased allowance: cap $= \min(MaxRate_d,\ \sum\text{subtree
demands})$), then **top-down to allocate capacity** (root capacity
poured down by weight, layer by layer). Both §3.2.2's $\hat s_f$ and
this section's $e_f$ are products of these same two walks; they differ
only in what is fed into the first one — $\hat s_f$ is fed infinite
demands everywhere (so only $MaxRate$ still binds), $e_f$ is fed
$\{D_f\}$.

The allocation walk performs, at each node, **bounded weighted
water-filling**: picture pouring into a row of cups. The water level
$t$ rises uniformly, child $j$ draws $w_j\,t$ but never more than its
own cap $c_j$; whoever hits its cap first "freezes", and what it
leaves keeps being poured to the others by weight, until capacity runs
out or everyone freezes. The three layers' capacities and caps:

```
root capacity C' = downlink line rate × (1 − headroom)  ← tracks live speed changes
layer 1 (VM):       split C' across dst VMs by tenant weight W_d
                    VM d's cap = min(MaxRate_d, sum of its subtree demands)
layer 2 (class):    split the VM's share across {TCP, RDMA} by class weights
                    a class's cap = the sum of its demands
layer 3 (flow-set): split the class's share across sources by per-sender weight u
                    a flow-set's cap = D_f
```

**Borrowing, worked out.** Tenant A's TCP wants only 5G
($D_{\rm TCP} = 5$G), its RDMA is backlogged ($D$ set high), tenant B
is backlogged:

- **Layer 1**: A's subtree demand sum = 5 + large ⇒ cap
  $\min(60,\,\text{large}) = 60$G; same for B. Pouring 97G at 1:1, the
  level reaches 48.5 when capacity runs out and neither hits its 60G
  cap ⇒ **A and B get 48.5G each**.
- **Layer 2**: A's 48.5G is poured between TCP (weight 1, cap 5) and
  RDMA (weight 1, cap large). At level 5 TCP freezes (taking 5G, 10G
  consumed so far); the remaining 38.5G all goes to RDMA ⇒ **RDMA gets
  43.5G**, against a pure weighted share of only 24.25G — the extra
  19.25G is borrowed from TCP. Borrowing needs no protocol and no
  explicit message: it **is** the water-filling's redistribution of
  the surplus.
- **Layer 3**: with one sender per class, the share passes straight
  through. With several senders in one class (many-to-many), the same
  pour repeats among them by per-sender weight.

**Reclaim.** On the tick TCP's demand returns to saturation, layer 2
re-pours ⇒ 24.25G each. RDMA's $\hat e$ drops from 43.5G to 24.25G in
one step, the target $u$ follows it down (§3.4) and the tracking law
pulls the rate back; meanwhile its arrival rate 43.5G > $e$ = 24.25G,
so the ledger starts charging and the audit discount adds pressure.
**Reclaim is driven mainly by the re-division of shares (effective in
one tick); the audit only adds a penalty for overstaying.**

This example also explains why §3.3's drain must use $\hat e_f$: while
borrowing, $e_{\rm TCP}$ is only 5G, so using it as the drain
reference gives a drain rate of $e - r = 0$ and the ledger could never
empty; $\hat e_{\rm TCP} = 24.25$G (computed next), giving a drain
rate of 19.25G — one tick clears it.

#### 3.2.5 The ceiling $\hat e_f$ and the single-pass cascade

**Definition and use.** $\hat e_f$ is what the allocation gives $f$
when $f$'s own demand is replaced by infinity while its siblings keep
their current demands — this flow's **policy-permitted ceiling right
now**. In the example, for $\hat e_{\rm TCP}$: with TCP's demand set
to infinity, layer 1's cap for A becomes $\min(60,\infty) = 60$G and A
still gets 48.5G; at layer 2 neither class is capped by its own
demand, so each gets 24.25G ⇒ $\hat e_{\rm TCP} = 24.25$G while
$e_{\rm TCP} = 5$G. Always $\hat e_f \ge e_f$, with equality when all
flows are backlogged. Its two consumers are the ledger drain (§3.3)
and the target anchor (§3.4) — both need a reference that **does not
collapse with the flow's own rate**.

**The naive algorithm** re-runs the whole tree once per flow-set with
that flow's demand set to infinity: $N$ full allocations, $O(N^2)$ or
worse.

**The single-pass cascade.** At any node, the water level $t^*$
solves $F(t) = \sum_j \min(c_j,\ w_j t) = C$. $F$ is a piecewise-linear
increasing function with breakpoints at $t_j = c_j/w_j$. Replacing
child $i$'s cap with $\infty$ changes only $i$'s own term:

$$g_i(t) = F(t) - \min(c_i,\ w_i t) + w_i t$$

still piecewise-linear and increasing over the *same* breakpoints. So:
sort the breakpoints once and precompute prefix sums ($O(m\log m)$);
after that **each** child's new level is a single binary search over
that grid ($O(\log m)$), and its ceiling is $w_i t^*_i$. One sort
plus $m$ binary searches yields all $m$ children's ceilings.

Chaining the three layers: once $f$'s demand is infinite, at layer 1
the cap of the VM containing $f$ becomes $\min(MaxRate, \infty) =
MaxRate$, and at layer 2 the cap of the class containing $f$ becomes
$\infty$ — both are exactly the "replace one child's cap" form. So the
computation above is run once per layer from root to leaf, each layer
taking the previous layer's ceiling as its capacity.

The cascade follows $f$'s **root-to-leaf path**, and leaves sharing an
ancestor share its result: layer 1's query result is shared by every
leaf under that VM, layer 2's by every leaf under that class.

**What "same batch" precisely means.** Sorting once and building
prefix sums once per node yields a table; everything at that node is
then a binary search on it: **one** query finds the ordinary level
$t^*$ ($F(t^*) = C$) ⇒ every child's $e = \min(c_j,\ w_j t^*)$; **one
query per child** finds the level after replacing that child's cap ⇒
every $\hat e$. For $m$ children that is $m+1$ queries sharing one
sort and one prefix-sum array — *that* is the shared batch, not "$\hat
e$ falls out for free while computing $e$".

**Cost.** $N$ is the number of currently active flow-sets (the
leaves). The child counts across a three-layer tree sum to $O(N)$ and
each node costs $O(m\log m)$, so a single pass yielding all $e_f$ and
all $\hat e_f$ costs $O(N\log N)$ — against $N$ full allocations
($O(N^2)$ or worse) for the naive route. Measured: 448 µs per pass at
20 flow-sets (§7).

**Headroom = 3%** keeps the ledger's capacity slightly below the
physical capacity, guaranteeing that under incast the virtual queues
alarm before the switch's physical queue does — by the time the signal
reaches the senders, no physical queue has formed yet.

### 3.3 Audit: virtual-queue integration

One integrator per flow-set:

$$vq_f \leftarrow \mathrm{clip}\big(vq_f + \Delta_f\, T,\; 0,\; V\big),
\qquad s_f = \min(vq_f / V,\ 1)$$

$$\Delta_f = \begin{cases}
r_f - e_f & r_f > e_f \quad\text{(charge: by true excess)}\\[2pt]
-(\hat e_f - r_f) & 0 < r_f \le e_f \quad\text{(drain: by unused fair-share ceiling)}\\[2pt]
-V/T & r_f = 0 \quad\text{(idle: fast flush)}
\end{cases}$$

Each of the three rules has one irreplaceable reason.

**Charging integrates rather than thresholding an instantaneous
ratio**: only persistent excess raises the alarm; single-period
measurement spikes are absorbed by the integral (the AVQ lineage). The
full scale $V$ is a wall-clock integral quantity (excess × time) and
therefore **does not scale with the control period** — the same
history of excess accrues the same debt whether it is ticked as fifty
1 ms periods or one 50 ms period. $V$ is set to 600 Mbit (= 0.1 s × line rate × headroom: it takes
roughly 100 ms of headroom-scale sustained excess to saturate the
mark); the integral is clipped at $V$ itself — a deeper reservoir
only makes the mark keep firing after the excess has stopped,
deepening the undershoot.

**Draining must use $\hat e_f$, not $e_f$**, or the system deadlocks:
$e_f$ is capped by estimated demand, and demand is derived from the
measured rate — for a deeply suppressed flow, wherever $r_f$ collapses
to, $e_f$ follows, so the drain rate is forever ~15% of the actual
rate; a deep integral never drains and the audit signal stays pinned
at full scale — the flow carries the maximum discount indefinitely and
the signal loses all discrimination (under the previous law form this
manifested as a flow permanently pinned to the floor, reproduced in
experiment). With $\hat e_f$, "the share you are
not using" is the drain rate: a suppressed flow has a large unused
share, drains fast, unmarks, and recovers; a flow using its full share
($r_f \approx \hat e_f$) drains at essentially zero and gets no free
unmarking.

**Idle flows flush fast**: a flow that sends nothing neither charges
nor can drain by share, so its queue would freeze at a stale value.
Let it flush at $V$ per period — a flow that stops sending abandons
its congestion claim; stale infrastructure state must not keep
punishing the application when it resumes (the fail-open spirit).

Three scenarios unify inside this one mechanism. **Policy-share
contention** (zero physical congestion): the VM-layer cap makes the
over-quota flow-set's $r_f > e_f$ — the ledger synthesizes scarcity.
**Incast**: root capacity is oversubscribed and the shortage is
apportioned into every $e_f$ by weight; headroom guarantees the ledger
alarms before the physical queue. **Borrowing and reclaim**: a direct
product of layer-2 work-conservation — the borrower is pushed back by
re-divided $e_f$ and the resulting marks the moment the lender's demand
returns, with no explicit protocol.

### 3.4 Target synthesis and telemetry

The receiver compresses its entire ruling for each flow-set into **one
target rate**:

$$u_f = \hat e_f\,(1 - \gamma\, s_f)$$

$\hat e_f$ is the policy-ruled share ceiling (§3.2), $s_f$ is the
ledger-audited persistent excess (§3.3), and $\gamma$ = 0.25 caps the
audit discount. With no excess, $u_f = \hat e_f$ — the target is the
fair-share ceiling; persistent excess pushes the target down to at
most $(1-\gamma)\hat e_f$ — the audit dose is bounded, and it enters
the target as a discount, never as a jump. Why the anchor is
$\hat e_f$ and not $e_f$ is explained at the end of §4.2.

Every period, telemetry sends each sender DPU one datagram with **two
numbers** per active flow-set: $\{u_f,\ r_f\}$. $u_f$ drives the
tracking law (§4.2); $r_f$ is the receiver-measured class-level
arrival rate, which the sender forwards to the RDMA executor as its
rate feedback (§5.2) and uses as the demand input of the sender tree
(§4.3).

Two protocol-level decisions. **The record itself is the explicit
"fresh and permitted" permit** — $u_f$ is sent every period, including
when there is no excess; every sender action is conditioned on
receiving it, so a telemetry blackout automatically lands the system
in fail-open (§4.4). **Membership in the report is
decided by the existence of flow-table keys, not by instantaneous
rate** — a paced flow shows zero-rate gaps at millisecond granularity,
and dropping its record on such a gap would make the sender mistake the
gap for a blackout, a positive feedback into fail-open. Records are
fixed-size binary (at a millisecond period, encode/decode cost must be
negligible).

Telemetry shares the physical path with the controlled data (in-band;
0.1 ms steady-state round trip): production environments have no
separate management network, and the design must not assume one.
Telemetry interruption caused by path failure is absorbed by the
fail-open semantics.

## 4 Sender: the unified tracking law

### 4.1 Reframing: setpoint tracking, not fairness search

In classic congestion control the sender does not know its fair share;
the law must bring its own "converge to fairness" dynamics — all the
complexity of increase/decrease skeletons (the AIMD family) stems from
this. HPFT is not that setting: the fair share is computed explicitly
by the receiver's water-filling and, together with the audit discount,
compressed into a target $u_f$ broadcast every period. Only one job
remains for the sender —

> track the rate cap $R_f$ quickly, stably, and without overshoot onto
> a target $u_f$ that is externally known but 1–2 periods stale.

The classic Chiu–Jain analysis of increase/decrease fairness does not
apply here because its premise — all sources share one binary feedback
signal — is dissolved by per-flow differential rulings: every flow
receives its own target, and fairness is already encoded in the target
set. Change the criterion and the optimal rule changes with it: no
"fairness-searching" increase/decrease dynamics are needed, only a
low-pass tracker of a stale broadcast target — §4.2 is the minimal
form of that tracker.

### 4.2 The law

Each flow-set carries a rate cap $R_f$, initialized to
$R_f = Tree_f$ (optimistic start: short flows pay no ramp-up tax; if
the flow lands on a congested resource, the target arrives within 1–2
periods and pulls it back, and the transient excess is absorbed by the
tenant's own CC). On every telemetry record the sender does exactly
one thing — **it takes one step toward the target on the log axis**:

$$R_f \leftarrow R_f\left(\frac{u_f}{R_f}\right)^{kT}
\qquad\Longleftrightarrow\qquad
\dot R_f = k\, R_f \ln\frac{u_f}{R_f}$$

This is first-order low-pass tracking in log coordinates (with
$z = \ln R_f$ it is the textbook $\dot z = k(\ln u_f - z)$): no
branches, no clamps, a single parameter $k$ = 20 s⁻¹ (time constant
50 ms; within the 5% band in about $3/k$ = 150 ms). Log coordinates
for the same reason as everywhere in this design: they are the natural
space of multiplicative dynamics — shares of any size see the same
relative step and the same convergence wall-clock, both classes share
one constant, and the law contains no weights (they live only where
shares are computed, §2.1). The implementation adds one absolute floor
orthogonal to the law (50 Mbps, §6) against executor stall; it takes
no part in the dynamics.

The three problems for which the previous form (an increase/decrease
skeleton) needed "walls" do not exist in the tracking form:

**No cap needed.** Tracking approaches asymptotically instead of
crossing — approaching $u_f$ from below, the step is proportional to
the remaining log distance and $R_f$ never passes the target. The
"climb at full speed, hard-stop at the ceiling" clamp disappears,
while the instability family "probe past the share → signal storm →
deep cut" remains structurally impossible (no state ever sits above
the target).

**No floor needed.** The downward approach is equally asymptotic, and
the audit dose is itself bounded — $u_f \ge (1-\gamma)\hat e_f$
(§3.4), so enforcement can push a flow at most to 75% of its
fair-share ceiling, and "a persistent signal compounds the rate
through the target" is structurally impossible. The old bottoming
clamp, and the monotonicity patch it required (finding 1 of the fluid
analysis), retire as a family.

**No jump protection needed.** When the target moves, $R_f$ follows
smoothly with time constant $1/k$ and never jumps; the accident
entrance "several flows jump onto mutually referenced, stale ceilings
at once" (the experimentally condemned direct jump) is closed by the
time constant itself.

Two tuning properties (derivations in `design_theory.md`):

**$k$'s ceiling comes from target staleness.** Every flow's $u_f$ is
computed under a "siblings stay put" assumption and is
$\tau_{\rm eff}$ ≈ 15–25 ms stale. Far from the target, log tracking's
absolute climb speed is self-bounded by $x\ln(u/x) \le u/e$ ($e$ =
Euler's number), so the joint climb of all flows never exceeds
$(k/e)\sum u$ — self-normalized by the targets, not amplified by flow
count. Staleness budget: $k\,\tau_{\rm eff} \lesssim 0.4$; the
production value sits inside it.

**The audit loop's damping is tunable.** The tracking term $-kz$
supplies exactly the damping that the "integral ledger + proportional
discount" loop lacks (without it, that loop is a zero-damping
oscillator; `design_theory.md` §1.4). Linearized, the loop is a
standard second-order system with closed-form damping ratio
$\zeta = \tfrac12\sqrt{kV/(\gamma\hat e_f)}$: $\gamma$ = 0.25 makes
typical shares (~10 G) overdamped; large shares are underdamped, but
the residual oscillation is confined to the discount band
$[(1-\gamma)\hat e_f,\ \hat e_f]$. The old form held oscillation down
with walls; the new form removes it with structural damping.

Anchoring the target at the infinite-demand $\hat e_f$ gives nothing
away: $R_f$ is a cap, not a grant — raising the cap of a
demand-limited flow sends not one extra byte, and the ledger allocates
from the measured $r_f$, lending out whatever the flow does not use
(§3.2). The anchor must also be $\hat e_f$ rather than the
demand-capped $e_f$ — the latter collapses with the flow's own rate
and would reproduce the self-referential trap of §3.3's drain
reference.

### 4.3 The sender tree and the two-layer model

The sender DPU owns the uplink. The same water-filling runs a local
tree (src VM $MaxRate$ and class weights → $Tree_f$), and enforcement
takes $pace_f = \min(R_f,\ Tree_f)$ — "the tightest bottleneck binds"
(§2.1) lands here: the receiver's constraint arrives via $R_f$, the
sender's via $Tree_f$, and the tighter one takes effect. The tree's demand
input is the telemetry-carried $r_f$ plus the growth margin, with a
saturation probe: a flow-set using ≥ 85% of its pace advertises demand
$pace(1+\delta)$, expressing appetite to grow. $Tree_f$ is the
sender-side fair-share **ceiling** ($f$'s demand set to infinity), not
the demand-capped share — both the new-flow start ($R_f = Tree_f$) and
the fail-open ramp target require it to be "the allowance policy
permits": under demand-capping, a new flow with $r=0$ would get
$Tree \approx 0$, a contradiction.

Sender-side policy is enforced in two layers, separating semantics from
the safety net:

- **Layer 1 (hard cap).** Per-VM $MaxRate_i$ is enforced by the NIC's
  hardware rate limiter — class-blind, static, independent of the
  software stack. The **selling principle** (no VM ever sends beyond
  its purchased allowance, even when others are idle) is guaranteed in
  hardware; software failure cannot break the contract.
- **Layer 2 (allocation semantics).** The software tree above performs
  oversubscription shrinkage, inter-class soft isolation, and borrowing
  within the hard caps. Both classes' executors (RDMA / TCP) hang off
  the same sender agent, so inter-class borrowing is the tree's
  work-conserving refill itself — no separate arbiter component exists.

### 4.4 Fail-open

The law's increase is conditioned on fresh, uncongested telemetry
(§3.4), so a telemetry blackout has a well-defined degradation path: if
a flow-set receives no record for $N_1$ consecutive periods (0.25 s),
$R_f$ freezes (no increase, no decrease); after $N_2$ periods (2 s),
$R_f$ ramps at a fixed slope toward $Tree_f$. Semantics: when the
receiver DPU or the telemetry path fails, traffic degrades to being
constrained only by sender-local policy — tenants are not wedged
(fail-open), yet nothing runs away ($Tree_f$ caps the ramp, and tenant
CC still operates). Once the receiver returns, the first mark restores
control.

## 5 Executors

### 5.1 The contract and the two-loop split

Both executors obey one contract: **pace delays packets, never drops
them**; the actual send rate = min(what tenant CC wants, $pace_f$). The
tenant's congestion control keeps running underneath the pace, and the
two loops do not excite each other: to tenant TCP the pace looks like a
smooth pipe (the OnRamp-style argument); for RDMA, the
min(DCQCN, budget) composition has been validated by long-term
operation on this testbed. The division of labor cuts along
timescales — **sub-period bursts and fabric congestion belong to
tenant CC** (CNP reaction is sub-millisecond); **policy shares at
period scale and above belong to HPFT**.

### 5.2 RDMA: water-level control of per-pair budgets

RDMA pacing lands on the DPU's programmable congestion-control (PCC)
engine, invisible to tenants. The sender agent pushes $pace_f$ down as
a **per-pair budget**; the device maintains one water level $level$ per
pair, applies rate = min($cc$, $level$) to **every QP** of the pair,
and closes the loop on the measured pair-aggregate rate: shrink the
level proportionally when the aggregate exceeds the budget, raise it in
small steps when below. The essence of water-level control is **O(1)
state, no flow counting**: the equilibrium level lands on budget/N
automatically, so the aggregate converges to the budget for any QP
count — this is where §2.3's flow-count immunity is cashed in at the
executor (the anti-collapse floor must likewise clamp the *aggregate*
rate, not the per-QP rate, or floor × N becomes the escape channel).
Rate feedback prefers the receiver-measured value carried back in
telemetry: it shares a source with the marks and is immune to
sender-side event-sampling loss.

The $cc$ term is a device-side DCQCN-flavored state machine
(CNP-triggered multiplicative decrease normalized by QP count, periodic
additive recovery) serving as the event-speed backstop for fabric
congestion, orthogonal to the policy level (its own fairness measured
at 0.98–0.99 across 16–256 flows).

The rate difference between the policy channel and the congestion
channel is deliberate: budget updates ride the slow path (the device
mailbox absorbs batches at millisecond scale; the agent coalesces to
~13 ms per push and skips rewrites that move the budget by less than
3%), while congestion response stays at event speed. **Budgets must be
quasi-static** — the executor treats every budget write as a cap
change and resets its own settling window, so an outer loop rewriting
the budget at millisecond rates would keep the inner loop out of
steady state forever. Coalescing plus hysteresis quantizes the policy
channel to a cadence the executor can digest, and policy shares are
slow variables anyway, so the down-rating costs no semantics.

**The descent direction carries no rate limit.** When the target drops,
the budget follows in one step and the executor's proportional
feed-forward (scaling its level by the ratio of new to old budget)
catches up in one step — for a monotone descent that is both correct
and fastest. The outer loop is already smooth (the tracking law moves
with time constant $1/k$, §4.2), so there is no high-frequency
back-and-forth needing extra damping; adding a descent ramp would only
make the executor, rather than the law, the convergence bottleneck and
throw away the law's descent capability.

### 5.3 TCP: host-side EDT pacing

TCP pacing lands in the sender host's kernel: an eBPF program keeps
per-pair Earliest-Departure-Time tokens, stamps each packet's departure
time, and the fq qdisc enforces it (a mature delay-only shaping path;
zero copy, zero drops). The sender agent writes rates into the BPF map
across a one-hop UDP bridge (tens of microseconds steady state), so the
control-loop shape is identical to the RDMA side; only the enforcement
point differs. Sparse flows — inter-packet gaps above 40 µs — bypass
the shared pacing: latency-sensitive small flows pay no queueing tax
for the shaping of throughput flows. Placing the TCP executor on the
host rather than the DPU is a settled architectural conclusion (the
negative result on DPU-side TCP offload is recorded in
`docs/archive/superseded-designs.md`).

## 6 Parameters

The parameter structure follows three calibration principles:
**parameters are wall-clock quantities** (rate constants in s⁻¹,
integral quantities in bits, independent of the control period;
changing the period requires no retuning, §4.2); **integral quantities do not scale with the period**
($V$ is a wall-clock integral, §3.3); **fairness before utilization**
(options that trade ratio fidelity for utilization are rejected
outright; the ~87% aggregate utilization under two-class concurrency is
caused by the tenant TCP's own loss-recovery gaps and cannot be bought
back on the ledger side). Production values (ground truth is the
`e_params` section of `lab-registry.json`, migrated to these values on
2026-07-27 together with the law itself):

| Parameter | Value | Role |
|---|---|---|
| $T$ | 1 ms | control period; total loop lag ≈ 1–2 $T$ |
| $k$ | 20 s⁻¹ | tracking bandwidth (time constant 50 ms; staleness budget $k\tau_{\rm eff} \lesssim 0.4$) |
| $\gamma$ | 0.25 | audit-discount cap (target floor $(1-\gamma)\hat e_f$; damping $\zeta = \tfrac12\sqrt{kV/(\gamma\hat e_f)}$) |
| $\delta$ | 0.15 | demand growth margin (a zero-sum tax; must stay small, §3.2) |
| $\theta_b$ | 0.5 | backlog test threshold (fraction of pure weighted share, §3.2) |
| headroom | 3% | ledger capacity's concession to physical capacity |
| $V$ | 600 Mbit (= 0.1 s × line rate × headroom) | mark full scale; integral clip = $V$ |
| $N_1$, $N_2$ | 0.25 s, 2 s | fail-open freeze / ramp thresholds |
| pace floor | 50 Mbps | enforcement lower bound; keeps executors out of their stall regions |

## 7 Implementation at a glance

| Component | Runs on | Form |
|---|---|---|
| rx agent (measure · allocate · mark · telemetry) | receiver DPU Arm | Python, single-process 1 ms loop |
| vport sampler (hardware class-bucket counters) | receiver DPU Arm | C, shared-memory publisher |
| tx agent (response law · sender tree · dispatch) | sender DPU Arm | Python |
| RDMA executor (per-pair water-level control) | sender DPU DPA | DOCA PCC device C |
| TCP executor (EDT pacing) + bridge | sender host | tc eBPF + Python |

The control plane totals about 1,800 lines of Python (the v2 migration
deleted v1's three law skeletons and their clamps, a net −300 lines); the
executors about 1,100 lines of DPA C and 250 lines of eBPF C. The single source
of truth for policy and parameters is the registry file (the repo copy
is authoritative; the receiver hot-reloads the policy section on file
mtime). At the 1 ms period each agent stays below 9% of one core; one
scheduling pass takes 448 µs at 20 flow-sets; the comfortable scale is
several hundred flow-sets. The code map is in
`hpft-implementation/README.md`; platform defects and experiment
hygiene are in `ops_notes.md`.

## 8 Properties

### 8.1 Equilibrium allocation = policy shares

On a single controlled resource this is the classic setting of
"AQM with excess-differential marking + per-source response laws":
flow-sets above $e_f$ are pushed back by the audit discount; those
below $\hat e_f$ unmark and their targets recover; the equilibrium is
$r_f \approx e_f$, and water-filling guarantees $e_f$ equals the
hierarchical weighted max-min share. Borrowing needs no extra
mechanism: an idle class's small demand refills the sibling's $e_f$
through work-conservation, so the borrower's target rises undiscounted;
when demand returns, the shares are re-divided and the lowered target
pulls the borrower back.
Across multiple bottlenecks, "the tightest bottleneck binds" follows
from $pace = \min(R, Tree)$: each bottleneck rules independently and
enforcement takes the minimum, with no cross-host weight comparison.

### 8.2 Stability

The primary loop (tracking) is a first-order low-pass: unconditionally
stable, structurally overshoot-free, with no parameter cliff. The
audit loop (ledger integral + discount) linearizes to a standard
second-order system with tunable damping
$\zeta = \tfrac12\sqrt{kV/(\gamma\hat e_f)}$ (§4.2), and the bounded
discount confines any residual oscillation to the band
$[(1-\gamma)\hat e_f,\ \hat e_f]$. Two honest weaknesses. First, the
tracking law carries no built-in fairness safety net — correctness
depends on the quality of the receiver's ruling (shares and audit),
and the law will not self-correct a distorted target; this is why
measurement accuracy and fail-open are on the critical path rather
than optional (§3.1, §4.4). Second, the $k\tau_{\rm eff}$ staleness
budget and $\zeta$'s share dependence give analytic stability
boundaries (`design_theory.md`, Chinese working document), but the
measured margin of the production operating point has not yet been
mapped systematically.

### 8.3 Measured results

The table below was measured on the **v1 transitional form (the MIMD
law plus the executor descent ramp)**; provenance in
`evaluation_index.md` and the per-directory `results/*/summary.md`.
v2 leaves the fairness, measurement, fail-open, and overhead
dimensions mechanically unchanged; the convergence dimension changes
with the new law and the removal of the descent ramp — predictions
(~130 ms up, ~150 ms down, symmetric) and the revalidation plan are in
`design_theory.md`:

| Dimension | Result |
|---|---|
| Step convergence (20G↔10G, competitor leaves/joins) | up ~59 ms; down ~580 ms |
| Incast symmetric fairness (8 flows, sustained contention) | Jain 0.999–1.000 |
| Measurement accuracy | byte-exact reconciliation vs. host counters; 0.36% steady-state error under incast |
| Telemetry | 0.1 ms in-band round trip; one fixed-size record per flow-set per period |
| Fail-open timing | freeze at 0.25 s of blackout, ramp at 2 s; back under control within 1 s of recovery |
| Control-plane overhead | each agent < 9% of one core; one scheduling pass 448 µs at 20 flow-sets |

## 9 Boundaries and extensions

**Core-network bottlenecks (the non-guarantee of §1.3).** Real CE/RTT
signals in the core carry no policy scale; there, the sender should
switch to a *weighted* response law (weights configured by the
operator), naturally separated from the edge's synthesized marks —
synthesized marks ride the telemetry channel, fabric signals ride
measurement. The mechanism is laid out but not evaluated on this
testbed (see `archive/rd_fairness_design_c.md` §4).

**Named configuration modes.** Demand-side "share floors" (ledger
shares never shrink, at the cost of killing temporary borrowing during
dips) and per-class step sizes (utilization-first, at the cost of ratio
drift) are retained as named modes for specific SLA scenarios; neither
is default (trade-off data in `docs/archive/`).

**Known limitations.** Sub-period burst precision is not promised
(tenant CC absorbs it); policy changes take effect at period
granularity, with no atomic multi-bottleneck switch; VM-migration
handling and multi-receiver-owner scale validation (the current testbed
is single-sender single-receiver, 8 VFs) are future work.

**Planned components.** Three committed extensions are specified in
`planned_extensions.md` (a Chinese working document): (i) a third
marking owner and a measured demonstration of multi-owner composition
(turning §2.1's "every bottleneck keeps its own ledger, enforcement
takes the minimum" from a claim into a result); (ii) an
adversarial-tenant analysis that bounds the worst-case over-share
strategic traffic can extract from demand estimation and the VQ rules
(hardening the soft spot §8.2 admits); (iii) online policy
reprogramming — runtime weight/cap changes converging at step-response
speed, making policy a live control input.

## 10 Document genealogy

**Live documents** (maintained alongside this one):

- `design_theory.md` — fluid-model analysis of the response loop
  (tutorial-style, Chinese): general method, equilibrium/convergence/
  stability derivations reconciled against measurements, parameter
  rules. The rationale for §6's parameter values lives here; its
  critical findings (the clamp family's monotonicity defect,
  slew-bound down-steps, the drain dead-zone) prompted the
  track-and-audit law of §3.4 and §4.2.
- `ops_notes.md` (required reading before touching the lab) and
  `cc_mode_switching.md` (CC-mode switching).
- `planned_extensions.md` — the three committed but unstarted
  extensions.
- `evaluation_index.md` — index of paper material; its numbers date
  from the old configuration era, with generations and trustworthiness
  in `hpft-implementation/results/EXPERIMENT_STATUS.md`.

**Archived** (`archive/`, for provenance only — mechanisms and
parameters are all outdated): the original Scheme-E design with the
Q1–Q27 decision log, Scheme C, the AIMD/MIMD law analyses, abandoned
directions, tuning and diagnostics logs, handoff history. Law lineage:
AIMD → (2026-07-13) MIMD → (2026-07-24) the log-space tracking law;
MIMD is the thoroughly measured transitional form (§8.3's numbers are
its record), and the three-law comparison is in
`archive/response_law_mimd_analysis.md`.

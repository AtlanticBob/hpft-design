# HPFT: Virtual-Queue Fairness at the DPU Edge — Design

**Version 3.0 (2026-07-23).** This document is rewritten against the
system as currently implemented (the `hpft-implementation` repository:
MIMD response law, 1 ms control period, direct vport-counter
measurement) and is the sole authoritative design description of the
running system. The decision log from the design-freeze period (Q1–Q27)
is archived in `rd_fairness_design_e.md`; abandoned directions and
historical evolution live in `docs/archive/`; neither is maintained as
part of this document. Operational procedures are out of scope — read
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
| Mark $s_f$ | The signal the owner sends to $f$'s sender, in $[0,1]$, meaning "the degree to which $f$ has persistently exceeded its entitled rate" |
| Response law | The fixed rule by which a sender adjusts the rate cap $R_f$ from telemetry (§4.2) |
| pace | Limiting a flow-set's rate by spacing packet departures; delay only, never drop |
| Tree share $Tree_f$ | The share the sender's local tree gives $f$, i.e. the product of sender-side uplink policy (§4.3) |
| Work-conserving | Capacity never idles while unmet demand exists; "one class idle, the other may borrow" is this property |
| Headroom | Deliberately allocating slightly below physical capacity (3%) so virtual queues alarm before physical ones |
| Fail-open | On control-channel failure the system degrades to "no rate limiting" rather than wedging in a limited state |
| Control period $T$ | The beat of one measure→allocate→mark→respond loop, currently 1 ms; $T_0$ = 50 ms is the reference period for parameter calibration (§4.2) |

## 1 Problem

### 1.1 Goal: two-level fairness

In a public cloud, tenant VMs generate both TCP and RDMA (RoCEv2,
lossy, no PFC) traffic that shares the network without switch-queue
isolation. The operator must provide, simultaneously:

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
RDMA (DCQCN, reacting to ECN at millisecond timescales) split a shared
bottleneck as a *by-product* of two algorithms colliding: the outcome
is arbitrarily sensitive to configuration such as ECN thresholds, and
swings between extremes across scenarios (measured: under a 32-flow
incast, Cubic can crush native DCQCN to 0.3% of the bottleneck). No
amount of parameter tuning on either side aligns the collision outcome
with the operator's weight intent.

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

HPFT in one sentence: **compute the shares, manufacture the signal,
and let every sender respond with one rule.**

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
marks** (the response law, §4): unmarked, rise — up to your share
ceiling; marked, back off in proportion to the mark. The rule contains
no weights and no class distinction — the weights were already spent
when the shares were computed. TCP and RDMA are thus shaped by the
same hand (each one's own congestion control keeps running underneath
the pace, §5.1), and "who gets how much" at a shared bottleneck stops
being a by-product of two colliding congestion controls. This removes
the second obstacle of §1.2.

Multiple bottlenecks need no extra machinery: every bottleneck along a
flow's path keeps its own ledger and expresses its own constraint —
the receiver's arrives as marks, the sender's is its locally computed
tree share — and the enforced rate is the minimum of what the
bottlenecks ruled, so the tightest one binds automatically. At no
point does the system compare weights across hosts (each side
normalizes its own; they were never comparable).

The structural property the rest of this document leans on also falls
out here: **fairness lives in the marks, not in the rule.** The
deserved share is computed explicitly at the receiver and broadcast in
telemetry, so the sender never has to search for the fair point — it
only tracks the broadcast value. This single fact dictates the choice
of response law (§4.1).

### 2.2 The control loop

```
sender host          sender DPU                            receiver DPU
┌─────────┐   ┌────────────────────────────┐   wire   ┌───────────────────────────┐
│ tenant VM│──▶│ ④ pace_f = min(R_f, Tree_f) │─────────▶│ ① measure per-flow-set r_f │
│ (TCP/    │   │    R_f ← MIMD(s_f, ê_f) §4.2│          │ ② allocate: water-filling  │
│  RDMA CC)│   │    Tree_f ← local tree §4.3 │          │        → e_f, ê_f     §3.2 │
└─────────┘   └──────────────▲─────────────┘          │ ③ mark: VQ integral → s_f  │
      ▲          telemetry    │                        └────────────┬──────────────┘
      └── TCP pace ──(in-band)┴── {f: s_f, r_f, e_f, ê_f} per period ┘         §3.4
```

One round (period $T$ = 1 ms): the receiver DPU measures each
flow-set's arrival rate $r_f$ (①) → three-layer water-filling yields
the entitled rate $e_f$ and the fair-share ceiling $\hat e_f$ (②) →
virtual queues integrate the excess and produce marks $s_f$ (③) →
telemetry returns to the senders → each sender updates its rate cap
$R_f$ and enforces the pace (④) → next period's $r_f$ reflects the
effect. Total loop delay is roughly 1–2 periods; the marks compensate
for exactly this staleness (§4.2).

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
measure→apply lag, running above two live tenant CCs — integral
evidence (§3.3), quasi-static setpoints (§5.2) and the two-sided
clamped, overshoot-free law (§4.2) are all consequences of the plant
having dynamics of its own.

One sentence each for the remaining neighbors: VCC/AC-DC interpose a
uniform congestion control for TCP at the vSwitch, covering neither
RDMA nor policy hierarchy; EQDS solves coexistence by credit-based
admission with a data-plane replacement — this design is its
rate-based dual: zero data-plane change, class-granular policy.

## 3 Receiver: compiling policy into marks

The receiver DPU owns two bottlenecks — the dst VMs' policy shares and
the receiver downlink. Each period it does three things: measure,
allocate, mark.

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

**Demand estimation.** If the target were only minimum guarantees,
allocation could sidestep demand entirely — divide capacity among the
active parties in proportion to their guaranteed weights. This
design's target is weighted max-min with borrowing: how much each
flow-set currently wants must enter the allocation, so demand
estimation is unavoidable; yet all the system can observe is the
actual arrival rate $r_f$ — a quantity capped by the flow-set's own
current pace. The base estimate is
$D_f = r_f\,(1+\delta)$ with a growth margin $\delta$ = 15%: a
suppressed flow-set gets room to grow, its estimate rises with it next
period, until it reaches its fair share. $\delta$ must stay small:
share granted to one flow-set that it cannot yet use is share taken
from what a sibling could have actually used (allocation is zero-sum).

$r_f(1+\delta)$ alone has one systematic bias. A **backlogged** class
(one that wants to send more) with fewer flows or a lower realized rate
gets its class-demand sum capped below its weighted share; the sibling
class borrows the difference, and the class ratio drifts off the policy
weights. Demand estimation therefore performs a backlog test: let
$\hat s_f$ be $f$'s pure weighted share under the all-backlogged
assumption; if $r_f \ge \theta_b\,\hat s_f$ ($\theta_b$ = 0.5), $f$ is
judged backlogged and its demand is treated as infinite (it claims its
full weighted share). A genuinely idle flow-set
($r_f \ll \hat s_f$) keeps $r_f(1+\delta)$, so work-conserving
borrowing is unaffected.

**Hierarchical allocation.** The tree has three layers; each performs
bounded weighted water-filling (pour by weight; whoever's demand is
satisfied first freezes, and the remainder is re-poured, until capacity
is exhausted):

```
capacity C_root = downlink line rate × (1 − headroom)   ← tracks live speed changes
layer 1 (VM):       split C_root across dst VMs by tenant weight W_d,
                    each VM capped at min(MaxRate_d, sum of its demands)
layer 2 (class):    split the VM share across {TCP, RDMA} by class weights;
                    a class with insufficient demand cedes the surplus
                    to the other (this IS borrowing)
layer 3 (flow-set): split the class share across sources by per-sender
                    weights u
```

Allocation outputs **two** quantities: the entitled rate $e_f$ (the
demand-capped share; the charge reference for marking) and the
fair-share ceiling $\hat e_f$ (the share recomputed with $f$'s own
demand set to infinity; the drain reference for marking and the
setpoint of the response law). $\hat e_f$ is computed for all flow-sets
in the same pass as $e_f$ by a single-pass cascade, at O(N log N)
instead of one full refill per flow-set.

Headroom = 3% keeps the ledger's capacity slightly below the physical
capacity, guaranteeing that under incast the virtual queues alarm
before the switch's physical queue does — by the time marks reach the
senders, no physical queue has formed yet.

### 3.3 Marking: virtual-queue integration

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
1 ms periods or one 50 ms period. $V$ is anchored to the reference
period and set to 600 Mbit (2 $T_0 \times$ line rate $\times$
headroom); the integral is clipped at $V$ itself — a deeper reservoir
only makes the mark keep firing after the excess has stopped,
deepening the undershoot.

**Draining must use $\hat e_f$, not $e_f$**, or the system deadlocks:
$e_f$ is capped by estimated demand, and demand is derived from the
measured rate — for a deeply suppressed flow, wherever $r_f$ collapses
to, $e_f$ follows, so the drain rate is forever ~15% of the actual
rate; a deep integral never drains, the mark never clears, and the flow
is pinned to the floor permanently. With $\hat e_f$, "the share you are
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

### 3.4 Telemetry

Every period, the receiver DPU sends each sender DPU one datagram
carrying records for all of that sender's active flow-sets:
$\{f:\ s_f,\ r_f,\ e_f,\ \hat e_f\}$. Each field has a distinct job:
$s_f$ drives the response law; $\hat e_f$ is the law's setpoint
(§4.2); $r_f$ is the receiver-measured class-level arrival rate, which
the sender forwards to the RDMA executor as its rate feedback (§5.2)
and uses as the demand input of the sender tree (§4.3); $e_f$ is for
diagnostics.

Two protocol-level decisions. **$s_f = 0$ is transmitted too** — it is
the explicit "fresh and uncongested" permit; the sender's rate increase
is conditioned on receiving it, so a telemetry blackout automatically
lands the system in fail-open (§4.4). **Membership in the report is
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

## 4 Sender: the unified response law

### 4.1 Reframing: setpoint tracking, not fairness search

In classic congestion control the sender does not know its fair share;
the law must bring its own "converge to fairness" dynamics — that is
the entire basis on which the Chiu–Jain analysis favors AIMD over MIMD.
HPFT is not that setting: the fair share is computed explicitly by the
receiver's water-filling, and the fair-share ceiling $\hat e_f$ is
broadcast in telemetry every period. Only one job remains for the
sender —

> track the rate cap $R_f$ quickly, stably, and without overshoot onto
> a setpoint $\hat e_f$ that is externally known but 1–2 periods stale.

Chiu–Jain does not apply here because its premise — all sources share
one binary feedback signal — is dissolved by per-flow differential
marking: every flow receives its own signal, and fairness is already
encoded in the set of setpoints. Change the criterion and the optimal
law changes with it: no need for AIMD's fairness search, only for
dynamics that hug a known target — which is precisely MIMD's strength.

### 4.2 The law

Each flow-set carries a rate cap $R_f$, initialized to
$R_f = Tree_f$ (optimistic start: short flows pay no ramp-up tax; if
the flow lands on a congested resource, marks arrive within 1–2 periods
and push it back, and the transient excess is absorbed by the tenant's
own CC). On every telemetry record:

$$
R_f \leftarrow
\begin{cases}
\min\!\big(R_f\,(1 + \alpha\,T/T_0),\ \hat e_f\big) & s_f = 0
  \quad\text{(multiplicative increase, hard-capped at the setpoint)}\\[4pt]
\max\!\big(R_f\,(1 - \beta\, s_f)^{\,T/T_0},\ \hat e_f\big) & s_f > 0
  \quad\text{(multiplicative decrease, hard-floored at the setpoint)}
\end{cases}
$$

Production parameters $\alpha$ = 0.6 and $\beta$ = 0.15 are calibrated
as wall-clock doses at the reference period $T_0$ = 50 ms: increases
scale by $T/T_0$, decreases exponentiate by $T/T_0$, so per-second
compounding behavior is identical at any control period — moving from
50 ms to 1 ms required retuning nothing. The law contains no weights
(they live only where shares are computed, §2.1) and no per-class
constants: multiplicative steps are scale-invariant by nature, so both
classes and shares of any size run the same constants.

The law is a composition of "setpoint + mark", two signals with
distinct, non-substitutable jobs:

**$\hat e_f$ is the computed "entitlement" and provides two-sided
clamping.** Upward, the multiplicative increase stops dead at
$\hat e_f$ — $R_f$ never sits above the ceiling, so the entire AIMD
failure family "probe past the share → marking storm → deep cut →
relaxation oscillation" **structurally cannot arise** (collapse
immunity is obtained by construction, not by tuning). Downward, the
multiplicative decrease bottoms out at $\hat e_f$ — mark clearance
always lags the rate's return (the queue must drain first), and without
this floor a persistent mark would compound $R_f$ through the target,
paying the debt back later through a slow climb.

**$s_f$ is the measured "excess" and provides the downward dose.**
$\hat e_f$ is a computed value, one to two beats stale and
cross-referenced among flows; the mark is the factual record of excess.
The speed of descent is set by mark intensity, not by jumps in
$\hat e_f$, so a transiently distorted $\hat e_f$ does not translate
into a violent cut.

**Why the upward path still approaches gradually instead of jumping
$R_f := \hat e_f$.** The setpoint is stale and mutually referenced:
every flow's $\hat e_f$ is computed under the assumption that its
siblings stay put. If several flows jump to their own current ceilings
simultaneously, the joint result instantly overshoots root capacity and
triggers a synchronized marking storm — exactly the failure mode
gradual convergence exists to prevent (experimentally confirmed: the
direct jump produces non-deterministic severe starvation under
sustained incast contention). The geometric climb is, in essence, a
**low-pass filter tracking a stale broadcast setpoint**: it approaches
with a time constant about an order of magnitude above the loop lag,
buying robustness against setpoint staleness. Gradualism is a
necessary condition for stability, not conservative legacy.

In log coordinates the law is a piecewise-linear map with bounded steps,
monotone toward its respective anchors: multiplicative step size is
independent of share size (a 0.5G share and a 40G share see the same
relative step and the same convergence wall-clock), the composite
system has a stable fixed point, and the law by itself produces no
limit cycle. The only oscillation source is external marking lag, which
is bounded (measured convergence in §8.3; the full theory treatment is
in `response_law_mimd_analysis.md`).

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
~13 ms per push, with change hysteresis and a descent slew limit),
while congestion response stays at event speed. **Budgets must be
quasi-static** — the executor's inner loop has its own measure→apply
lag, and if the outer loop's target jitters at a comparable frequency
the two loops couple into a limit cycle. Policy shares are slow
variables anyway; down-rating them loses no semantics.

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
**calibrate in wall-clock terms** (all rate/timeout parameters
reference $T_0$ = 50 ms; changing the running period requires no
retuning, §4.2); **integral quantities do not scale with the period**
($V$ is a wall-clock integral, §3.3); **fairness before utilization**
(options that trade ratio fidelity for utilization are rejected
outright; the ~87% aggregate utilization under two-class concurrency is
caused by the tenant TCP's own loss-recovery gaps and cannot be bought
back on the ledger side). Production values (ground truth is the
`e_params` section of `lab-registry.json`):

| Parameter | Value | Role |
|---|---|---|
| $T$ | 1 ms | control period; total loop lag ≈ 1–2 $T$ |
| $T_0$ | 50 ms | reference period for calibration |
| $\alpha$ | 0.6 / $T_0$ | MI step (compounding doubles in ~59 ms) |
| $\beta$ | 0.15 / $T_0$ | maximum MD dose at full mark |
| $\delta$ | 0.15 | demand growth margin (a zero-sum tax; must stay small, §3.2) |
| $\theta_b$ | 0.5 | backlog test threshold (fraction of pure weighted share, §3.2) |
| headroom | 3% | ledger capacity's concession to physical capacity |
| $V$ | 600 Mbit (= 2 $T_0$ × line rate × headroom) | mark full scale; integral clip = $V$ |
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

The control plane totals about 2,000 lines of Python; the executors
about 1,100 lines of DPA C and 250 lines of eBPF C. The single source
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
flow-sets above $e_f$ are persistently marked and pushed back; those
below $\hat e_f$ unmark and rise; the equilibrium is
$r_f \approx e_f$, and water-filling guarantees $e_f$ equals the
hierarchical weighted max-min share. Borrowing needs no extra
mechanism: an idle class's small demand refills the sibling's $e_f$
through work-conservation, so the borrower rises unmarked; when demand
returns, $e_f$ is re-divided and the marks push the borrower back.
Across multiple bottlenecks, "the tightest bottleneck binds" follows
from $pace = \min(R, Tree)$: each bottleneck rules independently and
enforcement takes the minimum, with no cross-host weight comparison.

### 8.2 Stability

The law by itself has no limit cycle (the log-space argument in §4.2),
and the two-sided setpoint clamp makes upward overshoot structurally
zero. The closed loop's entire oscillation budget comes from marking
lag (VQ drain time + the 1–2 period telemetry loop), which is bounded.
Two honest weaknesses. First, MIMD carries no built-in fairness safety
net — correctness depends on the quality of per-flow differential
marking, and the law will not self-correct a distorted mark; this is
why measurement accuracy and fail-open are on the critical path rather
than optional (§3.1, §4.4). Second, no-overshoot is a property of the
law itself; gain × lag still has a Nyquist-type stability boundary, and
the margin between the production operating point and that boundary has
not yet been mapped systematically (experiment plan in
`response_law_mimd_analysis.md` §2).

### 8.3 Measured results

Representative numbers under the current configuration (1 ms period,
MIMD, direct vport measurement); provenance in `evaluation_index.md`
and the per-directory `results/*/summary.md`:

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
testbed (see `rd_fairness_design_c.md` §4).

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

- This document (`design_en.md`, Chinese edition `design.md`) — the
  authoritative design description of the running system, replacing the
  deleted `design_and_implementation.md`.
- `rd_fairness_design_e.md` — the original Scheme-E design at freeze
  time and the Q1–Q27 decision log; its mechanism narrative is
  superseded by this document, but it remains the source of record for
  decisions.
- `response_law_mimd_analysis.md` — theory analysis of the response law
  and the follow-up experiment plan; companion to §4 and §8.2.
- `rd_fairness_design_c.md` — Scheme-C archive; its weighted law is
  referenced by the §9 extension.
- `planned_extensions.md` — motivation, mechanism, and experiment plans
  for the three planned components (multi-owner composition,
  adversarial tenants, online policy reprogramming).
- `docs/archive/` — abandoned directions (Scheme A, receiver-driven,
  DPU TCP offload, …), tuning and diagnostics logs, handoff history.
- `evaluation_index.md` — the master index of experiment data; data
  generations and trustworthiness in
  `hpft-implementation/results/EXPERIMENT_STATUS.md`.

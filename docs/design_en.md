# HPFT: Virtual-Queue Fairness at the DPU Edge

This chapter describes the design of HPFT. §1 states the model, the objective, and why existing mechanisms cannot deliver it. §2 gives the design overview and one structural property that runs through the rest. §3–§6 are the four parts in turn: synthesising the congestion signal that policy bottlenecks do not produce, computing fairness as an explicit quantity, tracking it, and enforcing it underneath the tenant's own congestion control. §7 argues the system's properties and states its boundaries.

## 1 Problem

### 1.1 Model and objective

Consider a public cloud in which tenant virtual machines generate TCP and RDMA (RoCEv2, lossy, no PFC) traffic over a shared network. The operator must provide two layers of assurance at once. **Between tenants the isolation is hard**: each tenant $i$ buys a bandwidth cap $MaxRate_i$, and when contending at a bottleneck the tenants divide it by operator-configured weights $W_i$. **Within a tenant the isolation is soft**: TCP and RDMA share that tenant's allocation by class weights the tenant configures; when one class is idle the other may borrow, and it gives the capacity back when demand returns.

Together these define a **hierarchical weighted max-min allocation**: the bottleneck capacity is the root, the first level divides it among tenants by weight and cap, the second divides each tenant's share between the two classes by class weight, and every level obeys max-min's borrowing semantics — whoever wants less than its share gets what it wants, and what is left over is redivided by weight among those not yet satisfied.

Two constraints define the solution space. **Tenant transparency**: we may not modify the tenant's network stack, congestion control, or application. **No data-plane change**: we may not depend on switch modifications, new packet formats, or an extra signalling plane. These are not preferences but the reality of a public cloud — the operator does not own the tenant's software, and can rarely re-procure a fabric for one feature.

### 1.2 Why existing mechanisms cannot deliver it

There are two obstacles, and they are present together.

**First, the collision between heterogeneous congestion controls carries no policy semantics.** TCP responds to loss on an RTT timescale; RDMA's DCQCN responds to ECN in under a millisecond. They listen to different signals. Sharing one bottleneck queue, the switch's ECN marks land only on RoCE packets, while TCP — which does not run ECN — keeps the queue above the marking band. The one class that listens to the signal is starved by its own compliance. In an incast experiment holding the total flow count fixed and varying only the ratio between the classes, we observe that as soon as TCP has more than one flow, the RDMA class's aggregate is pushed below 2% of the bottleneck, almost independently of the ratio; changing RDMA's retransmission algorithm does not help, and neither does sweeping the switch's ECN threshold over a 16× range or DCQCN's recovery parameters over 30×. Who gets what is a by-product of two algorithms colliding, and no amount of parameter tuning on either side aligns it with the operator's weights.

Putting the two classes in separate switch traffic classes, with class weights configured through DWRR, cures exactly half of it. In the traffic-class arm of the same experiment, the **inter-class** split is exactly 50:50 — but **between tenants**, the tenant that has the TCP class to itself gets twice an equal share while the three tenants crowded into the RDMA class get 0.65× each, and this 3× structural unfairness among four equally weighted tenants is present unchanged at every ratio. This is not a scheduler defect but a ceiling on expressiveness: a traffic class offers exactly one knob, the class weight, which cannot express "one share per tenant, shared between two classes inside it", and the number of traffic classes (typically eight) does not scale with the number of tenants.

**Second, a policy bottleneck physically produces no congestion signal.** A VM allowed 20G by policy, on an idle 100G link, sends its excess into no queue: nothing is marked, nothing is dropped. There is no signal for any congestion control to respond to. The contention for a policy share is simply invisible to the transport layer. The first obstacle is that a signal exists but has no policy meaning; the second is that no signal exists at all.

### 1.3 Scope of the guarantee

HPFT's guarantee is for **edge bottlenecks** — bottlenecks that have an owner, and whose owner can see all of the traffic competing at them. The sender's uplink is owned by the sender; a destination VM's policy share, and the receiver's downlink, are owned by the receiver (the one place where an N-to-1 incast physically materialises is the last hop toward that machine). These three share a physical fact: all of the competing traffic passes through one DPU.

At those bottlenecks we guarantee that the steady-state allocation converges to the hierarchical weighted max-min share of §1.1. We do **not** guarantee fairness inside the core network (no bottleneck there has an owner who sees all competitors; the extension is sketched in §7.3), host-internal resources (PCIe, memory bandwidth), loss elimination, path load balancing, or burst accuracy below one control period — bursts shorter than a period are absorbed by the tenant's own congestion control, which is always running underneath our shaping.

## 2 Design overview

### 2.1 Three ideas

HPFT reduces to one sentence: **compute the shares where all of the competition is visible, send the targets down, and run one rule at every sender.** It unfolds into three ideas.

**Compute the share; do not search for it.** The physical fact behind edge bottlenecks — all competing traffic passes through one DPU — means the competitors are all in view. There is then no need to probe, and no need to infer one's own entitlement from congestion signals: each period, that DPU substitutes the policy (weights, caps, class weights) into one hierarchical water-filling pass and computes directly what each flow-set is entitled to at this instant.

**Synthesise the missing signal by bookkeeping.** A policy bottleneck neither queues nor drops, but it can be accounted for: give each flow-set a virtual queue that exists only in the ledger and integrate "arrival rate minus entitled rate" over time. A flow-set that persistently exceeds its share accumulates debt and is marked. In front of the ledger, "a 20G policy allowance is being overrun" and "a 100G physical port is being overrun by incast" are the same event — the ledger is overloaded. The second obstacle of §1.2 is gone.

**Every sender runs one weightless rule against its own target.** The owner compresses its policy ruling and its audit into a single target rate and sends it down; the sender only tracks its rate smoothly onto that target. The rule contains no weights and does not distinguish classes — the weights were spent when the shares were computed. TCP and RDMA are therefore shaped by the same hand, each with its own congestion control still running underneath, and who gets what at a shared bottleneck is no longer a by-product of two congestion controls colliding. The first obstacle is gone with it.

Multiple bottlenecks need no additional mechanism. Every bottleneck a flow traverses keeps its own ledger and expresses its own constraint: the receiver's constraint arrives as a target, the sender's is the share computed locally, and enforcement takes the minimum of the rulings, so the tightest bottleneck binds automatically. Nowhere does the system have to compare the two ends' weights — each end normalises its own, and they were never comparable.

The structural property that matters most for what follows is also here: **fairness lives in the target, not in the rule.** The share is computed explicitly by the owner, compressed together with the audit discount into a target, and broadcast; the sender does not have to *search* for the fair point, only to track its rate onto the target. This alone determines the choice of tracking rule (§5.1).

### 2.2 The control loop

Let the control period be $T$. One turn has four steps: the owning DPU measures the arrival rate $r_f$ of each flow-set; it substitutes the policy into hierarchical water-filling and obtains the entitled rate $e_f$ and the fair ceiling $\hat e_f$; it integrates the excess in a virtual queue, audits out a mark $s_f$, and compresses everything into a target $u_f = \hat e_f(1-\gamma s_f)$; the target rides telemetry back to the sender, which moves its rate ceiling $R_f$ one step toward $u_f$ and enforces $pace_f = \min(R_f, Tree_f)$, and the next period's $r_f$ reflects the effect.

The telemetry loop itself is only a period or two, but what governs the tuning is not the loop — it is the **effective lag** $\tau_{\rm eff}$. The target is computed under a "siblings hold still" assumption, and between being measured and being enforced it passes through the measurement window, the telemetry round trip, and executor batching. $\tau_{\rm eff}$ is an order of magnitude larger than $T$, and the tracking time constant must cover it (§5.3).

### 2.3 The granularity of fairness equals the granularity of enforcement

The smallest unit the system measures, allocates, marks and paces is the **flow-set** $f$: the triple (src VM, dst VM, class) with class ∈ {TCP, RDMA}. The choice needs an argument in both directions, and both are necessary.

**It cannot be coarser.** Suppose we controlled per VM pair, so that a pair's TCP and RDMA shared one rate ceiling. Any pace executor can only constrain the total at a boundary; it cannot arbitrate the composition inside that boundary. How much each class gets under the shared ceiling would still be decided by two congestion controls colliding — now in the executor's queue. And whatever discipline the executor adopts is partial: dropping favours TCP (RDMA ends in a retransmission storm), marking ECN favours TCP (with TCP not running ECN, only RDMA slows down), and backpressure favours RDMA (hardware-issued traffic does not feel it). The collision of §1.2 has merely moved from the switch into the executor. In one line: **for inter-class fairness, measurement, allocation and enforcement must all land on (flow pair, class).**

**It need not be finer.** (src VM, dst VM, class) is already the finest unit that carries policy meaning: tenant weights act on VMs, class weights on classes, per-sender weights on the source; anything finer (per connection, per queue pair) carries no policy semantics at all. It also brings two properties: the state is $O(\text{VM pairs} \times 2)$, independent of how many connections a tenant opens; and because the share is adjudicated per flow-set, **opening more flows wins nothing** — a flow-set with a thousand queue pairs gets the same share as one with a single queue pair, and the executor divides it inside the set (§6.3). Fixing the control unit here immunises the system against flow-count gaming by construction.

### 2.4 What we inherit and what we rebuild

The control skeleton has a clear lineage: edge arbitration, receiver-side measurement with sender-side enforcement, explicit feedback, and headroom all trace back to EyeQ and the edge bandwidth-guarantee systems around it; manufacturing a signal with a virtual queue comes from HULL's phantom queue and AVQ's integral marking. The skeleton is mature and we use it directly.

What is new is the three parts of that skeleton which have to be rebuilt for a world of two classes, each carrying its own congestion control, enforced in hardware. First, the unit of fairness has to be refined from the VM or the flow pair to (flow pair, class), or the inter-class collision merely relocates (§2.3). Second, the EyeQ line sidesteps demand estimation by dividing among active parties by guarantee weight — enough for a bandwidth guarantee, but a policy objective of weighted max-min with borrowing and oversubscription requires the *degree* of demand to enter the allocation, and demand estimation becomes unavoidable (§4.1). Third, the EyeQ-style loop assumes an executor without lag over traffic that submits passively (a software token bucket takes effect the moment it is written, and TCP is held directly by qdisc backpressure), whereas tenant-transparent RDMA pacing is, in any implementation, an inner-loop controller with its own measure-then-apply lag, with one or two live tenant congestion controls underneath it. The integral audit (§3), the lag-matched first-order tracking law (§5.2), and the coupling to the tenant CC (§6.2) are all consequences of the controlled object having dynamics of its own.

## 3 Synthesising the missing signal

A policy bottleneck produces no congestion signal (§1.2), so we synthesise one. Each flow-set gets a **virtual queue** — a queue that exists only in the ledger, with no real packet ever waiting in it:

$$vq_f \leftarrow \mathrm{clip}\big(vq_f + \Delta_f\, T,\ 0,\ V\big), \qquad s_f = \min(vq_f / V,\ 1)$$

$$\Delta_f = \begin{cases}
r_f - e_f & r_f > e_f & \text{fill: by the real excess}\\
-(\hat e_f - r_f) & 0 < r_f \le e_f & \text{drain: by the unused fair ceiling}\\
-V/T & r_f = 0 & \text{idle: clear in one period}
\end{cases}$$

The mark $s_f \in [0,1]$ measures how persistently the flow-set exceeds what it is entitled to. Each of the three rules has one reason that cannot be substituted.

**Fill by integration, not by an instantaneous ratio.** Only persistent excess should raise an alarm; a single period's measurement noise should be absorbed. The full-scale value $V$ is a **wall-clock integral** (excess times time) and therefore does not scale with the control period: the same history of excess accounts to the same debt whether it is cut into fifty periods of $T$ or one of $50T$. $V$ is not a constant picked by hand — it is solved backwards from the audit loop's target damping. The audit loop (an integrating ledger with a proportional discount) linearises to a standard second-order system with damping ratio $\zeta = \frac12\sqrt{kV/(\gamma\hat e^*)}$, so choosing the classical optimum $\zeta = 1/\sqrt2$ fixes

$$V = \frac{4\zeta^2\gamma\,\hat e^*}{k},$$

where $\hat e^*$ is the fair ceiling at a typical contention operating point. There is one easy mistake here: $\hat e^*$ scales with the **bottleneck capacity**, so $V$ must be anchored to the bottleneck capacity and not to the NIC port's line rate — anchored to the port, the damping changes silently when the bottleneck speed does. The integral is capped at $V$ itself and no deeper: a deeper reservoir only makes the mark ring for extra periods after the excess stops, and deepens the undershoot.

**Drain must use the fair ceiling $\hat e_f$, not the entitled rate $e_f$.** This is the difference between deadlock and no deadlock. $e_f$ is capped by the demand estimate, and demand is derived from the measured rate (§4.1) — so for a deeply suppressed flow, $e_f$ follows $r_f$ wherever it collapses to, the drain rate is forever a small fraction of the actual rate, a deep integral never drains, the audit signal saturates permanently, the flow is held at maximum discount indefinitely, and the mark loses all discriminating power. With $\hat e_f$ the drain rate is the *unused* share: a suppressed flow has a large unused share and recovers quickly, while a flow using its ceiling ($r_f \approx \hat e_f$) drains at nearly zero and does not get its mark cleared for free.

**A flow that stops sending clears in one period.** Such a flow neither fills nor has any share against which to drain, so its queue would freeze at its old value. Clearing it quickly reflects a judgement: a flow that has given up its claim on the network should not be punished by the infrastructure's stale state when the application resumes.

Three scenarios that look unrelated are unified by this one mechanism. **Contention for a policy share**: the VM-level cap makes an over-sending flow-set exceed $e_f$, and the ledger synthesises a scarcity that does not exist physically. **Incast**: the root capacity is overrun, the shortfall is shared out among the $e_f$ by weight, and headroom — the deliberate margin by which ledger capacity sits below physical capacity — guarantees the virtual queue raises the alarm before the switch's physical queue does, so the signal reaches the sender while no physical queue has formed. **Borrowing and reclaim**: work conservation at the class level is a direct product of water-filling, and the borrowing class is pushed back by the mark once the lender's demand returns and the shares are redivided — no explicit protocol is involved.

## 4 Computing fairness as an explicit quantity

Each period the owner compiles the policy into one number: the target rate $u_f$. This section is that compilation.

### 4.1 Why demand cannot be avoided

Weighted max-min contains "how much do you want" in its definition. Pour capacity to each party by weight; whoever wants less than its share receives only what it wants, and the remainder is redivided by weight among those not yet satisfied — that last clause is borrowing, and it is a judgement about demand. Concretely: capacity $C$ and four equally weighted tenants. If all want a great deal, each gets $C/4$; if one wants only $0.05C$, the max-min answer is that it gets $0.05C$ and the other three get $0.317C$ each. To produce that answer the allocator **must** know that one of them only wants $0.05C$. An allocator that ignores demand entirely — dividing by weight among the active, which is what guarantee systems do — always answers $C/4$ each, leaving nearly a fifth of the capacity idle while three tenants are starved.

The difficulty is that demand is not directly observable. All we can measure is the arrival rate $r_f$, and $r_f$ is capped by the very limit the system imposed on that flow — a suppressed flow looks like it has little demand, which is circular. The base estimate is therefore a probing allowance,

$$D_f = r_f\,(1+\delta),$$

a margin $\delta$ above current usage (15% in our implementation). If the flow can use it, next period's estimate rises by another $\delta$ and doubles within a few periods ($1.15^5 \approx 2$) — the climb is geometric, not slow. If it cannot, the estimate stays put and the surplus is borrowed by its siblings. $\delta$ has to be small: allocation is zero-sum, and the part of that margin a flow cannot use this period is taken from what a sibling could actually have used.

### 4.2 Falls fast, climbs slowly — and the backlog test

$r_f$ is noisy: a TCP loss recovery, a gap between RDMA bursts, or measurement jitter all make some period's $r_f$ dip. But $D_f = r_f(1+\delta)$ follows a dip **immediately** while it can only climb back one $\delta$ at a time — **the fall takes one period, the recovery takes several**. At the class level that asymmetry becomes the sibling class's gain at once: in the period where TCP under-runs because of a loss recovery, water-filling concludes that TCP wants less than its share and hands the difference to RDMA; RDMA is backlogged and takes it at once; next period TCP tries to climb back but must start from the collapsed value, and if it jitters again it starts over. Averaged over time, the noisier class sits systematically below its policy weight — **not because it wants less, but because the demand estimate reads "did not manage to send just now" as "does not want it".**

The correction is a **backlog test** on the demand estimate, with a single criterion: from root to leaf, if the measured total at any node on $f$'s path is already against that node's capacity, $f$ is deemed to be in contention, and its demand is set to the root capacity — a **finite** value larger than any node's share, which is enough to make the demand cap inactive at every level, equivalent to infinity. The criterion uses **totals only**, and totals are measured exactly (§6.4 explains why that matters); the test carries hysteresis, held per node, so it does not chatter at the threshold.

### 4.3 Hierarchical water-filling and its two outputs

The policy is a three-level tree: the root is the bottleneck capacity after headroom (the ledger capacity $C'$), the first level divides it among destination VMs by weight $W_i$ and cap $MaxRate_i$, the second divides each VM's share between TCP and RDMA by class weight, and the third divides a class's share among senders by per-sender weight. Each level is one weighted water-filling: find the level $t^*$ with $\sum_j \min(c_j, w_j t^*) = C$, and child $j$ receives $e_j = \min(c_j, w_j t^*)$, where $c_j$ is its demand or its cap.

This step produces **two** quantities, and the distinction is used repeatedly below. The **entitled rate** $e_f$ is what $f$ is entitled to right now given everyone's current demand. The **fair ceiling** $\hat e_f$ is what $f$ would receive if its own demand were infinite while its siblings kept their actual demands — the ceiling on its share, with $\hat e_f \ge e_f$ always. $e_f$ is the outcome of an allocation; $\hat e_f$ is the allowance policy permits. The ledger drains against the latter (§3), and the target is anchored to the latter (§4.4).

Both come out of a single pass. Each node is sorted once and given a prefix sum, after which everything on that node is a binary search on that table: one search gives the ordinary level and hence every child's $e$, and one search per child — with that child's cap replaced — gives its $\hat e$. The three levels cascade along $f$'s root-to-leaf path, and leaves under a common ancestor share the upper levels' results. The total cost is $O(N\log N)$ in the number of active flow-sets, rather than the $N$ full allocations a naive construction would need.

### 4.4 Composing the target

The owner compresses every ruling about a flow-set into **one number**:

$$u_f = \hat e_f\,(1 - \gamma\, s_f).$$

$\hat e_f$ is the policy ceiling on the share, $s_f$ is the audited persistent excess, and $\gamma$ bounds the strength of the audit discount (0.25 here). With no excess the target is the fair ceiling; persistent excess pushes it down to at most $(1-\gamma)\hat e_f$ — **the audit dose is bounded**, and it enters the target as a discount rather than a jump.

The anchor must be $\hat e_f$ and not $e_f$, for the same reason the drain reference must be (§3): $e_f$ collapses with the flow's own rate, and anchoring to it would replay that self-referential trap. Anchoring to the demand-infinite $\hat e_f$ does not over-grant — $R_f$ is a ceiling, not an allocation; raising the ceiling of a flow with finite demand does not make it send one more byte, the ledger looks only at measured $r_f$, and an unused share is borrowed as usual.

Telemetry sends one datagram per sender per period, carrying two numbers per active flow-set, $\{u_f, r_f\}$. $u_f$ drives the tracking law; $r_f$ is the owner's measured arrival rate, which the sender uses as the demand input to its local tree and forwards to the executor as feedback. Two protocol decisions are worth naming. **The record is itself the explicit permit that the ruling is fresh and granted** — $u_f$ is sent every period, including when there is no excess, everything the sender does is conditioned on having received it, and a telemetry outage therefore drops the system into fail-open by construction (§5.4). **Presence in the report is decided by the existence of the flow-table key, not by an instantaneous rate** — a paced flow has empty periods at millisecond granularity, and omitting its record then would make the sender read an empty period as an outage, feeding back into fail-open.

## 5 Tracking a known set point

### 5.1 Restating the problem

In classical congestion control the sender does not know its fair share, so the rule must carry its own dynamics for converging to fairness — the entire complexity of the increase/decrease family (AIMD and its relatives) comes from that. HPFT is not in that setting: the share is computed explicitly by the owner and broadcast, compressed with the audit discount, as a target $u_f$ each period. One thing is left for the sender to do — bring the rate ceiling $R_f$ quickly, stably and without overshoot onto a target that is **externally known but lagged**.

Chiu and Jain's classical analysis of increase/decrease fairness does not apply here, because its premise has been dissolved: that analysis assumes all sources share one binary feedback, whereas here each flow receives a target of its own and fairness is already encoded in the set of targets. Change the criterion and the optimal rule changes with it — no fairness-searching dynamics are needed, only a low-pass tracker for a lagged broadcast target.

### 5.2 The law

Each flow-set keeps a rate ceiling $R_f$, and a new flow-set starts at the locally permitted allowance (an optimistic start: short flows pay no ramp-up tax, and a flow-set that lands on a congested resource is pulled back within a period or two when the target arrives, with the transient absorbed by the tenant CC's immediate response). On each telemetry record it does exactly one thing — **take one step toward the target in the logarithmic axis**:

$$\dot R_f = k\, R_f \ln\frac{u_f}{R_f} \qquad\Longrightarrow\qquad R_f \leftarrow R_f\left(\frac{u_f}{R_f}\right)^{\,1-e^{-k\Delta t}}$$

The law is defined in continuous time (with $z=\ln R_f$ it is $\dot z = k(\ln u_f - z)$), and the discrete update on the right is its exact integral over the **interval actually elapsed**, $\Delta t$, rather than over the nominal period. The distinction is a correctness requirement, not a precision optimisation: ticks are never exactly uniform in an implementation and telemetry can arrive late, and integrating over the real interval fills the gap exactly. More importantly, the exponent $1-e^{-k\Delta t}$ lies strictly in $(0,1)$ for any interval, so a long gap can only converge toward the target and can never cross it — writing $k\Delta t$ instead, for convenience, lets one long stall drive the exponent far above 1 and send the rate far past the target.

This is a first-order low-pass tracker in logarithmic coordinates: no branches, no clamps, one parameter $k$. The logarithmic coordinate is chosen for the reason that runs through the whole design — it is the natural space of multiplicative dynamics: shares of any size take the same relative step and converge in the same wall-clock time, the two classes share one constant, and the rule contains no weights (weights live only where the shares are computed).

The tracking form makes three families of problem structurally absent rather than handled. **No ceiling clamp is needed**: tracking approaches asymptotically instead of crossing, since the step from below is proportional to the remaining logarithmic distance, so the instability family of "probe past the target, signal storm, deep cut" has no entry point. **No floor clamp is needed**: descent is equally asymptotic, and the audit dose is itself bounded ($u_f \ge (1-\gamma)\hat e_f$), so enforcement can push a flow no lower than $(1-\gamma)$ of its fair ceiling and "a persistent signal compounds the rate through the floor" is structurally impossible. **No jump protection is needed**: when the target moves, $R_f$ follows with time constant $1/k$ and never jumps.

### 5.3 Tuning: lag bounds $k$, damping fixes $V$

Two properties fix the parameters.

**The upper bound on $k$ comes from the target's staleness.** Every $u_f$ is computed under a "siblings hold still" assumption and is lagged by $\tau_{\rm eff}$. Far from the target, logarithmic tracking has a natural ceiling on its absolute climb, $x\ln(u/x) \le u/e$, so the joint climb of all flows is at most $(k/e)\sum u$ — self-normalised against the targets and therefore **not amplified by the number of flows** — and the joint instantaneous overshoot within the stale window is about $(k/e)\tau_{\rm eff}$. That fixes the upper bound on $k$: $k\,\tau_{\rm eff} \lesssim 0.4$, corresponding to roughly 15% instantaneous overshoot. To go faster, the thing to shrink is $\tau_{\rm eff}$ (the measurement window or executor batching), not to raise $k$.

**The audit loop's damping fixes $V$.** The tracking term $-kz$ is exactly the damping this loop lacks — without it, an integrating ledger with a proportional discount is an undamped oscillator. Linearised, it is a standard second-order system with $\zeta = \frac12\sqrt{kV/(\gamma\hat e_f)}$, and $V$ follows from the target damping (§3). $\gamma$ appears in two places at once: it bounds the discount and it enters the damping — making the audit dose bounded also confines the residual oscillation to the band $[(1-\gamma)\hat e_f,\ \hat e_f]$.

### 5.4 Local constraints and graceful failure

The sending DPU owns the uplink, so it runs the same water-filling locally over a second tree (the source VM's cap and class weights) to obtain a local share $Tree_f$, and enforces $pace_f = \min(R_f, Tree_f)$ — this is where "the tightest bottleneck binds" from §2.1 lands. $Tree_f$ is the **fair ceiling** from the sender's viewpoint rather than the demand-capped value, for the same reason as in §4.4: both the start of a new flow and the failure path require it to be "the allowance policy permits", and a demand-capped value gives the self-contradiction that a new flow with $r=0$ has $Tree \approx 0$.

Sender-side policy is enforced in two layers, separating semantics from the backstop. The **hard cap**: per-VM $MaxRate_i$ is enforced by the NIC's hardware rate limiter — class-blind, static, independent of the software stack — so the **selling principle** (no VM ever sends beyond what it bought, even when every other VM is idle) is guaranteed by hardware and no software failure can break the contract. The **allocation semantics**: the software tree performs oversubscription shrink, inter-class soft isolation and borrowing inside that hard cap; both classes' executors hang off the same sender agent, so inter-class borrowing is nothing but the tree's own work-conserving fill and needs no separate arbiter.

The tracking law is conditioned on receiving a fresh target, so a telemetry outage has a well-defined degradation path. After a flow-set has been silent past a short threshold, $R_f$ freezes; past a long threshold, it **climbs toward $Tree_f$ using the same tracking law** — only the target is replaced by the locally permitted allowance, introducing no second rate constant and needing no extra clamp. The semantics: when the owner or the telemetry path fails, traffic degrades to being constrained by sender-local policy only — the tenant is not stalled (fail-open) and does not run wild ($Tree_f$ caps it, and the tenant CC is still there). The first target after recovery restores control.

One question remains: "never coming back". A finished flow-set should not be driven forever, but **forgetting must be conditioned on the telemetry channel being confirmed alive**, the test being that other flow-sets from the same sender are still receiving records. This gate cannot be omitted — looking at one flow-set's record stream, "this flow ended" and "the whole channel died" are indistinguishable, and forgetting without distinguishing them means one channel failure forgets every flow-set at once and drops the $Tree_f$ cap with them, exactly the runaway that fail-open exists to exclude. In **aggregate** the two are distinguishable: a real channel failure silences every flow-set together. Forgetting does not release executor state — the executor keeps the last rate it was given, so a forgotten flow-set remains bound by it and can only be slower, never faster.

## 6 Enforcing underneath the tenant's congestion control

### 6.1 The contract

Both executors (RDMA and TCP) obey one contract: **pace only delays packets, it never drops them**, and the actual sending rate is the smaller of what the tenant CC wants and $pace_f$. The tenant's congestion control keeps running underneath the pace, and the two loops do not drive each other into oscillation because the division of labour is cut by timescale: **sub-period bursts and fabric congestion belong to the tenant CC**, **policy shares over periods belong to HPFT**.

The contract constrains transitions as well as steady state: the enforced rate may not change so abruptly that the controlled transport reads it as a failure, since in effect that is a drop and defeats the point of the contract. The tracking law's own output satisfies this by construction (the target moves with time constant $1/k$); what needs limiting is a step injected from outside, typically a large downward policy change, which reaches the executor through the sender tree without passing through tracking. The risk is concrete: a single large-ratio reduction can drive an RDMA connection into terminal failure, while **running steadily at that same low rate is entirely safe** — what is dangerous is the ratio of the fall, not the destination rate. Large reductions are therefore walked down over a few periods in software. This does not weaken the selling principle: the hardware limiter applies the new allowance immediately, and the software gradient only shapes the trajectory.

The contract also has a requirement about units that is easy to miss: **pace and measurement must use the same byte accounting**, both counting wire bytes including every layer's headers and the interpacket gap. If the executor bills bytes from the application's or the kernel's point of view (a segmentation-offload super-packet counted as one header, with no interpacket gap) while measurement reads the NIC's wire counters, there is a constant systematic gain error between them, which appears directly as that class steadily exceeding its share. A sensor and an actuator that agree on units are the precondition for a loop without static bias. Under tunnel encapsulation the whole accounting shifts to **inner** wire bytes: measurement and both executors are naturally on the inner side, so the loop stays self-consistent, and the capacity cost of the outer headers does not enter the per-flow accounting but is conceded once at the root through headroom — compensating for outer bytes in one executor alone would recreate the asymmetry between the classes, since hardware pacing cannot bill them.

### 6.2 Coupling to the tenant's congestion control

Tenant transparency means we may not modify the tenant's congestion control. The executor's entire interaction with one is therefore three steps: **read the rate it has arrived at, combine that with the policy share, and shape the wire to the result.** Under that constraint the combiner is not a free choice — it is determined by what a tenant CC's rate can and cannot tell us.

The **absolute value** of a tenant CC's rate is of no use to us. It is calibrated against the link, not against a share it has never been told about, and it **drifts steadily downward with no way back**: many congestion controls recover toward "the rate I had before the congestion", so every cut also lowers the level it will be able to return to. The minimum combiner $\min(cc,\ level)$ inherits that behaviour whole — a run of cuts drives the flow to near zero and holds it there. At an edge bottleneck shared with another class the process is self-sustaining — RDMA yields its share, TCP expands into it and keeps the queue above the marking threshold, and RDMA is marked again before it finishes recovering.

What the rate does carry is its **decreases**. Each one is that CC's own judgement that the network asked it to yield. Multiplicative decreases are scale-free, so a cut means the same thing after conversion into new units. The combiner is therefore built like this: accumulate the CC's decreases into a **deviation relative to the policy share**, $d \in [f, 1]$, and shape to

$$rate = d \cdot level,$$

where $level$ is the policy share. **The decrease is taken from each CC untouched** — that half carries the CC's entire reading of the network: which signal it trusts, how hard it reacts, on what timescale. **The recovery belongs to the coupling**, because recovery needs a destination and only the coupling knows one: the policy share, $d \to 1$, halving the remaining gap each step. The whole construction reads only ratios, so whatever constant sits between the sample and the rate the CC would actually have paced at divides out — which is precisely what lets one combiner serve any CC. In one line: **the CC owns the decrease, the coupling owns the recovery.**

Both halves are necessary, and each is taught by a CC that fails without it. **Leave recovery to the CC's own target and it drifts to the floor**: a CC that climbs back toward "its pre-congestion self", when every cut drags that target down with it, is driven to zero by a run of cuts and pinned there, unable to return while the queue is still being marked. **Leave recovery to the CC's own step size and it starves — it does not drift downward, but it cannot climb back either**: a CC that decreases by a factor on each RTT sample and increases by a small fixed step will, in an edge queue that another class keeps above its delay target, keep receiving decreases that its additive increase can never catch up with, and the flow sits near the floor.

The coupling taking over recovery is safe exactly where it has authority: $d \le 1$ bounds every flow to its own share, and the shares sum by construction to the root capacity, so returning arbitrarily many flows to their shares cannot overwhelm the bottleneck the shares were computed against. One case of self-driving must also be excluded: **a decrease that occurs while the flow is held by our own shaping (achieved rate ≈ granted share) is not counted** toward $d$ — otherwise the slowdown our shaping causes is read as a congestion signal and drives deeper shaping.

The floor $f$ on $d$ answers a different question: **should this signal be trusted?** The criterion uses something the owner knows and the CC cannot — whether the bottleneck the signal came from is one we own. The owner sends a saturation bit with the telemetry and the sender forwards it to the executor with the budget. **When the edge bottleneck is saturated**, $f$ takes a substantial fraction: the CC may express congestion but may not surrender its share, because the scarcity at that moment is exactly the one we are adjudicating, and letting the CC give the share away would let it overrule the policy. **When the edge bottleneck still has room**, the marking can only have come from somewhere we neither see nor schedule (deep in the fabric), and $f$ approaches zero: the CC has full authority.

The construction has a checkable corollary: **adopting a new congestion control requires only its rule for decreasing**, and nothing else.

### 6.3 When the target is known, assignment beats convergence

The executor receives $pace_f$ as a **per-flow-pair budget** and divides it inside the pair by the number of queues actually carrying data, giving each queue its level — **assigned directly, not searched for**. The budget is known and the queue count is known; nothing here needs to be approached by feedback. This is the same principle as §5.1: **when the target is known, computing it beats converging to it**; the executor is not an independent convergence process but a translation of a known policy target into hardware parameters.

The cost of approaching a known value by integration is concrete: it needs a step size, saturation, a floor to prevent unbounded digging, and a settling window to avoid integrating against measurements that have not caught up — and one large budget reduction excites all four at once, producing a limit cycle lasting seconds. With assignment the limit cycle disappears and aggregate utilisation rises, because the under-sending during the search is gone too.

Dividing inside the pair is where §2.3's immunity to flow-count gaming is realised in the executor: opening more queues only slices the same budget more finely. One definitional detail must be stated: **count only the queues carrying that pair's data, excluding the management queues the transport reserves** — they exist as soon as the port does and produce events continuously while carrying almost no bytes, so counting them gives each a share it does not use and systematically under-delivers the budget. This is not an artefact of one implementation; every real deployment has them.

The rate difference between the policy channel and the congestion-response channel is deliberate: budget updates go on the slow channel (batched, and not rewritten at all when the change is below a threshold), while congestion response stays at event speed. **The budget must be quasi-static** — the executor treats every budget change as a change of ceiling and resets its own settling window, so an outer loop rewriting the budget at millisecond rates would never let the inner loop settle. Batching and hysteresis quantise the policy channel to a rhythm the executor can digest, and since the policy share is a slow variable anyway, lowering its rate costs no semantics. **The downward direction is not rate-limited**: when the target drops, the budget follows in one step, which for a monotone descent is both correct and fastest; the outer loop is already smooth, and adding a ramp on the way down would only make the executor, rather than the law, the bottleneck of convergence.

### 6.4 Which part is measured and which is estimated

The $r_f$ the owner sends back is not a homogeneous quantity. The **total** per destination comes from a hardware counter, exactly and freshly every period; the **division** of that total among several senders is an estimate that depends on flow-table byte ratios and lags by seconds. An error in the latter is not a loss of precision but a qualitative error — before a new sender's bytes appear, all of its traffic is charged to its peer.

Hence a usage rule: **a test may use totals, but anything driving the executor's inner loop must be immune to a mis-split.** The backlog test (§4.2) uses only totals and is therefore exact; the per-sender rate fed to the executor, while the membership has not all been measured yet, should fall back to apportioning by the previous period's granted shares rather than trusting a byte ratio that is not yet credible.

### 6.5 The two enforcement points

**RDMA** is paced by the DPU's programmable congestion-control engine, transparently to the tenant: the budget is delivered per flow pair, the device combines it with the tenant CC as in §6.2, and assigns the result to each queue as in §6.3.

**TCP** is paced in the sending host's kernel: an Earliest Departure Time token bucket per flow pair stamps each packet with a departure time, executed by fq — a mature delay-only shaping path that is zero-copy and lossless. The shape of the control loop is identical to the RDMA side and only the enforcement point differs; the combination with the tenant CC is the same one (read the kernel CC's window as a rate, same combiner, land on the timestamp).

The TCP side has one extra design point: a **sparse-flow bypass**. Latency-sensitive small flows should not pay the queueing tax of shaping meant for throughput flows, and may be released directly. The key is that the predicate must be **one a throughput flow cannot satisfy** — "the inter-packet gap exceeds a threshold" is not enough: with segmentation offload enabled, a throughput flow presents as large bursts with large gaps and passes that test too, leaving the shaping with nothing to do. The predicate is therefore the conjunction of three necessary conditions: the flow really has been idle for a while, this send really is small (not an offload super-packet), and the pair's accumulated debt is within one burst. The first two exclude a throughput flow disguised as sparse; the third caps the bypass itself, so that the aggregate escape of many small flows is bounded as well.

## 7 Properties and boundaries

### 7.1 The equilibrium allocation is the policy share

At a single controlled resource this is the classical setting of "an AQM that marks in proportion to excess, plus a response law at each source": flow-sets above $e_f$ are pushed back by the audit discount, those below $\hat e_f$ are unmarked and their targets rise, the equilibrium is $r_f \approx e_f$, and water-filling guarantees $e_f$ is the hierarchical weighted max-min share. Borrowing needs no additional mechanism: an idle class has small demand, the share it does not use is redivided to the other class and enters that class's $e_f$, and the borrower's target rises undiscounted; when demand returns the shares are redivided and the lowered target pulls the borrower back. Across multiple bottlenecks, "the tightest binds" is given by $pace = \min(R, Tree)$ — each bottleneck rules independently and enforcement takes the minimum, with no need to compare the two ends' weights.

### 7.2 Stability

The main loop (tracking) is a first-order low-pass: unconditionally stable, structurally free of overshoot, with no parameter cliff. The audit loop (an integrating ledger with a discount) linearises to a standard second-order system whose damping is tunable (§5.3), and the bounded discount confines any residual oscillation to the band $[(1-\gamma)\hat e_f,\ \hat e_f]$.

There are two honest weaknesses. First, the tracking law carries no fairness safety net of its own — correctness rests on the quality of the owner's rulings (shares and audit), and it will not self-correct a distorted target. Measurement accuracy and fail-open are therefore components on the critical path, not optional extras. Second, the staleness budget $k\tau_{\rm eff}$ and the share-dependence of $\zeta$ give analytical stability boundaries, but the measured margin around the operating point has not been systematically characterised.

### 7.3 Boundaries

**Core-network bottlenecks.** Real congestion signals in the core carry no policy scale; there, the response law should be weighted (weights configured by the operator), which separates naturally from the edge's synthesised marks — synthesised marks travel on the telemetry channel, fabric signals are measured. The mechanism holds but is outside the scope evaluated here.

**Two optional modes.** On the demand side, a "share floor" (the accounted share does not shrink when the rate dips, at the cost of losing the temporary borrowing during that dip) and a per-class step size (utilisation first, at the cost of ratio drift) are kept for particular SLA scenarios. Neither is the default.

**Known limitations.** Burst accuracy below the control period is not promised and is left to the tenant CC; online policy changes take effect at the granularity of a control period, with no atomic multi-bottleneck switch; adaptation to VM migration, and validation at scale of composed ownership (one flow adjudicated by two owners at once), are left for future work.

## 8 Implementation

The three parts run on the owning DPU's Arm cores, on the sending DPU's Arm cores and its programmable congestion-control engine, and in the sending host's kernel. The language split follows one principle: per-period arithmetic over arrays of flow-sets is C, while I/O, parsing, configuration and control flow are Python — so the hot path of the three-level allocation is C, the rest of the control plane is Python, and the two executors are device-side C and eBPF respectively. A pure-Python implementation of the allocation is kept as a reference and checked point-by-point against the C. The single source of truth for policy and parameters is one configuration file, whose policy section the owner hot-reloads on change. Control-plane overhead is under a tenth of one core at each end.

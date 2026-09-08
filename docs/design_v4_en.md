# HyperFront Control Design (v4, 2026-09-08)

HyperFront gives each tenant sharing a receiver port the bandwidth its policy promises, without relying on, or modifying, the tenant's own congestion control. This document is the design and nothing else: what the components are, what each one does, and why it is built that way. It does not depend on any particular hardware, topology or implementation, and it should not need changes when the system is moved to a different platform.

The companion documents each cover one thing. The control theory, the guarantees and the tuning method are in `design_theory_v4_en.md`. Numbers measured on our own testbed are in `platform_notes.md`. Where each rule lives in the code, and where the code still departs from this design, is in `hpft-implementation/docs/IMPLEMENTATION.md`. Validation scenarios, pass criteria and current results are under `hpft-implementation/validation/`.

## 1 Goal and principles

**Goal.** Traffic from several tenants arrives at one receiver port, each tenant running its own congestion control. HyperFront must give every tenant, and every traffic class within a tenant, the share its policy specifies; let unused share be lent to others and reclaimed within a second when needed; and do both without overrunning the port and without crushing any tenant's congestion control.

**Five principles.**

First, the sender sends no signal of any kind to the receiver. The receiver decides using only what it can see on its own port: how many bytes arrived, and to which virtual machine, which class and from which sender they belong.

Second, there is exactly one fairness signal: the length of a virtual queue. No physical queue in the network can tell you who is taking more than their share, so the receiver keeps a virtual queue per flow set: arrivals above the flow set's entitlement go into the queue, arrivals below it drain the queue. The only thing sent to the sender is the length of that queue. It is not a rate, and it is not anything a rate could be recovered from. The queue is never allowed to build up for long: in steady state it is a small sawtooth around zero. It is a bounded memory, not a debt.

Third, no assumption is made about which congestion control a tenant runs, and none of its private signals are used. Only two facts hold for every congestion control, and only they are relied on: it maintains a quota that the executor can read, meaning how fast it is currently willing to send; and it raises that quota when it sees no congestion and lowers it when it does.

Fourth, HyperFront's own rate control is multiplicative. With no queue it probes upward, taking larger steps the longer things have been quiet; with a queue it brakes in proportion to how fast the queue is changing and repays in proportion to how much has accumulated. Toward the tenant's congestion control HyperFront only caps: the executor never lifts a flow above its own quota, and never lets a flow set exceed the permitted rate. Every rate in this document is a ceiling on how fast something may send, never a command; if the application has nothing to send, nobody sends on its behalf.

Fifth, only capacity that physically exists is allocated. Traffic that reaches the port but is not scheduled by HyperFront is subtracted from the capacity.

## 2 Terms and notation

A **flow set** is a triple (source virtual machine, destination virtual machine, class), where the class is RDMA or TCP. Policy has three levels: the receiver port is divided among destination virtual machines by tenant weight, each with its own policy cap; within a virtual machine, between the two classes by class weight; within a class, among flow sets by sender weight.

The **period** is the receiver's clock: once per period it reads the counters and obtains the bytes that arrived during that period. The period should be the length of one complete measurement: shorter than one refresh of the counters and you read duplicates, longer than the loop delay and you are adding delay of your own. Quantities in this design are expressed in bytes per period; a rate is that number divided by the period.

The **delivery tolerance** $\delta$ is one margin used throughout. An executor always delivers a few percent less than its setting, and measurements jitter, so whenever the question is "has this flow used up its quota", reaching $1-\delta$ of the quota counts as full; and whenever room for growth must be left above a measured usage, the same $\delta$ is added.

| Symbol | Meaning |
|---|---|
| $A$, $E$ | Bytes a flow set actually delivered this period; bytes the receiver entitles it to this period |
| $S$ | Share: the bytes a flow set is entitled to when every present flow set demands infinitely much |
| $Q$, $q$ | Virtual queue in bytes; normalised queue $q=Q/E\in[0,D]$, "how many periods' worth of entitlement is backed up" |
| $D$ | Repayment time scale in periods; also the cap on the virtual queue |
| $R$ | Permitted rate: the ceiling the sender maintains per flow set from the queue feedback |
| $R_{\max}$ | Probe ceiling: the line the permitted rate may not probe past |
| $U$ | Local cap: what the sender's own uplink allocates to this flow set |
| $r_i$, $c_i$ | Wire rate ceiling the executor enforces on the $i$-th flow of a flow set; that flow's own congestion-control quota |
| $\bar A_s$ | The sender's own measurement of this flow set's sending rate, averaged over about one observation window |
| $m$ | Empty-queue count: consecutive feedbacks with an empty queue, capped at $m_{\max}$, so $m\in[0,m_{\max}]$ |
| $\alpha$, $\kappa$ | Growth rate of the probe step; brake gain |
| $C$, $h$ | Receiver port rate; headroom fraction |
| $R_0$ | Starting rate for a new flow set |
| $\tau$ | Total loop delay in feedbacks |

## 3 Overall structure

There are three components. The **receiver**, once per period, reads the counters to get each flow set's arrivals $A$, computes the period's physical capacity, allocates to obtain each flow set's entitlement $E$, updates the virtual queues, and sends each sender the normalised queue length $q$. The **sender**, on every feedback, updates the permitted rate $R$, takes the minimum with its own uplink allocation, and hands the result to the executor. The **executor** feeds the permitted rate into the flow set's token bucket, from which each flow draws at its own congestion control's quota, and shapes per connection or per queue pair.

Two time scales run inside these components. The fast loop acts on every feedback and manages the permitted rate. The uplink allocation is recomputed once per observation window. The two must be kept apart, and the fast loop must not be driven by the allocation; the theory document states this as a condition. The tenant's congestion control reacts below the permitted rate on the scale of a round-trip time and belongs to neither loop.

## 4 Receiver

### 4.1 Counting

The receiver needs one kind of measurement: how many bytes arrived per period for each flow set, that is, arrival counts bucketed by source virtual machine, destination virtual machine and class. The period's arrival $A$ is the difference between two readings. Counting is the hardware's job; the design only asks that it be bucketed per flow set. If the hardware cannot resolve that finely, what to do about it is a platform matter, covered in `platform_notes.md`.

### 4.2 Presence and expiry

A flow set is present from the first byte it delivers. When no byte has arrived for one full expiry period (Section 7), it has expired: it is dropped from the ledger, and if it reappears it is a new flow set starting from scratch. Any arrival at all counts as presence, however small: a tenant sending a trickle is still a tenant, and the share it does not use is lent to others under Section 4.4.

### 4.3 Physical capacity

The port's capacity is expressed in bytes per period: the bytes the port can carry in one period, minus the bytes that arrived in that period which HyperFront does not schedule, times $(1-h)$. Unscheduled traffic is anything that is neither RDMA nor TCP; it occupies the port all the same. It is measured with about one observation window of smoothing so that a single burst does not move the capacity. $h$ is the headroom; it makes the ledger raise the alarm before the switch queue does, and it absorbs encapsulation overhead. The capacity also has a small floor, a few percent of the port rate: however much unscheduled traffic there is, it cannot push the ledger to zero, since a zero entitlement would leave the queue with nothing to be measured against.

Each virtual machine's cap is the amount its policy promises, taken at face value and reduced only by the unscheduled traffic addressed to that virtual machine. No headroom is applied there: headroom exists to keep the switch queue from building, and queues build only at the physical port, whereas a virtual machine cap is not a physical bottleneck. The cost of this choice is described in Section 5.4.

This rule fixes how HyperFront behaves when its own port is squeezed by outside traffic: the ledger shrinks with the physical capacity, tenants still divide it by policy, the switch queue does not build up from an over-committed ledger, and no tenant's congestion control gets crushed at this port. It does not cover the case where the port is not full and the congestion is elsewhere; see Section 8.

### 4.4 Shares and lending

Every period the receiver runs water-filling twice (fill the smallest demands first, then split what is left evenly among the larger ones until the capacity is gone), always in bytes for this period. The first run gives every present flow set an infinite demand; the result is each flow set's **share** $S$, the bytes it is entitled to this period. The second run gives lenders a finite demand and everyone else an infinite one; the result is the **allocation**.

A flow set is a **lender** when its average arrival over about one observation window, scaled up by $(1+\delta)$, is still below its share. The decision uses the average; the queue uses the raw per-period value. The per-source split of arrivals jitters by a few percent from one period to the next, and a decision that flipped with that jitter would make the borrowers' entitlement jump back and forth. In the second run a lender's demand is $\bar A(1+\delta)$, and what it does not use flows to its siblings. Its own entitlement $E$, however, stays equal to its share $S$: sending less does not shrink it. Every other flow set's $E$ is the result of the second run, its share plus whatever it borrowed.

The reason for this arrangement is that from arrivals alone the receiver cannot tell "the application only wants this much" from "something is holding it down to this much", so it does not try. In either case the lender's $E$ stays at its share and it can grow back to that share whenever it wants; the borrower gets genuinely spare capacity, and the moment the lender returns, the borrower's allocation shrinks back to its share, its excess sending goes into its own virtual queue, and the queue reclaims the lent capacity. Lending needs no separate threshold, because it happens inside the same allocation, and the capacity being allocated is physical, so what is lent can never exceed what the port really has.

Can a flow that sends little because its congestion control has backed off be mistaken for a lender? Only if it backs off by more than the tolerance $\delta$. At the managed port a congestion control backs off because the port has a queue, and the measured backing-off stays within the tolerance; backing off beyond it means something is on the port that the ledger does not manage (Section 8.1), the flow really cannot send more, and lending its spare capacity to its siblings is the right outcome. $\delta$ is therefore also how much backing-off at the managed port the ledger tolerates.

### 4.5 Virtual queue

$$Q\leftarrow\operatorname{clip}\big(Q+A-E,\ 0,\ D\,E\big),\qquad q=Q/E\in[0,D] .$$

Arrivals above the entitlement accumulate; arrivals below it drain, down to zero. The queue is capped at $D$ periods' worth of entitlement, so the normalised queue $q$ always lies between $0$ and $D$, and the formulas below use that range directly. The reason for capping at $D$: the longer the queue, the harder the sender is pushed down, and beyond $D$ further accumulation no longer deepens the repayment (Section 5.2).

### 4.6 Feedback

Exactly one quantity goes to the sender: the normalised queue length $q$, once per period. The queue length tells the sender how much it has over-taken and must repay. The change in the queue from one feedback to the next tells the sender how fast it is over-taking right now; that is the loop's damping, and it need not be sent, because the sender can subtract two consecutive values of $q$ itself.

When the queue is zero the sender has no information pointing at its share: the share may be a hair above or far away, and the queue reads zero either way, so the only thing to do is probe upward (Section 5.3). Zero does still say one thing: this period's arrival did not exceed the entitlement, which means the rate the flow set was running on the wire was one the receiver approved of. That is useless in steady state and useful at start-up (Section 5.4). No separate direction signal is sent, because everything a direction signal could say is already in the sequence of queue values: when the queue is non-empty its rise or fall is the direction, and when it is empty the only thing to do is probe.

## 5 Sender

### 5.1 Two quantities from the feedback

On each feedback the sender has two quantities: the queue length $q$ and the queue change $\Delta q=q-q_{\text{prev}}$. The change is signed and is only taken when this feedback or the previous one carried a queue; otherwise it is zero. Both are dimensionless, so the period length appears nowhere in the formulas.

### 5.2 The rate law: three factors

$$R\leftarrow R\cdot e^{\alpha m}\cdot e^{-\kappa\,\Delta q}\cdot e^{-\frac{\kappa}{D}\,q},\qquad
m\leftarrow\begin{cases}\min(m+1,\ m_{\max}),&q=0,\\[2pt]0,&q>0.\end{cases}$$

Each factor does one job.

**Probe**, $e^{\alpha m}$. $m$ counts consecutive feedbacks with an empty queue, this one included, capped at $m_{\max}$, and resets to zero the moment a queue appears, so with a queue this factor is exactly one and the law needs no separate switch. Without a queue the permitted rate probes upward geometrically. The step per feedback is $\alpha m$, growing linearly with how long the queue has been empty and capping at $m_{\max}$ feedbacks; $\alpha$ is the growth rate of the step, not the step itself. In the first few feedbacks after a repayment the step is close to zero, and the longer things stay quiet the larger it gets. The full probing rule and its justification are in Section 5.3.

**Brake**, $e^{-\kappa\,\Delta q}$. Whenever there is a queue, adjust multiplicatively by how much the queue moved in this one feedback, with sign. A growing queue means the flow set is over-taking harder right now, so brake harder; a draining queue means it is already below its entitlement, so give the permitted rate back at the rate the queue is draining. This is the loop's damping, and while a queue exists it is the only term that knows how far the permitted rate is from the entitlement, because the queue's change per feedback is exactly the ratio of arrival to entitlement minus one. $\kappa$ is the brake gain; the loop delay bounds it, $\kappa\,\tau\lesssim0.4$ (theory document, Section 2).

**Repayment**, $e^{-(\kappa/D)\,q}$. Reduce multiplicatively by the queue length: repay as much as was over-taken, until the queue drains to zero. $D$ is the repayment time scale, fixed by the damping ratio; $q$ is at most $D$ (Section 4.5), so this factor is never smaller than $e^{-\kappa}$ in one feedback.

All three factors depend only on dimensionless quantities, so the period length appears nowhere: when the period changes, the per-feedback constants are rescaled and the loop's behaviour in seconds is unchanged. $\alpha$ is the growth rate of the step, with units of one over feedbacks squared, so it scales with the square of the period; $\kappa$ scales linearly.

### 5.3 Probing: the full rule

Probing has a line it may not cross, $R_{\max}$, and that line only blocks the probe, never pushing the permitted rate down. The full probing rule is:

$$R\ \leftarrow\ \max\Big(R,\ \min\big(R\,e^{\alpha m},\ R_{\max}\big)\Big),\qquad
R_{\max}=\min\Big(\max\big(R_0,\ 2\bar A_s\big),\ U\,(1+\delta)\Big).$$

This step replaces the probe factor of Section 5.2; the brake and the repayment act after it as before. The inner $\min$ says the probe does not cross $R_{\max}$; the outer $\max$ says $R_{\max}$ only blocks the probe and never lowers the permitted rate. $R_{\max}$ is the smaller of two terms: twice the flow set's own sending rate, but never less than the starting rate $R_0$ (before any sending-rate measurement exists, this term is absent); and the local cap $U$ (Section 5.5) with one tolerance added. The empty-queue count keeps accumulating while the ceiling blocks the probe; it is not reset.

The reasons follow.

**Why the step grows with the quiet time.** While probing there is no information pointing at the share (Section 4.6); the one extra thing the sender knows is how long the queue has been absent, and the step can only be defined from that. Probing too hard costs something only when the permitted rate was already sitting at the share: that step crosses the line, the excess goes into the queue and takes time to repay. If the permitted rate was far below the share, the step was free. And the likelihood of "still sitting at the share" falls monotonically with quiet time: in the first feedbacks after being pushed down by a repayment it almost certainly is; after a long silence with no queue, the share has most likely moved up. So the step starts near zero and grows linearly with quiet time. The cap bounds the worst overshoot: on crossing the line the excess is about the step at that moment times the loop delay, so with the cap it is at worst $\alpha_{\max}\tau$. Why the growth is linear, and why probing fast and overshooting little are two sides of the same coin, is in the theory document, Section 3.

**Why there is a ceiling.** The stop signal for probing is the queue, but an application-limited flow always arrives below its entitlement and its queue is always zero, so its probe never gets told to stop: the permitted rate would climb all the way to the policy ceiling, and when the application's demand returned, the executor would release it at that ceiling, far above its share, in one step.

**The two terms of the ceiling.** Twice the flow set's own sending rate: that factor of two is not a tuned parameter; it is the meaning of "how much may the flow expand when the application comes back". The permitted rate stops at twice the actual usage, so the excess on return is at most one usage. Never less than $R_0$, because a flow that has just started and sent nothing yet would otherwise be locked in by $2\bar A_s\approx0$. The local cap with one tolerance added: permitted rate above the local cap can never be used and would only make the fall deeper when the share drops; the tolerance cannot be omitted, because Section 5.5 takes a capped flow set's demand to be its permitted rate, and if the permitted rate were pinned exactly at the local cap, the demand would equal the cap, the allocation would settle at a value it set for itself, and the local cap could never grow again.

**Why the ceiling never lowers the permitted rate.** If the permitted rate were clipped to $R_{\max}$ outright, a single dip in the measured sending rate would lower the permitted rate, the lower permitted rate would lower the sending rate, and the lower sending rate would lower the ceiling again, all the way down.

**Why the empty-queue count is not reset.** The step is already capped, so a long block only means the flow restarts at the maximum step when demand returns, and a flow that has long used only a fraction of its share really is far from it, which is exactly the intended behaviour.

### 5.4 Start-up

A new flow set has no feedback to start from. Its initial values are the following, all derived from policy and platform quantities; no rate is hard-coded.

- **Permitted rate** $R=\min(R_0,\ U)$, where $R_0=h\,C$ is the receiver port's headroom and $U$ is the local cap (Section 5.5).
- **Before the first permitted rate arrives, the executor releases at $R_0$**, so that when the permitted rate does arrive, nothing on the wire changes. Where the executor shapes at a finer grain than the flow set (RDMA is shaped per queue pair), it cannot yet tell which flow set a queue pair belongs to, so it divides $R_0$ by the number of queue pairs a flow set is expected to have; the expected number is a deployment quantity.
- **The absolute minimum of the permitted rate** is the lowest rate the executor can shape correctly (a platform quantity). It is not a starting value; it only keeps the permitted rate out of the region where the executor is inaccurate during repayment.
- **Empty-queue count** $m=m_{\max}$.

**The start window.** From the moment a flow set appears until the first observation that its sending rate is no more than $(1+\delta)$ times the permitted rate, the permitted rate has not yet actually taken hold of the flow. That interval is the start window, and once it ends it ends for good. Two rules hold inside it, and only inside it:

1. A queue produced during the window does not reset the empty-queue count; the brake and the repayment act as usual.
2. With an empty queue the permitted rate adopts the rate already running on the wire: $R\leftarrow\max(R,\ \min(\bar A_s,\ R_{\max}))$.

The reasons follow.

**Why the permitted rate starts at the headroom.** Two reasons. First, the headroom is precisely the capacity the receiver holds back for transients. A new flow set arriving at that rate cannot push the port past line rate even if everyone else has already used their full share, so it is the only starting point that is guaranteed harmless to others before any feedback exists. Second, it is almost always above the newcomer's true share, because shares shrink as more flow sets are present while the headroom stays fixed. So after start-up it is the ledger that brings the permitted rate down to the share, and that is the damped closed loop with a queue, which settles in $D$ periods. Starting far below and probing up would rely on the open-loop probe with its step capped at $\alpha_{\max}$, an order of magnitude slower over the same distance. The starting point can also be below the share (an almost empty port, a share near the virtual machine cap); then probing proceeds as usual, and the second rule lifts the permitted rate straight to whatever rate the executor is already releasing. The minimum with the local cap is taken because what the local uplink cannot provide could not be sent anyway, and a permitted rate above it would let the start window close before the permitted rate had actually taken hold of the flow.

This starting value has a cost. Virtual machine caps carry no headroom (Section 4.3), so when a new flow set joins a virtual machine that is already at its cap, the cap is exceeded by $R_0$ for the short time before the receiver notices, and the excess is dropped or queued wherever that cap is enforced.

**Why the empty-queue count starts at its cap.** A flow that has never seen a queue has no evidence at all about where its share is. That is the state of maximum uncertainty, and the one case where the maximum step is the correct step.

**Why the two start-window rules.** The first: the count means "how long since the last proof that the permitted rate is high enough", and a queue produced by a flow that was not yet under the permitted rate's control says nothing about where the permitted rate stands; that queue must still be repaid, so the brake and the repayment act as usual. This rule must be tied to a one-time window and not turned into a standing condition of the form "is the sending rate currently above the permitted rate": when the permitted rate falls quickly the sending rate lags behind it as a matter of course, and that queue genuinely was caused by the permitted rate and must reset the count. The second: the basis is the observation in Section 4.6 that an empty ledger means the receiver approved of the current arrival. It may act only inside the window, because $\bar A_s$ is a short-window measurement, and if it acted permanently every upward jitter would be recorded into the permitted rate by the $\max$ and never taken back, leaving the permitted rate driven by noise; and the problem it solves, a new flow spending hundreds of milliseconds probing toward a rate the executor was already releasing, only occurs there.

### 5.5 The local uplink

The ceiling handed to the executor is $\min(R,\ U)$, and never below the minimum rate. $U$ is the local cap: what the sender's own uplink allocates to this flow set, computed as described below. The reason this layer exists: the permitted rate says how much the receiver lets this flow set take, but the sender's own uplink is finite too, it is shared by every flow set on that host, and each receiver sends its permitted rates independently without knowing what else hangs off that link. For a host serving several receivers at once, a sum of permitted rates larger than the NIC can carry is the normal case, not the exception. Without a local layer, how the excess gets divided is decided by the NIC's and the executor's scheduling, and tenant policy fails precisely when resources are short. A flow's rate is constrained by every link it crosses; each link's owner computes a share, and the flow takes the smallest.

The local allocation has the same three-level policy structure as the receiver's: source virtual machines divided by weight and bounded by their own caps, classes within a virtual machine by class weight, flow sets within a class equally; the capacity is the local link rate times $(1-h)$. The algorithm is the same two rounds of water-filling as Section 4.4. The first round gives every flow set an infinite demand and yields each flow set's share. The second round has to separate the flow sets that want more from the ones that do not, using only what the sender itself has. A flow set whose sending fills the ceiling we gave it (within the delivery tolerance) wants more; call it **capped**. Its demand is its permitted rate $R$: the receiver lets it have that much, and it could not use more. A flow set that does not fill its ceiling is limited by its own application; its demand is its own usage plus one tolerance. $U$ is the larger of the second-round allocation and the share.

A capped flow set gets its share plus whatever its siblings leave unused. An uncapped flow set keeps its share; it is not shrunk for sending less now, and it can grow back the moment demand returns. That is the same rule Section 4.4 applies to lenders. It also settles what the ceiling means for a flow set that has not started sending: it is uncapped, its $U$ equals its share, which is positive, so a new flow has room to start.

Why the demand cannot simply be the measured sending rate is a matter of loops. Sending depends on the ceiling, the ceiling on $U$, and $U$ on the demand. For a flow that we ourselves are holding down, using "how much did it send" as its demand closes a loop with no damping and no stability criterion, and when that loop is coupled to the damped permitted-rate loop, the transient is dominated by the undamped one. The rule above breaks that loop: a capped flow set's demand comes from the receiver's damped loop, an uncapped flow set's sending is set by its application, and neither is an output of the local allocation. The capped decision is a downstream observation, but it is used only to classify, never to produce a number. Being a ratio test, it can flip between recomputations for a flow set sitting right on the threshold. That is harmless: the uncapped side keeps its share, the two classifications differ only in the borrowed part, and the descent of $U$ is rate-limited anyway (next paragraph).

**Recomputation and taking effect.** The local allocation is recomputed once per observation window, and once immediately whenever the set of flow sets changes. An increase in $U$ takes effect at once; a decrease is rate-limited by ratio, one step per executor update interval and at most a halving per step. It is not recomputed on every feedback because a ceiling that moves at the same frequency as the thing it constrains is not a ceiling but a second controller, and the structure of contention (who is sending, to whom) changes far more slowly. The descent limit constrains the ratio of the fall, not the destination: a fall that is too abrupt is read by the tenant's transport as a failure, while running steadily at the same low rate is safe. This layer also arbitrates borrowing between classes across the two executors, since both hang off the same sender. Every quantity the test uses is in the sender's own hands: the ceilings we issued, the NIC's per-virtual-machine and per-class transmit counts, the policy and the link rate. Nothing from inside the tenant's virtual machine is needed.

### 5.6 Loss of feedback

Feedback from the receiver can stop: the receiver restarted, a link failed, or the flow set is no longer there. The sender handles this in three stages. While feedback has been missing for less than one hold period, the permitted rate stays where it is. Missing a feedback or two is normal, the rate law needs a fresh queue to act on, and acting on a stale one is acting on a state that no longer exists. Past the hold period, the permitted rate glides toward the local cap $U$; from then on the flow is bounded only by the sender's own policy, so it can neither run away nor starve. Past the purge period with still no feedback, the flow set is removed from the sender, but only while the feedback channel itself is alive, meaning feedback for other flow sets has arrived recently. A flow set that alone stops getting feedback has expired at the receiver; if all feedback stops together, the receiver is gone, and in that case nothing is removed: every flow set runs at $U$ until feedback returns. All three times are in Section 7.

## 6 Executor

The executor is the layer closest to the wire. From the sender it receives one thing: the flow set's rate ceiling, the $\min(R,U)$ of Section 5.5, written $R$ in this section. On its own it can read the current quota $c_i$ of every flow in the flow set, where a flow is a queue pair for RDMA and a connection for TCP. Its output is a wire rate ceiling $r_i$ per flow.

### 6.1 The token bucket

Each flow set has one token bucket, filled at rate $R$, from which every flow of the set draws tokens at its own quota $c_i$. While the bucket has tokens every flow sends at $c_i$; when it is empty, the flows share it in proportion to how fast they draw:

$$r_i=c_i\cdot\min\Big(1,\ \frac{R}{\sum_j c_j}\Big),$$

where the sum runs over the flows that are actually drawing. That is the whole rule: HyperFront supplies $R$ and the bucket, has no policy inside the flow set, and leaves everything below $R$ to the tenant's congestion control.

The quota $c_i$ is defined per class. For RDMA, $c_i$ is the current rate of the queue pair's own congestion control, a window divided by the control's own round-trip estimate for a window-based one; the executor shapes per queue pair and recomputes the denominator over the queue pairs that are actually sending, on a tick much shorter than the period. For TCP, $c_i$ is the connection's congestion window divided by its round-trip time; the connections of one flow set share a path and therefore a round-trip time, so the draw ratio is the window ratio, $r_i=R\,w_i/\sum_j w_j$ with $w_i$ the connection's congestion window. Every connection has its own departure clock, so one connection's backlog never becomes another's delay.

### 6.2 Why it is defined this way

**The division of labour follows who knows what.** The ledger knows the shares and divides the port capacity into $R$ once per period; the congestion control knows the path and reacts to marks, delay and loss on the scale of a round-trip time. The bucket draws the line between them at $R$: above $R$ the ledger rules, below $R$ the congestion control rules, and neither rewrites the other's output.

**The reason for keeping the congestion control is honoured here.** The flows of one flow set may belong to different applications; a common congestion control keeps them fair to one another and is the layer that per-flow load balancing in a datacenter relies on. The bucket leaves that layer alone: how the flows split the flow set is decided by how fast each draws, exactly as they would split a bottleneck of capacity $R$ among themselves; a flow with nothing to send draws nothing and holds no share.

**The congestion control must actually reach the wire.** The ledger allocates per period and only governs averages; queues below the period can only be held down by the congestion control. An executor that pushes a flow set back to $R$ no matter how far its congestion control backs off leaves the managed port's queue in nobody's hands: measured, such an executor multiplies the receiver port's ECN marks tens of times over, whereas the bucket lets every retreat of the congestion control land on the wire unchanged, and the port marks exactly as it does with the congestion control running alone. This is also why HyperFront does not simply replace the congestion control: the ledger's period is too long for anything but flow control, and congestion below the period is the congestion control's job.

**No lifting.** At the managed port every competitor is already shaped to its share, so a congestion control can only be backing off there because the port has a queue, and that queue drains only if all the congestion controls back off together; pushing one of them back to its share means pouring into a queue it can see. The size of the retreat is absorbed by the ledger's tolerance $\delta$ (Section 4.4).

**No parameters.** The formula contains only $R$ and the flows' $c_i$: no window, no threshold, no time constant. It does not judge whether the congestion control's rate can be trusted, and it reads neither loss, nor marks, nor queues.

### 6.3 Properties and boundaries

Four properties, proved in Section 4 of the theory document. **Never above:** a flow set's wire rate never exceeds $R$, for any congestion control and any number of flows. **Full delivery:** when the flows together ask for at least $R$, the wire carries $R$; what one flow asks less, the others take in proportion to their draw. **Proportional:** while the bucket binds, the flows' rates stand in the ratio of their quotas; while it does not, every flow runs at the rate its congestion control would run alone. **Neither delayed nor amplified:** every move of the congestion control below $R$ lands on the wire as it is; the executor neither smooths it nor sends more on its behalf.

Three boundaries. **The bucket cannot take a flow set above the sum of its congestion controls:** when the quotas add up to less than $R$ the flow set delivers less than $R$; the ledger counts what arrives, and only a flow set below $(1-\delta)$ of its share is judged a lender (Section 4.4). At the managed port the measured retreat stays within the tolerance; a retreat beyond it means something on the port that the ledger does not manage, the boundary of Section 8.1. **Start-up:** before the first $R$ arrives, flows are released as in Section 5.4. **Granularity of the emulation:** RDMA hardware has no bucket shared across queue pairs, so the executor writes $r_i$ into each queue pair's shaper at every event and recomputes the denominator on a tick much shorter than the period; the difference between a $c_i$ taken now and a denominator taken a tick earlier is visible at that tick and invisible at the period.

## 7 Parameters

Parameters are not design: on a different platform they must be set again. The table gives how each one is set; the last column is our platform's value, as an example only. Which constraint each one comes from, and in what order to redo them at a different scale, is in the theory document, Section 6: two measurements and two dimensionless choices, and the rest is algebra.

| Parameter | Meaning | How it is set | Our platform |
|---|---|---|---|
| $T_p$ | Period | One complete measurement; no shorter than one counter refresh, much shorter than the loop delay | 10 ms |
| $\tau$ | Total loop delay | Measured on the platform | 3 feedbacks |
| $\kappa$ | Brake gain | $\kappa\tau\lesssim0.4$ | 0.1 |
| $D$ | Repayment time scale, queue cap | From the damping ratio, $\zeta=\tfrac12\sqrt{\kappa D}\approx0.87$ | 30 periods |
| $\alpha$ | Growth rate of the probe step | $\alpha=\alpha_{\max}/m_{\max}$, with $\alpha_{\max}$ from the overshoot budget ($\alpha_{\max}\tau$ at most one tenth) | $3\times10^{-4}$ |
| $m_{\max}$ | Cap on the empty-queue count | How long a silence before the share is believed to have moved; a policy choice | 100 feedbacks (1 s) |
| $\delta$ | Delivery tolerance | Clearly larger than the executor's delivery shortfall plus measurement jitter; the larger it is, the less siblings can borrow | 15% |
| $h$ | Headroom | Encapsulation overhead plus the margin that lets the ledger warn before the switch queue | 8% |
| Capacity floor | Capacity retained no matter how much unscheduled traffic there is | A few percent of the port rate | 5% |
| Expiry period | How long a flow set must be silent before the receiver declares it gone | Longer than an application's normal pauses; the longer it is, the longer a departed flow set holds its share | 2 s |
| Observation window | Shared by the lender decision, the probe ceiling and the uplink recomputation | Much longer than any congestion control's adjustment period, much shorter than the time scale on which shares change | 100 ms |
| Minimum rate | Absolute floor on the permitted rate | Lowest rate the executor shapes correctly | 1 G |
| Expected queue pairs per flow set | Allowance per unknown queue pair before the first permitted rate $=R_0/$this | A deployment quantity | 4 |
| Local cap descent | Step size when $U$ falls | At most a halving per step, one step being the executor's update interval | Halving per 13 ms |
| Channel liveness, hold, purge | The three times after feedback stops | Liveness slightly longer than one lost feedback; hold longer than a normal receiver restart; purge longer than any reasonable outage | 0.25 s, 2 s, 30 s |

The starting rate is not a parameter but a derived quantity, $R_0=h\,C$. The factor of two in the probe ceiling and the delivery tolerance are not tuning knobs.

## 8 Boundaries and open problems

### 8.1 Inherent boundaries

**Congestion where the receiver cannot see it.** When the port is not full and the ledger is correct, only the congestion control knows the path is congested. Such a bottleneck is handled by the congestion controls on both sides below $R$ (Section 6.3); the flow set's permitted rate does not change because of it, and fairness over that stretch is decided by those congestion controls. The receiver will treat the throttled flow set as a lender and lend its spare capacity to its siblings, who may be crossing the same congested path; the behaviour of the lending rule over such a stretch has not been validated.

**How fast the loop can be is set by the delay.** The largest component of the delay is the receiver turning counters into rates. Getting past that constraint would require the sender to predict the consequences of its own actions, which this design does not do.

**The local ceilings can sum to more than the local capacity.** Uncapped flow sets keep their shares while capped siblings take what they leave unused. Unlike the receiver, the sender has no virtual queue to reclaim lent capacity; it can only wait for the next recomputation. So when a long-idle flow set suddenly sends at full rate, the local uplink is briefly over-committed, the excess is absorbed by the NIC's own queueing, and the condition lasts at most one recomputation interval.

### 8.2 Open problems

**Self-reference in the probe ceiling.** The ceiling of Section 5.3 contains the term $2\bar A_s$, derived from the flow's own sending. When a flow uses less than half its permitted rate for a long time for reasons unrelated to the permitted rate, the ceiling sits below twice its actual usage, the permitted rate can never climb, and the flow is locked in by its own usage. The control plane's behaviour in this situation is correct by definition: it sees a flow using a little with an empty queue and should call it application-limited. This is the ambiguity of Section 4.4, "cannot tell wanting less from being held down", appearing at the sender. The floor at $R_0$ only guarantees the permitted rate is not pinned below $R_0$; above $R_0$ it still can be. This design accepts the residual and requires that it be visible: the executor's report should show directly whether a flow set is currently pinned by the ceiling.

**The incumbents' repayment dip on a join.** The moment the receiver discovers a new flow set, the incumbents' entitlement shrinks, but their sending takes one loop delay to follow, and the bytes sent in between go into the ledger. During repayment their permitted rate dips to about one tenth below the new entitlement. The dip itself is the repayment behaviour of Section 5.2; its depth is set by the ledger accumulated before the newcomer was discovered, and the only way to reduce it is for the receiver to discover new flow sets faster.

## 9 Related documents

- `design_theory_v4_en.md`: what this control is mathematically, what it guarantees, and how to retune it at a different scale.
- `platform_notes.md`: numbers and environmental facts measured on our platform, to be re-measured on any other.
- `hpft-implementation/docs/IMPLEMENTATION.md`: where each rule lives in the code, known differences between implementation and design, switches.
- `hpft-implementation/validation/README.md` and `STATUS.md`: validation scenarios, criteria and current status.

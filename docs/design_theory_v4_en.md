# HyperFront Control Theory (v4)

The design document (`design_v4_en.md`) says what the system does and why each rule is the way it is. This document covers three things: what the control is mathematically, what it guarantees, and how to set the constants on a different platform. Notation follows Section 2 of the design document. The unit of time throughout is one feedback; multiply by the period $T_p$ to convert to seconds.

## 1 What the control is mathematically

It consists of three parts running on three different time scales.

The **fast loop** acts once per feedback. Its state is the log ratio of the permitted rate to the entitlement, $x=\ln(R/E)$, and the normalised queue $q$. It has two regimes. With a non-empty queue ($q>0$) the loop has a usable error signal and is a damped second-order loop. With an empty queue ($q=0$) the loop carries no information about the share at all, and the only option is to probe blindly, with a step set by how long the queue has been empty.

The **slow loop** acts on the executor's own tick. Its state is the confidence $T$. It does not change the permitted rate; it changes only how far below the permitted rate the executor lets the tenant's congestion control go. Its inputs are loss (raising it) and the queue and time (lowering it); its output goes only into the executor's lower bound.

The **local allocation** is recomputed once per observation window and is constant across any single feedback of the fast loop.

Keeping the three time scales apart is the premise of every result below; Section 5 states it as a condition.

## 2 Non-empty queue: a second-order loop

Assume the flow is bounded by the permitted rate ($A\approx R$, zero confidence). Per feedback, the queue changes by the ratio of arrival to entitlement minus one:

$$\Delta q=\frac{A-E}{E}\approx x .$$

This is the pivot of the whole analysis: the first difference of the queue is the log error of the permitted rate against the share. The receiver sends a single scalar $q$; the sender reads the error itself from the difference of two consecutive feedbacks and the integral of the error from the value of $q$. One scalar carries everything a second-order loop needs, which is why sending only the queue length does not weaken the control.

With $q>0$ the rate law reduces to the brake and the repayment. In logs,

$$\Delta x=-\kappa\,\Delta q-\frac{\kappa}{D}\,q\approx-\kappa x-\frac{\kappa}{D}\,q .$$

Combined with $\Delta q\approx x$ this is a standard second-order system:

$$\ddot q+\kappa\,\dot q+\frac{\kappa}{D}\,q=0,\qquad \zeta=\tfrac12\sqrt{\kappa D}.$$

$\kappa$ is the strength of the damping, $D$ the depth of the repayment; their product sets the damping ratio $\zeta$. With $\zeta\approx0.87$ the queue overshoots once and then drains monotonically to zero, with neither oscillation nor a long tail.

**Delay bounds $\kappa$ from above.** A change in the permitted rate shows up in the queue $\tau$ feedbacks later, so every step acts on a state that is already stale. The loop is most active at frequencies around $\kappa$, and there the delay costs about $\kappa\tau$ radians of phase. For the actual damping to stay close to the design value, that loss must stay within about 0.4 radians:

$$\kappa\,\tau\lesssim0.4 .$$

This is the first constraint on the whole parameter set, and the only one that ties a physical quantity of the platform (the delay) to a choice of the control (the damping strength). Raising $\kappa$ does not buy speed: beyond this line the delay itself becomes a source of oscillation, and the loop gets slower, not faster.

## 3 Empty queue: a conservation law for probing

With a zero queue the loop has no error signal and can only probe blindly. How well that can be done has an upper bound that does not depend on any parameter.

Suppose the permitted rate is a log distance $L$ below the share and the probe raises $x$ by a fixed step $s$ per feedback. Covering the distance takes $L/s$ feedbacks. At the moment the line is crossed, the receiver does not see it for another $\tau$ feedbacks, during which the permitted rate keeps climbing, so the overshoot is about $s\tau$. Multiply the two:

$$\frac{L}{s}\times s\tau=L\,\tau .$$

For any fixed-step blind probe, the product of discovery time and overshoot equals $L\tau$, independent of the step. Changing the step only slides along that hyperbola. Probing fast and overshooting little are two sides of one coin, and no better constant can improve both.

There is only one way out: the step must depend on information, and the only information available with an empty queue is how long it has been empty, the empty-queue count $m$. Section 5.3 of the design document accordingly sets $s=\alpha\hat m$, justified by how the cost is distributed: probing too hard costs something only when the permitted rate was already at the share, and the likelihood of that falls monotonically with quiet time. In steady state a line crossing comes right after a repayment, $\hat m$ is a few feedbacks, and the overshoot is small; when the share has genuinely moved up, the queue stays absent, $m$ keeps growing, and the step becomes aggressive on its own. One rule produces two behaviours in the two situations, which no fixed step can do. The cap $m_{\max}$ puts a hard bound on the worst overshoot, $\alpha_{\max}\tau$.

The steady-state sawtooth follows: the permitted rate overshoots the share by about the step at the moment of crossing times the delay, $\alpha\hat m_c\tau$.

## 4 Three properties

The following hold for any tenant congestion control satisfying the premises of Section 1 of the design document, regardless of parameter values.

**Property 1: fairness cannot be bought.** The wire rate is $r=\operatorname{clip}(c,(1-T)R,R)$, so for any confidence and any quota $c$ the congestion control produces, $r\le R$. The upper end is the permitted rate at every confidence; a more aggressive congestion control only raises $c$, and the excess is cut off with no effect on the wire. This turns fairness from an equilibrium that holds only if the other side also plays by the rules into an invariant enforced from one side. The excess has two bounds, both set directly by parameters: the instantaneous excess is at most $\alpha_{\max}\tau$ (Section 3), and the cumulative excess is at most $D$ periods of entitlement, because the virtual queue is clipped at $D\,E$. The ledger is a bounded memory, not a debt.

**Property 2: transparent to the tenant's congestion control, with bounded yielding.** When $c$ lies inside the interval, $r=c$, untouched: the tenant's congestion control sees the network responding to exactly the rate it chose, and its control loop stays intact. A weighted average instead would show the tenant the network's response to a rate we altered, and the two loops would interfere. The other half is the lower bound $r\ge(1-T)R$: a flow can be pushed below its share by its own congestion control only by the amount we explicitly grant, so a congestion control that only ever ratchets down cannot drag the flow to zero. The upper bound protects others from this flow; the lower bound protects this flow from its own congestion control; in between, nothing is touched.

**Property 3: confidence is a one-way permission, hence fail-safe.** Confidence lets a flow send less, never more, so granting it is always safe for everyone else: if the estimate is wrong, the flow itself pays. The error in the other direction, a hidden bottleneck that really exists while we keep pushing the permitted rate, is corrected by loss, a fact of the network and not the private signal of any congestion control. The same asymmetry forces confidence to expire on its own. Loss can prove that a bottleneck exists, but when the bottleneck disappears no new signal appears anywhere; the only change is that loss stops. And a flow with high confidence, whose upper bound is still the permitted rate, can never trigger the "took too much" path down. So confidence is held up by recurring evidence and, when the evidence stops, decays to zero with time constant $\tau_d$. The cost is that a bottleneck that still exists but has stopped dropping is gently re-probed from time to time.

## 5 Scale invariance

**Proposition.** The stability of the fast loop and the relative size of the steady-state sawtooth depend neither on link capacity nor on the number of flow sets.

Independence from capacity follows from the law being purely multiplicative: the state is the log $x$, the feedback is the normalised queue $q$, and no term of the second-order loop of Section 2 carries the dimension of an absolute rate. Multiply the capacity by ten, every $E$ is multiplied by ten, and $x$ and $q$ are unchanged point for point.

Independence from the number of flows follows from each flow set having its own ledger and its own permitted rate, with no direct coupling between fast loops; the only link is that the receiver's allocation sets $E$. With the port saturated, $E$ is determined by how many flow sets are present and their weights, not by anyone's arrivals, so what other flows do does not change this flow's $E$. Aggregate overshoot behaves the same way: each flow exceeds its own share by the fraction $\alpha\hat m\tau$, and the sum exceeds the port capacity by the same fraction.

Two conditions are needed, and both must be checked when moving platforms: the lender decision must be an order of magnitude slower than the fast loop, or $E$ stops being a slowly varying parameter and becomes a coupling; and no share may be so small that it is of the same order as the minimum rate, since such a flow no longer operates in the multiplicative regime.

**Corollary.** Changing scale requires no retuning. The only quantities that must be re-measured are two physical quantities of the platform: the period $T_p$ and the loop delay $\tau$.

## 6 Tuning recipe

Scale invariance shrinks tuning to a small job: two measurements, two design choices, and the rest is algebra.

**Two measurements.** $T_p$, the period: the length of one complete measurement, no shorter than one counter refresh and much shorter than the loop delay. $\tau$, the loop delay: the time from a change in the permitted rate to the receiver seeing it in the queue and the feedback returning to the sender, expressed in feedbacks, $\tau=\tau_{\text{wall}}/T_p$.

**Two design choices.** $\zeta$, the damping ratio, 0.87: one overshoot, then a monotone drain; take 1 to be more conservative. $\varepsilon$, the overshoot budget, the worst-case fraction by which the permitted rate may exceed the share, 0.1.

**Algebra.**

$$\kappa=\frac{0.4}{\tau},\qquad D=\frac{4\zeta^{2}}{\kappa},\qquad \alpha_{\max}=\frac{\varepsilon}{\tau},\qquad \alpha=\frac{\alpha_{\max}}{m_{\max}} .$$

$m_{\max}$ is the one quantity set by policy rather than physics: how long a silence before you are confident the share has really moved. The causes of share changes (flows joining and leaving, tenants added and removed) are all much slower than a second, so one second is used.

The remaining parameters are independent of the loop and are set on their own. $\delta$ is clearly larger than the executor's delivery shortfall plus measurement jitter. $h$ is the encapsulation overhead plus the margin that lets the ledger warn before the switch queue. $\tau_r$ is much larger than the fast loop's settling time $D$, or the two loops interfere. $\tau_d$ is much larger than $\tau_r$, or the gap between two losses would drain confidence that has just been raised.

**Worked example.** $T_p=10$ ms, $\tau_{\text{wall}}\approx30$ ms, so $\tau=3$ feedbacks: $\kappa\approx0.13$, rounded to 0.1; $D\approx30$; $\alpha_{\max}\approx0.03$; $m_{\max}=100$ and $\alpha=3\times10^{-4}$. These match the design document's parameter table line by line.

**What changes on a new platform.** A change in capacity or tenant count: nothing. A change in period: the delay in feedbacks changes with it, so recompute $\kappa$, $D$, $\alpha_{\max}$ and $m_{\max}$. A change in delay: recompute $\kappa$, $D$ and $\alpha_{\max}$. A different executor: re-measure $\delta$ only. Only the delay genuinely moves the whole parameter set, and it does so in a fully determined way: $\kappa$ is inversely proportional to $\tau$ and $D$ is proportional to it. Triple the delay and the loop's settling time goes up nine-fold. That is the price of delay, and tuning cannot avoid it.

**Self-check.** Once the parameters are set, stop the sender agent and drive the executor directly with a square wave, and measure two things. The executor's own latency from command to change on the wire should be much smaller than $\tau_{\text{wall}}$; if not, the delay is made up differently from what was assumed and $\tau$ must be re-measured. The closed-loop sawtooth height should land near $\alpha\hat m_c\tau$; if it is larger, measurement noise is usually leaving the queue empty for a few extra feedbacks.

## 7 What this theory does not cover

**The empty-queue regime has no stability to speak of.** The conservation law is an upper bound on it, not a stability guarantee; its behaviour is set by the rule that the step grows with quiet time, and that rule rests on the distribution of cost, not on a proof.

**The second-order loop is a small-signal approximation.** $e^{x}-1\approx x$ holds for moderate $|x|$. A large disturbance such as the share halving is at the edge of the approximation; the actual repayment is somewhat more aggressive than the linear prediction, which is the safe direction, but quantitative sawtooth predictions should not be trusted under large disturbances.

**The separation of the three time scales is uneven.** The fast loop settles in $D$ feedbacks; the local allocation is recomputed once per observation window, an order of magnitude slower than one feedback but faster than the fast loop's settling; confidence rises only a few times slower than the fast loop settles. Strict separation holds only at the level of "the local allocation is constant across one feedback". That is why the local cap applies increases at once and rate-limits decreases: the asymmetry is what keeps it from fighting the fast loop in practice.

**Property 1 relies on an honest executor.** $r\le R$ is the executor's arithmetic and holds only if the executor actually shapes to $r$. A defect inside the executor can breach the bound without violating any design rule, and only measurement can find it.

**The receiver's measurement accuracy is outside the model.** The analysis takes $A$ to be the flow set's true arrival. Measurement error enters $q$ directly; it is the source of the residual steady-state jitter and the reason that jitter cannot be tuned away.

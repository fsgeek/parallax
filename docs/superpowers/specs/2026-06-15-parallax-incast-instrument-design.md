# Parallax — Leg B: The Incast Instrument

**Date:** 2026-06-15
**Status:** Design — pre-implementation
**Repo lineage:** formerly `vmtp` (the February VMTPsim seed); sibling of `eidolon` (the far-end, published leg)

---

## 0. Why this exists (the thesis in one breath)

**Transaction-oriented, reciprocal transport is the correct model wherever closed-loop feedback control is unavailable — and that failure occurs at both ends of the *feedback-staleness axis* (sub-RTT incast and mega-RTT deep-space), while the WAN middle camouflages it.**

That sentence is the *thesis*, not a paper. Parallax is named for the method it embodies: true structure is revealed by observing one phenomenon from two widely separated vantage points. Here the two vantages are **datacenter incast** (the event is shorter than one RTT — feedback too slow *relative to the event*) and **interplanetary delay** (feedback too slow in absolute terms, and intermittent). The ordinary WAN middle hides the coupling precisely because feedback-latency ≈ control-timescale.

### The legs of the thesis
| Leg | Claim | Status |
|---|---|---|
| **A — Far end** | Quorum consensus survives extreme latency + intermittent disruption; quorum *shape* encodes topology | **Done, published** (`eidolon`, arXiv/Zenodo) |
| **B — Near end** | Transaction-aware + reciprocal scheduling beats greedy streams under incast | **This spec.** The missing leg. |
| **C — The axis** | A and B are *one* phenomenon; feedback staleness is the unifying variable | Synthesis. Gated on B existing + a novelty pass. |
| **D — Mechanism** | A QUIC-*recognizable* (incompatible) stack whose unit is the declared transaction | Later. The artifact, not the argument. |

**Leg B's job is to be the *teeth* for the axis:** produce the single demonstration where the feedback-staleness lens predicts a win the greedy/closed-loop lens cannot even represent. Not "a faster transport" — a result one lens sees and the other is structurally blind to.

---

## 1. The void this fixes

The February seed's `network.py` is a **delay/loss oracle**: constant `loss_probability`, fixed delay ± jitter, per-packet independent delivery. No link capacity, no buffer, no queue. **Congestion cannot occur in it by construction** — loss is exogenous and constant, not load-dependent. The prior "VMTP doesn't help" conclusion measured consensus on an instrument where congestion was *structurally inexpressible*; that verdict is **void for the congestion claim**, not merely over-scoped.

The signature of a congestion-capable simulator is one property: **loss and delay must be functions of offered load.** Leg B installs exactly that.

---

## 2. Architecture (single bottleneck, two transports, N-sweep)

Topology — the only thing that matters for incast:

```
                       request (small, fan-OUT — uncongested)
   aggregator  ────────────────────────────────────────────────▶  N workers
       ▲                                                               │
       │            responses (fan-IN — THE congested direction)       │
       └────────────────────────[ BottleneckLink ]◀───────────────────┘
                            finite rate C, bounded buffer B
```

One request fans out to N workers; all N reply with a fixed-size response at ~the same instant; the replies converge on the aggregator's inbound link — the single shared bottleneck where incast lives. **Sweeping N (fan-in degree) is the experiment.**

| Component | Status | Role |
|---|---|---|
| `BottleneckLink` | **NEW (heart)** | Finite rate, bounded buffer, serialization, **load-dependent drop**, optional ECN |
| `GreedyStream` | **NEW/adapted** | Probe-based baseline (TCP-like; DCTCP variant in wave 2) — the stream-oblivious lens |
| `DeclaredTransaction` | **NEW** | VMTP-style; receiver-granted, declared response group — the axis lens |
| `IncastScenario` | **NEW** | Wires partition-aggregate, drives the N-sweep, seeds |
| entities / packet / client-server state machines | **reused** | Untouched transaction machinery |
| `metrics` | **extended** | + goodput-vs-N, p99, queue occupancy, collapse point, synchronized-loss |
| TCP/QUIC baselines | **reused** | Reference, adapted to traverse the bottleneck |

**Honest boundary:** this models *the fan-in point*, not a datacenter fabric. Leaf-spine (Approach 2) and ns-3/htsim (Approach 3) are the **named, deferred escalation path** if a fabric-level skeptic ever demands it.

---

## 3. The `BottleneckLink` (Section 2)

**Parameters:** `rate_C` (bytes/s), `buffer_B` (**bytes** — the incast knob), `packet_size` (fixed for now), `prop_delay` (one-way), `ecn_K` (CE-mark threshold; **off by default**).

**Mechanics (SimPy):** a FIFO queue plus one server process — dequeue, `yield timeout(size/rate_C)` (serialization), deliver after `prop_delay`. The only place loss happens: enqueue checks `queued_bytes + size > buffer_B` → **tail-drop**. That single line is load-dependent loss.

**Decisions, with rationale:**
1. **Drop-tail FIFO is the default discipline** — the canonical incast environment the textbook collapse curve assumes (calibration needs it). ECN-marking and strict-priority are *pluggable variants* so baselines get a fair fight.
2. **Buffer accounted in bytes**, not packets — collapse is sensitive to buffer-vs-BDP in bytes; mixed response sizes will matter later.
3. **Only the fan-in direction is modeled in detail.** The request (fan-out) is small/uncongested — a plain delay.

**Credibility invariant — the link is a BLACK BOX to transports.** They never read queue depth; they see only acks, timeouts, delay, and (if ECN on) CE marks. The transaction transport's edge must come from information it *legitimately has* — that it issued one request to N known responders — never from oracular queue visibility. Forbidden at the interface, by construction.

---

## 4. The two transports (Section 3)

**A. `GreedyStream` (baseline — stream-oblivious lens).** Each worker's response is an independent window-based stream: slow-start, congestion-avoidance, multiplicative decrease on loss. Fairly tuned (modern IW, standard RTO) — *not* a strawman; it must reproduce the textbook collapse. The defining property is what it lacks: **no stream knows the other N−1 exist.** All N ramp into the same buffer at once. That obliviousness *is* incast.

**B. `DeclaredTransaction` (axis lens — declared, not probed).** The request is one logical transaction issued to a *known set* of N responders. The aggregator issues **grants** pacing the aggregate response to ≤ its own link rate, so the buffer never overflows — it can, because it knows the full response set: it asked them. Built from QUIC-recognizable nouns (Leg-D principle, so the sim doubles as the stack blueprint): declared-size response units (≈ streams/datagrams), grants (≈ flow control), entity handles (≈ connection IDs), one transaction ID.

**The teeth = an asymmetry, not a speedup.** `GreedyStream` *cannot* do the pacing — not "worse," cannot — because it has no group to schedule; to it there are only N strangers. The coordination is a thing the transaction *primitive* makes expressible and the stream primitive makes unsayable. Same physics; one lens can represent the win, the other is blind to it.

**Fairness invariants:** both black-box to the queue; the transaction transport uses only its own link rate (an endpoint knows its own NIC speed) and the responder set it issued the request to. Granting-to-your-own-downlink is exactly what Homa/NDP do — defensible, not oracular.

**Homa honesty (designed in, not discovered in review):** receiver-granted pacing *resembles Homa* deliberately. Leg B does **not** claim to beat Homa on bare incast — it claims the win *falls out of the transaction primitive for free* where the stream primitive can't express it. Wave 2 includes a **Homa-class baseline**; the expected honest result is `DeclaredTransaction ≈ Homa` on pure incast, with the transaction primitive's *extra* (at-most-once safety, multicast-response semantics) measured as a **separate** question in its own regime. We subsume Homa's win and add safety; we never pretend to outrun it on its home turf.

**Coordination mechanism (chosen):** **receiver-driven grants** for Leg B (clean, recognizable, the known-good Homa shape). The **sender-reciprocal (ayni)** variant — responders coordinate among themselves with no central grantor — is held as its own *later* leg, because that is the mechanism that must work when the coordinator is 22 light-minutes away (the bridge to the far end).

---

## 5. Calibration, metrics, kill criterion (Section 4)

**Calibration (gate on the instrument, before any comparison):** run `GreedyStream` alone, sweep N, shallow buffer. It must reproduce textbook TCP incast collapse — goodput climbs then *cliffs* as synchronized overflow drives timeouts and RTO-idle dead air. Validate **shape and trends** (deeper buffer → later cliff; smaller RTO → partial mitigation), not a magic number. If it won't collapse, the instrument is wrong and nothing downstream counts.

**Metrics:** goodput-vs-N (headline); p99/p99.9 transaction completion; queue occupancy over time (show the mechanism); collapse point; **synchronized-loss count at the burst instant** (the legibility metric — greedy's retransmit explosion vs. the granted transaction's near-zero drops).

**Kill criterion — pre-registered, written before any run.** Survives **only if** `DeclaredTransaction` holds goodput flat (or defers collapse substantially) where `GreedyStream` cliffs, **and** the gap persists when greedy is fairly armed (DCTCP/ECN, small RTO). **Dies if any of:**
- **(a)** the win evaporates once greedy gets real incast mitigations → we only beat a strawman;
- **(b)** the advantage required illegitimate info → caught by the black-box invariant;
- **(c)** the granted transaction merely **trades collapse for latency** — paces so conservatively that no buffer overflows but the transaction completes *no sooner* than greedy's collapse-and-recover.

> **(c) is the self-deception a solo researcher walks into.** Pacing trivially eliminates drops — "zero overflow!" feels like victory. It isn't. The win must be **net**, on goodput AND tail completion, not on the drop counter. "I prevented the overflow" and "I produced a better outcome" are different claims, and the gap between them is where you'd fool yourself with no reviewer in the room.

---

## 6. Pre-registration with teeth (governance OTS discipline)

Adopt the `governance` repo's proven, tamper-evident pre-registration verbatim. OpenTimestamps anchors a commit hash in the Bitcoin blockchain — un-backdatable — so the ledger proves predictions existed *before* the data. Honesty becomes a cryptographic fact, not a character claim. This is the substitute for peer review a solo researcher needs.

**Wiring:**
1. Copy `install-hooks.sh`, `hooks/post-commit`, `ots-upgrade.sh` (portable — they use git toplevel); add `opentimestamps-client` to the uv env; run `install-hooks.sh` once.
2. **Commit convention:**
   - `incast-teeth: pre-registration` — predicted goodput-vs-N curves + the (a)/(b)/(c) kill thresholds, committed **before a single run**. Auto-stamped.
   - run.
   - `incast-teeth: RESULT (scorecard)` — results + honest tally. Auto-stamped. `ots-upgrade.sh` hours later to anchor.
3. **Scorecard format:** `HIT / directional HIT / informative MISS / FAILED control`.
4. **Positive control with a predicted null** (straight from governance practice): pre-register a no-incast workload (single responder, or load well under link rate) where the transaction transport should show **no advantage**. If it "wins" there, the instrument is rigged in its favor (the `plant backwards` failure mode). Without this control, a solo "win" is unfalsifiable.

> The design spec defines the *method*. The numeric *predictions* get their own OTS-stamped `pre-registration` commit immediately before experiments run — distinct from this document.

---

## 7. Build waves

- **Wave 1 (the bare result):** `BottleneckLink` (drop-tail) → calibrate the collapse → `GreedyStream` vs `DeclaredTransaction` → the headline goodput-vs-N curves + the net-outcome check.
- **Wave 2 (arm the rivals):** DCTCP/ECN baseline, Homa-class baseline, the positive control, and the safety/multicast secondary question.

---

## 8. Out of scope (named, not forgotten)

- Sender-reciprocal (ayni) coordination → its own later leg (the bridge to deep-space).
- Leaf-spine / ns-3 fidelity → the deferred escalation path (Approaches 2/3).
- Inter-DC and the axis synthesis (Leg C), the recognizable stack (Leg D) → downstream of B.

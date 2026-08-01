# Network-Attached CDP Cannot Produce Plausible Pointer Input

**A measurement report and a proposed API for browser-infrastructure providers.**

## Summary

Remote browser providers expose a browser over CDP across a network. Clients drive the
pointer by sending `Input.dispatchMouseEvent` from their own process. **This
architecture cannot produce a pointer event stream with human-plausible timing, and no
amount of client-side modelling changes that**, because the timing a page observes is
determined by packet arrival rather than by anything the client computes.

The failure is not that the network is slow. It is that:

1. **CDP-injected pointer events are not frame-coalesced**, so every transport artifact
   reaches the page's listener directly. Real hardware input is coalesced to at most one
   event per compositor frame before any listener sees it.
2. **Arrival jitter is extremely heavy-tailed** — median 0.7ms, p99 63ms, max 1,280ms in
   our measurements — so a small fraction of events arrive wildly out of schedule.
3. **Every stall is followed by a burst.** The delayed events are released together,
   producing sub-millisecond inter-event gaps. In our sample this happened on 4 of 4
   stalls.

The fix has to run on the same machine as the browser. This document proposes a two-layer
API for that, plus a conformance test so implementations can be verified rather than
trusted.

All numbers below are our own measurements. Method and reproduction are in §7.

---

## 1. What a real pointer stream looks like

**Real pointer input has a hard floor on how close two events can be.** Across 1,539
measured gaps — including on a page whose main thread was blocked 70% of the time — the
shortest was **8.60ms**, and not one fell below half a frame period. Injected input over a
network breaks that floor up to 32% of the time.

(The test is half a frame, not one. Comparing against the full nominal period counts
ordinary one-frame gaps as violations, because the measured period carries a few tenths of
a millisecond of slop. §1.1 has the detail.)

Here is an unedited excerpt from a recorded human mouse movement — a `mousemove` listener
on local hardware, 60Hz display, recording `performance.now()` and the client coordinates.
The right-hand column is the elapsed time divided by the frame period:

```
  t=   0.0ms  x=1196  y=855   frame  0.00
  t=  16.7ms  x=1126  y=802   frame  1.00
  t=  33.4ms  x=1082  y=773   frame  2.00
  t=  50.1ms  x=1042  y=750   frame  3.01
  t=  66.8ms  x=1023  y=740   frame  4.01
  t=  83.4ms  x=1010  y=731   frame  5.00
  t= 100.0ms  x=1008  y=730   frame  6.00
  t= 116.9ms  x=1008  y=730   frame  7.01
  t= 133.5ms  x=1006  y=726   frame  8.01
  t= 150.1ms  x= 999  y=716   frame  9.01
  t= 166.9ms  x= 986  y=699   frame 10.01
  t= 183.3ms  x= 972  y=681   frame 11.00
  t= 200.2ms  x= 944  y=651   frame 12.01
  t= 216.8ms  x= 918  y=624   frame 13.01
```

The frame index increments by exactly one, every time, for all 56 events in this burst.
Note frames 6 and 7: the pointer did not move at all, and an event still fired — on its
own frame. The hand's speed varies wildly across the excerpt (70px in the first frame,
2px at frame 8) and the cadence does not vary at all.

**This is a property of the pipeline, not of hands.** The OS samples the device at 125Hz
or more; the browser coalesces those samples down to at most one `mousemove` per
compositor frame before any listener runs. The observed cadence is the *display clock*.
On a 120Hz display the period is 8.33ms, so a detector recovers the base period per
session from the modal gap rather than assuming 16.67ms.

Across the full baseline — three sessions, 782 within-burst gaps — mean deviation from the
nearest frame multiple is **0.029 frames**, and 92% of gaps land within 0.1 frames of a
multiple. Uniformly distributed gaps would give 0.25 and 20%.

Three consequences for any synthetic implementation:

- **There is a floor.** On an idle page p10 equals p50 (16.6 vs 16.7); across every load
  condition the shortest gap observed was 8.60ms, or 0.51 frames, with **0 of 1,539** below
  half a frame. An earlier capture through instrumented harness code showed 1.4% below that
  line, which we now attribute to the capture path rather than to the input.
- **Gaps longer than one frame happen (17.6%) and mostly are not dropped frames.**
  Displacement does not scale with the gap — three-frame gaps carry *half* the
  displacement of one-frame gaps (4.2px vs 8.1px) — because the gap comes from the
  *device* reporting nothing that frame. A mouse sensor emits discrete counts; a frame
  with no counts produces no event. (A count smaller than one CSS pixel still produces an
  event, which is why the excerpt above has two at identical coordinates.) An
  implementation that manufactures multi-frame gaps by *dropping* samples gets this
  backwards: random loss makes gap and displacement positively correlated, where real
  input is flat-to-negative.
- **The gap series is mildly positively autocorrelated** (lag-1 **+0.094**), because a
  hand's speed varies smoothly while its cadence does not.

### 1.1 What main-thread load does to it — and what it cannot do

Lattice adherence is not robust to page load, and any check built on it alone will fail
real users. Real mouse input, one machine, three load levels, 15 seconds each:

Throughout this document **"sub-frame" means shorter than *half* a frame period**, not
shorter than a frame. The nominal period carries a few tenths of a millisecond of slop, so
a strict "under one frame" test flags 23% of ordinary idle gaps against 0.8% that are
genuinely short. Half a frame sits clear of that noise.

| main thread | gaps | one-frame adherence | all gaps | **sub-frame** | **long-then-short** | gap p50 |
|---|---|---|---|---|---|---|
| idle | 485 | 95.1% | 94.4% | **0.0%** | **0.0%** | 16.9ms |
| blocked 30ms/100ms | 629 | 73.6% | 72.3% | **0.0%** | **0.0%** | 16.9ms |
| blocked 70ms/100ms | 425 | **46.9%** | **30.8%** | **0.0%** | **0.0%** | 16.7ms |

A hand on a heavily loaded page scores 30.8%, which is inside the range synthetic input
produces. Adherence therefore separates cleanly only on an idle page.

**What load actually does — it is not randomisation.** Under 70ms blocks the gaps stay
tightly clustered; they simply cluster somewhere other than the frame grid:

```
34.1%  at 12.5ms   (0.75 frames)
30.1%  at 16.7ms   (1 frame)
29.4%  at 71.0ms   (4.25 frames)
 5.2%  at 87.7ms   (5.25 frames)
```

145 gaps on 12.5ms and 125 on 71.0ms is structure, not noise. And 71.0ms is the *block
duration*, not four frames (66.8ms) — because the main thread frees when the block ends
rather than at the next vsync, and queued input then drains at ~12.5ms, faster than the
display. **A saturated main thread displaces vsync as the dispatch clock.** That is why
adherence measured against a fixed 16.67ms collapses: the metric assumes a clock the page
is no longer running on.

**Two properties survive it anyway, and they are the ones worth checking:**

- **The floor holds.** Not one gap in 1,539 fell below half a frame period, at any load
  level; the shortest was 8.60ms. A busy thread removes events and shifts the clock — it
  cannot deliver two events in quick succession, because there is nothing to deliver until
  the device reports again.
- **Compensating pairs stay at exactly zero.** A busy thread delays and merges. It never
  holds a backlog and releases it, which is precisely what a network does.

## 2. What CDP-over-network produces

Same instrumentation, same 900px straight-line intent, hosted browser, sends paced one
frame apart. This excerpt is real, taken from a run around one of the stalls:

```
  t=   16.9ms  x=423   frame  1.01    (+16.9ms)
  t=   33.2ms  x=426   frame  1.99    (+16.3ms)
  t=  150.5ms  x=435   frame  9.03    (+117.3ms)
  t=  150.6ms  x=444   frame  9.04    (+0.1ms)
  t= 1005.7ms  x=456   frame 60.34    (+855.1ms)
  t= 1005.8ms  x=462   frame 60.35    (+0.1ms)
  t= 1006.0ms  x=468   frame 60.36    (+0.2ms)
  t= 1006.6ms  x=474   frame 60.40    (+0.6ms)
  t= 1007.0ms  x=480   frame 60.42    (+0.4ms)
  t= 1007.5ms  x=486   frame 60.45    (+0.5ms)
```

**Five events inside frame 60.** The frame indices are fractional and clustered rather
than incrementing by one. A 117ms stall is paid back with a 0.1ms gap; an 855ms stall is
paid back with five events in under two milliseconds. Nothing in the client's intent
produced this — the sends were evenly paced one frame apart. The transport produced all
of it.

Compare the two blocks directly: the human stream has one event per frame index and the
network stream has none of that structure. That difference is the entire subject of this
document.

### 2.1 Injected pointer events are not frame-coalesced

The five-events-in-one-frame behaviour above is only possible because injected input skips
the coalescing stage. 300 moves dispatched 1ms apart over 357ms — roughly 21 frames —
produced **218 listener invocations** on a local browser. Frame coalescing would have
produced about 21.

Two independent confirmations that events are delivered rather than merged: the probe
walks a 900px line in 300 samples (3.01px stride), and received displacement was **3px at
the median**, i.e. one sample per delivered event; and on a live production page carrying
a major commercial bot-management sensor, an instrumented trace of that sensor's own
handler recorded **380 events delivered, 380 handler invocations, sampling ratio 1.0**,
with inter-event gaps of 0.1–1.5ms.

So the detector's listener sees sub-frame events that no coalesced hardware pipeline can
produce. **Every transport artifact is visible to the page.**

### 2.2 Ordering survives, timing does not

Zero order inversions in **1,153 consecutive event pairs** across six configurations
(local and hosted; 0ms, 1ms and 16ms send spacing), including configurations where 99% of
gaps were sub-2ms. CDP commands on one session traverse a single WebSocket and are
processed in send order.

This is worth stating because it bounds the problem: the channel damages **timing and
completeness**, never sequence. Path geometry arrives intact. Only the clock is corrupted.

### 2.3 The jitter distribution is heavy-tailed, not Gaussian

1,010 gaps across four runs against a hosted provider at 16ms send spacing. "Excess
delay" is the gap minus the sends it actually spanned, so lost events are factored out
and what remains is pure lateness.

| p50 | p90 | p99 | p99.9 | max |
|---|---|---|---|---|
| 0.7ms | 2.7ms | 63ms | 111ms | **1,280ms** |

Mean 1.3ms, standard deviation **49.6ms** — 37× the mean. The p99.9/p50 ratio is **989**;
a Gaussian gives about 4.

| excess delay | share |
|---|---|
| under half a frame | 83.4% |
| half to one frame | 6.2% |
| 1–3 frames | 8.0% |
| 3–12 frames | 2.2% |
| over 200ms | 0.2% |

**"Average jitter" is not a usable summary of this distribution.** The median is
excellent. The tail decides everything.

### 2.4 Every stall is followed by a burst

This is the core finding. Stalls over 100ms occurred on 0.40% of gaps (4 of 1,010). The
gap **immediately following** each one:

```
0.1ms      0.5ms      0.1ms      0.2ms
```

Four out of four. A 1,344ms gap spanning 4 sends was followed by a 0.1ms gap. The link
holds events and then releases them together; the release is always sub-frame.

This is a *mechanism*, not a tendency, and it is what produces the compensating signature
a detector can find with two accumulators:

| | long-gap-then-short-gap rate | gap lag-1 autocorrelation |
|---|---|---|
| real hardware | 0.3% | **+0.094** |
| same client code, local browser | 0.0% | −0.049 |
| same client code, over network | **4.1%** | **−0.140** |

The autocorrelation **changes sign**. A hand's cadence is positively correlated because
speed varies smoothly. A network's is negatively correlated because a stall creates a
debt that gets paid back in the next interval. No input device owes time.

## 3. Why the client cannot fix this

### The network, and only the network, is what breaks the lattice

Identical client code, identical send pacing, one frame apart. Lattice adherence is the
share of gaps landing within 0.1 frames of an integer multiple:

| | adherence |
|---|---|
| real hardware | 92% |
| **same client code, local browser** | **92%** |
| **same client code, hosted browser** | **56%** |
| uniform noise (no lattice at all) | 20% |

Driving a browser on the same machine reproduces real hardware exactly. Move that browser
across a network and adherence collapses by more than half, with nothing else changed.
This is the central result: **the client is already capable of producing a correct pointer
stream, and the transport destroys it in flight.**

### Two things that look like fixes and are not

**Pacing on round milliseconds.** A 20ms interval is 1.2 frames, so every gap is
off-lattice *by construction*, before jitter is involved. Measured, 20ms sends score 42%
— close to the 20% noise floor. This one is genuinely avoidable and any implementation
must pace on frame multiples, but fixing it does not address the network.

**Widening the interval.** Tempting, since a longer gap seems like it should absorb a
fixed jitter budget. It does not, for two reasons. The lattice period is set by the
display and does not scale with the send interval, so ±8ms is ±0.5 frames whether the
target gap is 16.7ms or 400ms. And §2.4 is spacing-independent: the post-stall burst
arrives at 0.1–0.5ms no matter what the nominal interval was, because the backlog is
released as fast as the link can drain it.

*Stated as an argument, not a measurement.* Lattice adherence was measured at one-frame
and 20ms spacings only; there is no 33ms or 50ms figure here. The spacing-independence of
the post-stall burst is measured (§2.4); the fixed-lattice-period claim is arithmetic.

### The only client-side lever is emitting fewer events

Each gap is an independent chance to be caught by a stall. At the measured 0.4% stall
rate, the probability a session contains **zero** transport-generated sub-frame gaps is
0.996ⁿ:

| events in session | clean |
|---|---|
| 25 | 90% |
| 100 | 67% |
| 200 | 45% |
| 1,200 (20s at one frame) | 0.8% |

A pointer stream dense enough to look like continuous human motion is, over a network,
almost certain to contain intervals that a coalesced pipeline cannot produce. **Plausible
density and plausible timing are mutually exclusive from a remote client.**

### CDP offers no primitive that would help

There is no batch or scheduled input command. `Input.dispatchMouseEvent` dispatches
immediately on receipt. Only `Input.synthesizeScrollGesture` and `synthesizeTapGesture`
perform browser-side timing, and neither covers pointer trajectories.

Spoofing the `timestamp` parameter does not work either and is worse than the problem:
it desynchronises the event's claimed time from when handlers actually run. Measured, a
1,070ms claimed span executed over 8.2ms of real handler time, giving per-event skew down
to −1,056ms — the handler running before the event claims to have occurred. Honest skew
is +0.1 to +6.5ms. A single event that contradicts itself is a stronger signal than
irregular cadence.

## 4. What consumes this, and what therefore matters

An instrumented trace of a major commercial bot-management sensor on a live production
page, to bound which properties are worth specifying:

- Handler runs on **every** pointer event; no `requestAnimationFrame` throttling.
- Per event it reads `pageX`, `pageY`, `isTrusted`, and calls `Date.now()` about 1.6
  times. It does **not** read `event.timeStamp` — it reads the wall clock itself.
- It buffers roughly **200 events** and then stops recording (coordinate reads cease
  while the handler keeps running).
- The transmitted payload is **invariant in size**: 4,396 / 4,406 / 4,398 bytes for 50 /
  200 / 1,000 events. A 20× change in event count moves it ±10 bytes.
- Under one arithmetic operation per event in the handler, and it is `floor` —
  quantisation. No typed arrays, no transcendental functions. A featurizer, not a model.

Two things follow. **Fine path geometry is probably not scoreable** — Bezier-versus-sampled
shape, overshoot and micro-pause placement cannot survive reduction to a fixed-width
digest. And **timing moments certainly are**: every statistic in §2.4 is O(1) streaming
state, a handful of bytes, and computable from exactly the `(x, y, wall-clock)` triples
this sensor collects.

That asymmetry is the argument for putting timing under provider control and leaving
trajectory shape under client control.

---

## 5. Proposed API

### 5.1 Design constraints

- **Dispatch must happen on the browser host**, driven by the browser's own clock. This is
  the entire point; everything else is detail.
- **Clients must keep control of behaviour.** A provider that applies one built-in motion
  model to every client makes that model a shared fingerprint. We disabled one provider's
  humanisation feature after measuring it inject ~720 pointer events we did not request,
  which made our own behaviour unmeasurable and identical to every other customer's.
  [Appendix: measuring a provider's built-in humanisation](appendix-provider-humanization.md)
  measures one such implementation in detail — it already synthesises provider-side and
  varies between runs, and misses conformance mainly on the frame lock.
- **Two layers**, because they have different adoption costs. The low layer is a small,
  mechanical change that fixes the timing problem. The high layer is more work and more
  valuable.
- **Results must be observable.** Clients currently cannot tell what the page received
  without instrumenting the page.

### 5.2 Layer 1 — scheduled trajectory

The client supplies the path and the intended timing; the provider owns dispatch.

```
Input.dispatchTrajectory {
  points: [{ x, y, frames }],   // absolute viewport coords; `frames` = gap BEFORE this point
  onLag?: "skip" | "compress"   // default "skip"
}
-> { emitted, skipped, framePeriodMs, gapsFrames: [...], durationMs }

Input.pressPointer {
  button?:     "left" | "right" | "middle",   // default "left"
  holdFrames?: number   // omitted: provider draws a human-plausible hold
}
-> { holdMs }
```

Gaps are expressed in **frames**, not milliseconds, because the target lattice is the
display clock and the client does not know the host's refresh rate. The provider resolves
frames against its actual compositor period.

Coordinates are absolute rather than deltas: the provider already knows the current pointer
position, and absolute coordinates avoid accumulating rounding error along a long path.

`frames` is the gap *before* each point, and values above 1 carry an obligation the client
must meet, because this layer puts the client in charge of the joint distribution. A gap
longer than one frame means the device reported nothing during those frames, which happens
when the pointer is barely moving — so `frames: 3` must be paired with a *small* step, not
a large one. Real three-frame gaps carry 4.2px against 8.1px for one frame (§1). Fast
motion reports every frame; a long gap followed by a large jump is a combination real input
never produces. A provider MAY reject or flag trajectories that invert this relationship.

The press is a **separate call**, not an index into the trajectory, because the two have
very different timing requirements. A press must follow a hover, and real hover-before-press
is a median 468ms with a wide spread — a network round-trip inside that window is invisible.
What cannot cross the network is the down-to-up hold, measured at a 106ms median: sending
`mousePressed` and `mouseReleased` as two CDP commands puts a full RTT inside the press and
produces holds no finger makes. `holdFrames` keeps that one interval on the host.

This layer is a small change: accept an array, run a frame-aligned dispatch loop, report
what happened. It takes the network out of the timing path without the provider having to
model behaviour at all.

### 5.3 Layer 2 — intent-level interaction

The client says what it wants to do; the provider synthesises the whole interaction.

```
Interaction.click {
  target:  { selector | backendNodeId | { x, y } },
  speed?:  "slow" | "normal" | "fast",            // default "normal"
  noise?:  "none" | "low" | "normal" | "high",    // default "normal"
  profile?: <inline distributions>,               // overrides speed/noise
  seed?:   <integer>
}

Interaction.type   { target, text, speed?, noise?, profile?, seed? }
Interaction.scroll { target | delta,  speed?, noise?, profile?, seed? }
Interaction.drag   { from, to,        speed?, noise?, profile?, seed? }

-> { emitted, skipped, gapsFrames, targetResolvedAt, durationMs }
```

**`speed`** scales displacement per frame — how fast the pointer covers ground. For
calibration, real motion measured a median of roughly 366px/s, with slow phases around
85px/s. Speed is not independent of the frame-count distribution: slower motion produces
*more* multi-frame gaps, because a slow-moving device reports no counts in some frames and
those frames produce no event (§1). An implementation that changes step size without
changing the gap distribution will get the joint structure wrong.

**`noise`** controls departure from a direct path — turn-angle magnitude, overshoot and
reversal rate, and micro-motion while hovering. At `normal` this should land near the
measured baseline: turn angle median ~10°, reversals (>135°) ~5%, and ~13% of samples with
zero displacement. **`none` produces a straight line and exists for testing only** — a
perfectly direct path is itself a strong signal and should not be a production default.

Both are deliberately coarse enums so the common case is one word. `profile` is the escape
hatch for clients that have fitted their own distributions and want to supply them rather
than accept a preset; see the fingerprint argument in §5.1.

Target resolution happens **at dispatch time on the host**, which also fixes a second
class of bug: a client computing coordinates from a stale layout clicks the wrong place
when the page reflows mid-approach.

`seed` matters. Without it, either every session gets identical motion (a fingerprint) or
runs are unreproducible (undebuggable). With it, motion is per-session distinct and
replayable.

`profile` should accept client-supplied distributions, not only named presets, so that a
client with its own fitted model can use it. A reasonable minimum: the frame-count
distribution, a displacement distribution conditioned on frame count, a turn-angle
distribution conditioned on displacement, hover-before-press and press-hold durations.

### 5.4 Worked examples

The same task three ways: move to a sign-in button at (640, 512) from (412, 588) and
click it. Messages are shown in CDP wire form.

**Today.** Every point is its own message, and every message's arrival time becomes the
page's observed cadence:

```jsonc
--> {"id":1,"sessionId":"A1","method":"Input.dispatchMouseEvent",
     "params":{"type":"mouseMoved","x":448,"y":573,"button":"none"}}
--> {"id":2,...,"params":{"type":"mouseMoved","x":495,"y":556,"button":"none"}}
// ... 40 more, each a separate round trip
--> {"id":43,...,"params":{"type":"mousePressed","x":640,"y":512,"button":"left","clickCount":1}}
--> {"id":44,...,"params":{"type":"mouseReleased","x":640,"y":512,"button":"left","clickCount":1}}
```

The client controls the *order* of these and nothing else. The 43rd and 44th messages are
the press, and the hold duration is whatever the network puts between them — typically
under a millisecond, because they are sent back to back.

**Layer 1.** One message carries the whole path; the client still owns the trajectory:

```jsonc
--> {"id":1,"sessionId":"A1","method":"Input.dispatchTrajectory","params":{
      "points":[
        {"x":448,"y":573,"frames":1},
        {"x":495,"y":556,"frames":1},
        {"x":541,"y":541,"frames":1},
        {"x":578,"y":530,"frames":1},
        {"x":605,"y":523,"frames":1},
        {"x":622,"y":518,"frames":1},
        {"x":632,"y":515,"frames":1},
        {"x":637,"y":513,"frames":2},   // slowing: small step, two frames
        {"x":639,"y":512,"frames":1},
        {"x":640,"y":512,"frames":2}
      ],
      "onLag":"skip"}}

<-- {"id":1,"result":{
      "emitted":10,"skipped":0,"framePeriodMs":16.667,
      "gapsFrames":[1,1,1,1,1,1,1,2,1,2],"durationMs":200}}
```

Abridged — a real approach is 20–60 points. Note the response echoes what was actually
emitted, so the client can verify the contract held rather than assume it.

When the host falls behind, rule 4 shows up in the response as dropped points rather than
a flushed backlog:

```jsonc
<-- {"id":1,"result":{
      "emitted":8,"skipped":2,"framePeriodMs":16.667,
      "gapsFrames":[1,1,2,1,1,3,1,1],"durationMs":200}}
```

Two points were skipped and the gaps absorbed them (a 2 and a 3 where 1s were requested).
Total duration is unchanged. **This is the correct behaviour** — it is what the browser
does to real input under load, and it keeps every gap on the lattice.

Then the press, as its own call so the hold stays on the host:

```jsonc
--> {"id":2,"sessionId":"A1","method":"Input.pressPointer",
     "params":{"button":"left","holdFrames":6}}
<-- {"id":2,"result":{"holdMs":100}}
```

**Layer 2.** The client states the intent and supplies nothing else:

```jsonc
--> {"id":1,"sessionId":"A1","method":"Interaction.click","params":{
      "target":{"selector":"#sign-in"},
      "speed":"normal","noise":"normal","seed":91424}}

<-- {"id":1,"result":{
      "emitted":47,"skipped":1,
      "gapsFrames":[1,1,1,2,1,1,1,1,3,1,...],
      "targetResolvedAt":{"x":640,"y":512},
      "durationMs":812}}
```

The client never computes a coordinate. `targetResolvedAt` reports where the element
actually was when the pointer arrived, which is how the caller learns the page reflowed
mid-approach instead of silently clicking the wrong thing.

Typing is the same shape, and note that the per-keystroke timing is what moves on-host:

```jsonc
--> {"id":2,"sessionId":"A1","method":"Interaction.type","params":{
      "target":{"selector":"#username"},
      "text":"a.member@example.com",
      "speed":"normal","seed":91425}}
<-- {"id":2,"result":{"emitted":20,"durationMs":2840}}
```

### 5.5 Timing contract (normative)

An implementation of either layer:

1. **MUST** dispatch from the browser host, timed by the host clock.
2. **MUST NOT** emit two pointer events for the same pointer within one compositor frame.
   This is the single most important requirement.
3. **MUST** align inter-event gaps to integer multiples of the compositor frame period.
4. **MUST**, when the dispatch loop falls behind schedule, *advance the trajectory and
   drop the intermediate samples* — never emit the backlog. This is `onLag: "skip"`, and
   it is the default because it is what the browser itself does to real input under load.
   Flushing a backlog reproduces the exact network pathology this API exists to remove.
5. **MUST** produce events with `isTrusted === true` (i.e. real input injection, not
   `dispatchEvent`).
6. **SHOULD** reproduce a realistic frame-count distribution rather than emitting every
   frame — real input is roughly 81% one-frame, 7% two-frame, 3% three-frame — and
   **SHOULD** correlate displacement with frame count in the observed direction: gaps
   longer than one frame carry *equal or less* displacement, because they arise from slow
   motion rather than dropped samples.
7. **SHOULD** hold pointer-down for a nonzero, varying duration. Real presses measured
   106ms at the median. `Input.dispatchMouseEvent` pairs sent back-to-back produce
   zero-duration presses, which no finger makes.
8. **MUST NOT** apply motion the client did not request. If a provider adds interactions
   of its own, that must be opt-in and documented.

### 5.6 Feedback

Both layers return the **actual emitted gaps in frames** and the host's frame period.
Without this, clients cannot distinguish "the provider dispatched correctly" from "the
provider dispatched correctly and the page was busy," and cannot verify conformance at
all. This is cheap to implement and it is what makes the rest auditable.

---

## 6. Conformance test

A provider can self-certify with the following procedure. It requires no proprietary
detector — it compares against real-hardware behaviour.

**Procedure.** On a blank document with an idle main thread, install a `mousemove`
listener recording `performance.now()`, `clientX`, `clientY`. Request a 900px straight
trajectory of 300 samples at one frame per sample. Repeat four times. Compute inter-event
gaps and displacements.

**Thresholds** fall into two groups, and the distinction matters more than any individual
number.

**Group A — load-invariant. These are the real checks.** Measured on real hardware they
hold at the same value whether the page is idle or its main thread is blocked 70% of the
time (§1.1), so an implementation has no excuse and no defence.

| metric | real hardware | pass |
|---|---|---|
| **gaps below half a frame period** | **0.0%** (0 of 1,539; floor 8.60ms) | **≤ 2%** |
| **long-gap-then-short-gap pairs** | **0.0%** | **≤ 1%** |
| **modal gap** | one frame period, at every load level | within 5% of the frame period |
| order inversions | 0 | 0 |
| gap lag-1 autocorrelation | +0.094 | ≥ −0.05 |
| displacement correlation with frame count | flat/negative | not positive |

**Group B — informative on an idle page only. Do not use these as a gate.**

| metric | idle | 30ms blocks | 70ms blocks |
|---|---|---|---|
| one-frame gaps within 0.1 frames | 95.1% | 73.6% | 46.9% |
| all gaps within 0.1 frames | 94.4% | 72.3% | 30.8% |

Lattice adherence is the most intuitive measure and the least robust one. A real hand on a
heavily loaded page reaches 30.8%, which is inside the range synthetic input produces — so
a conformance gate built on adherence would fail real users and could be passed by an
implementation that merely tested on a quiet page. Measure it, report it, and treat a low
score as a prompt to check Group A rather than as a failure in itself.

The half-frame floor is where the whole test lives. The check is not "one event per frame": that holds on an idle page (0.8% of gaps fall
meaningfully short) but breaks under load, where 34.1% land at 0.75 frames as the renderer
drains its queue. The load-invariant statement is that nothing arrives in under half a
period, which held at every level we tested. Synthetic input
breaks it routinely: 23–25% for one provider's built-in humanisation, and 3–32% run to run
for client-side pacing over a network.

The autocorrelation threshold catches a subtle and common implementation bug. Pacing with
`setTimeout(fn, 16)` against a 16.67ms frame period alternates 16/17ms and aliases against
the compositor; we measured **−0.414** doing exactly that on a local browser — worse than
the network figure it was meant to avoid. A conforming implementation must lock to the
frame clock, not to a millisecond timer.

---

## 7. Method and limits

**Measurement.** All page-side figures come from a `mousemove` listener recording
`performance.now()` and client coordinates — identical instrumentation for human and
synthetic input, so the comparison is like-for-like. The probe dispatches a monotone
horizontal line, which makes displacement a direct measure of how many samples a gap spans
and makes any ordering inversion visible as a negative delta. Hosted measurements used one
commercial provider over an ordinary internet path, no tunnel or proxy.

**Limits worth stating.**

- The human baseline is **three sessions from one person, 782 gaps**, on one machine at
  60Hz. The frame-lattice result is a property of the browser pipeline and should
  generalise; the specific frame-count distribution (81/7/3/2) is one person's motion and
  should not be treated as a population norm.
- The 0.4% stall rate rests on **four events**. The 95% interval is roughly 0.1%–1.0%, so
  the session-cleanliness arithmetic in §3 is an order of magnitude, not a threshold.
- Run-to-run variance on the network path is large. The same configuration produced 6.1%
  and 32.2% sub-2ms gaps in two runs an hour apart. Any single measurement of the timing
  signature is one draw.
- Channel measurements were taken on a blank document. On a contended real page, delivery
  saturated around 120–130 events per burst — a different loss regime, uncharacterised.
- §4 establishes what a sensor **collects and transmits**. Nothing observable from the
  client establishes what a server **scores**, and we make no claim about that.
- We have not tested whether the burst-after-stall pattern of §2.4 also appears on a
  contended production page, where the renderer contributes its own delay.

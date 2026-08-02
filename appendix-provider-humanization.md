# Appendix: measuring a provider's built-in humanisation

Several remote-browser providers offer an "humanize interactions" option that synthesises
a pointer approach from a single click instruction. This is the right architectural idea —
the provider is on the same machine as the browser, which is exactly where §3 of the
[specification](README.md) says the work has to happen.

This appendix measures one such implementation, **Steel's `stealthConfig.humanizeInteractions`**,
against the real-hardware baseline in §1. It is published because the specification is more
useful with a worked example of how close an existing implementation already is, and where
it falls short. The gap turns out to be small and specific.

Measured August 2026. Method and limits at the end. Nothing here is proprietary — it is
what any customer's page observes.

## What it does

A single `page.mouse.click(x, y)` at a target 537px away, with the pointer parked at a
known origin first. Two repetitions, plus a control with the option off.

| | humanize off | humanize on | real hardware |
|---|---|---|---|
| pointer events per click | 1 | **13–14** | — |
| wall time for the click | 67ms | 229ms | — |
| same path twice? | — | no | — |

The control matters: with the option off the page receives exactly what was dispatched —
one move. So all 13–14 events are the provider's, not ours.

**Three things it gets right**, and they are the hard parts:

- **It synthesises provider-side.** The approach exists at all, which no client-side
  implementation over a network can do correctly.
- **It varies between runs.** Across two identical clicks only the first and last positions
  repeated. A deterministic built-in path would be a fingerprint shared across every
  customer of the provider, which is worse than no humanisation at all.
- **Events are `isTrusted`.** It is real input injection, not `dispatchEvent`.

It also gets the event *budget* about right. 229ms is 13.7 frames at 60Hz and it emitted
13–14 events — approximately one per frame, which is what real input produces.

## What it gets wrong

### It emits more events than the display can deliver

This is the substantive defect, and it is categorical rather than a matter of degree.

*n* consecutive events require *n* distinct frames, so their span must contain at least
*n−1* frame boundaries. Measured against the frame timeline **on the provider's own host**
(16.70ms, 59.9Hz — the same as the reference hardware), across three runs:

| | events | frame boundaries in span | needs | surplus |
|---|---|---|---|---|
| rep 1 | 14 | 7 | 13 | **+6** |
| rep 2 | 12 | 7 | 11 | **+4** |
| rep 3 | 14 | 9 | 13 | **+4** |

**Fourteen events inside seven frames** — twice what the pipeline can carry. Real input
never exceeds one event per frame: verified over 1,792 events on real hardware, idle and
with the main thread blocked 70% of the time, zero violations at any window size.

This does not depend on a nominal refresh rate, on where inside a frame an event fired, or
on any distributional threshold. A page that receives two pointer events in one frame is
receiving something no pointing device produced.

### It is not locked to the compositor clock

The same defect seen through gap statistics rather than counts.

| | humanize on | real hardware, idle | real hardware, heavy load | no lattice at all |
|---|---|---|---|---|
| gap p50 | 12.5 / 14.1ms | 16.9ms | 16.7ms | — |
| **gaps below half a frame** | **23% / 25%** | **0.0%** | **0.0%** | — |
| gaps within 0.1 frames of a multiple | 30% / 22% | 94% | 31% | 20% |

Lattice adherence of 22–30% is statistically indistinguishable from having no frame
structure whatsoever — though on its own that is a weak charge, because a real hand on a
heavily loaded page also drops to 30.8% (§1.1 of the specification). **The half-frame floor is the one that does
not overlap:** across 1,539 real gaps at every load level tested, none fell below half a
frame period — the shortest was 8.60ms. This implementation goes below it a quarter of the
time.

A raw excerpt shows the shape:

```
  t=1456.8  x=160  y=180
  t=1467.9  x=207  y=205     (+11.1ms)
  t=1478.0  x=286  y=243     (+10.1ms)
  t=1499.7  x=346  y=267     (+21.7ms)
  t=1515.5  x=371  y=276     (+15.8ms)
  t=1515.9  x=393  y=283     (+0.4ms)   <-- two events inside one frame
  t=1532.1  x=432  y=294     (+16.2ms)
  t=1544.6  x=467  y=306     (+12.5ms)
```

Real input produces gaps that are integer multiples of the frame period, with consecutive
one-frame gaps stable to about 0.17ms. These are 11.1, 10.1, 21.7, 15.8, 0.4 — arbitrary
values, including one pair inside a single frame.

**Note where that jitter lives.** These timestamps come from the page's own
`performance.now()` inside a `mousemove` listener, so they record when the *browser*
dispatched each event. The irregularity is therefore inside the provider's own
generation-to-dispatch path, not in the customer's connection to the provider. Whatever
generates the trajectory is not running in the browser's frame loop.

### The press is untouched

| | humanize off | humanize on | real hardware |
|---|---|---|---|
| pointer-down to pointer-up | 1.5 / 0.7ms | **0.6 / 0.3ms** | **106ms** |

Turning humanisation on does not change the press at all. `mousePressed` and `mouseReleased`
still go out back to back, producing a hold no finger can make. This is the cheapest thing
in the whole specification to implement — normative rule 7 — and it is simply absent.

### It moves about six times too fast

537px in 229ms is roughly 2,340px/s. Real motion measured a median of 366px/s. The
consequence shows up in step size: 40–42px per event against 8.1px for a real one-frame
step. The path is plausible in shape and implausible in pace.

## Why this is worth fixing even though sessions currently pass

The obvious objection is that this configuration works. In our own testing the same provider
authenticated against a hardened commercial portal repeatedly — including with humanisation
switched **off**, i.e. with a single teleport move per click and a sub-millisecond press,
which is the worst behavioural profile available.

That observation does not establish what people take it to establish. **A bot score is an
aggregate, and you cannot read the weight of one input off a passing total.** A poor pointer
sub-score carried by a strong fingerprint, TLS profile and IP reputation looks exactly like
a pointer sub-score that carries no weight. From outside, a thin margin and no margin are
indistinguishable.

What follows is only that the total currently sits under a threshold. That is a statement
about today's margin, not about the coefficient — and margins move when a detector retunes,
when a vendor's fingerprint patches age, or when IP reputation shifts. Distinguishing the two
would require varying the pointer channel while holding everything else fixed and observing
the score, which no customer can do: the only signal available is a single bit, delivered with
minutes of latency, rate-limited by real credentials, and confounded by provider identity.

So the honest position is that the pointer channel is **unmeasured**, not unimportant.

## What would change it

The implementation is one property short of conforming, not a rewrite:

1. **Lock the dispatch loop to the compositor clock** (§5.5 rule 3). The event budget is
   already right; the timing is not. The measure that matters is the sub-frame rate — real
   input never puts two events in one frame at any load level, and this puts a quarter of
   them there. Frame-quantised dispatch takes that to zero by construction.
2. **Hold the press** (rule 7). A drawn duration around a 106ms median.
3. **Slow the default** to something near 366px/s, which follows from (1) anyway once gaps
   are frame-quantised.

Rules 4 and 6 — drop rather than flush when behind, and keep displacement from scaling with
gap length — matter once (1) is in place.

## Method and limits

Single click at a known target on a controlled blank page with an idle main thread, pointer
parked at a fixed origin beforehand, all pointer events captured by a page listener recording
`performance.now()` and client coordinates. Two repetitions with the option on, two with it
off as a control. Direct internet path, no proxy or tunnel.

- **Two repetitions per condition.** Enough to establish the defects, which are large and
  consistent, and not enough to characterise their distributions. The determinism finding in
  particular deserves more runs before anyone relies on it.
- **One provider, one option, one version, one target geometry.** A 537px traverse on a 60Hz
  host. Behaviour may differ for short approaches, other refresh rates, or other providers.
- Measurements are of observable page-side behaviour only. Nothing here reflects any
  provider's internal design beyond what the timing implies.
- The real-hardware baseline is three sessions from one person on one machine (§7 of the
  specification). The frame-lattice property is a browser-pipeline property and should
  generalise; the specific speed and step figures are one person's motion.

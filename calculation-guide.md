# Calculation guide

Everything the visual computes, and every rule it applies where the maths runs
out. Written to be readable by a workforce analyst, not just by a developer.

## Inputs

| Symbol | Meaning | Source |
|---|---|---|
| `V` | Forecast contacts offered in the interval | Field well |
| `AHT` | Average handle time, seconds | Field well |
| `I` | Interval length, seconds | Format pane |
| `SL_target` | Target service level, decimal | Format pane |
| `T` | Answer-time threshold, seconds | Format pane |
| `S` | Shrinkage, decimal | Format pane |
| `O_max` | Maximum occupancy, decimal | Format pane |

## Offered load

```
A = V * AHT / I
```

`A` is traffic intensity in Erlangs — the average number of agents busy at any
instant if there were no queueing.

## Erlang B, and why the recurrence matters

Erlang C is derived from Erlang B. The closed form of Erlang B divides `A^N` by
`N!`, and both overflow a double inside ordinary contact-centre range: `171!`
is `Infinity`, so the ratio becomes `NaN` from 171 agents up — roughly 1,020
contacts in a half-hour at a 300-second AHT.

This engine uses the recurrence instead:

```
B(0) = 1
B(n) = A * B(n-1) / (n + A * B(n-1))
```

Every intermediate value stays inside `(0, 1]`, so there is no overflow at any
load. At extremely low load and very high agent counts the value underflows to
exactly `0`, which is the correct limit and stays benign — service level then
evaluates to 1, not `NaN`.

## Probability of waiting

For `N > A`:

```
rho = A / N
C = B(N) / (1 - rho + rho * B(N))
```

For `N <= A` the queue has no steady state. The engine returns `C = 1` and
treats the interval as **overloaded** rather than applying a formula whose
assumptions do not hold.

## Service level

```
SL = 1 - C * exp(-(N - A) * (T / AHT))
```

An overloaded interval reports `SL = 0`. An idle interval — no contacts —
reports service level as **not applicable**, not 100%. See below.

## Occupancy

```
Occupancy = A / N
```

Capped at 100%. A ratio above 1 means the interval is overloaded rather than
that agents are more than fully utilised, and the `stability` field carries
that distinction, so nothing is lost by the cap.

## Required productive agents

The smallest integer `N` satisfying **all three** conditions:

```
N > A                     the queue has a steady state
A / N <= O_max            the staffing level is physically achievable
SL(N) >= SL_target        the service target is met
```

The search starts at:

```
N = max(floor(A) + 1, ceil(A / O_max))
```

which satisfies the first two by construction. Occupancy falls monotonically as
`N` grows, so every later candidate satisfies them too — incrementing can only
tighten service level, and the two constraints can never be traded off against
one another.

**This is the part most Erlang C implementations get wrong.** A search that
stops at the first `N` meeting service level alone will, at moderate to high
load, return a headcount whose occupancy is unachievable. At `A = 60` Erlangs
with an 80%/20s target, a service-level-only search returns 67 agents running
at 89.6% occupancy; enforcing an 85% cap returns 71. At 30% shrinkage that is
96 scheduled agents versus 102 — six per interval.

The search carries the Erlang B recurrence forward one term per increment
rather than rebuilding it, which keeps it linear in the answer. In practice it
converges in one to a few dozen iterations. A defensive bound of 10,000
iterations returns an `ITERATION_LIMIT` error rather than a best-effort number.

## Shrinkage

Applied **last**, never to the offered load, the AHT, or inside the Erlang C
formula:

```
RequiredScheduled = ceil(RequiredProductive / (1 - S))
```

## Estimated performance from scheduled staffing

When scheduled agents are supplied, the productive-agent assumption is:

```
AvailableProductive = floor(ScheduledAgents * (1 - S))
```

That integer feeds the Erlang C calculations. If it does not exceed offered
load, the interval is reported as overloaded with a failing service level —
never as `NaN`, infinity, or a misleadingly high percentage.

### A floating-point trap worth knowing about

These two conversions must round-trip: staffing that exactly meets requirement
has to yield the required productive agents back. Naively they do not, because
`1 - S` is not exactly representable in binary. At the default 30% shrinkage,
63 productive agents grosses up to 90 scheduled, and `90 * 0.7` evaluates to
`62.999999999999993` — so the interval would report one agent short and a
failing service level for staffing that is exactly right. It recurs 143 times
below 5,000 agents.

The engine applies a `1e-9` tolerance to both conversions, which fixes the
round trip without changing any correct answer. A property test guards it
across the full parameter grid.

## Staffing gap

```
Gap = ScheduledAgents - RequiredScheduled
```

Negative is understaffed, zero is exact, positive is surplus.

## Edge-case rules

These are choices, not derivations. They are listed here because the answer has
to be consistent and documented.

| Case | Rule | Why |
|---|---|---|
| `V = 0` | Required staff `0`; service level and occupancy **not applicable** (null) | Reporting 100% would count an overnight interval with no contacts as a service-level success, inflating the "target met" summary metric. A null keeps it out of both numerator and denominator. |
| `V > 0` and `AHT <= 0` | Input error on the AHT field | A handle time of zero against real volume is a data problem, not a staffing answer. |
| `V = 0` and `AHT` null | Accepted | There is nothing to handle, so there is no handle time to require. |
| Negative `V`, `AHT`, `I` or scheduled agents | Input error | |
| Null, `NaN` or infinite input | Typed error, surfaced as a data-quality message | Never coerced to a default. |
| `SL_target > 99.9%` | Setting rejected as out of range | Service level approaches 100% but never reaches it. A 100% target is only ever "met" once `C * exp(...)` underflows the double, which makes the answer an artefact of precision rather than a staffing result. |
| Scheduled agents supplied but zero, against real volume | Overloaded; service level 0 | |

## Aggregation, and the one field that needs care

Three of the four numeric fields are straightforward:

- **Forecast contacts** — sum.
- **Scheduled agents** — must represent staffing *at the interval grain*, not a
  sum of repeated row-level values.
- **Interval length** — a setting, not inferred. A user-selected value is
  better than silently calculating the wrong requirement.

**Average handle time is the exception, and it is the most likely source of a
wrong answer in the field.** A naive `AVERAGE(AHT)` averages averages across
whatever grain the visual is displaying, and the offered load comes out wrong
in a way that looks entirely plausible. Use a correctly weighted measure:

```
AHT = DIVIDE(SUM(HandleSeconds), SUM(Contacts))
```

The sample dataset stores `HandleSeconds` rather than an average precisely so
this measure is the natural thing to write.

## Tolerances

| Constant | Value | Purpose |
|---|---|---|
| `SERVICE_LEVEL_EPSILON` | `1e-12` | Absorbs the last bit of noise when comparing service level to target. The tightest real fixture misses by 0.44 of a point, so this can never change a meaningful answer. |
| `OCCUPANCY_EPSILON` | `1e-9` | Keeps an exact ratio such as `8.5 / 0.85` from rounding up to an extra agent. |
| `SHRINKAGE_EPSILON` | `1e-9` | Makes the two shrinkage conversions round-trip. |
| `MAX_SERVICE_LEVEL_TARGET` | `0.999` | The highest target with a finite answer. |
| `MAX_SEARCH_ITERATIONS` | `10,000` | Defensive bound; never reached in practice. |

## How these numbers are verified

The engine's Erlang B recurrence is checked against an **exact** implementation
of the same mathematics, written in BigInt rational arithmetic
(`test/fixtures/exactErlang.ts`). The two are deliberately opposite:

|  | Engine | Oracle |
|---|---|---|
| Formula | Erlang B recurrence | Erlang B closed form |
| Arithmetic | IEEE-754 doubles | Exact integer rationals |
| Overflow | Impossible by construction | Impossible — BigInt has no ceiling |
| Rounding | Every operation | None, except the final conversion |

They share no code path, no formula and no number representation, so agreement
between them is evidence rather than a tautology. At 500 Erlangs and 589
agents the oracle's numerator runs to about 1,590 decimal digits — a
calculation a double cannot express at all, and the reason the closed form had
to be abandoned in the engine in the first place.

Service level adds one transcendental term. Its exponent is exactly rational,
so the oracle computes it exactly and evaluates the exponential by fixed-point
Taylor series to 60 decimal places — about 44 digits beyond a double.

**Worst observed disagreement: 1.1e-15 relative**, a few units in the last
place. The test threshold is 1e-13, roughly two orders of magnitude clear of
the noise floor.

The oracle is also tested for discrimination — that it *rejects* the factorial
implementation at the loads where that overflows, accepts it below its ceiling,
and rejects a recurrence with a deliberate sign error. A comparison everything
passes would prove nothing.

### What this does not establish

Exact arithmetic confirms the formulas were **implemented** correctly. It
cannot confirm they are the **right** formulas. These remain specification
decisions, documented above and unconfirmed against real-world staffing
outcomes:

- that maximum occupancy belongs as a hard constraint on required staffing
- that shrinkage is applied last rather than folded into offered load
- that `floor(scheduled * (1 - S))` is the right bridge from scheduled
  headcount to productive agents
- the zero-volume and 99.9%-ceiling rules in the table above

Confirming those needs either a published reference or an analyst who already
knows what the answer should be for a handful of real intervals.

## What this does not model

Version 1 is deliberately narrow. Out of scope: abandonment and Erlang A,
retrials and callbacks, multi-skill or skill-based routing, queueing
simulation, forecast generation, schedule or break optimisation, and hiring
plans.

Erlang C assumes Poisson arrivals, exponentially distributed handle times,
no abandonment, and infinite queue patience. Real queues abandon, which means
Erlang C is generally **conservative** — it tends to recommend slightly more
staff than a model with abandonment would. That is usually the safer direction
for a staffing plan, but it is an assumption worth stating to anyone using the
numbers.

# Field mapping

What to put in each field well, and the one mistake that produces confident,
wrong staffing numbers.

## The five fields

| Field well | Required | Put in it |
|---|---|---|
| **Interval** | Yes | The *start* of each staffing interval. A date/time column is best. Any sortable category works, provided it separates one day from the next — see below. |
| **Forecast contacts** | Yes | Contacts offered in the interval. A plain `SUM`. |
| **Average handle time (seconds)** | Yes | A weighted average at the interval grain. **See below.** |
| **Scheduled agents** | No | Agents rostered for the interval. Staffing *level*, not a sum of row values. |

### The interval field has to separate the days

Intervals are keyed on the value you map here. A bare time-of-day with the date
discarded keys every Monday 09:00 to the same interval as every Tuesday 09:00,
so a week folds into a single day and every folded interval carries several
days' worth of contacts. Nothing errors; the chart just looks like one day.

A date/time column avoids this by construction. If your model splits date and
time into separate columns, make sure the field you map here is still the one
that distinguishes them.

To check: hover any interval and read **Queues** in the tooltip. It should
equal the number of queues you run. Three queues over five days reporting 15 is
the signature of folded days.
| **Queue or group** | No | Splits the calculation per queue. Used for filtering, selection and tooltips. |

Without the first three the visual tells you which one is missing rather than
drawing anything.

## Average handle time is the field to get right

Three of the four numbers are simple sums. This one is not, and it is the most
likely source of a wrong answer that still looks plausible.

Dropping a column called `AHT` into the field well makes Power BI average it —
and an average of averages is not the average. If one interval handled 500
contacts at 200 seconds and the next handled 5 contacts at 900 seconds, the
true weighted AHT is 207 seconds. Averaging the two averages gives 550. Offered
load comes out nearly three times too high, and the visual will confidently
recommend staffing for it.

The visual cannot detect this. Nothing in the data view distinguishes a
correctly weighted measure from a badly weighted one.

**Store handle *seconds*, not handle *time*, and divide:**

```dax
Weighted AHT =
DIVIDE(
    SUM( Interactions[HandleSeconds] ),
    SUM( Interactions[Contacts] )
)
```

The sample dataset in `sample-data/` stores `HandleSeconds` for exactly this
reason — it makes the correct measure the natural one to write.

If your source only gives you an average per interval, reconstruct the total
first:

```dax
Weighted AHT =
DIVIDE(
    SUMX( Interactions, Interactions[AvgHandleTime] * Interactions[Contacts] ),
    SUM( Interactions[Contacts] )
)
```

## Scheduled agents is a level, not a total

Required and scheduled staffing are both *headcounts at a moment*, not
quantities that accumulate over an interval. If your roster table has one row
per agent per interval, `COUNTROWS` is right. If it already stores a headcount
per interval, `SUM` over that one row is right. What is wrong is any measure
that grows when the interval gets longer.

A quick check: switch the interval length setting from 30 to 60 minutes. Your
scheduled staffing should stay roughly the same. If it doubles, the measure is
summing rather than levelling.

## Interval length is a setting, not an inference

The **Interval length** setting in the Format pane has to match the grain of
your Interval field. It is not detected from the data.

That is deliberate. Detection would have to guess from the spacing between
values, and it would guess wrong in exactly the cases that matter — overnight
gaps, missing intervals after filtering, a queue that only opens at 08:00.
A wrong guess silently changes every staffing number on the chart, so the
setting is explicit and the default is 30 minutes.

## Queues

Mapping a queue field calculates each queue separately and adds the resulting
agent counts together on the chart.

This is the no-pooling assumption: each queue is staffed by its own people.
It is the conservative reading, and the one that matches how most contact
centers actually schedule. If your agents genuinely handle all queues from one
pool, do not map the queue field — pooling more traffic into one queue needs
*fewer* total agents, and calculating them separately will overstate what you
need.

Service level and occupancy are not aggregated across queues, because there is
no honest single number for "the service level across three queues". A
multi-queue interval shows the per-queue breakdown in its tooltip instead.

## Reading the chart

| Mark | Means |
|---|---|
| Solid line | **Required scheduled agents** — the answer, after shrinkage |
| Dashed line | **Scheduled agents** — what you have rostered |
| Shaded band | The gap between them. Red and hatched when short, green when surplus |
| Faint dotted line | Required *productive* agents, before shrinkage. Off by default |

The **hatch** on an understaffed band is deliberate, not decoration.
Color-vision deficiency, grayscale printing and Windows high-contrast mode
each remove hue as a channel, so shortfall is marked by texture as well as
color and survives all three.

## Letting report readers adjust the parameters

The Format pane is **author-only**. Someone reading a published report cannot
open it, so out of the box they cannot ask "what if shrinkage were 35%?" — the
exact question this visual exists to answer.

Four optional field wells fix that. Bind a Power BI what-if parameter to one
and the reader gets a slicer.

| Field well | Overrides |
|---|---|
| Target service level % | The share half of the SLA |
| Answer within (seconds) | The seconds half of the SLA. **Seconds, not a percentage.** |
| Shrinkage % | The Format pane's shrinkage |
| Maximum occupancy % | The occupancy cap |

Whatever is not mapped keeps its Format-pane value, so you can expose just
shrinkage and leave the rest fixed.

### Setting one up

1. **Modeling → New parameter → Numeric range.**
2. Name it `Shrinkage`, minimum `0`, maximum `95`, increment `5`, default `30`.
3. Leave **Add slicer to this page** ticked.
4. Drag the generated `Shrinkage Value` measure into the **Shrinkage %** well.

The reader moves the slicer; required staffing recalculates.

### Units: whole-number percentages

These wells take **80**, not **0.8** — the same convention as the Format pane.

A what-if parameter built over `0` to `1` is the easy mistake, and it would
otherwise compute staffing for a 0.8% service-level target quite happily. So a
percentage field carrying a value between 0 and 1 is rejected with a message
naming the fix, rather than calculated with. Zero is allowed: 0% shrinkage is
a real setting.

### The chart says which parameters it used

Under the summary metrics is a line reading something like:

```
80% in 20s · 30% shrinkage · 85% max occupancy · 30-min intervals
```

It reflects whatever actually drove the calculation, so a reader moving a
what-if slicer sees the assumptions move with it. A staffing number is not a
fact on its own — it is a fact *given* those parameters — and a screenshot of
the chart would otherwise be unattributable.

It trims to the two parameters that move staffing most on a narrow visual, and
carries the full line as a tooltip. Turn it off under **Chart → Show
assumptions line**. Screen readers get it too, as part of the chart's label.

Interval length is deliberately **not** overridable. It has to match the grain
of your data, which is a property of the model rather than a question a reader
should be able to change.

## What you can change

Every card in the Format pane:

| Card | Controls |
|---|---|
| **Staffing parameters** | *Service level (SLA)*: the share of contacts to answer and the seconds to answer within. *Staffing and intervals*: shrinkage, maximum occupancy, interval length |
| **Chart** | Scheduled line, gap shading, required-productive line |
| **X axis** | Label format, rotation, and text formatting |
| **Y axis** | Show/hide, text formatting, number format |
| **Axis titles** | Show/hide, text formatting |
| **Legend** | Show/hide, text formatting |
| **Lines** | Style (solid / dashed / dotted) and width for required and scheduled |
| **Data labels** | Show/hide, which series, text formatting, number format |
| **Summary metrics** | Show/hide, text formatting, number format |
| **Assumptions line** | Show/hide, text formatting |
| **Colors** | Required, scheduled, understaffed, surplus |

### Text formatting

Every card that draws words carries the same set: **font family, size, bold,
italic, underline, alignment** and **color**.

Color has a **Match report theme** switch, on by default. Leave it on and the
text follows the report's own color, so the visual stays readable on a dark
theme. Turn it off and the color picker takes over — an explicit choice
rather than something that silently breaks when someone applies a dark theme.

Two colors ignore the setting on purpose, because they carry information
rather than decoration: an understaffed metric stays red, and so does the
evaluation-mode truncation notice.

Alignment means what it can for each element. In the summary tiles and the
assumptions line it is alignment within the box. For axis ticks, data labels
and the legend — text with no box — it shifts the label relative to the point
it is pinned to. Y-axis ticks stay right-anchored regardless, because any
other anchor walks them into the plot.

### Number format

**Summary metrics**, **data labels** and the **Y axis** each take display units
and decimal places.

Display units default to **Auto**, which only abbreviates once the digits stop
being individually meaningful — six figures. Turning "1,200 agents" into "1.2K"
loses precision a staffing plan needs, so that is not something the visual does
uninvited. Set **Thousands** or **Millions** explicitly if you want it sooner.

Decimal places default to **-1**, meaning "let the value decide": whole agent
counts print whole, percentages get one decimal, and an abbreviated value keeps
a decimal so "1.2K" does not collapse to "1K".

### The summary metrics

| Metric | Means |
|---|---|
| **Forecast SLA** | The service level the scheduled staffing is expected to deliver, across everything visible. Red when it falls below your target. |
| **Understaffed** | Intervals where scheduled staffing is below requirement |
| **Largest deficit** | The worst single-interval shortfall |
| **Peak required** | The highest scheduled requirement in the range |
| **Intervals on target** | The share of intervals that individually clear the target |

**Forecast SLA and Intervals on target are not the same number**, and confusing
them is easy. One says what service level the day will deliver; the other says
how many intervals clear the bar. A day can have two thirds of its intervals on
target and still deliver well under target overall, if the misses land where
the volume is.

Forecast SLA is **weighted by contacts**, because service level is a ratio of
contacts answered in time to contacts offered. Averaging the per-interval
percentages instead would let a quiet overnight interval at 100% cancel out a
peak interval at 40% — on the sample data that inflates the figure from 62.4%
to 75.3%, which would be a comfortable and completely wrong number to plan
against.

Zero-volume intervals carry no weight, so an overnight stretch cannot drag the
forecast either way. An overloaded interval counts as 0%, not as missing —
dropping it would flatter the result.

### Data labels

Off by default, because an interval chart is dense and the shape usually
matters more than the individual numbers. Turn them on under **Data labels**
and choose required, scheduled or both.

Two things happen automatically:

- **They thin out.** 96 numbers across 600 pixels is a smear, not information,
  so labels drop to every second or third interval the same way axis labels do.
  A larger text size means fewer labels.
- **They avoid each other.** With both series labeled, each number sits on the
  far side of its own line from the other one — so they do not stack up where
  the lines run close together, which is exactly when someone turns labels on.

Each label carries a halo in the background color so digits stay legible over
a gridline or a shaded gap band.

### Axis label format

**X axis → Label format** offers presets — time only, date only, weekday and
time, and so on — or **Custom** for your own pattern.

| Token | Gives | Token | Gives |
|---|---|---|---|
| `yyyy` `yy` | 2026, 26 | `HH` `H` | 14, 14 |
| `MMMM` `MMM` | March, Mar | `hh` `h` | 02, 2 |
| `MM` `M` | 03, 3 | `mm` `m` | 30, 30 |
| `dddd` `ddd` | Monday, Mon | `ss` `s` | 00, 0 |
| `dd` `d` | 02, 2 | `tt` | PM |

**Put literal text in single quotes.** `HH'h'mm` gives `14h30`; without the
quotes the `h` is read as a token and you get `14230`. `''` is an apostrophe,
so `HH:mm 'o''clock'` gives `14:30 o'clock`. Month and weekday names follow the
report's locale.

Non-date intervals are left alone — a category axis is already whatever you
made it.

### Axis label rotation

**X axis → Label rotation**: Automatic, Horizontal, Angled (45°) or Vertical
(90°). Rotated labels take far less horizontal room, so many more of them
survive before the axis starts dropping every second or third one. The chart
reserves the extra height automatically.

Automatic stays horizontal until the labels stop fitting, then angles them. It
never goes vertical on its own — 90° costs a lot of chart height, and that is a
trade worth opting into rather than having sprung on you by a resize.

The y-axis is titled **Agents**; the x-axis carries the display name of
whichever column you mapped to Interval, so a mismapped field shows up at a
glance. Both can be switched off under **Chart → Show axis titles**, and both
drop out automatically below 300 x 140.

The legend appears above the chart when the visual is at least 380px wide and
150px tall. Below that the chart is a sparkline and there is little left to
label.

## Getting a sanity check

The quickest test that a mapping is right: pick your busiest interval and check
it by hand.

1. Read the forecast contacts and weighted AHT from the tooltip.
2. Offered load is `contacts × AHT ÷ interval seconds`. For 300 contacts at 280
   seconds in a 30-minute interval: `300 × 280 ÷ 1800 = 46.7` Erlangs.
3. Required productive agents will be somewhere just above that — typically 10
   to 20 percent higher at an 80/20 target.
4. Required scheduled will be that divided by `1 − shrinkage`.

If required staffing comes out at three times offered load, or below it, the
AHT measure is the first thing to check.

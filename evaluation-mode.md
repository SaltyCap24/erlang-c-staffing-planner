---
title: Evaluation mode
---

# Evaluation mode

Without a licence, the visual runs in **evaluation mode**. It is not a timed
trial and it does not expire — you can use it for as long as you like.

The point of it is to let you check the arithmetic against your own data before
you spend anything. It calculates twelve intervals of your real forecast, with
your real parameters, and shows the full working. If those twelve intervals
reconcile against your spreadsheet, the rest will too.

What it will not do is run a shift. Twelve intervals is six hours at half-hour
grain, and none of the roster comparison is there.

---

## What you get without a licence

**Your own data.** No sample-data restriction, no demo mode. Load your forecast
and it calculates your forecast.

**Twelve intervals**, calculated and charted. That limit is the same whatever
your interval length.

**Every staffing parameter honoured** — target service level, answer threshold,
shrinkage, maximum occupancy and interval length. Both the Format pane settings
and the what-if override fields work, so you can move a slicer and watch the
requirement move.

**Required productive and required scheduled staffing** — the two numbers the
calculation exists to produce.

**The full calculation chain in the tooltip.** Contacts, handle time, offered
load in Erlangs, required productive, required scheduled, and the shrinkage
applied. This is deliberate: a staffing figure you cannot reconcile is a
staffing figure you cannot defend, and checking the maths is the whole purpose
of evaluating.

**The assumptions line**, so you can confirm which parameters produced the
numbers you are checking.

Axes, axis titles and the legend all render normally.

---

## What a licence adds

**Your whole range.** A day, a week, a month — whatever your filters select,
rather than twelve intervals.

**Your roster, compared.** Scheduled staffing drawn against required staffing,
with the gap shaded interval by interval — hatched red where you are short,
green where you are over.

**Summary metrics.** Forecast service level, understaffed interval count,
largest single-interval deficit, peak requirement, and the share of intervals
clearing target.

**Multiple queues.** Each queue calculated separately and summed, with a
per-queue breakdown in every tooltip.

**Expected performance from your roster** — the service level and occupancy
your scheduled staffing is forecast to deliver, rather than only what the
target demands.

**Overload and data-quality warnings**, where an interval's roster cannot
reach a steady state.

---

## Two things worth knowing

**A queue field will not calculate without a licence.** Rather than quietly
ignoring the field, the visual stops and says so. Staffing queues separately
produces a higher total than pooling them, so ignoring the field would give
you a number matching neither your spreadsheet nor the licensed version.
Remove the field to see pooled staffing, or add a licence.

**A scheduled-agents field is ignored rather than refused.** Unlike a queue
field it does not change required staffing, so leaving it mapped costs you
nothing — the numbers you are checking stay correct. A banner explains why the
comparison is missing.

If a licence check cannot complete — an offline moment, an unsupported
embedding context — the visual falls back to evaluation mode rather than
showing nothing. A lookup that failed is not evidence that you are unlicensed.

---

## Getting a licence

Licences are bought and assigned through Microsoft, not through us. Assign them
per user in the [Microsoft 365 admin center](https://admin.microsoft.com).

Everyone who **views** a report needs a licence, not only the person who built
it. An unlicensed viewer sees evaluation mode.

After assigning a licence it can take up to an hour to be recognised. Refresh
the browser, or restart Power BI Desktop.

Questions: [support](support), or **cap@work4wifi.com**.

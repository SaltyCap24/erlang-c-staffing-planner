# Evaluation Mode — specification

**Status: built, dormant.** `src/licensing/features.ts` gates every feature on
licence state, and the rest of the visual honours it. It has no effect yet
because `LICENSED_SERVICE_PLANS` is empty, so every session resolves to
`not-enforced` and gets the full feature set. Filling that array in — once the
Partner Center plan exists — turns Evaluation Mode on.

## The decision

Ship a **permanent, deliberately limited free tier** rather than a timed trial.
Paid licences unlock the rest.

A genuine 7–14 day unlimited trial would require tracking activation dates per
user, which means a server, accounts, hosting, a privacy policy covering
collected data, and outbound network access. Outbound network access alone
disqualifies the visual from Power BI certification — the audit checks for it
explicitly, and `pbiviz package --certification-audit` currently reports "no
external requests found". That is not worth trading away.

A free tier costs nothing architecturally: the licence state is already known
client-side, and gating features on it needs no network, no storage and no
identity.

This also sidesteps a one-way door recorded in the Phase 0 report: Microsoft
states a free trial **cannot be disabled later** for the same published plan.
Not enabling one avoids the commitment entirely.

## A Microsoft-managed timed trial is not available for this offer type

Confirmed 26 August 2026 against the Microsoft Marketplace publisher FAQ, and
it settles the question rather than reopening it.

The built-in free-period trial — customer charged $0.00 for 1 to 180 days, then
rolling to paid automatically — applies to **SaaS and VM offers only**. For a
Power BI visual, Microsoft's only trial route is a *listing-only* offer that
sends the customer to a web address of your choice to run a trial **outside**
the marketplace, hosted and managed by the publisher.

That is the server-and-accounts path this document already rules out on
certification grounds. So Evaluation Mode was not merely the better option for
this offer type — it is the only in-product one.

## What Microsoft supports

Confirmed against [License models for Power BI AppSource visuals](https://learn.microsoft.com/en-us/power-bi/developer/visuals/custom-visual-licenses):

> Some visuals have free trial versions, while others have a **basic version
> available for free with extra functionality available for purchase**.

And the matching notification:

> If you have a free version of the visual, a banner appears with a link to
> upgrade your license. This banner will disappear after a while.

That banner is `notifyFeatureBlocked(tooltip)`. The corner icon is
`notifyLicenseRequired(LicenseNotificationType.General)`. Both are host-drawn;
the visual must not build its own purchase UI.

## The split

### Free — Evaluation Mode

Enough to check the arithmetic against a spreadsheet with real data, not
enough to run a shift.

- Their own data, their own model, no sample-data restriction
- **12 intervals** calculated and charted
- All staffing parameters honoured: target, threshold, shrinkage, occupancy,
  interval length — both Format-pane and what-if fields
- Required productive and required scheduled staffing
- The assumptions line, so the parameters can be checked against the workbook
- Axis titles, legend, axes
- **The calculation chain in the tooltip** — see *Problem 1* below
- The Microsoft upgrade indicator

### Paid

- The complete day, week or whatever is filtered
- Scheduled staffing, the required-versus-scheduled comparison and the gap
- Deficit hatching and surplus bands
- Summary metrics — peak, understaffed count, largest deficit, target met
- Queue / group support
- Expected service level and occupancy from scheduled staffing
- Overload and data-quality warnings in the tooltip

---

## Three problems found in the first draft — all resolved

### Problem 1: the split as written made evaluation useless

The stated goal is that someone can *"verify the math against Excel using real
data in ten minutes"*. But "full tooltips and diagnostics" sits on the paid
side, and the tooltip is where the calculation is visible.

Without it an evaluator sees a line with no numbers behind it. They cannot
check offered load, cannot see required productive versus required scheduled,
and therefore cannot reconcile anything. The free tier would demonstrate that
the visual draws a chart, not that it computes correctly — which is the one
thing it needs to prove.

**Resolved: split the tooltip rather than withholding it.**

| Evaluation tooltip | Paid tooltip adds |
|---|---|
| Interval | Queue |
| Forecast contacts | Scheduled agents |
| Average handle time | Staffing gap |
| Offered load (Erlangs) | Expected service level |
| Required productive agents | Expected occupancy |
| Required scheduled agents | Overload and data-quality warnings |

The left column *is* the audit trail. It is also exactly the part a
spreadsheet reproduces, so it is what makes the ten-minute check possible.
Everything on the right depends on scheduled staffing, which is already a paid
feature — so the split falls along a line that is easy to explain and needs no
special-casing.

### Problem 2: "the first 12 intervals" is often the wrong 12

Most interval datasets start at midnight, so a centre open 07:00–21:00 would
show a flat line at zero.

**Resolved: leave it.** The first 12 in sort order, full stop. A report author
who wants a different window filters to it — which they can already do, and
which is one fewer rule to explain than "the first 12 with volume". The
truncation notice states the window, so nothing is hidden.

### Problem 3: in Reading mode the reader may get no indicator at all

This is the one that matters most, and it comes out of the Phase 3 work rather
than the docs.

`LicenseNotificationType.General` is **only enforced in Edit mode** in a
supported environment — Microsoft's own note says calling it in Reading mode or
on a dashboard "doesn't apply the icon and returns `false`".
`notifyFeatureBlocked` similarly no-ops in unsupported environments.

So a report *consumer* on an unlicensed tenant could see a chart showing 12
intervals of a 96-interval day **with nothing indicating why**. That is not a
paywall, it is a wrong chart. Someone would staff a shift off it.

**Resolved: render a neutral, factual truncation label** whenever the data is
cut, regardless of whether the host drew its notification:

```
Evaluation mode · showing 12 of 96 intervals
```

This is not licensing UX and does not compete with Microsoft's notification.
It carries no price, no call to action and no upgrade link — it explains why
the chart shows less than the data contains. Withholding that explanation
would be the actual violation of good faith, and the guidance Microsoft gives
is against homemade *purchase dialogs*, not against a visual describing its own
state.

The `notifyLicenseRequired` / `notifyFeatureBlocked` return values are already
read and stored (`LicenseManager.needsOwnMessage`), so the visual already knows
when the host declined to draw anything.

---

## Resolved rules

**A queue field without a licence refuses to calculate.** Per-queue staffing
sums higher than pooled, so silently ignoring the field would produce numbers
matching neither the workbook nor the paid version. The visual shows a state
message asking for the field to be removed or the licence upgraded, and
calculates nothing. Refusing is honest; quietly changing the model is not.

**A scheduled-agents field is ignored, not refused.** Unlike a queue field it
does not alter required staffing, so ignoring it changes nothing about the
numbers being verified. The feature-blocked banner says why.

**Twelve intervals, fixed**, at every interval length.

**Nothing blocks outright any more.** `unlicensed`, `unavailable` and
`unsupported-environment` all fall back to the free tier rather than a blank
rectangle. For `unavailable` in particular that is the only defensible answer:
a lookup that could not complete is not evidence that anyone is unlicensed.
The old `VisualIsBlocked` overlay is deliberately no longer requested — it
hides the visual entirely, which is the opposite of Evaluation Mode.

**Paid outputs are not computed at all**, not computed and hidden. The interval
limit is applied before calculation, and a paid scheduled value is kept out of
the y-axis range so it cannot leak through the axis scale.

## What is tested

`test/licensing/features.test.ts` and `test/licensing/evaluationMode.test.ts`
cover the interval limit, the suppression of paid outputs including the y-axis
leak, the tooltip split in both directions, and the truncation notice —
including an assertion that it contains no price, button or upgrade wording,
since that is the line between describing state and building a purchase
dialog.

## What this needs from the owner

Nothing to start building. Two things before it can ship:

1. The Partner Center offer must exist so `LICENSED_SERVICE_PLANS` in
   `src/licensing/licenseTypes.ts` can be filled with the plan's Service ID.
   Until it is, enforcement stays off and everything renders as licensed.
2. The private-plan purchase test, which is the only way to exercise the real
   licensed path end to end. Every state is already mocked in
   `test/licensing/`, so that test should be a confirmation rather than a
   discovery exercise.

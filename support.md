---
title: Support
---

# Support

## Before contacting us

Most problems are one of these, and the [user guide](user-guide) covers each in
full:

| What you are seeing | Usually |
|---|---|
| Staffing looks far too high | Average handle time is a plain column instead of a weighted measure. [Step 3](user-guide#step-3-the-one-measure-you-must-build) |
| Only one point per day on the chart | The interval field is bound to the Date Hierarchy instead of the plain field |
| Staffing doubles when you change interval length | The scheduled measure is summing rather than levelling. Scheduled agents is a headcount at a moment, not a running total |
| "This field takes whole-number percentages" | A what-if parameter was built over 0 to 1 instead of 0 to 100 |
| A gap in the scheduled line | Those intervals have no roster value. A blank is treated as unknown, not as zero agents, so the line breaks rather than dropping to the floor |
| Numbers higher than an online Erlang calculator | The occupancy ceiling. Most calculators solve only for service level; this one also keeps occupancy under your limit |

## Contact

**cap@work4wifi.com**

We aim to respond within **two business days**.

Please include:

- What you expected, and what you saw instead
- The version, from the Format pane under **About**
- Whether you are in Power BI Desktop or the Power BI service
- A screenshot, if the problem is something you can see

**Please do not send customer data.** A description of the shape of your data is
almost always enough. If we genuinely need a sample we will ask for one, with
volumes scaled or anonymised.

## Licensing and billing

Purchases, licence assignment and cancellation are handled by Microsoft, not by
us. Assign or remove licences per user in the [Microsoft 365 admin
center](https://admin.microsoft.com).

After a licence is assigned it can take up to an hour to be recognised. Refresh
the browser, or restart Power BI Desktop.

For anything to do with billing, invoices or refunds, contact Microsoft
support rather than us.

## Known limitations

Licence enforcement is not available in Publish to Web, PaaS embedding, Power BI
Report Server, national or regional clouds, or REST-API export to PDF and
PowerPoint. That is a Power BI platform limitation rather than a choice.

This release sizes staffing for a forecast you supply. It does not generate
schedules, optimise breaks, handle multi-skill routing, or model abandonment.
The [user guide](user-guide#what-this-visual-does-not-do) has the full list.

---

Erlang C Staffing Planner is published by **WORK4WIFI LLC**, 5101 Compass Way,
Christiana, TN 37037.

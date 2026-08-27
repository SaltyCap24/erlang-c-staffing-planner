---
title: Erlang C Staffing Planner
---

# Erlang C Staffing Planner

Interval-level contact-centre staffing for Power BI. The visual takes your
forecast and tells you how many agents each interval needs to hit your service
level — using Erlang C, entirely inside Power BI, with no data leaving your
report.

Published by **WORK4WIFI LLC**.

## Documentation

| | |
|---|---|
| [User guide](user-guide) | Setup, field mapping, the weighted handle-time measure, what-if parameters, troubleshooting |
| [Field mapping](field-mapping) | What each field well expects, and the measures behind them |
| [How the calculation works](calculation-guide) | Erlang C, offered load, occupancy and shrinkage, worked through |
| [Evaluation mode](evaluation-mode) | What the free tier includes and what a licence unlocks |

## Support

Questions, problems, or a staffing number you cannot reconcile:
**cap@work4wifi.com**

Please include the visual version — it is shown in the Format pane under
**About** — and what you expected versus what you saw. Please do not send
customer data; a description of the shape of your data is almost always
enough.

## What it does not do

Version 1 models a single channel with no abandonment, no multi-skill routing
and no channel blending. Queues are calculated separately and summed, which
assumes no pooling between them. The [user guide](user-guide) sets out the full
list, because knowing where a model stops is part of using it properly.

---
title: Erlang C Staffing Planner
---

# Erlang C Staffing Planner

Interval-level contact-center staffing for Power BI. The visual takes your
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
| [Evaluation mode](evaluation-mode) | What the free tier includes and what a license unlocks |

| [Support](support) | Common problems, how to reach us, licensing and billing |
| [Privacy policy](privacy) | What the visual accesses, and what it transmits |

## Support

Questions, problems, or a staffing number you cannot reconcile:
**cap@work4wifi.com**

Check the [support page](support) first — most problems are one of half a dozen
things, and each has a short answer there.

## What it does not do

Version 1 models a single channel with no abandonment, no multi-skill routing
and no channel blending. Queues are calculated separately and summed, which
assumes no pooling between them. The [user guide](user-guide) sets out the full
list, because knowing where a model stops is part of using it properly.

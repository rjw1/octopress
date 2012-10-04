---
layout: post
title: "Metric retention just add one"
date: 2012-10-04 19:53
comments: true
categories: ops monitoring metric retention
---

Something occurred to me a while back when I was setting up some monitoring. Maybe it's obvious to everyone else, but it's not something that I've seen discussed or written before:

> Retain your metrics for `$unit_of_time + 1`

Where retention is either the complete data or a lower resolution thereof. Unit of time may be a day, week, month, year, or even ten years. Quite often it seems logical to rotate or average at these round values.

## Don't, add one..

- 24 hours becomes 25 hours.
- 7 days becomes 8 days.
- 4 weeks becomes 5 weeks.
- 12 months becomes 13 months.

## Why?

Nearly every service experiences some form of time-based usage fluctuation. Your users probably wake up at a certain time of day, they might be more productive on a Tuesday, make more purchases at the beginning of the month, maybe they consume more media during the school holidays, whatever.

Data is near worthless without context. If you can't compare the metrics you're looking at now to another representative time period in the past, and at the same resolution, then how can you determine that what you're seeing is normal?

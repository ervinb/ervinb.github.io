---
layout: article
title: SLO Alerting for Mortals 
tags: sre
key: slo-alerting
---

I assume you're here with a spinning head from reading Google's SRE book/articles,
or you just need a quick refresher about the SLO alerting math.

I will keep the text to a minimum (there are [better places to get that](https://sre.google/workbook/alerting-on-slos/)), and mostly
focus on visual representation of the basic ideas behind SLO-based alerts.


## SLOs and SLIs

Service level objective (SLO). It represents how reliably the service is delivering "value" to its users.
Service level indicator (SLI). A measurement of a specific service metric. We're using SLIs and math to define an SLO.

A 99.9% SLO per month means if 0.1% of requests fail, that's *acceptable*, and it will not cause panic around the globe.
That allowed error rate is also known as the *error budget*.

![slo-req-count](/assets/images/slo-alerting/slo-count-999.png)

We're ok with the fact that 1 in every 1K requests will fail.

An SLI can be the error rate of the incoming requests.
![slo-vs-sli](/assets/images/slo-alerting/sli-slo-init.png)


## Converting time

Most of the math involved here is about converting various time units (i.e 1 month to hours)
and checking the ratio between them (i.e. 1h is 0.14% of a month).

![time-conversion](/assets/images/slo-alerting/time-conv.png)

Time wise, a 0.1% error rate for a month means a 43 minute complete downtime.


## Burn rate

When our budget of 43 minutes a month starts burning, we want to know about it fairly quickly.

![error-budget-graph](/assets/images/slo-alerting/error-budget-monthly-graph.png)

The green line represents the border between good and evil: if the error rate is *exactly* 0.1% throughout the month, we're still fine, but barely.
As soon as the line starts to tilt left (into the danger zone) it means that the error rate is higher than the allowed 0.1%, and we should
focus more on improving the service.

That "tilting" is also called the *burn rate*.


## Defining the first alert

As a start, we define the following alert condition.

![initial-alert-condition](/assets/images/slo-alerting/slo-query-first.png)

This translates to "if the green line starts tilting left: alert!"

And it is fine until it starts to alert 19 times a day. The slightest increase of error rate will trigger an alert. If we have a constant
error rate just above our threshold for an hour, we will burn through roughly 0.14% of the error budget.
(1 hour is 0.14% of 1 month.)

Not really worth to wake up someone over it.

To improve on this, let's say we want 2% of our monthly error budget to be burned within an hour, before the alert fires.
So the budget burn is currently 0.14% per hour but we want 2%, so we need to multiply our threshold with some number to reach this new target.

Dividing the target with the current value gives us the multiplier: `2%/0.14% = 14.3`

![alert-condition-multiplied](/assets/images/slo-alerting/slo-query-first-144.png)

This magic multiplier is called the burn rate (or "tilting of the green line to the left" as we defined it a couple of lines above).


# Simulating an alert

Here's the scenario:
- 10 requests per 5 minutes is our traffic (constant)
- 10% error rate for 10 minutes (1 request out of 10 will fail)
- A snapshot of the state is taken every 5 minutes (`scrape_interval` in Prometheus terms). In the real world, you will probably have snapshots every 30 seconds or every minute. We're using 5 minutes here for easier calculation, and to be able to draw square error rate lines, instead of sloping ones.

![1h-graph](/assets/images/slo-alerting/1h-graph-err-rate.png)

An error rate of 10% in 2 subsequent snapshots (0m-5m, 5m-10m) is enough to trigger the alert.

![1h-graph-calc](/assets/images/slo-alerting/1h-graph-err-rate-calculation.png)

What immediately stands out here is the long running alert. It will be active for 55 minutes, even tough we're not constantly in an erronous state.

![1h-moving-window-smooth](https://imgur.com/q5CUnRk.gif)

As long as both erronous snapshots are inside the window, the alert will be active. When one of them leaves, the error rate drops
to 0.8% and the alert stops.


## Multi-window alerts

One way to combat the long running alert is to introduce another, shorter time window. It will make sure to end the alert, not long after the error rate goes back to normal. A 5 minute time window with the same error rate as before does just that.

![1h5m-query](/assets/images/slo-alerting/1h5m-query.png)

![1h5m-graph](/assets/images/slo-alerting/1h5m-graph-err-rate.png)

The alert is active only when the snapshots with a high error rate are in both time windows - in our case this is true for 10 minutes.

![1h5m-moving-window-smooth](https://i.imgur.com/2NFzF39.gif)

With the addition of the shorter time window, we made sure that alert is matching reality more closely, i.e. it reacts only when there's an ongoing issue.

These were the basics, the next level would be multi-level, multi-burn rate alert which are a bit out of the scope of this post, so refer to
the [Google SLO alerting documentation for more details](https://sre.google/workbook/alerting-on-slos).
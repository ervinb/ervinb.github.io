---
layout: article
title: SLO Alerting for Mortals 
tags: sre
key: slo-alerting
---

This is an attempt to break down the concept of SLO alerting as much as possible.
Step-by-step, each concept will be illustrated and occasionally animated.

My hope is that the short material presented here is intuitive enough for it to stick, and to serve as a
base for further research.

I will try to demistify concepts such as burn rate, error budget, and multi-window alerts.


## SLOs and SLIs

Service level objective (SLO). It represents how reliably the service is delivering "value" to its users.


Service level indicator (SLI). A measurement of a specific service metric. We're using SLIs and math to define an SLO.


A 99.9% SLO per month means if 0.1% of requests fail, that's *acceptable*, and it won't raise any fuss.

![slo-req-count](/assets/images/slo-alerting/slo-count-999.png)

We're ok with the fact that 1 in every 1K requests will fail.

An SLI can be the error rate of the incoming requests.
![slo-vs-sli](/assets/images/slo-alerting/sli-slo-init.png)

The 0.1% wiggle room is the limit above which we don't want to go - the error budget.

![error-budget-timeline](/assets/images/slo-alerting/error-budget-timeline.png)


## Converting time

Most of the math involved here is about converting various time units (i.e 1 month to 720 hours)
and checking the ratio between them (i.e. 1h is 0.14% of a month). Then using those ratios
with existing SLIs to come to an SLO condition.

![time-conversion](/assets/images/slo-alerting/time-conv.png)

Time wise, a 0.1% error rate for a month means a 43 minute complete downtime.

![slo-time](/assets/images/slo-alerting/metric-slo-timeline.png)


## Burn rate

When our budget of 43 minutes a month starts burning, we want to know about it fairly quickly.

![error-budget-graph](/assets/images/slo-alerting/error-budget-monthly-graph.png)

The green line represents the border between good and evil: if the error rate is *exactly* 0.1% throughout the month, we're still fine, but barely.
The burn rate is 1 in this case. As soon as the line starts to tilt left (into the danger zone), the error rate is higher than the allowed 0.1% and in
turn, and the burn rate also increases - the system eats the error budget faster than it should.


## Defining the first alert

As a start, we define the following alert condition, which is identical to the SLO.

![initial-alert-condition](/assets/images/slo-alerting/slo-query-first.png)

Keeping the graph above in mind, this translates to "if the green line starts tilting left: alert!"

To reiterate on our timeline, the error budget spans out across the whole month. The alert we defined operates in an hour long time window.
It follows that the 1 hour time window is of course not the whole month, but only 0.14% percent of it.

At the error rate of 0.1% (our SLO), 0.14% of the budget is consumed during 1 hour.

![slo-vs-sli](/assets/images/slo-alerting/metric-slo-sli-timeline.png)

Not really worth to wake up someone over it.

Another issue is that the alert will be active for almost an hour. (we will revisit this a bit later)

To improve on this, instead of the 0.14% budget burn in an hour, the alert should have a higher threshold and aim at a 2% burn.
We need to multiply our threshold with some number to reach this new target.

Dividing the target value with the current one gives us the multiplier: `2%/0.14% = 14.3`

![alert-condition-multiplied](/assets/images/slo-alerting/slo-query-first-144.png)

This magic multiplier is actually the burn rate (or "tilting of the green line to the left" as we defined it a couple of lines above).


# Simulating an alert

Here's the scenario:
- 10 requests per 5 minutes is our traffic (constant)
- 10% error rate for 10 minutes (1 request out of 10 will fail)
- A snapshot of the state is taken every 5 minutes (`scrape_interval` in Prometheus terms). In the real world, you will probably have snapshots every 30 seconds or every minute. We're using 5 minutes here for easier calculation, and to be able to draw square error rate lines, instead of sloping ones.

![1h-graph](/assets/images/slo-alerting/1h-graph-err-rate.png)

An error rate of 10% in 2 subsequent snapshots (0m-5m, 5m-10m) is enough to trigger the alert.

![1h-graph-calc](/assets/images/slo-alerting/1h-graph-err-rate-calculation.png)

What immediately stands out here is the long running alert. It will be active for 55 minutes, even tough we're not constantly in an erronous state.
The `http_error_rate[1h]` metric considers the samples in the last 1 hour, and this time window is shifted with each snapshot. The snapshots are happening at the markers on the animation, every 5 minutes (remember, this is how the scenario was defined).

![1h-moving-window-smooth](https://imgur.com/q5CUnRk.gif)

As long as both erronous snapshots are inside the window, the alert will be active. When one of them leaves, the error rate drops
to 0.8% (as per the calculation above) and the alert stops. This is why the alert is firing for 55 minutes instead of 1 hour.
In a more realistic scenario, with a 30s second snapshot interval, it will be active for 1 hour.


## Multi-window alerts

One way to combat the long running alert is to introduce another, shorter time window. It will make sure to end the alert, not long after the error rate goes back to normal. A 5 minute time window with the same error rate as before does just that.

![1h5m-query](/assets/images/slo-alerting/1h5m-query.png)

![1h5m-graph](/assets/images/slo-alerting/1h5m-graph-err-rate.png)

A single bad request out of 10 is plenty to trigger the first, 5 minute condition. The condition with the 1 hour window sets off
after the second subsequent snapshot with an elevated error rate, as it needs 2 erronous requests out of 120 to surpass the threshold, as we saw in the previous section.


Both conditions have the same threshold of 1.4% (0.1% * 14.4); the difference is that the 5 minute one takes 10 samples into consideration, and the 1 hour one takes
120 samples. A bad request has naturally a bigger impact on the smaller sample size than on the bigger one - 1 in 10 vs. 1 in 120. The smaller window
is more jittery, where the longer one is slugish, but as they meet at the middle, the result is almost the best of both worlds: we have reasonable
sensitivity and decent reset time (i.e the alert stops when the coast is clear).

The alert is active only when the snapshots with a high error rate are in both time windows - in our case this is true for 10 minutes.

![1h5m-moving-window-smooth](https://i.imgur.com/2NFzF39.gif)

With the addition of the shorter time window, we made sure that alert is matching reality more closely, i.e. it reacts only when there's an ongoing issue.

These were the basics, the next level would be multi-level, multi-burn rate alert which are a bit out of the scope of this post, so refer to
the [Google SLO alerting documentation for more details](https://sre.google/workbook/alerting-on-slos).

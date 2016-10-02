---
layout: post
title: SLA driven metrics
tags:
- python
---

Every organization's internal monitoring setup has a variety of environments it caters to, with varying levels of alerting mechanisms (email, slack, pagerduty, etc.) and response time expectations. There are environments which are monitored with the same SLA as production systems and environments where in, after the initial setup, developers take responsibilities of getting alerts and acting on them. In the midst of focusing solely on external customer facing SLAs it is easy to loose track of response time tracking for other environments.

I have been playing with [icinga2](https://www.icinga.org/products/icinga-2/) and it is pretty neat in the sense that it has a solid REST [API](http://docs.icinga.org/icinga2/latest/doc/module/icinga2/chapter/icinga2-api), a good [API client](https://github.com/saurabh-hirani/icinga2_api) and in built features like [high availability and distributed checks](https://www.icinga.org/products/icinga-2/distributed-monitoring/). One of the interesting end points exposed by the API are the service states which givies insight into a very crucial metric - the last_ok state. A sample output of the API is:

```
"last_state_ok": 1475377506.875772,
```

which means the last time this service was in the "OK" state was this timestamp. This one metric can give a very piercing insight into how alert response times are handled.

If the last time a service was in "OK" state 3 hours ago - it means that it has not been acted upon. And that might be because the service is running in a non-prod, non-stage environment (I am looking at you QA). In a real world scenario you don't want to alert the on-call person with a pager every time QA breaks (and most of the times - it is broken deliberately to check out some scenarios) but such a long ignored alert should also surface up. This is the point at which the drilling stops because you have a hard choice to make - pager all environments or pager none. There is no middle ground.

There can be one - if instead of alerting the ops team on a specific service - alerts/reports are sent when SLAs are violated

For example:

1. A service in environment X was in warning/critical state for more than N minutes
2. SLA violated
3. Slack the escalation points of contact
4. If the number goes beyond a limit, raise a pager. But the saving grace of that pager would be that it would be an aggregated one (10 services in warning state for more than N minutes) and not for a single service.

The last point is what makes all the difference. Non critical environments should not pager on a single service failure basis - they should pager on an aggregate number.

And you wouldn't mind a reminder that environment E is broken because while you were sipping coffee, your alerts/xyz email folder was being bombed, which you never check because - who checks emails for alerts?

This data is easy to plot. Define a bunch of time slots (10-30 minutes, 30-60 minutes, 1-2 hours, etc.) and plot the number of warning/critical services in those slots every N minutes. Define response time slots for various environments and alert only when those slots are crossed. This could also be an insight for the higher management into the response times for alerts. Because no matter what you say - if an alert is not resolved on the monitoring system, the problem still exists on record.

Talk is cheap. Show me the code.

This is the [graphite plugin](https://github.com/saurabh-hirani/graphite-plugins/tree/master/icinga2_plugins) which talks to icinga2 and gets these numbers.

It combines two of my works - [icinga2_api](https://github.com/saurabh-hirani/icinga2_api) and [slotter](https://github.com/saurabh-hirani/slotter)

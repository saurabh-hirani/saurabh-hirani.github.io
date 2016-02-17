---
layout: post
title: Monitoring transition - manual => automated => distributed
author: Saurabh
tags: 
- devops
- icinga2
---

*"All happy families are alike; each unhappy family is unhappy in its own way."
- Leo Tolstoy.*

*"All automated monitoring systems work similarly; each manual nagios setup is manual in its own way."*

If you have ever maintained a manual monitoring setup, you know the gist captured by the above statement. And if you have spent enough time in infrastructure, you will see that the monitoring systems of an organization are often polarized in 2 camps - clean systems built from ground up and handcrafted wizardry fed with generous additions of bandaids every other day. 

Everyone wants to have the former and criticize the later, but rarely do you see someone sharing their experiences on the transition. I am going to give a crisp talk on how we transitioned a manually maintained [nagios](https://www.nagios.org/) setup to an automated [icinga2](https://www.icinga.org/icinga/icinga-2/) setup at [Rootconf 2016](https://rootconf.talkfunnel.com/2016/13-the-transition-manual-automated-distributed-monito). This is an introductory post on the same.

The costs incurred by having manually maintained system are immense and worst of all, immeasureable. They gnaw into the productivity of the people maintaining them (never out of choice) and feed on themselves to become unmanageable monsters. At some point you have to take a step back and just kill the beast. But if that beast is your monitoring system - the eyes and ears of your infrastructure - you cannot just shut it down. Because no matter how manual it is - it works.

A transparent phased cutover can be planned and executed. But how do you measure the components of the manual setup - how many hosts, hostgroups, checks are running on this system? How many of these hosts are decomissioned or worse, unwanted and still consuming resources? How do you start consolidating unmonitored sources while you are at it? And how do you keep track of changes trickling in the manual setup while you are building an automated solution?

These are just few of the critical questions that you need to answer before you put on your cape and save the world. There are logical baby steps that can help you show measurable progress at each stage and address these issues. 

1. Measure what you have

2. Select a tool to address those limitations

3. Consolidate your checks

4. Know your sources of truth - AWS, VMware, network devices, etc. and how you can talk to them through APIs.

5. Break with permission

6. Automated rules to catch future deviations.

I have expanded on these points in my work-in-progress slide deck [here](http://go-talks.appspot.com/github.com/saurabh-hirani/talks/monitoring-transition/monitoring-transition-wip.slide#1)

All of this is easier said than done, but with the right approach and restraint, it is doable. And the paybacks are worth it. Catch me talk about how we accomplished it at [Rootconf 2016](https://rootconf.in/2016/)! 

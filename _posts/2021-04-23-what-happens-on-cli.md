---
layout: post
title: What happens on cli (no longer) stays on cli
tags:
- Linux
- cli
---

<style type="text/css">
pre {
	width: 1000px;                          /* specify width  */
}
</style>

We have all been there. I have been there so many times that I am no longer welcome
there. That place is - "Hack up X quickly".

X = pattern matching pipeline commands or
<br/>
    docker-compose containers to test a REST API or
<br/>
    some cli variant which involves using `jq`, `sed`, `awk` and other Avengers.
<br/>

A systems engineer's command line muscle memory is built everyday through deliberate, engaging
practice. Assumptions are often broken when new tools replace older ones. For example,
[ag](https://github.com/ggreer/the_silver_searcher) changed the way I grep. One needs to have both -
the flexibility to adopt new tools and the perseverance to explore the old ones to their depths
before chasing the next shiny new thing.

The second part - perseverance, can only be built with discipline. By discipline, I do not mean the
one that is the foundation of flame wars like memacs is infinitely better than youvim. I mean holding
yourself accountable for your learning, enforcing mental reminders so that you don't fall back into the
easy mouse-clicky shortcuts that you wanted to escape from in the first place. These reminders can be - avoiding
using arrow keys when learning `vim` OR avoiding the temptation of using `grep` when learning `jq`.

Mental reminders, like alarms, are often snoozed and seldom revived. They are your shell's new year resolutions.
And it is 1st January everytime you see someone whip out a clever, clean, cli solution and you take that
dreaded vow again - "I **will** master tool X." You throw away your mouse, camp in the wild woods of the cli and learn
some survival skills. But life happens - projects change, roles change, governments change (well...) and you lose track. You still
find solace in the fact that at least what you learnt is preserved in your shell's history. Time ticks away.

One day you get stuck in that similar situation and you do a Control-R (Command-R for the Darwinians?) and try to
find that thing you did to get that pattern. Best case scenario - you get the command but don't fully understand
why you did what you did to customize it for the current problem. Worst case scenario - that command is lost because your shell's
history file had a limit of 10,000 commands and as luck would have it, today was that rollover day.

**Why am I ranting about all of this?** Because I feel that in today's world of serverless and yaml-driven-development, the
opportunities to get better at the command line are fewer and far in between. Couple that with the above strokes of bad
luck, I dread that young engineers may not see solid enough payoffs to keep getting better at it. I just wrote - young
engineers. Self-realization: might want to delete "learning parkour while surfing with the sharks" from that todo list.

**What do I intend to do to fix this?** I will, in my limited capacity, try to blog about hacky one liners, simple docker-compose
setups, functional shell scripts which do the job and nothing more. In the process, if I can help you get at least 10% better
at loving your command line tools, I would have accomplished my goal.

**What if I this is just another high-on-a-thought mid-year resolution?** Fair point. If I can't build up a good enough list of posts in
the next 6 months, I will hold myself fully accountable and do the right thing - I will delete this post. No evidence. No crime. :)

Here's my first instalment:

**Problem:**

Dump EC2 instances of a region having `tag=Name` in a nicely formatted cli table.

**Sample output:**

{% highlight text %}
+----+--------------+-----------------------+----------------------------------------+
| h0 |      h1      |           h2          |                   h3                   |
+----+--------------+-----------------------+----------------------------------------+
| 1  | "us-east-1a" | "i-a23456789abcdefgh" |             "server1"                  |
| 2  | "us-east-1a" | "i-b23456789abcdefgh" |             "server2"                  |
| 3  | "us-east-1a" | "i-c23456789abcdefgh" |             "server3"                  |
| 4  | "us-east-1a" | "i-d23456789abcdefgh" |             "server4"                  |
| 5  | "us-east-1a" | "i-e23456789abcdefgh" |             "server5"                  |
+----+--------------+-----------------------+----------------------------------------+
{% endhighlight %}

**Solution:**

  {% highlight text %}
  aws ec2 describe-instances | \
  jq -rc '.Reservations[].Instances[]  [.Placement.AvailabilityZone, .InstanceId, (.Tags[] | select(.Key=="Name") | .Value)] | @csv' |\
   TABLE_ADD_HEADER=0 TABLE_ADD_SR_NO=1 csv2table
  {% endhighlight %}

**Key takeways:**

- `jq` is powerful.
- Check [csv2table](https://github.com/saurabh-hirani/bin/blob/master/csv2table) - good enough to convert csv to table using the
  very awesome [PrettyTable](https://pypi.org/project/prettytable/]) Python module.
- Use `--filters` flag to reduce the output to running instances only e.g. `aws ec2 describe-instances --filters 'Name=instance-state-name,Values=running'`
- We can also use aws command line [--query](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html) param to filter on the server
  side, but here's the catch - the `--query` param takes a [jmespath](https://jmespath.org/) query. Although, I started with using jmespath due
  to it's availability as both a cli and a programming language module, I had to swich gears to [jq](https://stedolan.github.io/jq/) because most
  of json query google searches yielded `jq`. Not an ideal excuse, but one life is too short to learn both `jmespath` and `jq`.

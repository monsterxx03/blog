---
title: "Handle outage"
date: 2017-12-10T11:13:53+08:00
categories:
  - Tech
tags:
  - lang-en
  - server-infra
---

A few weeks ago, production environment came to an outage, solve it cost me 8 hours (from 3am to 11am) although total down time is not long, really a bad expenrience. Finally, impact was mitigated, and I'm working on a long term solution. I learned some important things from this accident.

## The outage

I received alarms about live performance issue at 3am, first is server latency increaing, soon some service's health check failed due to high load.

I did following:

1. Check monitor
2. Identify the problem is caused by KV system

Okay, problem is here, I know the problem is KV system's performance issue. But I can't figure out the root case right now, I need a temporary solution.
Straightward way is redirect traffic to slave instance. But I know it won't work (actually it is true), I come to similar issue before, did a fix for it, but seems it doesn't work. 

The real down time was not long, performance recovered to some degree soon, but latency was still high, not normal. I monitored it for long time, and tried to find out the root case until morning. Since traffic was growing when peak hour coming, performance became problem again.

I decided to upgrade the server to a large tier with more cpu cores, since the problem is caused by high cpu usage. After it, latency back to normal, at least users won't feel anyting unusual, but KV server's cpu usage is still high, so the problem needs more inspectation.

In the following working day, I worked on investigating the root case. And finally figured out the problem is caused by a lot of keys in KV expired at the same time. And the underlying storage engine can't hanle this condition well. I'm working on migrate KV system to another one(AWS DynamoDB).

## Review

There is something I did wrong when reviewing this accident.

Our campany's original KV system is hosted by redis (with AOF enabled), but the total size is limited by RAM, and data size grows quickly, can't fit in RAM anymore. So I need to migrate to a disk based choice, and I perfer to migrate a system compatible with redis protocol, then I needn't change live code and migrating data will be simple.

After investigating several choices, I choosed current one, since its compatibility is best. I came to several bugs when testing, reported it to author,he fixed it very quickly (really appreciated). The reason why I didn't migrate to DynamoDB is also migration effort is big, although I must do it now. When running on live, performance is great, but I came to crash issue sometime, the author also fixed it. Looking back today, I feel it's too bugy, and I should consider migratition earlier. The codebase is too large and I'm not familiar with C++, every time came to bug, I can't figure out the root case indenpendently.

When reviewing my process of handling outage, I'm lack of mitigation solution, most thing I can do is just waiting and monitoring.

## Conclusion

When choosing core technology, we should be more careful, not only considering the migration effort, but also backup, recovering, monitor and outage handling.

So the right process if handling outage is:

1. Notice the problem (receive alerm).
2. Reponse to it quick. (Let team know I'm handling it)
3. Migitation, make business back to normal. (Maybe fail over to slave)
4. Cleanup and fix previous workaround.
5. Postmortem.

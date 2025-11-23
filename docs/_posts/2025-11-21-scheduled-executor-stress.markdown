---
layout: post
title:  "Putting ScheduledExecutorService under Stress"
date:   2025-11-21
categories: Java Concurrency
excerpt: "Putting ScheduledExecutorService under Stress"
---

In this blog post I want to closely look into the behaviour exhibited by standard ScheduledExecutorService implementations shipped
with Java 25 when faced with impossible, or highly challenging demands. More specifically, I want to discuss what happens if
* `ScheduledExecutorService.scheduleAtFixedRate` is called with a runnable, that occasionally runs longer than the configured 
  rate.
* or scheduled and immediate tasks have to compete with other tasks for spare resources.

Apart from that I want to briefly discuss a recent change in that area: Since JDK 25, `ScheduledThreadPoolExecutor`
is no longer the only implementation of `ScheduledExecutorService` shipped with standard Java, as [ForkJoinPool has been 
updated to implement the interface as well](https://bugs.openjdk.org/browse/JDK-8362881).

Last but not least, I want to look into timed waiting, or put differently: How long does `sleep(100)` really sleep?

Let's start with the simplest of the aforementioned edge cases:

## ScheduledExecutorService.scheduleAtFixedRate: When desire meets reality

According to the [documentation](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html#scheduleAtFixedRate(java.lang.Runnable,long,long,java.util.concurrent.TimeUnit)), 
`ScheduledExecutorService.scheduleAtFixedRate`

> submits a periodic action that becomes enabled first after the given initial delay, and subsequently with the given period; 
> that is, executions will commence after initialDelay, then initialDelay + period, then initialDelay + 2 * period, and so on.

But what if the submitted runnable takes longer to execute than period? This case is covered by the documentation as well:

> If any execution of this task takes longer than its period, then subsequent executions may start late, 
> but will not concurrently execute.


![Behaviour if task runs for 3 periods](/assets/drawings/2025-11-21-schedule-at-fixed-rate-overflow.drawio.png)






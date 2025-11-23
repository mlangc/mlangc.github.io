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
updated to implement ScheduledExecutorService as well](https://bugs.openjdk.org/browse/JDK-8362881).

Last but not least, I want to look into timed waiting, or put differently: How long does `sleep(100)` really sleep?

Let's start with the simplest of the aforementioned edge cases:

## ScheduledExecutorService.scheduleAtFixedRate: When desire meets reality

According to the [documentation](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html#scheduleAtFixedRate(java.lang.Runnable,long,long,java.util.concurrent.TimeUnit)), 
`ScheduledExecutorService.scheduleAtFixedRate`

> submits a periodic action that becomes enabled first after the given initial delay, and subsequently with the given period; 
> that is, executions will commence after initialDelay, then initialDelay + period, then initialDelay + 2 * period, and so on.

But what if the submitted runnable takes longer to execute than period? This case is covered by the documentation, though it's
a bit vague about the exact behaviour:

> If any execution of this task takes longer than its period, then subsequent executions may start late, 
> but will not concurrently execute.

To figure out what the implementation actually does, I created a small experiment, where the running time of a periodically
submitted task exceeds the `period` in `ScheduledExecutorService.scheduleAtFixedRate` by a factor of 3 one time, and then falls 
back to runtimes very much below the `period`.

Here is what I got, visually summarized:

![overflowing-task-runs-for-3-periods](/assets/drawings/2025-11-21-schedule-at-fixed-rate-overflow.drawio.png)

Something else can be noticed when looking at the actual numbers that the experiment prints to the console (note that
`1ms = 1_000_000ns`). The executions are always a few milliseconds late:
```text
Executing schedule 000 at   510,239,350,875ns, which is         2,030,375ns too late
Executing schedule 001 at   510,546,730,833ns, which is       209,410,333ns too late
Executing schedule 002 at   510,557,060,458ns, which is       119,739,958ns too late
Executing schedule 003 at   510,568,988,208ns, which is        31,667,708ns too late
Executing schedule 004 at   510,640,559,791ns, which is         3,239,291ns too late
Executing schedule 005 at   510,744,208,250ns, which is         6,887,750ns too late
```

## ScheduledExecutorService.submit getting in the way of scheduled tasks






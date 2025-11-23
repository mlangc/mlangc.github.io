---
layout: post
title: "Putting ScheduledExecutorService under Stress"
date: 2025-11-21
categories: Java Concurrency
excerpt: "Putting ScheduledExecutorService under Stress"
---

In this blog post I want to closely look into the behaviour exhibited by standard ScheduledExecutorService implementations shipped
with Java 25 when faced with impossible, or highly challenging demands. More specifically, I want to discuss what happens if

* `ScheduledExecutorService.scheduleAtFixedRate` is called with a runnable, that occasionally runs longer than the configured
  rate.
* or scheduled and immediate tasks have to compete with other tasks for spare resources.

Apart from that I want to briefly discuss a recent change in that area: Since JDK 25, `ScheduledThreadPoolExecutor`
is no longer the only implementation of `ScheduledExecutorService` shipped with standard Java,
as [ForkJoinPool has been updated to implement ScheduledExecutorService as well](https://bugs.openjdk.org/browse/JDK-8362881).

Last but not least, I want to look into timed waiting, or put differently: How long does `sleep(100)` really sleep?

Let's start with the simplest of the aforementioned edge cases:

## ScheduledExecutorService.scheduleAtFixedRate: When desire meets reality

According to
the [documentation](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html#scheduleAtFixedRate(java.lang.Runnable,long,long,java.util.concurrent.TimeUnit)),
`ScheduledExecutorService.scheduleAtFixedRate`

> submits a periodic action that becomes enabled first after the given initial delay, and subsequently with the given period;
> that is, executions will commence after initialDelay, then initialDelay + period, then initialDelay + 2 * period, and so on.

But what if the submitted runnable takes longer to execute than period? This case is covered by the documentation, though it's a
bit vague about the exact behaviour:

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

I'll come back to that.

## Directly submitted tasks getting in the way of scheduled tasks

The Javadocs of `ScheduledExecutorService`, which covers both implementations, state:
> Commands submitted using the Executor.execute(Runnable) and ExecutorService submit methods are scheduled with a requested
> delay of zero.

Note the implications: If you submit 10 tasks, `t0, ..., t4`, each running for `200ms`, and then schedule another task `s`, to
be executed with a delay of `500ms`, some compromises are needed, at least for fewer than 6 threads.

Here is a visual summary about what happens in the above scenario, for exactly one thread:
![execute-vs-schedule](/assets/img/2025-11-21-execute-vs-schedule-copilot.png)

As you can see, and conform yourself


Let's briefly discuss how `ScheduledExecutorService` is 
implemented. JDK 25 ships two implementations:
`ForkJoinPool` and `ScheduledThreadPoolExecutor`.

`ScheduledThreadPoolExecutor` has been around for ages, and relies on a custom priority work queue (see
`ScheduledThreadPoolExecutor.DelayedWorkQueue`), where all tasks are submitted to, weather they are delayed or not.

`ForkJoinPool` was extended to implement `ScheduledExecutorService` in JDK 25, as part of an effort to [improve the performance
of delayed task handling](https://bugs.openjdk.org/browse/JDK-8350493). Instead of deeply integrating the scheduling
functionality into `ForkJoinPool`, the actual scheduling is performed by a single `java.util.concurrent.DelayScheduler` thread,
that is started on demand, which maintains a [4 ary heap](https://en.wikipedia.org/wiki/D-ary_heap) based on trigger times.
When its delay expires, the respective task is scheduled to execute on the connected `ForkJoinPool`. Regular task submissions
via `submit` or `execute` completely bypass the delay scheduler.







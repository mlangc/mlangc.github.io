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

Last but not least, I want to look into timed waiting: How long does `sleep(100)` really sleep?







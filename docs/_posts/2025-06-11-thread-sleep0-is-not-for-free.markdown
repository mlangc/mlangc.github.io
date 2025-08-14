---
layout: post
title:  "Thread.sleep(0) is not for free"
date:   2025-08-14 08:52:34 +0200
categories: "Java Performance"
published: true
---
In this short blog post, I want to clear up a common misconception about `java.lang.Thread.sleep`.
Calling `Thread.sleep(0)` is not for free. True, the [official documentation](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/Thread.html#sleep(long))
would allow a fast path for `millis == 0` that only checks the interrupted status of the current
thread to potentially throw an [InterruptedException](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/InterruptedException.html).
The actual implementation however, at first calls into the [native method Thread.sleepNanos0(long nanos)](https://github.com/openjdk/jdk/blob/445e5ecd98f41d4d625af5731f7b5d10c9225e49/src/java.base/share/classes/java/lang/Thread.java#L516)
which is defined in [Thread.c](https://github.com/openjdk/jdk/blob/445e5ecd98f41d4d625af5731f7b5d10c9225e49/src/java.base/share/native/libjava/Thread.c#L42),
and implemented by [JVM_SleepNanos](https://github.com/openjdk/jdk/blob/445e5ecd98f41d4d625af5731f7b5d10c9225e49/src/hotspot/share/prims/jvm.cpp#L2876).
The native code then checks for interrupts and indeed contains a special case for `nanos == 0`, however, it looks like

```cpp
  if (nanos == 0) {
    os::naked_yield();
  } else {
    // ...
  }
```

This means that calling `Thread.sleep(0)` is more or less the same as checking for interrupts, and calling 
[Thread.yield()](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/Thread.html#yield()).
On Linux, `os::naked_yield()` is implemented by the posix function
[sched_yield](https://github.com/openjdk/jdk/blob/445e5ecd98f41d4d625af5731f7b5d10c9225e49/src/hotspot/os/posix/os_posix.cpp#L945),
that clearly states in [its documentation](https://man7.org/linux/man-pages/man2/sched_yield.2.html)

> Avoid calling sched_yield() unnecessarily or inappropriately
> (e.g., when resources needed by other schedulable threads are
> still held by the caller), since doing so will result in
> unnecessary context switches, which will degrade system
> performance.

Indeed, with my local setup, calling `Thread.sleep(0)` is roughly as expensive as calling
[ThreadLocalRandom.nextBytes](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/util/Random.html#nextBytes(byte%5B%5D))
with `byte[128]`, according to [this JMH benchmark](https://github.com/mlangc/java-snippets/blob/refs/heads/thread-sleep0/src/jmh/java/at/mlangc/benchmarks/ExperimentalThreadSleep0Benchmarks.java#L93).
Moreover, `Thread.sleep(0)` is especially expensive when you need it the least, that is when the CPU is already overloaded, since
yielding will more likely result in context switches if there are many threads starving for CPU. Let me demonstrate this using a 
small JMH benchmark:
```java
@Fork(value = 1)
@Warmup(iterations = 3, time = 100, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 5, time = 500, timeUnit = TimeUnit.MILLISECONDS)
@BenchmarkMode(Mode.Throughput)
@State(Scope.Benchmark)
public class ThreadSleep0Benchmark {
    @Param("85")
    private int cpuTokens;

    @Benchmark
    public void burnCpu() {
        Blackhole.consumeCPU(cpuTokens);
    }

    @Benchmark
    public void burnCpuAndSleep0() throws InterruptedException {
        Thread.sleep(0);
        Blackhole.consumeCPU(cpuTokens);
    }
}
```
It contains two methods, both burning the same amount of CPU, however, one of them additionally calls `Thread.sleep(0)`.
I ran both tests on my laptop, with 1, 4, 10 and 20 threads. Microsoft Copilot was so nice to assemble this graph for me 
visualizing the results:

![image showing the impact of sleep0 depending on thread count](/assets/img/2015-06-11-copilot-sleep0-bench-1vs4vs10vs20threads.png)

As you can see `burnCpu` scales more or less linearly, till it plateaus at 10 threads. This makes perfect sense since my
CPU has 4 performance and 6 efficiency cores.

`burnCpuAndSleep0` though exhibits a very different behaviour. With 10 threads, the throughput is only slightly better than with
4 threads, and with 20 threads, it's worse than with 1 thread.

##### Cutting a long story short: 
If you have a piece of performance critical code, that under certain, very unlikely conditions employs `sleep` to
back off, for example after an error, or in an overload situation, replace
```java
int delay = allGood ? 0 : waitShorty;
Thread.sleep(delay);
```
with
```java
if (!allGood) {
    Thread.sleep(waitShortly);
}
```
or use [TimeUnit.sleep](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/TimeUnit.html#sleep(long))
that contains a similar, clearly documented, fast path. If you really rely on the undocumented `yield` implied by 
`Thread.sleep(0)`, better make that explicit, by calling [Thread.yield()](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/Thread.html#yield())
directly.




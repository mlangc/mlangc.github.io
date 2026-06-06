---
layout: post
title: "How to deadlock a Java executor service"
date: 2026-06-05
categories: Java Concurrency
excerpt: "TODO"
---

In this blog post I want to have a close look a type of deadlock you can run into with any Java
[ExecutorService](https://docs.oracle.com/en/java/javase/25/docs/api//java.base/java/util/concurrent/ExecutorService.html)
with a bounded number of threads. While rare in practice, the necessary ingredients to trigger it are always at your fingertips.
It's especially problematic and hard to debug when it affects the globally available
[ForkJoinPool.commonPool](https://docs.oracle.com/en/java/javase/25/docs/api//java.base/java/util/concurrent/ForkJoinPool.html#commonPool())
which is how I ran into this issue recently while working on the test-suite of
[more-log4j2](https://github.com/mlangc/more-log4j2).

### Minimal Reproduction

Let me try to explain the heart of the problem at an abstract level, before showing you how to reproduce it with a few lines of
code. What you need for a minimal reproduction is a single threaded executor service `E` and a task `S` that is scheduled on `E`.
`S` schedules another task `T` on `E`, and waits for its completion synchronously. Since `E` has only one thread that is already
occupied by `S`, `T` can never start, and `S` therefore waits forever. You might find it helpful to have a look at the following
diagram, that captures the scenario I've just described:

![executor-deadlock-minimal-reproduction](/assets/drawings/2026-06-05-minimal-reproduction.drawio.png)

Taking advantage of [Java Compact Source Files](https://openjdk.org/jeps/512), a minimal reproduction based on the 
aforementioned ideas is only 6 lines of code:
```java
void main() {
    var executor = Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory());
    Runnable taskT = () -> { };
    Runnable taskS = () -> CompletableFuture.runAsync(taskT, executor).join();
    CompletableFuture.runAsync(taskS, executor).join();
}
```
Note that `Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory())` is used in favor of
`Executors.newSingleThreadExecutor()` to make sure that executor threads cannot prevent the program from terminating. The only
potential reason for the program to hang is therefore the last line in `main`, that schedules and waits for `taskS`.

If you run this program, you can confirm that it hangs indeed.


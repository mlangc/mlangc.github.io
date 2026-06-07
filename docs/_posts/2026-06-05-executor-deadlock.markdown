---
layout: post
title: "How to Deadlock a Java Executor Service"
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
code. What you need for a minimal reproduction is a single threaded executor service `executor` and a task `taskS` that is
scheduled on `executor`. `taskS` schedules another `taskT` on `executor`, and waits for its completion synchronously. Since
`executor` has only one thread that is already occupied by `taskS`, `taskT` can never start, and `taskS` therefore waits forever.
You might find it helpful to have a look at the following diagram, that captures the scenario I've just described:

![executor-deadlock-minimal-reproduction](/assets/drawings/2026-06-05-minimal-reproduction.drawio.png)

A minimal reproduction based on the aforementioned ideas has only a few of code:
```java
void main() {
    var executor = Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory());
    Runnable taskT = () -> out.println("T done");

    Runnable taskS = () -> {
        CompletableFuture.runAsync(taskT, executor).join();
        out.println("S done");
    };

    CompletableFuture.runAsync(taskS, executor).join();
}
```

Note that `Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory())` is used in favor of
`Executors.newSingleThreadExecutor()` to make sure that executor threads cannot prevent the program from terminating. The only
potential reason for the program to hang is therefore the last line in `main`, that schedules and waits for `taskS`.

If you run this program, you can confirm that it hangs indeed, and all `println` statements are actually dead code.

Before we move on, let's look at two ways to fix the deadlock. The probably most obvious way is to add a thread to the executor,
as in
```java
var executor = Executors.newFixedThreadPool(2, Thread.ofPlatform().daemon().factory());
```
Then `taskT` can execute while `taskS` waits for it as in the following picture:

![executor-deadlock-minimal-reproduction-fixed-with-two-threads](/assets/drawings/2026-06-05-minimal-reproduction-fixed-with-2-threads.png)

However, the implementation is easily adapted to deadlock also with 2 threads:

```java
void main() {
    var executor = Executors.newFixedThreadPool(2, Thread.ofPlatform().daemon().factory());
    IntFunction<Runnable> newTaskT = i -> () -> out.printf("T%s done%n", i);

    IntFunction<Runnable> newTaskS = i -> () -> {
        CompletableFuture.runAsync(newTaskT.apply(i), executor).join();
        out.printf("S%s done%n", i);
    };

    CompletableFuture.allOf(
            CompletableFuture.runAsync(newTaskS.apply(1), executor),
            CompletableFuture.runAsync(newTaskS.apply(2), executor)
    ).join();
}
```

What happens if you execute the code above is captured in the diagram below:

![executor-deadlock-minimal-reproduction-with-2-threads](/assets/drawings/2026-06-05-minimal-reproduction-with-2-threads.drawio.png)

The important thing in order to reproduce the deadlock, is that `S1` and `S2` are started before any of `T1` and `T2` have
a chance to run. I hope that is clear at this point, how we could continue with 3, 4, 5,... threads.

Luckily, there is another, more reliable way to get rid of the deadlock, without adding threads. The code can be unblocked
by letting `taskS` wait for `taskT` *asynchronously*, like in this snippet:

```java
void main() {
    var executor = Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory());
    Runnable taskT = () -> out.println("T done");

    Supplier<CompletableFuture<Void>> taskS = () ->
            CompletableFuture.runAsync(taskT, executor).thenRun(() -> out.println("S done"));

    CompletableFuture.supplyAsync(taskS, executor)
            .thenCompose(f -> f) // Flattens Future<Future<...>> into Future<...>
            .join();
}
```


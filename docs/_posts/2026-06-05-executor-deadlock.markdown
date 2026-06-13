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

This fixes the problem because `taskS` now releases its hold on the executor thread after scheduling `taskT`, and therefore
alows `taskT` to run. As you can confirm by instrumenting our executor with
```java
var executor = new Executor() {
    final Executor impl = Executors.newSingleThreadExecutor(Thread.ofPlatform().daemon().factory());

    @Override
    public void execute(Runnable command) {
        out.printf("execute(%s)%n", command);
        impl.execute(command);
    }
};
```
there are still two `Runnable`s being submitted. The first one, implemented by `CompletableFuture.AsyncSupply`, corresponds
to the first part of `taskS`, that schedules `taskT`. The second one, implemented by `CompletableFuture.AsyncRun` executes
`taskT`, and then the remaining part of `taskS`. Assembled in a diagram, you can picture what is going on as follows:

![executor-deadlock-fix-with-async-ops](/assets/drawings/2026-06-05-fix-with-async-ops.drawio.png)

### Implications and Impact in Practice

The examples I've presented so far are of course artificial. While interesting in theory, nobody and no-thing would ever write
such code in practice, right?

![deadlocking-executors-is-fine-meme](/assets/img/This-is-Fine-Meme.jpg)

Closely looking at the minimal reproduction again
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
we can observe at least the following:
* Many applications use single-threaded, or at least bounded executors.
* Many applications use a mixture of asynchronous and synchronous code.
* Many applications directly or indirectly submit tasks to executors from multiple places.

These points together however are exactly what is needed to run into a real world version of the minimal reproduction above. 
Unlike in our toy example, `taskS` and `taskT` could originate from different source files, modules or even libraries and the 
affected `executor` could be managed by a framework. 

Last but not least: This problem is not limited to `CompletableFuture#join`. A deadlock is possible whenever `taskS` running
on a bounded `executor` synchronously waits for a `taskT` on the same `executor` to do something. If the synchronous waiting is
implemented by `CompletableFuture#join`, `CountDownLatch#await` or `BlockingQueue#put` doesn't matter in this context.

### Case Study of a Real-World Example

Let me add some depth to the discussion by having a closer look at how I ran into this issue recently in a realistic setting
when working on [more-log4j2](https://github.com/mlangc/more-log4j2). Back then at the end of April 2026, I was addressing
bugs pointed out by [Claude Sonnet 4.6](https://www.anthropic.com/news/claude-sonnet-4-6). To make sure that I fixed them 
properly, I
[added few tests](https://github.com/mlangc/more-log4j2/commit/7acd211#diff-da6eb56b57edce2e06ec766c3a5d83b19b841f5b615947e52ab710d2e92f0071R1851)
that all passed locally. However, when I pushed these changes a few hours later, togehter with other commits, the 
[CI builds started hanging and timing out after 6 hours](https://github.com/mlangc/more-log4j2/actions/runs/24911662871).

It took me some time to figure out what was going on exactly. Luckily I did not follow some reasonable sounding, but misleading 
explanations provided by Claude Opus 4.7, since I anyway wanted to find out what was going on for myself.

Here is brief summary: The problematic test `AsyncHttpAppenderTest#shouldRespectMaxBatchBytesIncludingSeparatorsIfOverloaded`
added with [7acd211491b6ab46b78b4215a9a70d8d7d942eb4](https://github.com/mlangc/more-log4j2/commit/7acd211491b6ab46b78b4215a9a70d8d7d942eb4)
instantiates a log4j2 configuration that contains an [AsyncHttpAppender](https://github.com/mlangc/more-log4j2#Async-HttpAppender)
that is configured to block practically indefinitely when it can't keep up with the incoming logs. This appender is than hammered
via

```java
var logsPerJob = 1000;
var numJobs = 4;
// ...
var jobs = IntStream.range(0, numJobs)
    .mapToObj(ignore -> CompletableFuture.runAsync(() -> {
        for (int i = 0; i < logsPerJob; i++) {
            log.info("x");
        }
    })).toArray(CompletableFuture<?>[]::new);

assertThat(CompletableFuture.allOf(jobs)).succeedsWithin(1, TimeUnit.SECONDS);
// ...
```
As mentioned, this test passed without issues locally, however it hang completely on the CI server. After some experimentation,
I found out that I can reproduce the hanging locally by setting 
```
-Djava.util.concurrent.ForkJoinPool.common.parallelism=2
```
which restricts the number of threads in the [ForkJoinPool#commonPool](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ForkJoinPool.html#commonPool()).

Now you need to know that the [AsyncHttpAppender](https://github.com/mlangc/more-log4j2#Async-HttpAppender) uses a 
Semaphore to keep the number of HTTP requests in flight below a configurable value. By default, at most 5 concurrent HTTP requests
are allowed. Simplified, paraphrased, and stripped down to the part that is relevant for this discussion, the code that is
responsible for sending off batches of logs looks like

```java
final long backoffCapNanos = TimeUnit.SECONDS.toNanos(5);
final Semaphore allowedInFlight = new Semaphore(5);

void drainBufferedBatches(long maxBackoffNs) {
    while (true) {
        var nextBatch = pollNextBatch();

        if (nextBatch == null) {
            break;
        } else if (!allowedInFlight.tryAcquire()) {
            var backoffNs = ThreadLocalRandom.current().nextLong(maxBackoffNs + 1);
            var newMaxBackoffNs = Math.min(maxBackoffNs * 2, backoffCapNanos);
            executor().schedule(() -> drainBufferedBatches(newMaxBackoffNs), backoffNs, TimeUnit.NANOSECONDS);
            break;
        } else {
            sendBatch(nextBatch).whenComplete((_, _) -> allowedInFlight.release());
        }
    }
}
```

`sendBatch(Batch batch)` calls into a `java.net.http.HttpClient` to do the heavy lifting roughly like this:

```java
final HttpClient httpClient;

CompletableFuture<?> sendBatch(Batch batch) {
    return httpClient.sendAsync(buildRequest(batch), HttpResponse.BodyHandlers.ofString());
}

```

At this point, we need to remember that the problematic test hammers the appender, which is configured with
`maxBlockOnOverflowMs=Integer.MAX_VALUE` and an intentionally ridiculously small batch size, using
```java
var jobs = IntStream.range(0, 4)
        .mapToObj(ignore -> CompletableFuture.runAsync(() -> {
            for (int i = 0; i < 1000; i++) {
                log.info("x");
            }
        })).toArray(CompletableFuture<?>[]::new);
```

Since `-Djava.util.concurrent.ForkJoinPool.common.parallelism=2`, this quickly blocks all threads of the common `ForkJoinPool`.
Now you might think that this won't affect the `httpClient`, which is configured with a dedicated, single-threaded executor,
that the library is very careful to never block. However, the implementation of `HttpClientImpl#sendAsync(...)` contains
```java
// makes sure that any dependent actions happen in the CF default
// executor. This is only needed for sendAsync(...), when
// exchangeExecutor is non-null.
return res.isDone() ? res
        : res.whenCompleteAsync((r, t) -> { /* do nothing */}, ASYNC_POOL);
```
were `res` is a `CompletableFuture<HttpResponse<T>>` containing our response. As hinted by the comment, the sole purpose of
`res.whenCompleteAsync((r, t) -> { /* do nothing */}, ASYNC_POOL)` is to make sure that dependent, non-async completions, are not
run on the executor of the `HttpClient`, but on the common `ForkJoinPool`. What are "dependent, non-async completions"?
Well, futures that are returned by `CompletionStage#thenApply`, `CompletionStage#thenRun`, `CompletionStage#thenAccept`,
`CompletionStage#thenCombine` and so on, but not `CompletionStage#thenApplyAsync`, `CompletionStage#thenRunAsync`,
`CompletionStage#thenAccept`, `CompletionStage#thenCombineAsync` and so on. The lambdas you pass to any of these non-async methods
are run by the thread that completed the future these transformations are applied to. This 

```java
CompletableFuture.completedFuture(42)
        .thenRun(() -> out.println("Execucted by caller thread of thenRun"))
```

or they are executed by the thread that completes the dependant

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

The important thing in order to reproduce the deadlock, is that `S1` and `S2` are started before any of `T1` and `T2` have a
chance to run. I hope that is clear at this point, how we could continue with 3, 4, 5,... threads.

Luckily, there is another, more reliable way to get rid of the deadlock, without adding threads. The code can be unblocked by
letting `taskS` wait for `taskT` *asynchronously*, like in this snippet:

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

This fixes the problem because `taskS` now releases its hold on the executor thread after scheduling `taskT`, and therefore alows
`taskT` to run. As you can confirm by instrumenting our executor with

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

there are still two `Runnable`s being submitted. The first one, implemented by `CompletableFuture.AsyncSupply`, corresponds to the
first part of `taskS`, that schedules `taskT`. The second one, implemented by `CompletableFuture.AsyncRun` executes
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

Last but not least: This problem is not limited to `CompletableFuture#join`. A deadlock is possible whenever `taskS` running on a
bounded `executor` synchronously waits for a `taskT` on the same `executor` to do something. If the synchronous waiting is
implemented by `CompletableFuture#join`, `CountDownLatch#await`, `Semaphore#acquire` or `BlockingQueue#put` doesn't matter in 
this context.

### Case Study of a Real-World Example

Let me add some depth to the discussion by having a closer look at how I ran into this issue recently in a realistic setting when
working on [more-log4j2](https://github.com/mlangc/more-log4j2). Back at the end of April 2026, I was addressing bugs pointed
out by [Claude Sonnet 4.6](https://www.anthropic.com/news/claude-sonnet-4-6). To make sure that I fixed them properly, I
[added a few tests](https://github.com/mlangc/more-log4j2/commit/7acd211#diff-da6eb56b57edce2e06ec766c3a5d83b19b841f5b615947e52ab710d2e92f0071R1851)
that all passed locally. However, when I pushed these changes a few hours later, togehter with other commits, the
[CI builds started hanging and timing out after 6 hours](https://github.com/mlangc/more-log4j2/actions/runs/24911662871).

It took me some time to figure out what was going on exactly. Luckily I did not follow some reasonable sounding, but misleading
explanations provided by Claude Opus 4.7, since I anyway wanted to find out what was going on for myself.

Here is brief summary: The problematic test `AsyncHttpAppenderTest#shouldRespectMaxBatchBytesIncludingSeparatorsIfOverloaded`
added
with [7acd211491b6ab46b78b4215a9a70d8d7d942eb4](https://github.com/mlangc/more-log4j2/commit/7acd211491b6ab46b78b4215a9a70d8d7d942eb4)
instantiates a log4j2 configuration that contains an [AsyncHttpAppender](https://github.com/mlangc/more-log4j2#Async-HttpAppender)
that is configured to 
* block practically indefinitely via `maxBlockOnOverflowMs=Integer.MAX_VALUE` when it can't keep up with the incoming logs
* and with an intentionally ridiculously small value for `maxBatchBytes`.

This appender is then hammered via
```java
var jobs = IntStream.range(0, 4)
        .mapToObj(ignore -> CompletableFuture.runAsync(() -> {
            for (int i = 0; i < 1000; i++) {
                log.info("x");
            }
        })).toArray(CompletableFuture<?>[]::new);

assertThat(CompletableFuture.allOf(jobs)).succeedsWithin(1,TimeUnit.SECONDS);
```

As mentioned, this test passed without issues locally, however it hung completely on the CI server. After some experimentation, I
found out that I can reproduce the hanging on my laptop by setting

```
-Djava.util.concurrent.ForkJoinPool.common.parallelism=2
```

which restricts the number of threads in
the [ForkJoinPool#commonPool](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ForkJoinPool.html#commonPool()).

In order to understand what happened and why, a closer look into the implementation of the `AsyncHttpAppender`
is necessary. I'm trying to keep this as high level as possible, focusing only on the parts relevant for our discussion.

#### Relevant AsyncHttpAppender Mechanics

Let's have a look at what happens in the appender when you call `log.info(x)`:
![executor-deadlock-append-x](/assets/drawings/2026-06-05-append-x.drawio.png)

The most important thing about the drawing above is what is doesn't contain: As you can see, the buffer is only appended to, but
not drained. Draining is done asynchronously by a single threaded executor that is shared with the `HttpAppender`
implementation:

![executor-deadlock-drain-buffers](/assets/drawings/2026-06-05-drain-buffers.drawio.png)

This drawing needs more explanation than the last one, so let's go through it step by step:

1. The `AsyncHttpAppender` limits the number of concurrently executing HTTP requests to `5` by default. This is implemented by a
   semaphore, that needs to be acquired before invoking the HTTP client. However, the implementation does not call
   `Semaphore#acquire()`, because that can block. Blocking is anyway something to do very cautiously in this context, since the
   thread we are running on is shared with the HTTP client. In this case it would be fatal though: Since the semaphore is
   only released after the HTTP client returns, blocking the thread would make sure that this never happens, and we would be
   deadlocked immediately. For that reason the implementation uses
   `Semaphore#tryAcquire()` instead, which never blocks. If `tryAcquire` fails, the appender reschedules a drain attempt on the
   executor using an exponential backoff logic, and releases the thread. This is what the diagram colloquially paraphrases as "
   async acquire".
2. Batches that are ready to be sent are kept in a queue, which is polled here. Note that in the actual implementation, the 
   semaphore is only acquired if there is something to poll.
3. This is the step where `java.net.http.HttpClient#sendAsync` is invoked.
4. After `HttpClient#sendAsync()` is done, the semaphore that we acquired previously needs to be released. Simplified, this is 
   done like 
   ```java
   httpClient.sendAsync(buildRequest(batch), HttpResponse.BodyHandlers.ofString())
      .whenComplete((_, _) -> allowedInFlight.release());
   ```

At this point we have discussed everything you need to know about the `AsyncHttpAppender` implementation. To get the full 
picture, we need to have a brief look at a surprising detail of the `HttpClient` implementation as well.

#### How and why java.net.http.HttpClient uses the common ForkJoinPool

I already mentioned that the `AsyncHttpAppender` shares an executor with the used `HttpClient`, that is passed into the client 
at construction time via `HttpClient.Builder#executor`. Therefore, you might be tempted to assume, that the `HttpClient` 
implementation is not using the common `ForkJoinPool` at all. That's at least what I thought - until the aforementioned 
problem led my attention to `HttpClientImpl#sendAsync`. In there you can find

```java
// makes sure that any dependent actions happen in the CF default
// executor. This is only needed for sendAsync(...), when
// exchangeExecutor is non-null.
return res.isDone() ? res
        : res.whenCompleteAsync((r, t) ->{ /* do nothing */}, ASYNC_POOL);
```

were `res` is a `CompletableFuture<HttpResponse<T>>` containing our response and `ASYNC_POOL` points to the common `ForkJoinPool`.
As hinted by the comment, the sole purpose of

```
res.whenCompleteAsync((r, t) -> { /* do nothing */}, ASYNC_POOL)
```
is to make sure that dependent, non-async completions, are not
run on the executor of the `HttpClient`, but on the common `ForkJoinPool`. What are "dependent, non-async completions"? Well,
futures that are returned by 
* `CompletionStage#thenApply`, 
* `CompletionStage#thenRun`, 
* `CompletionStage#thenAccept`,
* `CompletionStage#thenCombine` and so on

but not

* `CompletionStage#thenApplyAsync`,
* `CompletionStage#thenRunAsync`,
* `CompletionStage#thenAccept`, 
* `CompletionStage#thenCombineAsync` and so on. 

The lambdas you pass to any of these non-async methods are run 
* either by the thread that completes the future these transformations are depending on
* or directly by the caller of `thenApply`, `thenRun`, ... if invoked on futures that are already completed.

Thus,
```
res.whenCompleteAsync((r, t) -> { /* do nothing */}, ASYNC_POOL)
```

ensures that the future returned to callers is completed by the common `ForkJoinPool`. This is a defensive measure against users
of the library that chain heavyweight operations to the returned future, which could starve the executor of the
`HttpClient`, and severely degrade its functionality.

In my opinion, it would be nice to give users a way to intervene, by allowing them to optionally configure a second executor 
in addition to `HttpClient.Builder#executor`, that would then be used instead of `ASYNC_POOL`, but that's another topic and 
probably not best addressed in this blog.

#### Connecting the Dots

Finally, we have all we need to understand why the test hangs with
```
-Djava.util.concurrent.ForkJoinPool.common.parallelism=2
```
but runs fine with the default number of threads 
```java
Math.max(1, Runtime.getRuntime().availableProcessors() - 1)
```

on typical consumer hardware. What we are seeing is exactly the kind of executor deadlock I've explained in isolation in the first
part of this blog post. Since

```java
var jobs = IntStream.range(0, 4)
        .mapToObj(ignore -> CompletableFuture.runAsync(() -> {
           for (int i = 0; i < 1000; i++) {
              log.info("x");
           }
        })).toArray(CompletableFuture<?>[]::new);
```

deliberately overloads the appender that is configured to block if it can't keep up, both threads in the common `ForkJoinPool`,
that is used by `CompletableFuture#runAsync` if no executor is provided, end up being blocked. At the same time, the previously
discussed semaphore, that limits the number of concurrent HTTP requests, runs out of permits. These permits are never released,
because their release is chained after

```java
res.whenCompleteAsync((r, t) -> { /* do nothing */}, ASYNC_POOL)
```

which is not executed, since all threads in the common pool are blocked. You can picture it like this:

![executor-deadlock-waiting-circle](/assets/drawings/2026-06-05-waiting-circle.drawio.png)

If the number of threads in the common `ForkJoin` pool is greater than 4, the test passes without issues, since then there are 
threads that can pick up the work chained after the completion of the HTTP request, and release the semaphore.

To fix the tests, I therefore [adapted them](https://github.com/mlangc/more-log4j2/commit/7057301b1e34abe2a26aaff554a4fb2efb74bfdc)
to run potentially blocking tasks in another, test specific exeuctor.

### Final Remarks and Recommendations

I hope that this discussion gave you a sense of the problem, both in its abstract form and how it might play out in realistic
code. Though it's hard to come up with a simple guideline that will reliably prevent it, here are two recommendations that should
make it less likely at least:

* The common `ForkJoinPool` is a shared resource, that might be used in unexpected ways. Make sure you don't overload it, and 
  use a dedicated executor if in doubt.
* Avoid synchronously waiting for something that can only be completed by a task in the same, bounded executor. 

The best remedy though is a profound understanding of the problem, since that will help you both with preventing and debugging 
in the case this hits you.




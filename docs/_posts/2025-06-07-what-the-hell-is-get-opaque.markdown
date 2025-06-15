---
layout: post
title:  "What the Hell is GetOpaque in Java"
date:   2025-05-25 08:52:34 +0200
categories: "Java Concurrency"
---
In this blog post, that can be seen as a follow-up of my last post acquire release semantics, I want to shine some light
on [getOpaque](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/invoke/VarHandle.html#getOpaque(java.lang.Object...))
and [setOpaque](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/invoke/VarHandle.html#setOpaque(java.lang.Object...)).

I'll show how it is defined and what that means in practice, as well as how `get` and `setOpaque` relate
to the weaker `get` and `setPlain` and the stronger `getAcquire` and `setRelease`.

## Javadocs and their Interpretation
Unfortunately, the [Javadocs](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/invoke/VarHandle.html#getOpaque(java.lang.Object...))
of `get` and `setOpaque` are somewhat vague, stating

> public final Object getOpaque(Object... args)
> 
> Returns the value of a variable, accessed in program order, but with no assurance of memory ordering effects with respect to other threads. 

> public final void setOpaque(Object... args)
> 
> Sets the value of a variable to the newValue, in program order, but with no assurance of memory ordering effects with respect to other threads.

Rephrasing this, it means that the 
runtime or hardware must not reorder opaque operations with other opaque operations on the same variable. 
It is however perfectly legal to reorder opaque operations with other opaque operations on different variables.

Let's see what this actually means in the [following example](https://github.com/mlangc/java-snippets/blob/refs/heads/blog-2025-06-get-opaque/src/main/java/at/mlangc/concurrent/get/opaque/ObservedSequenceRace.java#L133):
```java
  static class SetGetRace {
      private final MemoryOrdering memoryOrdering;
      private final AtomicInteger a;
      private final AtomicInteger b;

      SetGetRace(MemoryOrdering memoryOrdering, AtomicInteger a, AtomicInteger b) {
          this.memoryOrdering = memoryOrdering;
          this.a = a;
          this.b = b;
      }

      RaceResult run() {
          a.setPlain(0);
          b.setPlain(0);

          var setJob = CompletableFuture.runAsync(() -> {
              memoryOrdering.set(a, 1);
              memoryOrdering.set(a, 2);

              memoryOrdering.set(b, 1);
              memoryOrdering.set(b, 2);
          });

          var getJob = CompletableFuture.supplyAsync(() -> {
              // Note that a is written before b,
              // but b is read before reading a.
              var b1 = memoryOrdering.get(b);
              var a1 = memoryOrdering.get(a);
              var b2 = memoryOrdering.get(b);
              var a2 = memoryOrdering.get(a);

              return new RaceResult(b1, a1, b2, a2);
          });

          setJob.join();
          return getJob.join();
      }
  }
  
  record RaceResult(int b1, int a1, int b2, int a2) {
      boolean interesting() {
          return b1 > a1 || b2 > a2;
      }

      boolean spectacular() {
          return a1 > a2 || b1 > b2;
      }
  }
```
Can we observe "interesting" (in the sense of `RaceResult#interesting`), or even "spectacular" (in the sense of
`RaceResult#spectacular`) results? The answer is that "interesting" results, where the writes to `b` are
observed before the writes to `a` are indeed permitted, and in fact observable on ARM systems, because `a`
and `b` are different atomics. "Spectacular" results can be ruled out though, because individual variables
must be accessed in program order. Note that interesting results are not permitted with 
`MemoryOrdering.ACQUIRE_RELEASE` or `MemoryOrdering.VOLATILE`, since then observing `b > 0` implies
a happens-before-relationship with `a = 2`, as I explain at great lengths in my [previous blog post](https://mlangc.github.io/java/concurrency/2025/05/25/volatile-vs-acq-rel.html) 
about getAcquire and setRelease.

### Additional Documentation
Unfortunately, I could not find any official documentation apart from the Javadocs about `getOpaque` and 
`setOpaque`. There is however a very well written, and comprehensive guide about 
[JDK 9 Memory Order Modes](https://gee.cs.oswego.edu/dl/html/j9mm.html) written by [Doug Lea](https://gee.cs.oswego.edu/dl/)
who is the official author of most classes in `java.util.concurrent`, that contains a paragraph on
"opaque mode". According to this document opaque mode is usable in the same contexts as 
[C++ atomic memory_order_relaxed](https://cppreference.com/w/cpp/atomic/memory_order.html#Relaxed_ordering), which is the same as
[relaxed ordering in Rust](https://marabos.nl/atomics/memory-ordering.html#relaxed).


### Use Cases for get and setOpaque
Opaque mode can be used whenever you want to pass and update to a single value to another thread. This includes
primitive types, like `boolean`, `int`, `long` as well as references to immutable objects.
Publishing references to mutable objects using `get` and `setOpaque` however is unsafe, since reading
a reference with `getOpaque` that has been published with `setOpaque` does not establish a happens-before-relationship.
This means that the thread reading the object reference with `getOpaque` might see a partially constructed object.

Having said that, let's talk about valid use cases, where opaque mode can be safely used instead of volatile mode.

#### Broadcasting a Stop Signal
Let's assume that you have one or more worker threads running code like
```java
while (!stopSignalRecieved()) {
    doSomeWork();
}
```
and one thread that at some point should stop the workers. Then opaque mode is an option, like in
```java
final AtomicBoolean stop = new AtomicBoolean();

void startWorkerLoop() {
    while (!stop.getOpaque()) {
        doSomeWork();
    }

    out.println("Stopped");
}

void sendStopSignal() {
    stop.setOpaque(true);
}
```
Does it buy you something over just using `volatile boolean`, or `AtomicBoolean.get` and `AtomicBoolean.set`?
Probably not. On X86, reads are compiled down to a single `mov` instruction, regardless of their type. On ARM,
plain and opaque reads are compiled to `ldr` instructions, whereas acquire and volatile reads are implemented using
[ldar instructions](https://developer.arm.com/documentation/102336/0100/Load-Acquire-and-Store-Release-instructions).
The minimal differences I could observe in the benchmarks here and here when comparing very tight worker loops
seem to be more the result of JIT implementation details than anything meaningful. Summarizing, using a `volatile boolean`
to implement a stop signal not only makes your code more readable, it might even be slightly faster than using opaque mode
because JIT implementers probably tend to think a lot more about efficient ways to implement `volatile` than about 
opaque mode.

If you read the last paragraph carefully, you might wounder why we couldn't use plain reads and writes, that is
plain mode, at least on X86 and ARM, to broadcast a stop signal. After all, the CPU instructions used for opaque access
on ARM and X86 are ordinary reads. However, CPU instructions only matter if they are executed, which might not be
the case in plain mode, since JIT will optimize the check for the stop flag away if it can prove that the loop body
never modifies it. This proof typically succeeds if the loop body can be fully inlined. Let
me illustrate this point by two examples:
```java
static long brokenBroadcast() throws InterruptedException {
    var stop = new AtomicBoolean();
    var job = CompletableFuture.supplyAsync(() -> {
        var spins = 0L;
        while (!stop.getPlain()) {
            spins++;
        }

        return spins;
    });

    Thread.sleep(100);
    stop.setPlain(true);
    return job.join();
}

public static void main() throws InterruptedException {
    var spins = brokenBroadcast();
    out.printf("Done after %s spins%n", spins);
}
```
This program will almost certainly never terminate, because the check for the stop flag will be optimized away
by the JIT compiler. This variation however probably will:
```java
static long brokenButAccidentallyWorkingBroadcast() throws InterruptedException {
    var stop = new AtomicBoolean();
    var job = CompletableFuture.supplyAsync(() -> {
        var spins = 0L;
        while (!stop.getPlain()) {
            if (++spins % 1_000_000_000 == 0) {
                out.printf("%s spins already...%n", spins);
            }
        }

        return spins;
    });

    Thread.sleep(100);
    stop.setPlain(true);
    return job.join();
}

public static void main() throws InterruptedException {
    var spins = brokenButAccidentallyWorkingBroadcast();
    out.printf("Done after %s spins%n", spins);
}
```
For me this reliably prints
```text
1000000000 spins already...
Done after 1000000000 spins
```
Am I always that lucky to terminate after exactly 1000000000 spins? Let's look into the generated assembler
code to understand what is happening here (how to do this would be worth a blog post of its own; for the time being
please have a look at the [Developers disassemble! Use Java and hsdis to see it all](https://blogs.oracle.com/javamagazine/post/java-hotspot-hsdis-disassembler)
blog post and learn about [-XX:CompileCommand=print](https://docs.oracle.com/en/java/javase/24/docs/specs/man/java.html#advanced-jit-compiler-options-for-java)):
```nasm
; 
; the loop starts here:
;   r9 corresponds to "spins" in Java
;
0x0000025c321d4300:   mov    %r9,%rbx
0x0000025c321d4303:   mov    0x30(%r15),%r8
;
; increments r9 (=spins), and then does some magic to calculate % 1_000_000_000
; without a div instruction, since div is slow. See 
; https://ridiculousfish.com/blog/posts/labor-of-division-episode-i.html if you are interested.
;
0x0000025c321d4307:   lea    0x1(%rbx),%r9
0x0000025c321d430b:   movabs $0x112e0be826d694b3,%rax
0x0000025c321d4315:   imul   %r9
0x0000025c321d4318:   mov    %r9,%r10
0x0000025c321d431b:   sar    $0x3f,%r10
0x0000025c321d431f:   sar    $0x1a,%rdx
0x0000025c321d4323:   sub    %r10,%rdx
0x0000025c321d4326:   imul   $0x3b9aca00,%rdx,%r10
0x0000025c321d432d:   sub    %r10,%rbx
0x0000025c321d4330:   add    $0x1,%rbx
0x0000025c321d4334:   cmp    %r11,%rbx
0x0000025c321d4337:   mov    $0xffffffff,%r10d
0x0000025c321d433d:   jl     0x0000025c321d4347
0x0000025c321d433f:   setne  %r10b
0x0000025c321d4343:   movzbl %r10b,%r10d                  ; ImmutableOopMap {rdi=Oop }
                                                          ;*ifne {reexecute=1 rethrow=0 return_oop=0}
                                                          ; - (reexecute) at.mlangc.concurrent.get.opaque.BroadcastStopExample::lambda$brokenButAccidentallyWorkingBroadcast$0@20 (line 48)
0x0000025c321d4347:   test   %eax,(%r8)                   ;   {poll}
0x0000025c321d434a:   test   %rbx,%rbx
;
; at this point, rbx contains spins % 1_000_000_000; jump back to the start of the 
; loop unless 0, or we are at a safepoint (test directly above)
;
0x0000025c321d434d:   jne    0x0000025c321d4300

0x0000025c321d434f:   mov    $0xffffff4d,%edx
0x0000025c321d4354:   mov    %rdi,%rbp
0x0000025c321d4357:   mov    %r9,0x20(%rsp)
0x0000025c321d435c:   mov    %r10d,0x28(%rsp)
0x0000025c321d4361:   xchg   %ax,%ax
;
; we are going back to interpreted mode since the body of the if in the loop
; was not compiled.
;
0x0000025c321d4363:   call   0x0000025c31bb4f60           ; ImmutableOopMap {rbp=Oop }
                                                          ;*ifne {reexecute=1 rethrow=0 return_oop=0}
                                                          ; - (reexecute) at.mlangc.concurrent.get.opaque.BroadcastStopExample::lambda$brokenButAccidentallyWorkingBroadcast$0@20 (line 48)
                                                          ;   {runtime_call UncommonTrapBlob}
0x0000025c321d4368:   nopl   0x10001e0(%rax,%rax,1)       ;   {post_call_nop}
```
Let's summarize the important parts:
* The generated code does not check the `stop` flag at all.
* As soon as it hits the if block with the `printf`, it goes back to the interpreter.

The interpreter then invokes `printf`, and checks the `stop` flag, which it this point is
already `true`. Thus the loop terminates after exactly `1_000_000_000` iterations.

You might wounder if JIT isn't going a bit over the top when simply removing the check for the
stop flag. However, in most cases, this kind of optimization is exactly what you want, and far more
common than you might initially think. Let me give you an example: Every time you iterate over an 
`ArrayList` in a tight loop, using ```for (var elem : arrayList)```, you would have to pay for
```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
in each iteration. This would add a significant performance penalty if JIT wasn't
allowed to optimize this check away as long as it can prove that the loop body doesn't modify the list.
The [Java Memory Model](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4)
has been specifically designed to allow this kind of optimizations. The only reliable way to make
sure that updates from one thread to a variable are eventually picked up by other thread without any
additional synchronisation, is to use opaque, or a stronger memory ordering mode, like volatile, for
both reads and writes.

#### Broadcasting Progress
Another legitimate use case for opaque mode






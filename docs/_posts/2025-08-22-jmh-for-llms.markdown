---
layout: post
title: "JMH for LLMs"
date: 2025-08-22 07:08:09 +0200
categories: "Java Performance"
published: false
---

## The Idea

I recently had a conversation about Java casting performance with Microsoft Copilot, mostly to test the model, but also open to
learning something new. Very soon, Copilot suggested to generate a JMH benchmark for me, and being curious, I agreed. What
followed was very interesting, because it became clear pretty quickly, that I had overstretched the capabilities of the model
significantly. This inspired the following idea: While JMH benchmarks are typically used to measure runtimes of Java code
snippets, why not use it differently, by comparing the ability of different LLMs to create JMH benchmarks to answer subtle
performance related questions.

## The Rules

Thus, I decided to run a competition between different models, which me being the referee. The conversation with each model would
start with the following prompt:

> I know that casts in Java are pretty cheap, since JVM implementations go at great lengths to make them as performant
> as possible. Still, even cheap operations might add up, if executed in large enough numbers. Given the fact that Java
> collections are operating on plain `Object` references, with implicit casts being inserted by the compiler as needed,
> typical Java programs contain far more casts than meets the eye. So, to make this more concrete, I'm wondering how much casting
> overhead is involved when iterating over an `ArrayList<Integer>`. Is it always negligible, or can it have a significant impact
> in tight loops?
>
> Please don't answer my questions directly: Instead, I would like you to generate a JMH benchmark class, that shines some light
> on this topic. The benchmark should be small and finish in less than 1 minute. Adding some comments to the source code as to
> how to interpret the results would be a bonus, but please try to be verbose. Also, please restrict your answer to the JMH
> benchmark class and note that I don't need instructions about how to run it. I will execute the benchmark, and send you the
> results, so that we can look into it together.

I would then run the benchmark, and respond with

> Thanks, here are the results of the benchmark:
>
> ${JMH results table}
>
> Would you like to refine the benchmark? If you think that the numbers make sense,
> and the benchmark should stay as is, please just answer with no. Otherwise, please restrict your answer to an updated
> JMH benchmark class that I should run instead.

After that I would end the conversation, and discuss potential problems in the generated benchmarks. Last but not least, I will
give a score out of `{0, 1, 2}` with

* `0` meaning useless, or even misleading
* `1` meaning OK, though there is room for improvement
* `2` meaning very good.

This score will be mostly affected by the second attempt of the model, if there is one, since this iterative approach also most
closely resembles how humans write benchmarks. I could of course give the models multiple chances to refine their benchmarks, but
I decided to stop after one iteration, since I'm doing this alone, and want to limit the scope of this article and the amount of
benchmark code to discuss.

## The Participants

The next problem choosing the participants. Again, due to resource constraints, I want to limit myself to 3 models, thus it's very
likely that your favorite LLM is not in the list. It should however be straight forward enough to replicate what I just did for
any model, at least if you don't wait for too long, because it's only a matter of time till models will learn about this article
in one way or another. So without further ado, here are the participants:

* Microsoft Copilot using `Smart (GPT-5)` mode
* Google Gemini 2.5 Pro
* deepseek V3 using `DeepThink` mode

## The Competition

### Microsoft Copilot

### Google Gemini 2.5 Pro

### deepseek V3

## My Benchmark to answer the Prompt

Before drawing any conclusions about the strengths and limits of current LLMs, let me give you my take on an answer to the
aforementioned prompt. Please note that I'm not pretending to be participant, not only because I'm also acting as the referee, but
also since I spent hours digging though benchmark results, profiler output and generated assembler code, before wrapping up my
results, and presenting them to you. So without further ado, here is the benchmark class that I finally came up with after quite a
few iterations of looking at results and verifying assumptions by checking the generated JIT assembler code:

```java
/**
 * This benchmark demonstrates how casting overhead can impact loops iterating over lists.
 *
 * <p>
 * All benchmark methods are prefixed with `groupX`. Always look at results from one group at a time.
 * Comparing results from different groups doesn't make sense if you are interested in the overhead
 * incurred by implicit casts. Here is a brief summary of the different groups:
 *
 *     <ul>
 *         <li>group0 compares reading a reference to reading and an implicit cast</li>
 *         <li>group1 compares copying a reference to copying and an implicit cast</li>
 *         <li>group2 compares calling hashCode to calling hashCode and an implicit cast</li>
 *         <li>group3 does the same as group2, but iterates over <code>Object[]</code> and <code>Integer[]</code></li>
 *     </ul>
 * </p>
 */
@Fork(1)
@Warmup(iterations = 3, time = 100, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 10, time = 200, timeUnit = TimeUnit.MILLISECONDS)
@BenchmarkMode(Mode.Throughput)
@State(Scope.Thread)
public class ListCastingOverheadBenchmark {
    @Param("100000")
    private int length;

    private ArrayList<Integer> integerArrayList;
    private ArrayList<Object> objectArrayList;
    private ArraySink arraySink;

    private Object[] objectArray;
    private Integer[] integerArray;

    @Setup
    public void setup() {
        integerArrayList = new ArrayList<>(length);
        objectArrayList = new ArrayList<>(length);
        arraySink = new ArraySink(length);
        objectArray = new Object[length];
        integerArray = new Integer[length];

        var rng = new Random(123);
        for (int i = 0; i < length; i++) {
            integerArrayList.add(rng.nextInt());
            objectArrayList.add(integerArrayList.getLast());
            objectArray[i] = integerArrayList.getLast();
            integerArray[i] = integerArrayList.getLast();
        }
    }

    @Benchmark
    public void group0ConsumeObjectArrayList(Blackhole bh) {
        for (Object o : objectArrayList) {
            bh.consume(o);
        }
    }

    @Benchmark
    public void group0ConsumeIntegerArrayList(Blackhole bh) {
        for (Integer i : integerArrayList) {
            bh.consume(i);
        }
    }


    @Benchmark
    public ArraySink group1CopyReferencesIntegerArrayList() {
        arraySink.reset();
        for (Integer i : integerArrayList) {
            arraySink.consumeInteger(i);
        }

        return arraySink;
    }

    @Benchmark
    public ArraySink group1CopyReferencesObjectArrayList() {
        arraySink.reset();
        for (Object o : objectArrayList) {
            arraySink.consumeObject(o);
        }

        return arraySink;
    }

    @Benchmark
    public void group2ConsumeIntegerArrayListHashCodes(Blackhole bh) {
        for (Integer i : integerArrayList) {
            bh.consume(i.hashCode());
        }
    }

    @Benchmark
    public void group2ConsumeObjectArrayListHashCodes(Blackhole bh) {
        for (Object o : objectArrayList) {
            bh.consume(o.hashCode());
        }
    }

    @Benchmark
    public void group3ConsumeIntegerArrayHashCodes(Blackhole bh) {
        for (Integer i : integerArray) {
            bh.consume(i.hashCode());
        }
    }

    @Benchmark
    public void group3ConsumeObjectArrayHashCodes(Blackhole bh) {
        for (Object o : objectArray) {
            bh.consume(o.hashCode());
        }
    }

    /**
     * A custom sink, that stores objects in an internal array.
     *
     * @implNote objects are written in reverse order to prevent smart compilers from figuring out that they could
     * just use {@link System#arraycopy(Object, int, Object, int, int)} instead of consuming objects one by one under
     * specific conditions.
     */
    public static final class ArraySink {
        final Object[] values;
        int pos;

        ArraySink(int size) {
            this.values = new Object[size];
            this.pos = values.length - 1;
        }

        void consumeObject(Object o) {
            values[pos--] = o;
        }

        void consumeInteger(Integer i) {
            values[pos--] = i;
        }

        void reset() {
            pos = values.length - 1;
        }
    }
}
```

This is quite some code, with benchmarks being split into groups. I could have created dedicated benchmarks for each group, but
found that I could iterate quicker this way. Also, with everything being in one file, it's easier to observe similarities &
differences between the groups. As an added bonus, some code duplication is being avoided as well. Let's discuss one group 
after the other.

### Group 0: Casting in Isolation
For your convenience, let me paste the benchmark methods from group 0 here again:
```java
    @Benchmark
    public void group0ConsumeObjectArrayList(Blackhole bh) {
        for (Object o : objectArrayList) {
            bh.consume(o);
        }
    }

    @Benchmark
    public void group0ConsumeIntegerArrayList(Blackhole bh) {
        for (Integer i : integerArrayList) {
            bh.consume(i);
        }
    }
```
Both methods iterate over an array list containing exactly the same elements, however the first one 
consumes them as plain objects, and the second one as `Integers` objects. If you look into the Java bytecode corresponding
to these two methods, you can confirm that they are identical, except for a single `CHECKCAST java/lang/Integer` instruction,
that appears only in `group0ConsumeIntegerArrayList` but not in `group0ConsumeObjectArrayList`. Since nothing else is
happening in the loop, comparing the results of these two benchmarks shows us how much casting costs compared to iteration
and the costs implied by the `Blackhole` implementation. Here is a plot generated by the courtesy of Microsoft Copilot
with the results from my Apple M4 laptop using `OpenJDK 64-Bit Server VM, 24.0.2`:

![bar plot with benchmark results group 0](/docs/assets/img/2025-08-22-group0-consume-array-list-benchmark.png)

As you can see, in this highly artificial scenario, casting is not for free at all, but adds a significant overhead. Digging
though the ARM assembly, I found `CHECKCAST java/lang/Integer` is being compiled to something like
```none
ldr	w12, [x4, #8]       // loads the class pointer at offset 8, from the object addressed by x4 into register w12
mov	x11, #0x190000
movk	x11, #0x1ee0        // these two movs store the address of java/lang/Integer into register x11
cmp	w12, w11            // compares the class of the object with java/lang/Integer
b.ne	0x0000000116e67724  // jumps to an uncommon call trap if the cast fails
```
Thus, it's not that surprising, that these 5 instructions can add overhead, if we are doing nothing else in the loop body.
Now, does it mean that typical Java programms are doomed to eternal slowness, due to implicit casts from the standard collectins
library and other generic classes? Let's have a look at the benchmark from group 1:
```java
    @Benchmark
    public ArraySink group1CopyReferencesIntegerArrayList() {
        arraySink.reset();
        for (Integer i : integerArrayList) {
            arraySink.consumeInteger(i);
        }

        return arraySink;
    }

    @Benchmark
    public ArraySink group1CopyReferencesObjectArrayList() {
        arraySink.reset();
        for (Object o : objectArrayList) {
            arraySink.consumeObject(o);
        }

        return arraySink;
    }

    /**
     * A custom sink, that stores objects in an internal array.
     *
     * @implNote objects are written in reverse order to prevent smart compilers from figuring out that they could
     * just use {@link System#arraycopy(Object, int, Object, int, int)} instead of consuming objects one by one under
     * specific conditions.
     */
    public static final class ArraySink {
        final Object[] values;
        int pos;

        ArraySink(int size) {
            this.values = new Object[size];
            this.pos = values.length - 1;
        }

        void consumeObject(Object o) {
            values[pos--] = o;
        }

        void consumeInteger(Integer i) {
            values[pos--] = i;
        }

        void reset() {
            pos = values.length - 1;
        }
    }
```
Again, both `group1CopyReferencesIntegerArrayList` and `group1CopyReferencesObjectArrayList` iterate over an array list,
and copy the object references one by one over to another `Object` array. The first method contains an implicit cast, the
second one doesn't. Note that the loop body is still very short, and involves just copying references, and decrementing a counter.
However, looking at the benchmark results, the picture looks very different:

![bar plot with benchmark results group 1](/docs/assets/img/2025-08-22-group1-consume-array-list-benchmark.png)

While previously, the version without the cast was almost 3 times faster, now we are taking about an overhead of roughly 5%.
Note that the loops in group 1 are still somewhat atypical, because the object based version of the loop, that is measured
by `group1CopyReferencesObjectArrayList` is able to get away by only manipulating object references, whereas the implicit cast
in `group1CopyReferencesIntegerArrayList` forces the runtime to access the referenced objects themselves. In the next group

## Conclusion



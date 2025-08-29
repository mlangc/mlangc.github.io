Got it — here’s a compact JMH benchmark class that compares iterating over an `ArrayList<Integer>` with and without explicit casting, so you can see if the extra casts have any measurable cost in a tight loop.

```java
package org.sample;

import org.openjdk.jmh.annotations.*;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * Benchmark to measure potential overhead of casting when iterating over ArrayList<Integer>.
 * 
 * - baseline(): uses primitive int[] (no boxing/unboxing, no casting)
 * - implicitCast(): iterates over ArrayList<Integer> with implicit casts from Object
 * - explicitCast(): same as implicitCast but with an explicit (Integer) cast
 * 
 * If implicitCast and explicitCast are slower than baseline, the difference may be due to
 * casting, boxing/unboxing, or both. The gap between implicitCast and explicitCast isolates
 * the cost of the explicit cast itself.
 */
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread)
public class CastOverheadBenchmark {

    private static final int SIZE = 1_000;
    private List<Integer> list;
    private int[] array;

    @Setup
    public void setup() {
        list = new ArrayList<>(SIZE);
        array = new int[SIZE];
        for (int i = 0; i < SIZE; i++) {
            list.add(i);
            array[i] = i;
        }
    }

    @Benchmark
    public int baseline() {
        int sum = 0;
        for (int v : array) {
            sum += v;
        }
        return sum;
    }

    @Benchmark
    public int implicitCast() {
        int sum = 0;
        for (Integer v : list) { // implicit cast from Object to Integer
            sum += v;
        }
        return sum;
    }

    @Benchmark
    public int explicitCast() {
        int sum = 0;
        for (Object o : list) {
            sum += (Integer) o; // explicit cast
        }
        return sum;
    }
}
```

This should complete well under a minute with default JMH settings.  
Once you run it, we can compare the throughput numbers to see if the explicit cast adds measurable overhead beyond the implicit one.
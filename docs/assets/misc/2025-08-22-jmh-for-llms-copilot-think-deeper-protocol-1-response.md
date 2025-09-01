```java
package benchmark;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.List;
import java.util.ArrayList;

/**
 * Compare the cost of iterating over a primitive int[] vs. ArrayList<Integer>.
 * ArrayList.get(...) incurs a JVM CHECKCAST + Integer.intValue() on each access.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 3, time = 200, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 5, time = 200, timeUnit = TimeUnit.MILLISECONDS)
@Fork(1)
public class CastOverheadBenchmark {

    @State(Scope.Thread)
    public static class Data {
        @Param({"1000", "1000000"})
        public int size;

        public int[] array;
        public List<Integer> list;

        @Setup(Level.Trial)
        public void setup() {
            array = new int[size];
            list = new ArrayList<>(size);
            for (int i = 0; i < size; i++) {
                array[i] = i;
                list.add(i);
            }
        }
    }

    /**
     * Pure primitive loop: no casts or object indirection.
     */
    @Benchmark
    public long sumArray(Data data) {
        long sum = 0;
        int[] a = data.array;
        for (int i = 0; i < a.length; i++) {
            sum += a[i];
        }
        return sum;
    }

    /**
     * Boxed loop: each get() does a CHECKCAST to Integer, then auto-unboxes.
     */
    @Benchmark
    public long sumList(Data data) {
        long sum = 0;
        List<Integer> l = data.list;
        for (int i = 0, n = l.size(); i < n; i++) {
            sum += l.get(i);
        }
        return sum;
    }
}
```
Here's a concise JMH benchmark to compare casting overhead in ArrayList iteration:

```java
import org.openjdk.jmh.annotations.*;
import java.util.ArrayList;
import java.util.concurrent.TimeUnit;

@State(Scope.Thread)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 3, time = 500, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 5, time = 500, timeUnit = TimeUnit.MILLISECONDS)
public class ArrayListCastBenchmark {
    
    private ArrayList<Integer> arrayList;
    private Integer[] array;
    
    @Setup
    public void setup() {
        int size = 10000;
        arrayList = new ArrayList<>(size);
        array = new Integer[size];
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            array[i] = i;
        }
    }

    // Baseline: Iterate over pre-cast Integer array (no casts in loop)
    @Benchmark
    public long baselineArray() {
        long sum = 0;
        for (Integer integer : array) {
            sum += integer;
        }
        return sum;
    }

    // Tests ArrayList iteration with implicit casts in get()
    @Benchmark
    public long arrayListGet() {
        long sum = 0;
        for (int i = 0; i < arrayList.size(); i++) {
            sum += arrayList.get(i);  // Includes cast from Object to Integer
        }
        return sum;
    }

    // Tests ArrayList iteration with iterator (implicit casts in next())
    @Benchmark
    public long arrayListForEach() {
        long sum = 0;
        for (Integer integer : arrayList) {  // Includes cast in iterator.next()
            sum += integer;
        }
        return sum;
    }
}
```

Key points:
1. **baselineArray**: Measures iteration without casts (lower bound)
2. **arrayListGet**: Measures classic for-loop with casts in `get()`
3. **arrayListForEach**: Measures enhanced for-loop with casts in iterator

The difference between array and ArrayList measurements shows the cast overhead. Typically, JIT optimizes casts efficiently, but this helps quantify it.
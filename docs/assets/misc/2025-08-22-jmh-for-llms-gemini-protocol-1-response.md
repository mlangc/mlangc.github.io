```java
import org.openjdk.jmh.annotations.*;
import java.util.ArrayList;
import java.util.concurrent.TimeUnit;
import java.util.Random;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(value = 1, warmups = 1)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 1)
public class CastingBenchmark {

    @Param({"1000"})
    private int size;

    private ArrayList<Integer> arrayList;
    private int[] primitiveArray;

    @Setup
    public void setup() {
        Random random = new Random(1234); // Use a fixed seed for reproducibility
        arrayList = new ArrayList<>(size);
        primitiveArray = new int[size];
        for (int i = 0; i < size; i++) {
            int value = random.nextInt();
            arrayList.add(value);
            primitiveArray[i] = value;
        }
    }

    /**
     * This benchmark iterates over an ArrayList<Integer>.
     * The Java compiler inserts an implicit cast from Object to Integer
     * for each element retrieved from the list, followed by unboxing to an int.
     * This measures the performance of a typical generic collection loop.
     */
    @Benchmark
    public int sumArrayList() {
        int sum = 0;
        for (Integer i : arrayList) {
            sum += i;
        }
        return sum;
    }

    /**
     * This benchmark iterates over a primitive int array.
     * There is no casting or unboxing involved, as we are dealing
     * directly with primitive types. This serves as a baseline to
     * measure the "best-case" performance without any object overhead.
     */
    @Benchmark
    public int sumPrimitiveArray() {
        int sum = 0;
        for (int i : primitiveArray) {
            sum += i;
        }
        return sum;
    }
}
```
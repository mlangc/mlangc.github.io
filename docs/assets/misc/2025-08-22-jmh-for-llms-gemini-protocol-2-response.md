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
    private Integer[] integerArray;
    private Object[] objectArray;
    private int[] primitiveArray;

    @Setup
    public void setup() {
        Random random = new Random(1234);
        arrayList = new ArrayList<>(size);
        integerArray = new Integer[size];
        objectArray = new Object[size];
        primitiveArray = new int[size];

        for (int i = 0; i < size; i++) {
            int value = random.nextInt();
            Integer integerValue = value; // Box once
            arrayList.add(integerValue);
            integerArray[i] = integerValue;
            objectArray[i] = integerValue;
            primitiveArray[i] = value;
        }
    }

    /**
     * Baseline: Iterates over a primitive int array.
     * No objects, no casting, no unboxing. This is the theoretical best-case scenario.
     */
    @Benchmark
    public int sumPrimitiveArray() {
        int sum = 0;
        for (int i : primitiveArray) {
            sum += i;
        }
        return sum;
    }

    /**
     * Iterates over an Integer[] array.
     * This involves object access and unboxing for each element, but no type casting,
     * as the array's type is already known to be Integer[].
     */
    @Benchmark
    public int sumIntegerArray() {
        int sum = 0;
        for (Integer i : integerArray) {
            sum += i;
        }
        return sum;
    }

    /**
     * Iterates over an Object[] array known to contain Integers.
     * This requires an explicit (or in a for-each loop, implicit) cast and then unboxing.
     * This is very similar to ArrayList's internal behavior but without the get() method call overhead.
     */
    @Benchmark
    public int sumObjectArrayWithCast() {
        int sum = 0;
        for (Object o : objectArray) {
            sum += (Integer) o;
        }
        return sum;
    }

    /**
     * The original test: iterates over an ArrayList<Integer>.
     * This involves the list's get() method call, a type cast, and unboxing.
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
}
```
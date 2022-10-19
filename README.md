# Handling Shared Data Part 1

## Learning Goals

- Define concurrency issue involving variable visibility.
- Define concurrency issue involving operation atomicity.
- Use `volatile` keyword to manage variable visibility.
- Use class `AtomicInteger` to manage operation atomicity.

## Shared Data Problem

Data can end up in an inconsistent state when multiple threads operate on the data.
Two or more threads may attempt to read or write the same variable, and the
concurrent execution of the operations may result in unexpected values being stored
in the variable.

## Visibility

For certain applications there may be a single thread that’s modifying data
while several other threads are only reading it. The reading threads can access
stale data even if only a single thread is writing the data, i.e., different
threads may have different views of the data. This can happen because of caching
in threads, compiler optimization, sequence of operations, and more.

Let's look at an example.  The `main` thread writes to the
instance variables, while the `reader` thread reads the variables.

```java
public class Writer {

   private static long a;
   private static long b;
   private static boolean okToAdd;

   public static void main(String[] args) {
      Thread reader = new Thread(() -> {
         if (okToAdd)
            System.out.println("a + b = " + (a+b));
         else
            System.out.println("a = " + a + " b = " + b);
      });

      reader.start();
      a = 3;
      b = 4;
      okToAdd = true;
   }
}
```

If we run the program it will most likely print `a + b = 12` or `a = 0 b = 0`, but it
could print something unexpected such as `a + b = 0` or `a = 3 b = 4`.  How could that happen?


![visibility issue with cache](https://curriculum-content.s3.amazonaws.com/6002/handling-shared-data-part-1/cache.png)

If your computer has multiple CPUs, each thread may run on a different CPU.
A CPU uses cache to improve performance, and data is copied between
main memory and cache.  When a thread updates a variable in cache,
the new value may not immediately be written to main memory
as write operations are often queued.  As a result, other threads may
read the stale value in their cache. 

Another issue that arises is that a thread may see writes occur in a
different order than the actual program instruction order!
The instruction that writes a new value in `okToAdd` may in fact
happen before the instructions that write new values to variables `a` and `b`.
Reordering of instructions is an optimization technique sometimes
done for performance improvements.
Another possibility is that the order of writes
flushed to main memory may occur in an order that
is different from the program order.

Unfortunately, caching and instruction reordering may lead to data inconsistency
among multiple threads.

## Volatile

Java provides a keyword `volatile` to ensure that an update
of a variable made by one thread must be made visible to
other threads.  Making a variable `volatile`
tells the JVM that reads and writes should happen only in the main memory
instead of caches, ensuring all threads see the same value.
The keyword also tells the JVM to not reorder any instruction involving the volatile variable.

```java
private static volatile long a;
private static volatile long b;
private static volatile boolean okToAdd;
```

In addition to fixing the visibility problem, the `volatile` keyword
ensures memory access is atomic.  This is important when working with
data types such as `long` and `double`, since reading and writing 64-bits
may require two write operations. 


### Atomic Operations

An atomic operation occurs as a single step, it
can't be further divided into smaller steps.
A non-atomic operation is one that consists of multiple steps.

A `volatile` variable ensures memory access is atomic, i.e. reading
and writing the variable to memory occurs as a single step.
However, `volatile` provides no guarantee that other operations
are atomic for the variable.

Consider a basic `Counter` class that provides methods to increment and get
the variable `count`.

```java
public class Counter {
    private int count;

    public int getValue() {return count;}

    public void increment() {count++;}
}
```

The `increment()` method contains a single statement `count++`.
While it may seem like this is a single operation, it actually
consists of multiple steps:

- Read the current value of the `count` variable.
- Increment the value.
- Write the new value to the `count` variable.

The `count++` operation is non-atomic since it can't be done as a single step.
Data inconsistency may occur if multiple threads execute the operation.  

The program below demonstrates the problem by creating
two threads that call `counter.increment()` 1000 times each.

```java
import java.util.stream.IntStream;

public  class Example {
   public static void main(String[] args) throws InterruptedException {
      Counter counter = new Counter();

      //each thread calls counter.increment() 1000 times
      Thread t1 = new Thread( () -> IntStream.range(0,1000).forEach( (i) -> counter.increment()) );
      Thread t2 = new Thread( () -> IntStream.range(0,1000).forEach( (i) -> counter.increment()) );

      t1.start();
      t2.start();

      //wait until t1 and t2 finish running
      t1.join();
      t2.join();
      System.out.println("counter value = " + counter.getValue());  //should be 2000
   }
}
```

We would expect to see `2000` as the final value,
but each time we run the program it may produce a different value:

```text
counter value = 1934
```

When multiple threads operate on the same data, it can
lead to thread interference, i.e., the sequences of the non-atomic operations
performed by multiple threads may overlap.

Let’s walk through a scenario like this with our code above.

1. Thread `t1`: read `count` value of 0.
2. Thread `t1`: increment `count` value by 1.
3. Thread `t2`: read `count` value. (this is read as `0` since thread `t1`
   did not yet write the new value from step 2).
4. Thread `t1`: write new value (`1`) in the `count` variable.
5. Thread `t2`: increment the value read in step 3 by 1.
6. Thread `t2`: write new value (`1`) in the `count` variable.

The `count` variable ends up with a value of `1` because the non-atomic
operations overlapped. The `t1` thread’s result in step 4
was overwritten by the `t2` thread in step 6.

One way to ensure atomic operations is to use
atomic classes such as `AtomicInteger`, `AtomicLong`, `AtomicBoolean`,
and `AtomicReference`.  Each has methods to support atomic operations and
volatility. 

We will update our counter to use `AtomicInteger` to store the value.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public int getValue() {return count.get();}

    public void increment() {count.incrementAndGet();}
}
```

An `AtomicInteger` object is also volatile, ensuring all threads
see the same integer value.

We replace `count++` in the `increment()` method with
`count.incrementAndGet()`.  The method performs an atomic
operation to increment the value by 1.
We use `count.get()` to retrieve the value stored in the atomic integer.

```java
import java.util.stream.IntStream;

public  class Example {
    public static void main(String[] args) throws InterruptedException {
        AtomicCounter counter = new AtomicCounter();

        //each thread calls counter.increment() 1000 times
        Thread t1 = new Thread( () -> IntStream.range(0,1000).forEach( (i) -> counter.increment()) );
        Thread t2 = new Thread( () -> IntStream.range(0,1000).forEach( (i) -> counter.increment()) );

        t1.start();
        t2.start();

        //wait until t1 and t2 finish running
        t1.join();
        t2.join();
        System.out.println("counter value = " + counter.getValue());  // 2000
    }
}
```

The program now produces the correct result:

```text
2000
```

## Conclusion

We’ve looked at how shared data between threads can lead to visibility and
atomicity issues.  We can use the `volatile` keyword to ensure all threads
use atomic access with main memory rather than cache.  We use atomic
classes such as `AtomicInteger` to ensure atomic operations when incrementing
a counter.

## Resources

[Java 11 AtomicInteger](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html)  
[Java Atomic Access](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html)  

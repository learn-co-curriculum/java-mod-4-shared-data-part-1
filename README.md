# Handling Shared Data Part 1

## Learning Goals

- Explain how shared data between threads can cause issues.
- Use the `volatile` keyword to handle shared data.

## Shared Data Problem

Let’s look at an example to see how data can end up in an unexpected state when
multiple threads perform operations on it at the same time.

```java
class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getValue() {
        return count;
    }
}

class CountThread extends Thread {
    private final Counter counter;

    public CountThread(Counter counter) {
        this.counter = counter;
    }

    @Override
    public void run() {
        counter.increment();
    }
}

class Example {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Thread t1 = new CountThread(counter);
        Thread t2 = new CountThread(counter);

        t1.start();
        t2.start();

        System.out.println(counter.getValue());
    }
}
```

If you run this program, the `counter.getValue()` may give you the wrong result.
It will also give you different results if you run it multiple times. In order
to understand the issue, we have to first learn about atomic and non-atomic
operations.

An atomic operation is an operation that can be consists of a single step
whereas a non-atomic operation is one that consists of multiple steps. For
example, in the `increment` method of the `Counter` class, we use the `count++`
operation. This is a non-atomic operation since it’s performing 3 operations:

- Read the current value of `count`.
- Increment the `count` value.
- Write the new value in the `count` field.

When multiple threads (`t1` and `t2`) are operating on the same data, it can
lead to thread interference, i.e., the sequences of the non-atomic operations
performed by multiple threads may overlap.

Let’s walk through a scenario like this with our code above.

1. Thread `t1`: read `count` value.
2. Thread `t1`: increment `count` value by 1.
3. Thread `t2`: read `count` value. (this is read as `0` since thread `t1`
   hasn’t written the new value to the `count` variable).
4. Thread `t1`: write new value (`1`) in the `count` variable.
5. Thread `t2`: increment the read value by 1 (current `count` value is `1`
   since this thread had read the value as `0` earlier).
6. Thread `t2`: write new value (`1`) in the `count` variable.

The `count` variable ends up with a value of `1` because the non-atomic
operations overlapped. The `t1` thread’s result was overwritten by the `t2`
thread.

Sometimes even reading data can be non-atomic. For example, reading 64-bit
values like `double` and `long` may not be atomic. A thread can access the value
while another thread hasn’t finished writing the new value (it may only have
written 32-bits).

## Visibility

For certain applications there may be a single thread that’s modifying data
while several other threads are only reading from it. The threads can access
stale data even if only a single thread is changing the data, i.e., different
threads may have different views of the data. This can happen because of caching
in threads, compiler optimization, sequence of operations, and more.

Here’s a program to demonstrate this phenomenon.

```java
class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getValue() {
        return count;
    }
}

class Example {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Thread t1 = new Thread(() -> counter.increment());
        Thread t2 = new Thread(() -> System.out.println(counter.getValue()));

        t1.start();
        t2.start();
    }
}
```

The `t2` thread might output `0` or `1` depending on when the `count` variable
is read and if it’s being read from a cache.

## Volatile

Java provides a keyword called `volatile` that can make variable reads and
writes happen only in the main memory instead of any caches. The variables
visible to the thread that writes changes are written out to the main memory and
any threads that are reading the variable will read from the main memory.

For example, if we declare a `double` or `long` variable as `volatile`, it would
make the read and write operations an atomic.

```java
class Example {
    volatile long num;
    volatile double doubleNum;
}
```

The primitive types (boolean, byte, short, int, char, float) are already
guaranteed to have atomic read and writes.

The `volatile` does not make the data thread-safe though as thread interference,
i.e., the overlapping of thread operation sequences, can still happen.

## Fixing Visibility Without Using Volatile

If a thread modifies a variable before another operation thread is started or
waits until after finishing its execution, then visibility problems can be
avoided. Let’s look at examples of both of these cases.

For the first, case we simply change the variables from the main thread before
starting up threads that has to read the variable. This isn’t always viable
since we will need multiple threads operating on the same variable in certain
cases.

```java
class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getValue() {
        return count;
    }
}

class Example {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
				counter.increment();
        Thread t1 = new Thread(() -> System.out.println(counter.getValue()));

        t1.start();
    }
}
```

In the second case, we use the `join` method to wait for a thread to finish its
execution before starting a new thread. Let’s modify the very first example in
this lesson to use this method.

```java
class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int getValue() {
        return count;
    }
}

class CountThread extends Thread {
    private final Counter counter;

    public CountThread(Counter counter) {
        this.counter = counter;
    }

    @Override
    public void run() {
        counter.increment();
    }
}

class Example {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Thread t1 = new CountThread(counter);
        Thread t2 = new CountThread(counter);

        t1.start();
				t1.join();
        t2.start();
				t2.join();

        System.out.println(counter.getValue());
    }
}
```

This program will always give the correct result since there’s no chance of the
non-atomic operations in different threads overlapping. Again, this method won’t
always be viable but it’s still important to be aware of it.

## Conclusion

We’ve looked at how shared data between threads can lead to issues and a few
different ways of handling them. In later lessons, we’ll look at how to
synchronize data between threads so that multiple threads can safely work with
shared data.

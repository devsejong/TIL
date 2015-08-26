# Java 8 Concurrency Tutorial: Atomic Variables and ConcurrentMap

Welcome to the third part of my tutorial series about multi-threaded programming in Java 8. This tutorial covers two important parts of the Concurrency API: Atomic Variables and Concurrent Maps. Both have been greatly improved with the introduction of lambda expressions and functional programming in the latest Java 8 release. All those new features are described with a bunch of easily understood code samples. Enjoy!

# 자바8 Concurrency 튜토리얼: Atomic 변수/ConcurrentMap

자바8을 활용한 멀티스레드 프로그래밍 튜토리얼의 3번째 파트에 오신것을 환영합니다. 이번 튜토리얼은 Concurrency API의 Atomic 변수와 ConcurrentMap에 대해서 설명하고 있습니다. 자바8에서 소개된 람다와 함수형 프로그래밍 기법을 도입하여 엄청나게 개선되었습니다. 아래에서는 자바8을 기반으로 코드예제를 작성하였습니다. 

***

- Part 1: Threads and Executors
- Part 2: Synchronization and Locks
- Part 3: Atomic Variables and ConcurrentMap

//TODO

***

For simplicity the code samples of this tutorial make use of the two helper methods `sleep(seconds)` and `stop(executor)` as defined [here](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java).

//TODO

***

## AtomicInteger

The package `java.concurrent.atomic` contains many useful classes to perform atomic operations. An operation is atomic when you can safely perform the operation in parallel on multiple threads without using the `synchronized` keyword or locks as shown in my [previous tutorial](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/.

## AtomicInteger

//TODO

***

Internally, the atomic classes make heavy use of [compare-and-swap](http://en.wikipedia.org/wiki/Compare-and-swap)(CAS), an atomic instruction directly supported by most modern CPUs. Those instructions usually are much faster than synchronizing via locks. So my advice is to prefer atomic classes over locks in case you just have to change a single mutable variable concurrently.

Now let's pick one of the atomic classes for a few examples: `AtomicInteger`

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(atomicInt::incrementAndGet));
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 1000

By using `AtomicInteger` as a replacement for `Integer` we're able to increment the number concurrently in a thread-safe manor without synchronizing the access to the variable. The method `incrementAndGet()` is an atomic operation so we can safely call this method from multiple threads.

`AtomicInteger` supports various kinds of atomic operations. The method `updateAndGet()` accepts a lambda expression in order to perform arbitrary arithmetic operations upon the integer:

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> {
            Runnable task = () ->
                atomicInt.updateAndGet(n -> n + 2);
            executor.submit(task);
        });
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 2000

The method `accumulateAndGet()` accepts another kind of lambda expression of type `IntBinaryOperator`. We use this method to sum up all values from 0 to 1000 concurrently in the next sample:

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> {
            Runnable task = () ->
                atomicInt.accumulateAndGet(i, (n, m) -> n + m);
            executor.submit(task);
        });
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 499500

Other useful atomic classes are [AtomicBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html), [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html) and [AtomicReference](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html).

# Java 8 Concurrency Tutorial: Atomic Variables and ConcurrentMap

Welcome to the third part of my tutorial series about multi-threaded programming in Java 8. This tutorial covers two important parts of the Concurrency API: Atomic Variables and Concurrent Maps. Both have been greatly improved with the introduction of lambda expressions and functional programming in the latest Java 8 release. All those new features are described with a bunch of easily understood code samples. Enjoy!

# 자바8 Concurrency 튜토리얼: Atomic 변수/ConcurrentMap

자바8을 활용한 멀티스레드 프로그래밍 튜토리얼의 3번째 파트에 오신것을 환영합니다. 이번 튜토리얼은 Concurrency API의 Atomic 변수와 ConcurrentMap에 대해서 설명하고 있습니다. 자바8에서 소개된 람다와 함수형 프로그래밍 기법을 도입하여 엄청나게 개선되었습니다. 아래에서는 자바8을 기반으로 코드예제를 작성하였습니다. 

***

- Part 1: Threads and Executors
- Part 2: Synchronization and Locks
- Part 3: Atomic Variables and ConcurrentMap


- Part 1: [Thread/Executor](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)
- Part 2: [Synchronize/Lock](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)
- Part 3: Atomic Variables and ConcurrentMap

***

For simplicity the code samples of this tutorial make use of the two helper methods `sleep(seconds)` and `stop(executor)` as defined [here](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java).

튜토리얼에서 작성할 예제코드에서는 단순함을 위해서 두개의 메서드 `sleep(seconds)`과 `stop(executor)`을 정의합니다. `sleep(seconds)`는 입력되는 초만큼 스레드를 정지합니다. `stop(executor)`는 입력받은 ExecutorService를 정지하는 역할을 합니다. 이 메서드는 [여기](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)에서 찾아볼 수 있습니다.


***

## AtomicInteger

The package `java.concurrent.atomic` contains many useful classes to perform atomic operations. An operation is atomic when you can safely perform the operation in parallel on multiple threads without using the `synchronized` keyword or locks as shown in my [previous tutorial](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/.

## AtomicInteger

패키지 `java.concurrent.atomic`에서는 Atomic Operation을 수행하는데 유용한 클래스들을 찾을 수 있습니다. Operation이 Atomic하다면 멀티스레드를 활용하는 병렬환경에서 [이전 튜토리얼](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)에서 `synchronized`키워드나 락을 사용하지 않고 안전하게 기능을 수행할 수 있다는 것을 의미합니다.

***

Internally, the atomic classes make heavy use of [compare-and-swap](http://en.wikipedia.org/wiki/Compare-and-swap)(CAS), an atomic instruction directly supported by most modern CPUs. Those instructions usually are much faster than synchronizing via locks. So my advice is to prefer atomic classes over locks in case you just have to change a single mutable variable concurrently.

내부적으로 Atomic 클래스는들은 [compare-and-swap(CAS)](https://en.wikipedia.org/wiki/Compare-and-swap)을 주로 사용합니다. 현대의 CPU에서 기본적으로 Atomic 명령을 바로 지원합니다. 이러한 명령들은 락을 활용한 동기화 방법보다 훨씬 유용합니다. 그러므로 만약 하나의 변수를 동시적으로 변경할 경우, atomic 클래스를 적극적으로 사용하는 것을 권장합니다.

***

Now let's pick one of the atomic classes for a few examples: `AtomicInteger`

이제 Atomic클래스 중 하나인 `AtomicInteger`를 사용한 예제코드를 잠깐 살펴봅시다.

***

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(atomicInt::incrementAndGet));
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 1000

By using `AtomicInteger` as a replacement for `Integer` we're able to increment the number concurrently in a thread-safe manor without synchronizing the access to the variable. The method `incrementAndGet()` is an atomic operation so we can safely call this method from multiple threads.

`AtomicInteger`를 `Integer`대신 사용하여 숫자가 동시에 변경되는 상황에서 변수의 접근을 동기화하지 않고도 스레드세이프하게 처리할 수 있습니다. 메서드 `incrementAndGet()`은 Atomic Operation이므로 우리는 멀티스레드에서 안전하게 메서드를 부를 수 있습니다.

***

`AtomicInteger` supports various kinds of atomic operations. The method `updateAndGet()` accepts a lambda expression in order to perform arbitrary arithmetic operations upon the integer:

`AtomicInteger`는 다양한 종류의 Atomic Operation을 제공합니다. 메서드 `updateAndGet()`은 람다 표현식을 사용하여 자유롭게 산술 표현식을 사용할 수 있습니다. 

***

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

메서드 `accumulateAndGet()`는 다른 종류의 람다표현식 타입인 `IntBinaryOperator`를 받아들일 수 있습니다. 이 메서드를 사용하여 0부터 1000까지 비동기를 사용하여 합계를 계산하는 코드를 아래의 샘플에서 확인할 수 있습니다.

***

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

다른 유용한 Atomic클래스에는 [AtomicBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html), [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html), [AtomicReference](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)가 있습니다.

***

## LongAdder

The class `LongAdder` as an alternative to `AtomicLong` can be used to consecutively add values to a number.

`LongAdder`클래스는 `AtomicLong`에 계속적으로 값이 더해지는 경우에 사용할 수 있습니다.

***

    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(adder::increment));
    
    stop(executor);
    
    System.out.println(adder.sumThenReset());   // => 1000

LongAdder provides methods `add()` and `increment()` just like the atomic number classes and is also thread-safe. But instead of summing up a single result this class maintains a set of variables internally to reduce contention over threads. The actual result can be retrieved by calling `sum()` or `sumThenReset()`.

`LongAdder`는 메서드 `add()`와 `increment()`를 제공합니다. 다른 Atomic클래스와 동일하게 이 메서드들 또한 스레드세이프합니다. 하지만 하나의 결과값을 클래스에서 유지하고 있는 다른 클래스와는 달리 내부적으로 변수값 세트를 가지며 reduce contention over threads. 실제값은 `sum()`이나 `sumThenReset()` 메서드가 호출 될 때 계산됩니다.

***

This class is usually preferable over atomic numbers when updates from multiple threads are more common than reads. This is often the case when capturing statistical data, e.g. you want to count the number of requests served on a web server. The drawback of `LongAdder` is higher memory consumption because a set of variables is held in-memory.

이 클래스는 주로 여러 스레드에서 수정작업이 발생할경우, 읽기작업보다는 쓰기작업이 많을경우에 활용할 수 있습니다. 통계 데이터를 계산할 경우가 대표적입니다. 예를들면 웹서버에서 요청이 얼마만큼 발생했는지를 측정하는 경우. `LongAdder`클래스의 단점으로는 변수의 셋을 관리하여야 하므로 많은 메모리를 차지한다는 점이 있습니다. 

***

## LongAccumulator

`LongAccumulator` is a more generalized version of LongAdder. Instead of performing simple add operations the class `LongAccumulator` builds around a lambda expression of type `LongBinaryOperator` as demonstrated in this code sample:

## LongAccumulator

`LongAccumulator`은 `LongAdder`보다 일반화된 버전의 클래스입니다. 단순한 덧셈작업만 제공하는 `LongAdder`와는 달리 `LongAccumulator`는 람다 표현식을 가진 타입 `LongBinaryOperator`를 사용하여 계산을 할 수 있도록 지원합니다. 아래는 해당 클래스를 사용한 코스예제입니다.

***
    
    LongBinaryOperator op = (x, y) -> 2 * x + y;
    LongAccumulator accumulator = new LongAccumulator(op, 1L);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 10)
        .forEach(i -> executor.submit(() -> accumulator.accumulate(i)));
    
    stop(executor);
    
    System.out.println(accumulator.getThenReset());     // => 2539

We create a `LongAccumulator` with the function `2 * x + y` and an initial value of one. With every call to `accumulate(i)` both the current result and the value `i` are passed as parameters to the lambda expression.

함수 `2 * x + y`를 사용하여 `LongAccumulator`를 초기화하였습니다. `accumulate(i)`가 호출될 때 마다 현재의 결과값과 `i`값이 람다 표현식의 값으로 전달될 것 입니다.

***

A `LongAccumulator` just like `LongAdder` maintains a set of variables internally to reduce contention over threads.

`LongAccumulator` 클래스는 `LongAdder`와 마찬가지로 내부적으로  변수의 셋을 가지고 있으며 to reduce contention over threads. 

***

## ConcurrentMap

The interface `ConcurrentMap` extends the map interface and defines one of the most useful concurrent collection types. Java 8 introduces functional programming by adding new methods to this interface.

## ConcurrentMap

인터페이스 `ConcurrentMap`은 기존의 `Map`인터페이스를 확장하여 동시성환경에서도 사용할 수 있도록 정의된 컬렉션 타입입니다. 자바 8에서는 이 인터페이스에 함수형 프로그래밍의 기능이 들어간 새로운 메서드가 정의되어있습니다.

***

In the next code snippets we use the following sample map to demonstrates those new methods:

다음의 코드에서 앞으로 예제에서 사용할 값들을 정의하였습니다.

***

    ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

The method `forEach()` accepts a lambda expression of type `BiConsumer` with both the key and value of the map passed as parameters. It can be used as a replacement to for-each loops to iterate over the entries of the concurrent map. The iteration is performed sequentially on the current thread.

`forEach()` 메서드는 람다 표현식을 `BiConsumer`타입으로 받을 수 있습니다. 이 functional object는 key/value의 파라미터 형태로 값을 Map에 전달합니다. 이 메서드는 기존의 for-each문을 대체할 수 있습니다. 이 반복문은 현재 스레드에서 순서대로 실행됩니다.

***


    map.forEach((key, value) -> System.out.printf("%s = %s\n", key, value));

The method `putIfAbsent()` puts a new value into the map only if no value exists for the given key. At least for the `ConcurrentHashMap` implementation of this method is thread-safe just like `put()` so you don't have to synchronize when accessing the map concurrently from different threads:

메서드 `putIfAbsent()`는 map에서 해당 key의 값이 존재하지 않을 경우에만 새 값을 추가합니다. `ConcurrentHashMap`에서는 `put`과 마찬가지로 스레드 세이프한 메서드이므로 여러 스레드에서 동시에 접근하더라도 동기화를 할 필요가 없습니다.

***

    String value = map.putIfAbsent("c3", "p1");
    System.out.println(value);    // p0

The method `getOrDefault()` returns the value for the given key. In case no entry exists for this key the passed default value is returned:

메서드 `getOrDefault()`는 해당하는 key에 대한 값을 반환합니다. 만약 key의 값이 존재하지 않을 경우 기본값을 반환합니다.

***

    String value = map.getOrDefault("hi", "there");
    System.out.println(value);    // there

The method `replaceAll()` accepts a lambda expression of type `BiFunction`. `BiFunctions` take two parameters and return a single value. In this case the function is called with the key and the value of each map entry and returns a new value to be assigned for the current key:

//이건 다시.

메서드 `replaceAll()`은 `BiFunction`타입의 람다 표현식을 파라미터로 받습니다. `BiFunctions`클래스의 두 파라미터를 받으며 하나의 값을 리턴합니다. 이 메서드가 호출될 경우 key/value를 활용하여 현재 키값에 새로운 값을 부여할 수 있습니다.

***

    map.replaceAll((key, value) -> "r2".equals(key) ? "d3" : value);
    System.out.println(map.get("r2"));    // d3

Instead of replacing all values of the map `compute()` let's us transform a single entry. The method accepts both the key to be computed and a bi-function to specify the transformation of the value.

map의 전체값을 replace하는 대신 `compute()`를 사용할경우 단일값을 변경할 수 있습니다. 이 메서드는 첫번째 파라미터로 key를 입력받으며, 두번째 파라미터로는 값을 변환하는 로직이 들어있는 `BiFunctions`타입을 받습니다.

***

    map.compute("foo", (key, value) -> value + value);
    System.out.println(map.get("foo"));   // barbar

In addition to `compute()` two variants exist: `computeIfAbsent()` and `computeIfPresent()`. The functional parameters of these methods only get called if the key is absent or present respectively.

Finally, the method `merge()` can be utilized to unify a new value with an existing value in the map. Merge accepts a key, the new value to be merged into the existing entry and a bi-function to specify the merging behavior of both values:

    map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
    System.out.println(map.get("foo"));   // boo was foo


## ConcurrentHashMap

All those methods above are part of the `ConcurrentMap` interface, thereby available to all implementations of that interface. In addition the most important implementation `ConcurrentHashMap` has been further enhanced with a couple of new methods to perform parallel operations upon the map.

Just like parallel streams those methods use a special ForkJoinPool available via `ForkJoinPool.commonPool()` in Java 8. This pool uses a preset parallelism which depends on the number of available cores. Four CPU cores are available on my machine which results in a parallelism of three:

    System.out.println(ForkJoinPool.getCommonPoolParallelism());  // 3

This value can be decreased or increased by setting the following JVM parameter:

    -Djava.util.concurrent.ForkJoinPool.common.parallelism=5

We use the same example map for demonstrating purposes but this time we work upon the concrete implementation `ConcurrentHashMap` instead of the interface `ConcurrentMap`, so we can access all public methods from this class:

    ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

Java 8 introduces three kinds of parallel operations: `forEach`, `search` and `reduce`. Each of those operations are available in four forms accepting functions with keys, values, entries and key-value pair arguments.

All of those methods use a common first argument called `parallelismThreshold`. This threshold indicates the minimum collection size when the operation should be executed in parallel. E.g. if you pass a threshold of 500 and the actual size of the map is 499 the operation will be performed sequentially on a single thread. In the next examples we use a threshold of one to always force parallel execution for demonstrating purposes.

### ForEach

The method `forEach()` is capable of iterating over the key-value pairs of the map in parallel. The lambda expression of type `BiConsumer` is called with the key and value of the current iteration step. In order to visualize parallel execution we print the current threads name to the console. Keep in mind that in my case the underlying `ForkJoinPool` uses up to a maximum of three threads.

    map.forEach(1, (key, value) ->
        System.out.printf("key: %s; value: %s; thread: %s\n",
            key, value, Thread.currentThread().getName()));
    
    // key: r2; value: d2; thread: main
    // key: foo; value: bar; thread: ForkJoinPool.commonPool-worker-1
    // key: han; value: solo; thread: ForkJoinPool.commonPool-worker-2
    // key: c3; value: p0; thread: main

### Search

The method `search()` accepts a `BiFunction` returning a non-null search result for the current key-value pair or `null` if the current iteration doesn't match the desired search criteria. As soon as a non-null result is returned further processing is suppressed. Keep in mind that `ConcurrentHashMap` is unordered. The search function should not depend on the actual processing order of the map. If multiple entries of the map match the given search function the result may be non-deterministic.

    String result = map.search(1, (key, value) -> {
        System.out.println(Thread.currentThread().getName());
        if ("foo".equals(key)) {
            return value;
        }
        return null;
    });
    System.out.println("Result: " + result);
    
    // ForkJoinPool.commonPool-worker-2
    // main
    // ForkJoinPool.commonPool-worker-3
    // Result: bar

Here's another example searching solely on the values of the map:

    String result = map.searchValues(1, value -> {
        System.out.println(Thread.currentThread().getName());
        if (value.length() > 3) {
            return value;
        }
        return null;
    });
    
    System.out.println("Result: " + result);
    
    // ForkJoinPool.commonPool-worker-2
    // main
    // main
    // ForkJoinPool.commonPool-worker-1
    // Result: solo

### Reduce

The method `reduce()` already known from Java 8 Streams accepts two lambda expressions of type `BiFunction`. The first function transforms each key-value pair into a single value of any type. The second function combines all those transformed values into a single result, ignoring any possible `null` values.

    String result = map.reduce(1,
        (key, value) -> {
            System.out.println("Transform: " + Thread.currentThread().getName());
            return key + "=" + value;
        },
        (s1, s2) -> {
            System.out.println("Reduce: " + Thread.currentThread().getName());
            return s1 + ", " + s2;
        });
    
    System.out.println("Result: " + result);
    
    // Transform: ForkJoinPool.commonPool-worker-2
    // Transform: main
    // Transform: ForkJoinPool.commonPool-worker-3
    // Reduce: ForkJoinPool.commonPool-worker-3
    // Transform: main
    // Reduce: main
    // Reduce: main
    // Result: r2=d2, c3=p0, han=solo, foo=bar

I hope you've enjoyed reading the third part of my tutorial series about Java 8 Concurrency. The code samples from this tutorial are `hosted on GitHub` along with many other Java 8 code snippets. You're welcome to fork the repo and try it by your own.


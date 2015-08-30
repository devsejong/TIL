# Java 8 Concurrency Tutorial: Atomic Variables and ConcurrentMap

Welcome to the third part of my tutorial series about multi-threaded programming in Java 8. This tutorial covers two important parts of the Concurrency API: Atomic Variables and Concurrent Maps. Both have been greatly improved with the introduction of lambda expressions and functional programming in the latest Java 8 release. All those new features are described with a bunch of easily understood code samples. Enjoy!

# 자바8 Concurrency 튜토리얼: Atomic 변수/ConcurrentMap

자바8을 활용한 멀티스레드 프로그래밍 튜토리얼의 3번째 파트에 오신것을 환영합니다. 이번 튜토리얼은 Concurrency API의 Atomic 변수와 ConcurrentMap에 대해서 함께 알아보도록 합시다. 두 영역 모두 자바 8에서 도입된 람다와 함수형 프로그래밍을 통해서 많은 개선이 이루어졌습니다. 아래에서 설명하는 예제들은 모두 자바8을 기준으로 작성하였습니다.

***

- Part 1: Threads and Executors
- Part 2: Synchronization and Locks
- Part 3: Atomic Variables and ConcurrentMap

- Part 1: [Thread/Executor](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)
- Part 2: [Synchronize/Lock](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)
- Part 3: Atomic Variables and ConcurrentMap

***

For simplicity the code samples of this tutorial make use of the two helper methods `sleep(seconds)` and `stop(executor)` as defined [here](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java).

튜토리얼에서 작성할 예제코드에서는 단순함을 위해서 두개의 메서드 `sleep(seconds)`과 `stop(executor)`을 정의합니다. `sleep(seconds)`메서드는 입력되는 초만큼 스레드를 정지합니다. `stop(executor)`메서드가 호출될 경우 파라미터의 ExecutorService를 정지합니다. 이 메서드들은 [여기](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)에서 확인할 수 있습니다.


***

## AtomicInteger

The package `java.concurrent.atomic` contains many useful classes to perform atomic operations. An operation is atomic when you can safely perform the operation in parallel on multiple threads without using the `synchronized` keyword or locks as shown in my [previous tutorial](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/.

## AtomicInteger

패키지 `java.concurrent.atomic`에서는 Atomic Operation을 수행하는데 유용한 클래스들을 찾을 수 있습니다. Atomic Operation은 다시말해 멀티스레드를 활용하는 병렬환경에서 [이전 튜토리얼](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)에서 `synchronized`키워드나 락을 사용하지 않고 안전하게 기능을 수행할 수 있다는 것을 의미합니다.

***

Internally, the atomic classes make heavy use of [compare-and-swap](http://en.wikipedia.org/wiki/Compare-and-swap)(CAS), an atomic instruction directly supported by most modern CPUs. Those instructions usually are much faster than synchronizing via locks. So my advice is to prefer atomic classes over locks in case you just have to change a single mutable variable concurrently.

내부적으로 Atomic 클래스들은 [compare-and-swap(CAS)](https://en.wikipedia.org/wiki/Compare-and-swap)을 사용하고 있습니다. 현시대 대부분의 CPU에서 Atomic 명령을 바로 사용할 수 있도록 지원합니다. 이러한 명령문들은 이전 튜토리얼에서 소개한 락을 활용한 동기화 방법보다 훨씬 성능이 좋습니다. 하나의 변수를 동시적으로 변경해야하는 경우에는 Atomic 클래스를 사용하는 것을 권장합니다.

***

Now let's pick one of the atomic classes for a few examples: `AtomicInteger`

이제 Atomic 클래스 중 하나인 `AtomicInteger`를 사용한 예제코드를 잠깐 살펴봅시다.

***

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(atomicInt::incrementAndGet));
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 1000

By using `AtomicInteger` as a replacement for `Integer` we're able to increment the number concurrently in a thread-safe manor without synchronizing the access to the variable. The method `incrementAndGet()` is an atomic operation so we can safely call this method from multiple threads.

`AtomicInteger`를 `Integer`대신 사용하여 숫자가 동시에 변경되는 상황에서 변수의 접근을 동기화하지 않고도 스레드 세이프하게 처리할 수 있습니다. `incrementAndGet()`메서드는 멀티스레드에서 안전하게 호출할 수 있는 Atomic Operation입니다.

***

`AtomicInteger` supports various kinds of atomic operations. The method `updateAndGet()` accepts a lambda expression in order to perform arbitrary arithmetic operations upon the integer:

`AtomicInteger`클래스에는 다양한 종류의 Atomic Operation 메서드를 제공합니다. `updateAndGet()`메서드는 다양한 산술 표현식을람다표현식으로 사용할 수 있습니다.

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

메서드 `accumulateAndGet()`은  `IntBinaryOperator`타입의 람다표현식을 사용할 수 있습니다. 아래는 0부터 1000까지 비동기를 사용하여 합계를 계산하는 코드입니다.

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

`java.concurrent.atomic` 패키지에서 선언된 Atomic 클래스는 [AtomicBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html), [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html), [AtomicReference](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)가 있습니다.

***

## LongAdder

The class `LongAdder` as an alternative to `AtomicLong` can be used to consecutively add values to a number.

## LongAdder

`LongAdder`클래스는 연속적으로 값을 더하는 상황에서 `AtomicLong`를 대체하여 사용합니다.

***

    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(adder::increment));
    
    stop(executor);
    
    System.out.println(adder.sumThenReset());   // => 1000

LongAdder provides methods `add()` and `increment()` just like the atomic number classes and is also thread-safe. But instead of summing up a single result this class maintains a set of variables internally to reduce contention over threads. The actual result can be retrieved by calling `sum()` or `sumThenReset()`.

`LongAdder`는 메서드 `add()`와 `increment()`를 제공합니다. 다른 Atomic클래스와 동일하게 내부의 메서드들은 멀티 스레드 환경에서 안전합니다. 다른 Atomic 클래스와 다른점은 하나의 결과값을 클래스에서 유지하고 있는게 아니라 내부적으로 변수값 세트를 가져 스레드간 경쟁상태가 일어나지 않도록 설계되어있습니다. 메서드를 실제값은 `sum()`이나 `sumThenReset()` 메서드가 호출 될 때 계산됩니다.

***

This class is usually preferable over atomic numbers when updates from multiple threads are more common than reads. This is often the case when capturing statistical data, e.g. you want to count the number of requests served on a web server. The drawback of `LongAdder` is higher memory consumption because a set of variables is held in-memory.

`LongAdder` 클래스는 주로 여러 스레드에서 수정작업이 발생하며 읽기작업보다는 쓰기작업이 많을 경우에 유리합니다. 웹서버에서 요청이 얼마만큼 발생했는지를 측정하는 경우와 같이 통계 데이터를 계산할 경우에 사용할 수 있을것 입니다. 대신에 `LongAdder` 클래스는 각 스레드에서의 변수셋을 각각 관리하므로 상대적으로 많은 메모리를 차지한다는 단점이 있습니다. 

***

## LongAccumulator

`LongAccumulator` is a more generalized version of LongAdder. Instead of performing simple add operations the class `LongAccumulator` builds around a lambda expression of type `LongBinaryOperator` as demonstrated in this code sample:

## LongAccumulator

`LongAccumulator`은 `LongAdder`보다는 일반적인 상황에서 사용할 수 있습니다. 간단한 덧셈작업만을 제공하는 `LongAdder`와는 달리 `LongAccumulator`는 `LongBinaryOperator`타입의 람다 표현식으로  연산작업을 할 수 있습니다.

***
    
    LongBinaryOperator op = (x, y) -> 2 * x + y;
    LongAccumulator accumulator = new LongAccumulator(op, 1L);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 10)
        .forEach(i -> executor.submit(() -> accumulator.accumulate(i)));
    
    stop(executor);
    
    System.out.println(accumulator.getThenReset());     // => 2539

We create a `LongAccumulator` with the function `2 * x + y` and an initial value of one. With every call to `accumulate(i)` both the current result and the value `i` are passed as parameters to the lambda expression.

함수 `2 * x + y`를 사용하여 `LongAccumulator`객체를 초기화하였습니다. `accumulate(i)`가 호출될 때 마다 현재의 결과값과 `i`값이 람다 표현식의 값으로 전달될 것 입니다.

***

A `LongAccumulator` just like `LongAdder` maintains a set of variables internally to reduce contention over threads.

`LongAccumulator` 클래스는 `LongAdder`와 마찬가지로 내부적으로  변수 셋을 가지고 있어 스레드간 경쟁상태를 최소화 합니다.

***

## ConcurrentMap

The interface `ConcurrentMap` extends the map interface and defines one of the most useful concurrent collection types. Java 8 introduces functional programming by adding new methods to this interface.

## ConcurrentMap

인터페이스 `ConcurrentMap`은 기존의 `Map`인터페이스를 확장하여 멀티스레드 환경에서 사용할 수 있는 컬렉션 타입입니다. 자바 8에서는 이 인터페이스에 함수형 프로그래밍의 기능이 들어간 새로운 메서드가 정의되어있습니다.

***

In the next code snippets we use the following sample map to demonstrates those new methods:

다음의 코드에서 앞으로 예제에서 사용할 맵을 정의합니다.

***

    ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

The method `forEach()` accepts a lambda expression of type `BiConsumer` with both the key and value of the map passed as parameters. It can be used as a replacement to for-each loops to iterate over the entries of the concurrent map. The iteration is performed sequentially on the current thread.

`forEach()` 메서드는 `BiConsumer`타입의 람다 표현식을 사용합니다. 이 functional object는 key/value 형태의 파라미터로 값을 Map에 전달합니다. `forEach()` 메서드는 기존 for-each문과 동일하게 동작합니다.

***


    map.forEach((key, value) -> System.out.printf("%s = %s\n", key, value));

The method `putIfAbsent()` puts a new value into the map only if no value exists for the given key. At least for the `ConcurrentHashMap` implementation of this method is thread-safe just like `put()` so you don't have to synchronize when accessing the map concurrently from different threads:

메서드 `putIfAbsent()`는 map에서 해당 key의 값이 존재하지 않을 경우에만 새 값을 추가합니다. `ConcurrentHashMap`에서는 `put`과 마찬가지로 스레드 안전한 메서드이므로 여러 스레드에서 동시에 접근하더라도 동기화 로직을 추가할 필요가 없습니다.

***

    String value = map.putIfAbsent("c3", "p1");
    System.out.println(value);    // p0

The method `getOrDefault()` returns the value for the given key. In case no entry exists for this key the passed default value is returned:

`getOrDefault()`메서드는 해당하는 key에 대한 값을 반환하며 key의 값이 존재하지 않을 경우 두번째 파라미터로 설정한 기본값을 반환합니다.

***

    String value = map.getOrDefault("hi", "there");
    System.out.println(value);    // there

The method `replaceAll()` accepts a lambda expression of type `BiFunction`. `BiFunctions` take two parameters and return a single value. In this case the function is called with the key and the value of each map entry and returns a new value to be assigned for the current key:

//이건 다시.

메서드 `replaceAll()`은 `BiFunction`타입의 람다 표현식을 파라미터로 받습니다. `BiFunctions`에는 두개의 값, 즉 key-value를  파라미터로 가지며 연산결과로 하나의 값을 리턴합니다. 이 메서드가 호출될 경우 key/value를 활용하여 현재 키값에 새로운 값을 부여할 수 있습니다.

***

    map.replaceAll((key, value) -> "r2".equals(key) ? "d3" : value);
    System.out.println(map.get("r2"));    // d3

Instead of replacing all values of the map `compute()` let's us transform a single entry. The method accepts both the key to be computed and a bi-function to specify the transformation of the value.

map의 전체값을 replace하는 대신 `compute()`를 사용할 경우 단일값을 변경할 수 있습니다. 이 메서드는 첫번째 파라미터로 key를 입력받으며, 두번째 파라미터로는 `BiFunctions`타입의 람다표현식을 받습니다.

***

    map.compute("foo", (key, value) -> value + value);
    System.out.println(map.get("foo"));   // barbar

In addition to `compute()` two variants exist: `computeIfAbsent()` and `computeIfPresent()`. The functional parameters of these methods only get called if the key is absent or present respectively.

`compute()`와 비슷한 기능을 가진 메서드가 두개 더 존재합니다. `computeIfAbsent()` 는 값이 존재하지 않을 경우에, `computeIfPresent()`는 값이 존재할 경우에 작업이 수행됩니다.

***

Finally, the method `merge()` can be utilized to unify a new value with an existing value in the map. Merge accepts a key, the new value to be merged into the existing entry and a bi-function to specify the merging behavior of both values:

마지막으로 메서드 `merge()`는 기존의 값과 새로운 값을 합쳐 연산을 할 경우에 사용할 수 있습니다. merge의 첫번째 파라미터는 key를 두번째 파라미터는 새로운 값, 세번째는 연산작업이 되는 BiFunction타입의 람다표현식을 입력할 수 있습니다.

***

    map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
    System.out.println(map.get("foo"));   // boo was bar


## ConcurrentHashMap

All those methods above are part of the `ConcurrentMap` interface, thereby available to all implementations of that interface. In addition the most important implementation `ConcurrentHashMap` has been further enhanced with a couple of new methods to perform parallel operations upon the map.

위에서 이야기했던 `ConcurrentMap` 인터페이스의 메서드들은 구현체를 활용하여 사용이 가능합니다. 이중에서 가장 많이 활용되는 구현체 `ConcurrentHashMap`에서는 추가로 병렬작업 메서드들을 더 확인할 수 있습니다.

***

Just like parallel streams those methods use a special ForkJoinPool available via `ForkJoinPool.commonPool()` in Java 8. This pool uses a preset parallelism which depends on the number of available cores. Four CPU cores are available on my machine which results in a parallelism of three:

Parallel Stream과 마찬가지로 이 메서드들은 자바8의 `ForkJoinPool.commonPool()`을 사용할 수 있습니다. 병렬처리의 프리셋으로 사용되는 이 스레드풀은 활용 가능한 코어의 숫자에 따라서 풀의 크기가 자동으로 정해집니다. 4개의 CPU코어를 가지고 있는 제 컴퓨터에서는 스레드풀의 크기가 3으로 나오고 있습니다.

***

    System.out.println(ForkJoinPool.getCommonPoolParallelism());  // 3

This value can be decreased or increased by setting the following JVM parameter:

참고로 `ForkJoinPool` 스레드 풀의 크기는 JVM파라미터값을 통해\ 수정이 가능합니다.

***

    -Djava.util.concurrent.ForkJoinPool.common.parallelism=5

We use the same example map for demonstrating purposes but this time we work upon the concrete implementation `ConcurrentHashMap` instead of the interface `ConcurrentMap`, so we can access all public methods from this class:

위의 예제에서 활용했던 map을 이번에는 내부의 메서드를 활용하기 위해 인터페이스가 아닌 구현체 `ConcurrentHashMap`으로 선언하겠습니다.

***

    ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

Java 8 introduces three kinds of parallel operations: `forEach`, `search` and `reduce`. Each of those operations are available in four forms accepting functions with keys, values, entries and key-value pair arguments.

자바8에서 3가지 유형의 병렬처리작업(`forEach`, `search`, `reduce`)이 소개되었습니다. 각각은 파라미터를 받아들이는 4가지 유형(key, value, entry, key-value)이 존재합니다.

***

All of those methods use a common first argument called `parallelismThreshold`. This threshold indicates the minimum collection size when the operation should be executed in parallel. E.g. if you pass a threshold of 500 and the actual size of the map is 499 the operation will be performed sequentially on a single thread. In the next examples we use a threshold of one to always force parallel execution for demonstrating purposes.

이 메서드들은 공통적으로 첫번째 아규먼트로 `parallelismThreshold`를 호출합니다. 이 threshold는  병렬작업이 실행될 때 최소 컬렉션 사이즈를 지정합니다. 예를 들어 threshold가 500으로 설정하였다면 실제 맵의 사이즈는 499가 됩며, 단일 스레드에서 순서대로 실행될 것입니다. 다음의 예시에서는 threshold를 하나만 설정하여 병렬작업이 항상 실행되도록 설정합니다. 

***

### ForEach

The method `forEach()` is capable of iterating over the key-value pairs of the map in parallel. The lambda expression of type `BiConsumer` is called with the key and value of the current iteration step. In order to visualize parallel execution we print the current threads name to the console. Keep in mind that in my case the underlying `ForkJoinPool` uses up to a maximum of three threads.

### ForEach

메서드 `forEach()`는 맵 내부의 key-value를 병렬로 순회하며 작업을 할 수 있는 기능을 제공합니다. `BiConsumer`타입의 람다표현식에서는 각 key-value을 활용하여 작업을 수행하게 됩니다. 다음 예제에서는 병렬작업이 수행되는 것을 시각화할 목적으로 현재 스래드의 이름을 콘솔에 출력합니다. 제 예제에서는 `ForkJoinPool`의 최댓값이 3이라는 사실을 기억해주세요.

***

    map.forEach(1, (key, value) ->
        System.out.printf("key: %s; value: %s; thread: %s\n",
            key, value, Thread.currentThread().getName()));
    
    // key: r2; value: d2; thread: main
    // key: foo; value: bar; thread: ForkJoinPool.commonPool-worker-1
    // key: han; value: solo; thread: ForkJoinPool.commonPool-worker-2
    // key: c3; value: p0; thread: main

### Search

The method `search()` accepts a `BiFunction` returning a non-null search result for the current key-value pair or `null` if the current iteration doesn't match the desired search criteria. As soon as a non-null result is returned further processing is suppressed. Keep in mind that `ConcurrentHashMap` is unordered. The search function should not depend on the actual processing order of the map. If multiple entries of the map match the given search function the result may be non-deterministic.


### Search

`search()`메서드는 파라미터로 `BiFunction`을 받습니다. 람다식에는 결과값에 대한 key-value를 반환하며 결과가 없을 경우 `null`을 반환하도록 작성합니다. 람다식에서 `null`이 아닌 값이 나오는 경우 즉시 모든 프로세스를 정지합니다. `ConcurrentHashMap`에서 실제 정렬순서와 무관합니다. 따라서, 맵에 여러개의 검색결과가 존재할 경우에 결과값이 항상 동일하지 않을것 입니다.

***

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

`searchValues()`메서드는 value값만 검색시 필요한 상황에서 사용합니다.

***

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

### Reduce

`reduce()`메서드에는 `BiFunction`타입의 람다표현식 두개를 선언하여 사용합니다. 첫번째 function은 key-value 쌍을 단일 value로 변환시키는 역할을 합니다. 두번째 function은 이 변환된 값을 조합하여 하나의 값으로 변경합니다. 만약 null이 발생하는 경우는 이 연산에서 무시됩니다.

***

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

튜토리얼의 3번째이자 마지막 파트는 여기까지입니다. 코드샘플은 [github](https://github.com/winterbe/java8-tutorial)에서 확인할 수 있습니다. 직접 포크를 한 뒤 실행해보시길 추천드립니다.




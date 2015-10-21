# 자바8 Concurrency 튜토리얼: Atomic 변수/ConcurrentMap

이 글은 [Benjamin Winterberg](http://winterbe.com/about/)가 작성한 [Java 8 Concurrency Tutorial](http://winterbe.com/posts/2015/04/07/java8-concurrency-tutorial-thread-executor-examples/)을 번역한 글입니다. 저자의 허락을 얻어 번역하여 공개합니다.

***

자바8을 활용한 멀티스레드 프로그래밍 튜토리얼의 3번째 파트에 오신 것을 환영합니다. 이번 튜토리얼은 Concurrency API의 Atomic 변수와 ConcurrentMap에 대해서 함께 알아보도록 합시다. 두 영역 모두 자바 8에서 도입된 람다와 함수형 프로그래밍을 통해서 많은 개선이 이루어졌습니다. 아래에서 설명하는 예제들은 모두 자바8을 기준으로 작성하였습니다.

[[MORE]]

- Part 1: [Thread/Executor](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)
- Part 2: [Synchronize/Lock](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)
- Part 3: Atomic Variables and ConcurrentMap

튜토리얼에서 작성할 예제코드에서는 단순함을 위해서 두 개의 메서드 `sleep(seconds)`과 `stop(executor)`을 정의합니다. `sleep(seconds)`메서드는 입력되는 초만큼 스레드를 정지합니다. `stop(executor)`메서드가 호출될 경우 파라미터의 ExecutorService를 정지합니다. 이 메서드들은 [여기](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)에서 확인할 수 있습니다.

## AtomicInteger

패키지 `java.concurrent.atomic`에서는 Atomic 연산에 관련된 클래스들을 찾을 수 있습니다. Atomic 연산은 [이전 튜토리얼](http://devsejong.tumblr.com/post/127226710107/자바8-concurrency-튜토리얼-synchronizelock)에서 이야기 하였던 `synchronized` 키워드나 락을 사용하지 않고도 스레드 안전하다는 것을 의미합니다.

내부적으로 Atomic 클래스들은 [compare-and-swap(CAS)](https://en.wikipedia.org/wiki/Compare-and-swap)을 사용하고 있습니다. 현시대 대부분의 CPU에서 Atomic 명령을 바로 사용할 수 있도록 지원합니다. 이러한 명령문들은 이전 튜토리얼에서 소개한 락을 활용한 동기화 방법보다 훨씬 성능이 좋습니다. 하나의 변수를 동시적으로 변경해야 하는 경우에는 Atomic 클래스를 사용하는 것을 권장합니다.

이제 Atomic 클래스 중 하나인 `AtomicInteger`를 사용한 예제코드를 잠깐 살펴봅시다.

    AtomicInteger atomicInt = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(atomicInt::incrementAndGet));
    
    stop(executor);
    
    System.out.println(atomicInt.get());    // => 1000

`AtomicInteger`를 `Integer`대신 사용하여 숫자가 동시에 변경되는 상황에서 변수의 접근을 동기화하지 않고도 스레드에 안전하게 처리할 수 있습니다. `incrementAndGet()`메서드는 멀티스레드에서 안전하게 호출할 수 있는 Atomic 연산입니다.

`AtomicInteger`클래스에는 다양한 종류의 Atomic 연산 메서드를 제공합니다. `updateAndGet()`메서드는 다양한 산술 표현식을람다표현식으로 사용할 수 있습니다.

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

메서드 `accumulateAndGet()`은  `IntBinaryOperator`타입의 람다표현식을 사용할 수 있습니다. 아래는 0부터 1000까지 비동기를 사용하여 합계를 계산하는 코드입니다.

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

`java.concurrent.atomic` 패키지에서 선언된 Atomic 클래스는 [AtomicBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html), [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html), [AtomicReference](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)가 있습니다.

## LongAdder

`LongAdder`클래스는 연속적으로 값을 더하는 상황에서 `AtomicLong`를 대체하여 사용합니다.

    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 1000)
        .forEach(i -> executor.submit(adder::increment));
    
    stop(executor);
    
    System.out.println(adder.sumThenReset());   // => 1000

`LongAdder`는 메서드 `add()`와 `increment()`를 제공합니다. 다른 Atomic클래스와 동일하게 내부의 메서드들은 멀티 스레드 환경에서 안전합니다. 다른 Atomic 클래스와 다른점은 하나의 결괏값을 클래스에서 유지하고  있는 게 아니라 내부적으로 변수값 세트를 가져 스레드 간 경쟁상태가 일어나지 않도록 설계되어있습니다. 메서드를 실제값은 `sum()`이나 `sumThenReset()` 메서드가 호출 될 때 계산됩니다.

`LongAdder` 클래스는 주로 여러 스레드에서 수정작업이 발생하며 읽기작업보다는 쓰기작업이 많을 경우에 유리합니다. 웹서버에서 요청이 얼마만큼 발생했는지를 측정하는 경우와 같이 통계 데이터를 계산할 경우에 사용할 수 있을 것 입니다. 대신에 `LongAdder` 클래스는 각 스레드에서의 변수 셋을 각각 관리하므로 상대적으로 많은 메모리를 차지한다는 단점이 있습니다. 

## LongAccumulator

`LongAccumulator`은 `LongAdder`보다는 일반적인 상황에서 사용할 수 있습니다. 간단한 덧셈작업만을 제공하는 `LongAdder`와는 달리 `LongAccumulator`는 `LongBinaryOperator`타입의 람다 표현식으로  연산작업을 할 수 있습니다.
    
    LongBinaryOperator op = (x, y) -> 2 * x + y;
    LongAccumulator accumulator = new LongAccumulator(op, 1L);
    
    ExecutorService executor = Executors.newFixedThreadPool(2);
    
    IntStream.range(0, 10)
        .forEach(i -> executor.submit(() -> accumulator.accumulate(i)));
    
    stop(executor);
    
    System.out.println(accumulator.getThenReset());     // => 2539

함수 `2 * x + y`를 사용하여 `LongAccumulator`객체를 초기화하였습니다. `accumulate(i)`가 호출될 때마다 현재의 결과값과 `i`값이 람다 표현식의 값으로 전달될 것 입니다.

`LongAccumulator` 클래스는 `LongAdder`와 마찬가지로 내부적으로  변수 셋을 가지고 있어 스레드간 경쟁상태를 최소화 합니다.

## ConcurrentMap

인터페이스 `ConcurrentMap`은 기존의 `Map`인터페이스를 확장하여 멀티스레드 환경에서 사용할 수 있는 컬렉션 타입입니다. 자바 8에서는 이 인터페이스에 함수형 프로그래밍의 기능이 들어간 새로운 메서드가 정의되어있습니다.

다음의 코드에서 앞으로 예제에서 사용할 맵을 정의합니다.

    ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

`forEach()` 메서드는 `BiConsumer`타입의 람다 표현식을 사용합니다. 이 functional object는 key/value 형태의 파라미터로 값을 Map에 전달합니다. `forEach()` 메서드는 기존 for-each문과 동일하게 동작합니다.

    map.forEach((key, value) -> System.out.printf("%s = %s\n", key, value));

메서드 `putIfAbsent()`는 map에서 해당 key의 값이 존재하지 않을 경우에만 새 값을 추가합니다. `ConcurrentHashMap`에서는 `put`과 마찬가지로 스레드 안전한 메서드이므로 여러 스레드에서 동시에 접근하더라도 동기화 로직을 추가할 필요가 없습니다.

    String value = map.putIfAbsent("c3", "p1");
    System.out.println(value);    // p0

`getOrDefault()`메서드는 해당하는 key에 대한 값을 반환하며 key의 값이 존재하지 않을 경우 두번째 파라미터로 설정한 기본값을 반환합니다.

    String value = map.getOrDefault("hi", "there");
    System.out.println(value);    // there

메서드 `replaceAll()`은 `BiFunction`타입의 람다 표현식을 파라미터로 받습니다. `BiFunctions`에는 두 개의 값, 즉 key-value를  파라미터로 가지며 연산결과로 하나의 값을 반환합니다. 이 메서드가 호출될 경우 key/value를 활용하여 현재 키값에 새로운 값을 부여할 수 있습니다.

    map.replaceAll((key, value) -> "r2".equals(key) ? "d3" : value);
    System.out.println(map.get("r2"));    // d3

map의 전체값을 replace하는 대신 `compute()`를 사용할 경우 단일값을 변경할 수 있습니다. 이 메서드는 첫 번째 파라미터로 key를 입력받으며, 두번째 파라미터로는 `BiFunctions`타입의 람다표현식을 받습니다.

    map.compute("foo", (key, value) -> value + value);
    System.out.println(map.get("foo"));   // barbar

`compute()`와 비슷한 기능을 가진 메서드가 두 개 더 존재합니다. `computeIfAbsent()` 는 값이 존재하지 않을 경우에, `computeIfPresent()`는 값이 존재할 경우에 작업이 수행됩니다.

마지막으로 메서드 `merge()`는 기존의 값과 새로운 값을 합쳐 연산을 할 경우에 사용할 수 있습니다. merge의 첫 번째 파라미터는 key를 두번째 파라미터는 새로운 값, 세 번째는 연산작업이 되는 BiFunction타입의 람다표현식을 입력할 수 있습니다.

    map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
    System.out.println(map.get("foo"));   // boo was bar


## ConcurrentHashMap

위에서 이야기했던 `ConcurrentMap` 인터페이스의 메서드들은 구현체를 활용하여 사용할 수 있습니다. 중에서 가장 많이 활용되는 구현체 `ConcurrentHashMap`에서는 추가로 병렬작업 메서드들을 더 확인할 수 있습니다.

Parallel Stream과 마찬가지로 이 메서드들은 자바8의 `ForkJoinPool.commonPool()`을 사용할 수 있습니다. 병렬처리의 프리셋으로 사용되는 이 스레드 풀은 활용 가능한 코어의 숫자에 따라서 풀의 크기가 자동으로 정해집니다. 4개의 CPU코어를 가지고 있는 제 컴퓨터에서는 스레드 풀의 크기가 3으로 나오고 있습니다.
    System.out.println(ForkJoinPool.getCommonPoolParallelism());  // 3

참고로 `ForkJoinPool` 스레드 풀의 크기는 JVM파라미터값을 통해\ 수정이 가능합니다.

    -Djava.util.concurrent.ForkJoinPool.common.parallelism=5

위의 예제에서 활용했던 map을 이번에는 내부의 메서드를 활용하기 위해 인터페이스가 아닌 구현체 `ConcurrentHashMap`으로 선언하겠습니다.

    ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
    map.put("foo", "bar");
    map.put("han", "solo");
    map.put("r2", "d2");
    map.put("c3", "p0");

자바8에서 3가지 유형의 병렬처리작업(`forEach`, `search`, `reduce`)이 소개되었습니다. 각각은 파라미터를 받아들이는 4가지 유형(key, value, entry, key-value)이 존재합니다.


//번역 수정 필요.
이 메서드들은 공통으로 첫 번째 아규먼트로 `parallelismThreshold`를 호출합니다. 이 threshold는  병렬작업이 실행될 때 최소 컬렉션 크기를 지정합니다. 예를 들어 threshold가 500으로 설정하였다면 실제 맵의 크기는 499가 되며, 단일 스레드에서 순서대로 실행될 것입니다. 다음의 예시에서는 threshold를 하나만 설정하여 병렬작업이 항상 실행되도록 설정합니다. 

### ForEach

메서드 `forEach()`는 맵 내부의 key-value를 병렬로 순회하며 작업을 할 수 있는 기능을 제공합니다. `BiConsumer`타입의 람다표현식에서는 각 key-value을 활용하여 작업을 수행하게 됩니다. 다음 예제에서는 병렬작업이 수행되는 것을 시각화할 목적으로 현재 스레드의 이름을 콘솔에 출력합니다. 제 예제에서는 `ForkJoinPool`의 최댓값이 3이라는 사실을 기억해주세요.

    map.forEach(1, (key, value) ->
        System.out.printf("key: %s; value: %s; thread: %s\n",
            key, value, Thread.currentThread().getName()));
    
    // key: r2; value: d2; thread: main
    // key: foo; value: bar; thread: ForkJoinPool.commonPool-worker-1
    // key: han; value: solo; thread: ForkJoinPool.commonPool-worker-2
    // key: c3; value: p0; thread: main

### Search

`search()`메서드는 파라미터로 `BiFunction`을 받습니다. 람다식에는 결과값에 대한 key-value를 반환하며 결과가 없으면 `null`을 반환하도록 작성합니다. 람다식에서 `null`이 아닌 값이 나오는 경우 즉시 모든 프로세스를 정지합니다. `ConcurrentHashMap`에서 실제 정렬순서와 무관합니다. 따라서 맵에 여러 개의 검색결과가 존재할 경우에 결괏값이 항상 같지 않을 것 입니다.

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

`searchValues()`메서드는 value값만 검색 시 필요한 상황에서 사용합니다.

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

`reduce()`메서드에는 `BiFunction`타입의 람다표현식 두 개를 선언하여 사용합니다. 첫 번째 function은 key-value 쌍을 단일 value로 변환시키는 역할을 합니다. 두 번째 function은 이 변환된 값을 조합하여 하나의 값으로 변경합니다. 만약 null이 발생하는 경우는 이 연산에서 무시됩니다.

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

튜토리얼의 3번째이자 마지막 파트는 여기까지입니다. 예제 코드는 [github](https://github.com/winterbe/java8-tutorial)에서 확인할 수 있습니다. 직접 포크를 한 뒤 실행해보시길 권장합니다.



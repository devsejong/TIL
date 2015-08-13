# 자바8 Concurrency 튜토리얼: Thread/Executor

이 가이드는 코드를 통해서 자바 8의 [concurrent(이하 병렬) 프로그래밍](https://en.wikipedia.org/wiki/Concurrent_computing)을 쉽게 이해하는 것을 목표로 만들어졌습니다. 자바의 [Concurrency API](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html)를 설명하는 튜토리얼 중 첫 번째 파트이며, 앞으로 15분 정도의 분량으로 thread, task, executor service를 사용하여 비동기를 어떻게 처리하는지에 대해서 이야기 할 것 입니다.

- 파트 1: Thread/Executor
- 파트 2: Synchronization/Lock
- 파트 3: Atomic Variables/ConcurrentMap

Concurrency API는 자바5에서 처음으로 소개되었으며, 새로운 자바 버전이 발표될 때마다 조금씩 보강되어 왔습니다. 아래의 가이드에 나오는 이러한 주요 개념들은 오래된 자바에서도 동일하게 사용할 수 있으나, 이 가이드는 자바8을 기준으로 설명하였습니다. 람다 표현식과 같이 자바8에 소개된 새로운 문법을 많이 사용하였으므로 문법에 친숙하지 않을 경우 우선 람다에 대한 [튜토리얼](http://winterbe.com/posts/2014/03/16/java-8-tutorial/) 먼저 참조하시기 바랍니다.

## Thread와 Runnable

우리가 사용하는 모든 현대적인 OS들은 [프로세스](https://ko.wikipedia.org/wiki/프로세스)와 [스레드](https://ko.wikipedia.org/wiki/스레드)를 사용한 병렬 처리를 지원합니다. 프로그램의 인스턴스인 프로세스는 일반적으로 서로 독립된 상태로 동작합니다. 예를 들어 자바 프로그램을 실행할 경우 OS는 새로운 프로세스를 생성하며, 이 프로세스는 다른 프로그램과 병렬로 실행됩니다. 이 프로세스들 내부에서 동시에 코드를 실행할 수 있는 스레드를 사용할 수 있으며. CPU에서 가용 코어를 모두 사용할 수 있게 됩니다.

자바는 JDK 1.0 부터 [스레드](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)를 지원해왔습니다. 새로운 스레드를 시작하기 전에 스레드 내부에서 실행되는 코드(이를 task라고 칭함)를 만들어야 합니다. 이 task는 `Runnable`을 상속함으로써 만들 수 있습니다. `Runnable`은 아래와 같이 반환 값과 아규먼트가 없는 매서드 `run()`을 가지고 있는 Functional Interface입니다.


	Runnable task = () -> {
	    String threadName = Thread.currentThread().getName();
	    System.out.println("Hello " + threadName);
	};

	task.run();

	Thread thread = new Thread(task);
	thread.start();

	System.out.println("Done!");

위의 코드에서는 `Runnable` 인터페이스에서 람다 표현식을 사용하여 현재 스레드 이름을 콘솔에 출력하였습니다. 우선 메인 스레드에서 새로운 스레드를 실행하기 전에 `Runnable`을 바로 실행하였습니다.

콘솔에서 출력되는 결과는 다음과 같을 것입니다.

	Hello main
	Hello Thread-0
	Done!

또는 아래처럼 나올 수 도 있습니다.

	Hello main
	Done!
	Hello Thread-0

위의 코드는 병렬로 실행되므로, 마지막에 위치하는 "done"이 Runnable을 활용한 스레드보다 먼저 출력될지 나중에 출력될지를 알 수 없습니다. 정렬순서는 결정할 수 없으므로, 병렬프로그래밍은 커다란 애플리케이션을 만들 경우 복잡한 작업입니다.

스레드에서는 특정한 시간 동안 스레드를 멈출 수 있는 sleep을 둘 수 있습니다. 다음과 같이 긴 작업을 시뮬레이션할 경우에 sleep을 편리하게 사용할 수 있습니다.

 	Runnable runnable = () -> {
	    try {
	        String name = Thread.currentThread().getName();
	        System.out.println("Foo " + name);
	        TimeUnit.SECONDS.sleep(1);
	        System.out.println("Bar " + name);
	    }
	    catch (InterruptedException e) {
	        e.printStackTrace();
	    }
	};

	Thread thread = new Thread(runnable);
	thread.start();

위의 코드를 실행할 경우, 첫 번째와 두 번째 출력사이에 1초간 딜레이가 발생하는 것을 확인할 수 있을 것 입니다. `TimeUnit`은 시간의 단위를 기준으로 작업을 진행할 수 있도록 도와주는 enum입니다. `Thread.sleep(1000)`과 같은 역할을 합니다.

`Thread` 클래스를 사용하는 방식은 매우 지루하며, 에러가 나기 쉬운 구조입니다. 때문에 2004년 후반 자바 5가 릴리즈에서 **Concurrency API**가 소개되었습니다. 이 API들은 `java.util.concurrent`패키지에서 찾을 수 있으며, 병렬 프로그램을 다룰 수 있는 유용한 클래스가 포함되어 있습니다. 새로운 자바가 발표될 때 마다 이 API는 계속 보강되어 왔으며, 자바 8에서는 병렬 프로그램을 다룰 수 있는 새로운 클래스와 메서드가 추가되었습니다.

아래에서는 이 Concurrency API의 가장 중요한 부분 중 하나인 `ExecutorService`에 대해서 자세히 알아보도록 합니다.

## Executor

Concurrency API에서는 직접 스레드를 사용할 수 있는 높은 레벨의 `ExecutorService`가 포함되어 있습니다. Executor는 비동기 작업을 실행할 수 있고, 스레드 풀을 관리할 수 있는 기능을 제공합니다. 그 덕분에 직접 스레드를 새롭게 생성하지 않아도 됩니다. 내부의 풀에 소속된 모든 스레드들은 재사용될 수 있으므로, 우리는 애플리케이션 생명주기 내에서 하나의 Executor service를 사용하여 많은 동행 작업을 처리할 수 있습니다.

아래는 Executor를 사용한 예제입니다.

    ExecutorService executor = Executors.newSingleThreadExecutor();
    executor.submit(() -> {
        String threadName = Thread.currentThread().getName();
        System.out.println("Hello " + threadName);
    });

    // => Hello pool-1-thread-1

Excutor에서 제공하는 클래스에서는 다양한 형태의 Executor Service를 만들 수 있도록 팩토리 메서드를 제공합니다. 위 예제에서는 하나의 스레드 풀을 사용하도록 지정하였습니다.

위의 `Thread`를 사용한 코드와 같은 결과가 나온 것처럼 보이지만 중요한 차이점이 있습니다. 자바 프로세스가 절대 종료되지 않는다는 것입니다. Executor는 명시적으로 종료되지 않습니다. (새로운 작업을 받을 수 있는 대기 상태로 유지됩니다.

`ExecutorService`는 종료를 위해서 두 가지 메서드를 제공합니다. `shutdown()`은 현재 진행 중인 작업들이 끝나는 실행 중것을 기다린 뒤, `shutdownNow()`는 모든 작업을 한꺼번에 멈춘 뒤 Executor를 종료합니다.

아래는 Executor를 종료하는 전형적인 방법입니다.

	try {
	    System.out.println("attempt to shutdown executor");
	    executor.shutdown();
	    executor.awaitTermination(5, TimeUnit.SECONDS);
	}
	catch (InterruptedException e) {
	    System.err.println("tasks interrupted");
	}
	finally {
	    if (!executor.isTerminated()) {
	        System.err.println("cancel non-finished tasks");
	    }
	    executor.shutdownNow();
	    System.out.println("shutdown finished");
	}

Executor는 먼저 종료 신호를 보내며, 지정된 시간만큼 현재 진행 작업이 끝나도록 대기합니다. 대기 시간 5초가 지나면, Executor는 모든 실행 중 태스크에 interrupt 예외를 발생시켜 강제종료 시키며 최종적으로 Executor를 종료합니다.

### Callable과 Future

Executor는 `Runnable`뿐만 아니라 `Callable`을 사용하여 task를 만들 수 있습니다. `Callable`은 `Runnalbe`과 마찬가지로 functional interface이지만, 반환 값을 가지고 있다는 차이점이 있습니다.

아래의 예제에서는 람다표현식을 통해서, 1초간 정지한 다음 Integer값을 반환하는 예제입니다.

***

	Callable<Integer> task = () -> {
	    try {
	        TimeUnit.SECONDS.sleep(1);
	        return 123;
	    }
	    catch (InterruptedException e) {
	        throw new IllegalStateException("task interrupted", e);
	    }
	};

`Callable`은  `Runnalbe`과 마찬가지로 `submit()`을 사용하여 작업을 실행할 수 있습니다. 그렇다면 실행 결과는 어떻게 받아올 수 있을까요? `submit()`은 작업이 완료되는것을 기다리지 않으므로, ExecutorService는 Callable을 실행한 결과를 바로 가져올 수 없습니다. 대신에 `Future`라는 특별한 타입의 값을 반환합니다. `Future`는 실행 결과를 원하는 시간에 가져올 수 있도록 제공합니다.

	ExecutorService executor = Executors.newFixedThreadPool(1);
	Future<Integer> future = executor.submit(task);

	System.out.println("future done? " + future.isDone());

	Integer result = future.get();

	System.out.println("future done? " + future.isDone());
	System.out.print("result: " + result);

`submit()`을 사용하여 `Callable`을 실행시킨 경우, 작업이 종료되었는지 `isDone()`메서드를 사용하여 확인할 수 있습니다. 이전 코드에서 우리가 만든 `Callable`에서는 1초간 sleep을 걸어놓았기에, 처음의 `isDone()`이 호출되는 시점에서는 작업이 진행 중인 상태일 것이라고 예상할 수 있을 것입니다.

메서드 `get()`을 호출할 경우, `Callable`의 결과인 `123`이 나올 때까지 스레드를 정지합니다. 이때서야 비로소 작업이 끝나게 되고, 아래와 같은 결과를 콘솔에서 확인할 수 있습니다.

	future done? false
	future done? true
	result: 123

`Future`는 `ExecutorService`와 강하게 결합해 있습니다. 그래서 Executor를 종료한 경우, 작업이 종료되지 않은 `Future`들은 에러를 발생시킨다는 사실을 기억하셔야 합니다.

	executor.shutdownNow();
	future.get();

위의 예제는 이전의 예제와 살짝 다른 부분이 있습니다. `newFixedThreadPool(1)`을 사용하여 `ExecutorService`의 스레드 풀을 하나만 가지게끔 지정하였습니다. 이전 예제의 `newSingleThreadExecutor()`와 같은 역할을 하고 있습니다. 파라미터의 숫자를 늘림으로써 스레드 풀의 숫자를 더 늘릴 수 있을 것 입니다.

### Timeout

`future.get()`을 호출할 경우 현재 스레드는 callable을 구현한 작업이 끝날 때 까지 정지한 상태로 대기합니다. callable의 작업이 종료되지 않고 계속 실행되는 경우, 애플리케이션이 동작하지 않게 될 수도 있습니다. 이러한 경우 아래의 예제와 같이 timeout을 지정하여 대비할 수 있습니다.

	ExecutorService executor = Executors.newFixedThreadPool(1);

	Future<Integer> future = executor.submit(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
        return 123;
    }
    catch (InterruptedException e) {
        throw new IllegalStateException("task interrupted", e);
    }
	});

	future.get(1, TimeUnit.SECONDS);

위의 코드를 실행할 경우 아래와 같이 `TimeoutException`이 발생하게 됩니다.

	Exception in thread "main" java.util.concurrent.TimeoutException
    at java.util.concurrent.FutureTask.get(FutureTask.java:205)

위의 예외가 왜 발생하는지에 대해서 벌써 알고 계시리라 생각합니다. 위의 코드에서 최대 대기시간을 1초로 설정하였지만, 실제 결과가 나오기까지는 2초가 걸리기 때문입니다.

***

### invokeAll

Executor는 여러 개의 `Callable`을 한 번에 실행할 수 있는 `invokeAll()`을 지원합니다. 이 메서드는 `Callable`의 컬렉션을 받을 수 있으며 리스트의 형식으로 `Future`타입의 값을 반환합니다.

	ExecutorService executor = Executors.newWorkStealingPool();

	List<Callable<String>> callables = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

	executor.invokeAll(callables)
    .stream()
    .map(future -> {
        try {
            return future.get();
        }
        catch (Exception e) {
            throw new IllegalStateException(e);
        }
    })
    .forEach(System.out::println);

위 예제에서는 자바8의 stream을 활용하여 `invokeAll`의 모든 결과의 반환 값이 나올 때 까지 실행하도록 하였습니다. 각각의 future들의 리턴값을 콘솔에 출력하도록 매핑하였습니다. 자바8의 stream문법에 익숙하지 않은 경우 [자바8 스트림 튜토리얼](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)을 참조하시기 바랍니다.

`Callable`을 실행할 수 있는 다른 방법은 `invokeAny()`를 사용하는 것입니다. `Future`타입의 컬렉션이 반환되는 `invokeAll()`과는 달리 이 메서드는 작업 중 하나의 값이 완료될 때 까지 기다린 뒤 결과를 반환하고, 다른 작업을 종료시킵니다.

위의 동작을 테스트해보기 위해서 callable의 실행시간을 다르게 설정해보도록 하겠습니다. 아래는 callable을 호출할 경우 지정된 시간 동안 멈춘 뒤 결과를 반환하도록 작성된 코드입니다.

	Callable<String> callable(String result, long sleepSeconds) {
    return () -> {
        TimeUnit.SECONDS.sleep(sleepSeconds);
        return result;
    };
}

위에서 만든 callable을 활용하여 1초부터 3초까지 각기 다른 시간이 걸리도록 설정하였습니다. Executor에서 `invokeAny()`를 통해서 실행될 경우 가장 빠른 실행결과가 나오게 됩니다. 아래의 경우에서는 1초를 설정한 `task2`가 출력됩니다.

	ExecutorService executor = Executors.newWorkStealingPool();

	List<Callable<String>> callables = Arrays.asList(
    callable("task1", 2),
    callable("task2", 1),
    callable("task3", 3));

	String result = executor.invokeAny(callables);
	System.out.println(result);

	// => task2

위의 예제에서는 `newWorkStealingPool()`를 사용하여 Executor를 생성하고 있습니다. 이 팩터리 메서드는 자바8에서부터 지원되며, `ForkJoinPool`을 Executor 타입으로 반환합니다. 기존의 고정된 스레드 풀을 지정하는 것과는 다르게 [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)은 프로그램을 실행하는 CPU 코어의 숫자를 기준으로 스레드 풀을 생성합니다.

`ForkJoinPool`은 자바7에서 추가되었습니다. 더욱 자세한 이야기는 이어지는 튜토리얼에서 자세하게 설명하도록 하겠습니다. 이제 이 튜토리얼의 마지막인 `Scheduled Executor`에 대해서 알아보도록 합시다. 

## ScheduledExecutor

지금까지는 Executor를 활용하여 일회성 작업들을 어떻게 만들고 실행하는지에 대해서 이야기했습니다. 만약 주기적으로 여러 번 실행되어야 하는 작업이 필요하다면 scheduled 스레드 풀을 사용하여 구현할 수 있습니다.

`ScheduledExecutorService`을 사용하면, 주기적으로 또는 작업이 끝난 뒤 일정 시간 후 다시 작업하는 등의 스케줄링 설정이 가능합니다.

아래의 예제에서는 초기의 딜레이 3초 뒤 작업이 진행됩니다.

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());
	ScheduledFuture<?> future = executor.schedule(task, 3, TimeUnit.SECONDS);

	TimeUnit.MILLISECONDS.sleep(1337);

	long remainingDelay = future.getDelay(TimeUnit.MILLISECONDS);
	System.out.printf("Remaining Delay: %sms", remainingDelay);

스케줄링 작업의 결과로는 조금은 독특한 `Future` 타입인 `ScheduledFuture`가 생성됩니다. 이 클래스에서는 `getDelay()`를 호출하여 얼마만큼의 딜레이가 남았는지 확인할 수 있습니다. 이 딜레이가 완료된 후에 작업이 시작되게 됩니다.

주기적으로 실행되는 스케쥴 작업을 위해서 Executor에서는 두 개의 메서드 `scheduleAtFixedRate()`과 `scheduleWithFixedDelay()`를 제공합니다. 첫 번째 메서드는 고정된 시간을 기준으로 작업을 진행합니다. 아래의 코드에서는 매초 현재 시간이 콘솔에 출력됩니다.

***

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> System.out.println("Scheduling: " + System.nanoTime());

	int initialDelay = 0;
	int period = 1;

	executor.scheduleAtFixedRate(task, initialDelay, period, TimeUnit.SECONDS);

추가적으로 이 메서드는 작업이 시작되기 전에 초기 딜레이 값을 지정할 수 있습니다. 

기억해야 하는 사실은 `scheduleAtFixedRate()`는 실제 작업이 얼마만큼 걸리는지에 대해서 고려하지 않는다는 것입니다. 만약 반복 주기를 1초라고 지정하였지만, 실제 작업이 2초가 걸린다면 스레드 풀은 곧 차버리게 될 것입니다.

이러한 경우라면 `scheduleWithFixedDelay()`를 사용하는 것을 고려해 보아야 합니다. 이 메서드는 위의 상황에서 대체재가 될 것입니다. `scheduleAtFixedRate()`와 다른 점은 작업이 종료된 후부터 지정된 시간을 기다린 후 다음 작업을 시작한다는 점입니다.

	ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

	Runnable task = () -> {
    try {
        TimeUnit.SECONDS.sleep(2);
        System.out.println("Scheduling: " + System.nanoTime());
    }
    catch (InterruptedException e) {
        System.err.println("task interrupted");
    }
};

	executor.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);

위 예제에서는 작업이 종료된 후 다음 작업이 시작되기 전까지 딜레이를 1초로 지정하였습니다. 초기 딜레이는 0이며, 작업은 2초가 걸립니다. 그러므로 위 예제에서는 3초를 주기로 작업이 진행되는 것을 확인할 수 있을 것입니다. 이처럼 `scheduleWithFixedDelay()`는 작업이 얼마만큼 걸릴지 모르는 상황에서 유용하게 사용할 수 있습니다.

이상으로 concurrency 튜토리얼의 첫 번째 파트를 마칩니다. 위의 코드를 스스로 실행해보시는 것을 강력하게 권장합니다. 또한, 예제에 대한 샘플은 [Github](https://github.com/winterbe/java8-tutorial)에서 찾을 수 있습니다. fork와 star를 주는것에 대해서 너무 부담가지지 않으셔도 됩니다.
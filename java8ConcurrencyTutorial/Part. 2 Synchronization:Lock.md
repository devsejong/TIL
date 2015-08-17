# 자바8 Concurrency 튜토리얼: Synchronize/Lock

Welcome to the second part of my Java 8 Concurrency Tutorial out of a series of guides teaching multi-threaded programming in Java 8 with easily understood code examples. In the next 15 min you learn how to synchronize access to mutable shared variables via the synchronized keyword, locks and semaphores.

병렬 프로그래밍 튜토리얼의 두번째 파트에 오신것을 환영합니다. 이 튜토리얼는 자바8을 기반으로 한 예제코드를 통해서 쉬운 이해를 돕는걸 목표로 합니다. 앞으로 15분간 synchronized 키워드, Lock, Semaphore 를 활용하여 공유변수에 안전하게 엑세스할 수 있는 방법을 이야기할 것 입니다.

***

- Part 1: [Threads and Executors](http://winterbe.com/posts/2015/04/07/java8-concurrency-tutorial-thread-executor-examples/)
- Part 2: Synchronization and Locks
- Part 3: [Atomic Variables and ConcurrentMap](http://winterbe.com/posts/2015/05/22/java8-concurrency-tutorial-atomic-concurrent-map-examples/)


- 파트 1: [Thread/Executor](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)
- 파트 2: Synchronize/Lock
- 파트 3: Atomic Variables/ConcurrentMap

***

The majority of concepts shown in this article also work in older versions of Java. However the code samples focus on Java 8 and make heavy use of lambda expressions and new concurrency features. If you're not yet familiar with lambdas I recommend reading my [Java 8 Tutorial](http://winterbe.com/posts/2014/03/16/java-8-tutorial/) first.

앞으로 이야기할 내용의 주요 개념들은 오래된 자바에서도 동일하게 사용할 수 있으나, 이 튜토리얼은 자바8을 기준으로 코드를 작성하였습니다. 람다 표현식과 같이 자바8에 소개된 새로운 문법을 많이 사용하였으므로 문법에 친숙하지 않을 경우 우선 람다에 대한 [튜토리얼](http://winterbe.com/posts/2014/03/16/java-8-tutorial/) 먼저 참조하시기 바랍니다.

***

For simplicity the code samples of this tutorial make use of the two helper methods `sleep(seconds)` and `stop(executor)` as defined [here](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java).

튜토리얼에서 작성할 예제코드에서는 단순함을 위해서 두개의 메서드 `sleep(seconds)`과 `stop(executor)`을 정의합니다. `sleep(seconds)`는 입력되는 초만큼 스레드를 정지합니다. `stop(executor)`는 입력받은 ExecutorService를 정지하는 역할을 합니다. 이 메서드는 [여기](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)에서 찾아볼 수 있습니다.

***

## Synchronized

In the [previous tutorial](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/) we've learned how to execute code in parallel via executor services. When writing such multi-threaded code you have to pay particular attention when accessing shared mutable variables concurrently from multiple threads. Let's just say we want to increment an integer which is accessible simultaneously from multiple threads.

[이전 튜토리얼](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)에서는 Executor Service를 사용하여 동시성을 어떻게 처리하는지에 대해서 이야기하였습니다. 이러한 멀티 스레드 환경에서 각 스레드가 참조하는 공유 변수에 접근하는 코드를 작성할때에는 각별히 주의를 기울여야 합니다. integer값을 멀티 스레드로 동시에 증가시키는 코드를 예시로 어떠한 문제점이 생기는지에 대해서 자세하게 살펴보도록 합시다.

***

We define a field `count` with a method `increment()` to increase count by one:

아래와 같이 필드 `count`와, 1씩 `count`를 증가시키는 메서드 `increment()`를 정의하였습니다.

***

	int count = 0;

	void increment() {
	    count = count + 1;
	}
	
When calling this method concurrently from multiple threads we're in serious trouble:

이 메서드를 멀티 스레드 환경에서 병렬로 호출할 경우 심각한 문제가 생깁니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);

	IntStream.range(0, 10000)
	    .forEach(i -> executor.submit(this::increment));

	stop(executor);

	System.out.println(count);  // 9965

Instead of seeing a constant result count of 10000 the actual result varies with every execution of the above code. The reason is that we share a mutable variable upon different threads without synchronizing the access to this variable which results in a [race condition](http://en.wikipedia.org/wiki/Race_condition).

위의 코드를 실행할 경우 예상하는 값 10000이 아니라 매번 다른 값이 출력되는 것을 확인할 수 있을 것입니다. 이는 서로 다른 스레드가 공유변수에 동기화되지 않은채로 접근하기 때문입니다. 이러한 현상을 경쟁상태([race condition](https://ko.wikipedia.org/wiki/경쟁_상태))라고 부릅니다.

***

Three steps have to be performed in order to increment the number: (i) read the current value, (ii) increase this value by one and (iii) write the new value to the variable. If two threads perform these steps in parallel it's possible that both threads perform step 1 simultaneously thus reading the same current value. This results in lost writes so the actual result is lower. In the above sample 35 increments got lost due to concurrent unsynchronized access to count but you may see different results when executing the code by yourself.

위의 코드에서 숫자를 1 증가시키는 로직은 다음과 같이 3번의 과정을 거쳐 진행됩니다. (1)현재의 값을 읽는다. (2)읽은값의 숫자에 1을 더한다. (3)결과를 변수에 설정한다. 위 과정을 거쳐 실행될 경우 두 스레드가 (1)의 과정에서 변수를 동시에 읽어 같은 값을 가져올 가능성이 있습니다. 그렇기 때문에 실제 실행결과 값이 10000보다 작은 숫자가 나오는 것입니다. 위 코드에서는 35만큼이 동기화되지 않은 접근 때문에 누락되었습니다.

***

Luckily Java supports thread-synchronization since the early days via the `synchronized` keyword. We can utilize synchronized to fix the above race conditions when incrementing the count:

다행히도 자바 초기버전부터 지원하는  `synchronized` 키워드를 활용하여 스레드 환경에서 동기화된 접근이 가능합니다. 1씩 값을 증가시키는 위의 경쟁상태 코드를 고쳐보도록 하겠습니다.

***

	synchronized void incrementSync() {
	    count = count + 1;
	}

When using `incrementSync()` concurrently we get the desired result count of 10000. No race conditions occur any longer and the result is stable with every execution of the code:

이제 `incrementSync()`를 병렬로 사용하더라도 우리가 예상했던 결과값 10000을 얻을 수 있습니다. 매번 실행을 하더라도 더이상 경쟁상태로 인한 문제가 생기지 않는 것을 확인할 수 있습니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);

	IntStream.range(0, 10000)
	    .forEach(i -> executor.submit(this::incrementSync));

	stop(executor);

	System.out.println(count);  // 10000

The `synchronized` keyword is also available as a block statement.

`synchroized` 키워드는 코드 블록에서도 직접 사용할 수 있습니다.

***

	void incrementSync() {
	    synchronized (this) {
	        count = count + 1;
	    }
	}

Internally Java uses a so called [monitor also known as monitor lock](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html) or intrinsic lock in order to manage synchronization. This monitor is bound to an object, e.g. when using synchronized methods each method share the same monitor of the corresponding object.

//내가 이해를 못하고 있음.

자바 내부적으로 [Monitor Lock](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)을 동기처리를 위해서 사용합니다. 또한 위와 같이 `synchronized` 키워드를 활용한 락 방식을 암묵적인 락(Intrinsic lock)이라고 부릅니다. 자바에서는 각 객체마다 모니터가 하나씩 존재합니다. synchronized 메서드가 동시에 호출될 경우 해당 객체의 동일한 모니터를 공유하여 하나의 스레드만 메서드를 사용할 수 있도록 만듭니다.

***

All implicit monitors implement the reentrant characteristics. Reentrant means that locks are bound to the current thread. A thread can safely acquire the same lock multiple times without running into deadlocks (e.g. a synchronized method calls another synchronized method on the same object).


//쪼금있다가 다시 해보자.

모든 암시적 모니터락은 [Reentrant](https://en.wikipedia.org/wiki/Reentrancy_(computing)) 특성을 가지고 있습니다. Reentrant란 다시 말해서 현재 스레드에 Lock이 결합되었다라는 뜻입니다. 스레드에서는 동일한 Lock을 데드락에 빠지는 일 없이 안전하게 가져올 수 있습니다. 동일한 객체에서 synchronized메서드는 다른 synchronized 메서드를 부를 수 있습니다.

***

## Locks

Instead of using implicit locking via the `synchronized` keyword the Concurrency API supports various explicit locks specified by the `Lock` interface. Locks support various methods for finer grained lock control thus are more expressive than implicit monitors.

## Lock

`synchronized` 키워드를 사용하는 대신에 Concurrency API 에서 지원하는 다양한 `Lock` 인터페이스를 활용할 수 있습니다. Lock 인터페이스에서는 각 시나리오별로 필요한 다양한 메서드를 제공합니다. 암묵적인 모니터락보다는 비용이 비싸다라고 볼 수 있을것 입니다.

***

Multiple lock implementations are available in the standard JDK which will be demonstrated in the following sections.

다양한 Lock은 표준 JDK에서 사용할 수 있습니다. 앞으로 나올 섹션에서 더욱 자세하게 이야기해 나가도록 하겠습니다.

***

### ReentrantLock

The class `ReentrantLock` is a mutual exclusion lock with the same basic behavior as the implicit monitors accessed via the `synchronized` keyword but with extended capabilities. As the name suggests this lock implements reentrant characteristics just as implicit monitors.

### ReentrantLock

`ReentrantLock`은 상호배제를 활용한 `Lock`입니다. 이는 `synchronized` 키워드를 사용한 방법과 동일하지만, 확장이 가능하다는 차이점이 존재합니다. 이름에서도 드러나듯이 암묵적인 모니터 락 활용과 마찬가지로 Reentrant한 특성을 가지고 있습니다.

***

Let's see how the above sample looks like using `ReentrantLock`:

앞서 나왔던 락은 `ReentrantLock`을 사용하여 다음과 같이 작성하여 줄 수 있습니다.

	ReentrantLock lock = new ReentrantLock();
	int count = 0;

	void increment() {
	    lock.lock();
	    try {
	        count++;
	    } finally {
	        lock.unlock();
	    }
	}

A lock is acquired via `lock()` and released via `unlock()`. It's important to wrap your code into a `try/finally` block to ensure unlocking in case of exceptions. This method is thread-safe just like the synchronized counterpart. If another thread has already acquired the lock subsequent calls to `lock()` pause the current thread until the lock has been unlocked. Only one thread can hold the lock at any given time.

`lock()`메서드를 호출하여 락을 시작하며 `unlock()`메서드를 호출하여 풀수 있습니다. 예외가 발생할 경우를 대비하여 공유자원이 들어간 코드 블록을 `try/finally`로 감싸는 것은 중요합니다. 이 메서드는 앞서 사용했던 `synchronized`와 동일한 기능을 할 수 있습니다. 만약 다른 스레드에서 이미 `lock()`을 호출하였을 경우에는 해당 스레드의 작업이 진행될 때 까지 잠시 작업을 정지합니다. 오직 하나의 스레드만이 락이 사용된 코드에 접근이 가능합니다.

***

Locks support various methods for fine grained control as seen in the next sample:

Lock인터페이스에서는 다음과 같이 다양한 메서드를 지원합니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);
	ReentrantLock lock = new ReentrantLock();

	executor.submit(() -> {
	    lock.lock();
	    try {
	        sleep(1);
	    } finally {
	        lock.unlock();
	    }
	});

	executor.submit(() -> {
	    System.out.println("Locked: " + lock.isLocked());
	    System.out.println("Held by me: " + lock.isHeldByCurrentThread());
	    boolean locked = lock.tryLock();
	    System.out.println("Lock acquired: " + locked);
	});
	
	stop(executor);

While the first task holds the lock for one second the second task obtains different information about the current state of the lock:

//해석다시 하기.
첫번째 executor에서는 1초간 락이 걸리기 때문에 대기하므로 두번째 태스크에서는 락의 현재 상태가 변화하게 된다.

***

	Locked: true
	Held by me: false
	Lock acquired: false

The method `tryLock()` as an alternative to `lock()` tries to acquire the lock without pausing the current thread. The boolean result must be used to check if the lock has actually been acquired before accessing any shared mutable variables.

메서드 `tryLock()`은 `lock()`의 대체제 입니다. 현재 스레드의 멈춤없이 락을 할 수 있습니다. boolen 결과값을 반드시 활용하여야 합니다. 락이 걸려있는지 확인하기 위해서 어떠한 공유변수이던지.

***

### ReadWriteLock

The interface `ReadWriteLock` specifies another type of lock maintaining a pair of locks for read and write access. The idea behind read-write locks is that it's usually safe to read mutable variables concurrently as long as nobody is writing to this variable. So the read-lock can be held simultaneously by multiple threads as long as no threads hold the write-lock. This can improve performance and throughput in case that reads are more frequent than writes.

### ReadWriteLock

`ReadWriteLock` 인터페이스는 읽기 엑세스와 쓰기 엑세스 한 쌍의 락을 관리할 수 있는 기능을 가지고 있습니다. 공유변수에 쓰기 작업이 없을 경우, 기본적으로 변수를 읽는 것은 안전할 것이라는 개념이 이 인터페이스에 나타나 있습니다. 쓰기락이 사용되지 않는 경우, 읽기 락은 동시에 적용될 수 있을것 입니다. 쓰기작업보다 읽기작업이 많이 사용되는 경우, 이 락을 사용할 경우 성능과 처리량을 향상시킬 수 있을 것 입니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);
	Map<String, String> map = new HashMap<>();
	ReadWriteLock lock = new ReentrantReadWriteLock();

	executor.submit(() -> {
	    lock.writeLock().lock();
	    try {
	        sleep(1);
	        map.put("foo", "bar");
	    } finally {
	        lock.writeLock().unlock();
	    }
	});

The above example first acquires a write-lock in order to put a new value to the map after sleeping for one second. Before this task has finished two other tasks are being submitted trying to read the entry from the map and sleep for one second:

위의 예제에서는 우선 쓰기 락의 건 다음 1초간 일시정지, 그 다음 Map에 새로운 값을 추가합니다. 아래의 코드에서는 쓰기 작업이 끝나기 전에 읽기작업을 실행하고 1초간 정지하도록 코드를 작성하였습니다.

***

	Runnable readTask = () -> {
	    lock.readLock().lock();
	    try {
	        System.out.println(map.get("foo"));
	        sleep(1);
	    } finally {
	        lock.readLock().unlock();
	    }
	};

	executor.submit(readTask);
	executor.submit(readTask);

	stop(executor);

When you execute this code sample you'll notice that both read tasks have to wait the whole second until the write task has finished. After the write lock has been released both read tasks are executed in parallel and print the result simultaneously to the console. They don't have to wait for each other to finish because read-locks can safely be acquired concurrently as long as no write-lock is held by another thread.

코드를 실행할 경우 두 읽기작업의 경우 쓰기작업이 종료될 때 까지 기다리는 것을 확인할 수 있을 것입니다. 쓰기작업이 끝날경우, 읽기작업은 동시에 실행되어 결과가 출력되는 것을 콘솔에서 확인할 수 있을 것 입니다. 읽기작업은 쓰기작업이 다른스레드에서 진행되지 않을 경우에 병렬로 사용할 수 있으므로, 두개의 읽기작업은 동시에 실행되는것 입니다.

***

### StampedLock

Java 8 ships with a new kind of lock called `StampedLock` which also support read and write locks just like in the example above. In contrast to `ReadWriteLock` the locking methods of a `StampedLock` return a stamp represented by a `long` value. You can use these stamps to either release a lock or to check if the lock is still valid. Additionally stamped locks support another lock mode called optimistic locking.

자바8에서는 `StampedLock`이라는 새로운 락이 추가되었씁니다. 이는 위의 `ReadWriteLock`과 동일한 개념을 가지고 있으나, 락의 여부인 stamp를 `long` 값으로 반환합니다. 이 stamp 값을 사용하여 현재 락의 상태와 락이 걸렸는지 여부를 확인할 수 있습니다. 추가적으로 `StampedLock`은 다른 낙관적 락이 가능하도록 지원하여 줍니다.

***

Let's rewrite the last example code to use `StampedLock` instead of `ReadWriteLock`:

아래는 이전 예제코드를 `StampedLock`을 사용하여 변경한 예시입니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);
	Map<String, String> map = new HashMap<>();
	StampedLock lock = new StampedLock();

	executor.submit(() -> {
	    long stamp = lock.writeLock();
	    try {
	        sleep(1);
	        map.put("foo", "bar");
	    } finally {
	        lock.unlockWrite(stamp);
	    }
	});

	Runnable readTask = () -> {
	    long stamp = lock.readLock();
	    try {
	        System.out.println(map.get("foo"));
	        sleep(1);
	    } finally {
	        lock.unlockRead(stamp);
	    }
	};

	executor.submit(readTask);
	executor.submit(readTask);

	stop(executor);

Obtaining a read or write lock via `readLock()` or `writeLock()` returns a stamp which is later used for unlocking within the finally block. Keep in mind that stamped locks don't implement reentrant characteristics. Each call to lock returns a new stamp and blocks if no lock is available even if the same thread already holds a lock. So you have to pay particular attention not to run into deadlocks.

 `readLock()`메서드와 `writeLock()`메서드를 사용하여 얻은 stamp값은 `finally`블록에서 락을 해제할 때 사용됩니다. `StampedLock`은 다른 락과는 다르게 Reentrant특성을 가지고 있지 않다는 점을 기억하여야 합니다. 동일한 스레드며 이미 락이 호출되었더라도 매번 호출될 때 마다 새로운 stamp값을 반환합니다. 그러므로 `StampedLock`을 사용할 때에는 데드락과 같은 상황에 빠지지 않도록 특별에 주의하여야 합니다.

***

Just like in the previous `ReadWriteLock` example both read tasks have to wait until the write lock has been released. Then both read tasks print to the console simultaneously because multiple reads doesn't block each other as long as no write-lock is held.

이전의 `ReadWriteLock`과 마찬가지로 쓰기락이 걸려있는 상태일 경우 락이 풀릴때 까지 읽기작업은 일시정지 상태가 됩니다. 읽기작업은 쓰기락이 걸려있지 않을경우에 동시에 실행할 수 있습니다.

***

The next example demonstrates *optimistic locking*:

아래는 *낙관적 락(optimistic lock)*에 대한 예제코드입니다.

***

	ExecutorService executor = Executors.newFixedThreadPool(2);
	StampedLock lock = new StampedLock();

	executor.submit(() -> {
	    long stamp = lock.tryOptimisticRead();
	    try {
	        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
	        sleep(1);
	        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
	        sleep(2);
	        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
	    } finally {
	        lock.unlock(stamp);
	    }
	});

	executor.submit(() -> {
	    long stamp = lock.writeLock();
	    try {
	        System.out.println("Write Lock acquired");
	        sleep(2);
	    } finally {
	        lock.unlock(stamp);
	        System.out.println("Write done");
	    }
	});

	stop(executor);

An optimistic read lock is acquired by calling `tryOptimisticRead()` which always returns a stamp without blocking the current thread, no matter if the lock is actually available. If there's already a write lock active the returned stamp equals zero. You can always check if a stamp is valid by calling `lock.validate(stamp)`.

낙관적 읽기락의 stamp `tryOptimisticRead()`을 사용하여 얻어올 수 있습니다. 현재 스레드가 락이 걸렸는지, 사용 가능한지와 상관없이 stamp 값을 리턴합니다. 만약 쓰기락이 걸려있는 상태라면 stamp값은 0을 반환합니다. lock이 가능한지 여부는 `lock.validate(stamp)`를 호출하여 언제나 확인할 수 있습니다.

***

Executing the above code results in the following output:

위의 코드를 실행하려 경우 아래와 같은 결과가 반환됩니다.

***

	Optimistic Lock Valid: true
	Write Lock acquired
	Optimistic Lock Valid: false
	Write done
	Optimistic Lock Valid: false

The optimistic lock is valid right after acquiring the lock. In contrast to normal read locks an optimistic lock doesn't prevent other threads to obtain a write lock instantaneously. After sending the first thread to sleep for one second the second thread obtains a write lock without waiting for the optimistic read lock to be released. From this point the optimistic read lock is no longer valid. Even when the write lock is released the optimistic read locks stays invalid.

// 머리가 복잡하다 해석이 안됭.. ㅇㅅㅇ

***

So when working with optimistic locks you have to validate the lock every time after accessing any shared mutable variable to make sure the read was still valid.

그러므로 만약 낙관적 락과 함께 작업을 할경우 매번 공유변수에 접근할 때 마다 읽기작업이 가능한지 확인해보아야 합니다.

***

Sometimes it's useful to convert a read lock into a write lock without unlocking and locking again. `StampedLock` provides the method `tryConvertToWriteLock()` for that purpose as seen in the next sample:

	ExecutorService executor = Executors.newFixedThreadPool(2);
	StampedLock lock = new StampedLock();

	executor.submit(() -> {
	    long stamp = lock.readLock();
	    try {
	        if (count == 0) {
	            stamp = lock.tryConvertToWriteLock(stamp);
	            if (stamp == 0L) {
	                System.out.println("Could not convert to write lock");
	                stamp = lock.writeLock();
	            }
	            count = 23;
	        }
	        System.out.println(count);
	    } finally {
	        lock.unlock(stamp);
	    }
	});

	stop(executor);

The task first obtains a read lock and prints the current value of field `count` to the console. But if the current value is zero we want to assign a new value of `23`. We first have to convert the read lock into a write lock to not break potential concurrent access by other threads. Calling `tryConvertToWriteLock()` doesn't block but may return a zero stamp indicating that no write lock is currently available. In that case we call `writeLock()` to block the current thread until a write lock is available.

## Semaphores

In addition to locks the Concurrency API also supports counting semaphores. Whereas locks usually grant exclusive access to variables or resources, a semaphore is capable of maintaining whole sets of permits. This is useful in different scenarios where you have to limit the amount concurrent access to certain parts of your application.

Here's an example how to limit access to a long running task simulated by `sleep(5)`:

	ExecutorService executor = Executors.newFixedThreadPool(10);

	Semaphore semaphore = new Semaphore(5);

	Runnable longRunningTask = () -> {
	    boolean permit = false;
	    try {
	        permit = semaphore.tryAcquire(1, TimeUnit.SECONDS);
	        if (permit) {
	            System.out.println("Semaphore acquired");
	            sleep(5);
	        } else {
	            System.out.println("Could not acquire semaphore");
	        }
	    } catch (InterruptedException e) {
	        throw new IllegalStateException(e);
	    } finally {
	        if (permit) {
	            semaphore.release();
	        }
	    }
	}

	IntStream.range(0, 10)
	    .forEach(i -> executor.submit(longRunningTask));

	stop(executor);

The executor can potentially run 10 tasks concurrently but we use a semaphore of size 5, thus limiting concurrent access to 5. It's important to use a `try/finally` block to properly release the semaphore even in case of exceptions.

Executing the above code results in the following output:

	Semaphore acquired
	Semaphore acquired
	Semaphore acquired
	Semaphore acquired
	Semaphore acquired
	Could not acquire semaphore
	Could not acquire semaphore
	Could not acquire semaphore
	Could not acquire semaphore
	Could not acquire semaphore

The semaphores permits access to the actual long running operation simulated by `sleep(5)` up to a maximum of 5. Every subsequent call to `tryAcquire()` elapses the maximum wait timeout of one second, resulting in the appropriate console output that no semaphore could be acquired.

This was the second part out of a series of concurrency tutorials. More parts will be released in the near future, so stay tuned. As usual you find all code samples from this article on [GitHub](https://github.com/winterbe/java8-tutorial), so feel free to fork the repo and try it by your own.
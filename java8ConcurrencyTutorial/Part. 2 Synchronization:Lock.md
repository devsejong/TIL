# 자바8 Concurrency 튜토리얼: Synchronize/Lock

이 글은 [Benjamin Winterberg](http://winterbe.com/about/)가 작성한 [Java 8 Concurrency Tutorial](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)을 번역한 글입니다. 저자의 허락을 얻어 번역하여 공개합니다.

***

병렬 프로그래밍 튜토리얼의 두 번째 파트에 오신 것을 환영합니다. 이 튜토리얼은 자바8을 기반으로 한 예제코드를 통해서 쉬운 이해를 돕는 걸 목표로 합니다. 앞으로 15분간 synchronized 키워드, Lock, Semaphore를 활용하여 공유변수에 안전하게 엑세스할 수 있는 방법을 이야기할 것입니다.

- 파트 1: [Thread/Executor](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)
- 파트 2: Synchronize/Lock
- 파트 3: Atomic Variables/ConcurrentMap

앞으로 이야기할 내용의 주요 개념들은 오래된 자바에서도 같게 사용할 수 있으나, 이 튜토리얼은 자바8을 기준으로 코드를 작성하였습니다. 람다 표현식과 같이 자바8에 소개된 새로운 문법을 많이 사용하였으므로 문법에 친숙하지 않을 경우 우선 람다에 대한 [튜토리얼](http://winterbe.com/posts/2014/03/16/java-8-tutorial/) 먼저 참조하시기 바랍니다.

튜토리얼에서 작성할 예제코드에서는 단순함을 위해서 두개의 메서드 `sleep(seconds)`과 `stop(executor)`을 정의합니다. `sleep(seconds)`는 입력되는 초만큼 스레드를 정지합니다. `stop(executor)`는 입력받은 ExecutorService를 정지하는 역할을 합니다. 이 메서드는 [여기](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)에서 찾아볼 수 있습니다.

## Synchronize

[이전 튜토리얼](http://devsejong.tumblr.com/post/126596600092/자바8-concurrency-튜토리얼-threadexecutor)에서 우리는 Executor Service를 사용해 어떻게 동시성을 처리하는지에 대해서 이야기하였습니다. 앞의 튜토리얼을 진행하면서 각 스레드가 참조하는 공유 변수에 엑세스하는 것에 대해서는 특별한 언급 없이 진행하였지만, 공유 변수를 참조하는 코드를 작성할 때에는 특별히 주의를 기울여야 합니다. 다음에 나올 예제코들 통해서 어떤 문제가 있는지에 대해서 알아보도록 합시다.

다음 예제에서는 int 타입 `count`를 정의하고 멀티 스레드로 변수를 1씩 더하도록 코드를 작성하였습니다. 1씩 `count`를 증가시키는 메서드는`increment()`로 정의하였습니다.

	int count = 0;

	void increment() {
	    count = count + 1;
	}
	
이 메서드가 병렬로 호출할 경우 심각한 문제가 생깁니다.

	ExecutorService executor = Executors.newFixedThreadPool(2);

	IntStream.range(0, 10000)
	    .forEach(i -> executor.submit(this::increment));

	stop(executor);

	System.out.println(count);  // 9965

위의 코드를 실행할 경우 예상하는 값 10000이 아니라 매번 다른 값이 출력되는 것을 확인할 수 있을 것입니다. 이는 서로 다른 스레드가 공유변수에 동기화되지 않은 채로 접근하기 때문입니다. 이러한 현상을 경쟁상태([race condition](https://ko.wikipedia.org/wiki/경쟁_상태))라고 부릅니다.

위의 코드에서 숫자를 1 증가시키는 로직은 다음과 같이 3번의 과정을 거쳐 진행됩니다. (1)현재 count 값을 읽는다. (2)읽은값의 숫자에 1을 더한다. (3)결과를 변수에 설정한다. 위 과정을 거쳐 실행되면서 두 스레드가 (1)의 과정에서 변수를 여러 스레드가 동시에 읽어 같은 값을 가져올 가능성이 존재합니다. 그러므로 실제 실행결과 값이 10000보다 작은 숫자가 나오는 것입니다. 위 코드에서는 35만큼이 동기화되지 않은 접근 때문에 빠졌습니다.

다행히도 자바에서는 초기 버전부터 지원하는  `synchronized` 키워드를 활용하여 스레드 환경에서 동기화된 접근이 가능하도록 하였습니다. 1씩 값을 증가시키는 위의 경쟁상태 코드를 수정해 보도록 하겠습니다.

	synchronized void incrementSync() {
	    count = count + 1;
	}

이제 `incrementSync()`를 병렬로 사용하더라도 우리가 예상했던 결괏값 10000을 얻을 수 있습니다. 매번 실행하더라도 더는 경쟁상태로 인한 문제가 생기지 않는 것을 확인할 수 있습니다.

	ExecutorService executor = Executors.newFixedThreadPool(2);

	IntStream.range(0, 10000)
	    .forEach(i -> executor.submit(this::incrementSync));

	stop(executor);

	System.out.println(count);  // 10000

`synchroized` 키워드는 코드 블록을 앞뒤로 감싸서 직접 활용할 수 있습니다.

	void incrementSync() {
	    synchronized (this) {
	        count = count + 1;
	    }
	}

자바 내부적으로 모니터락([Monitor Lock](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html))을 동기처리를 위해서 사용합니다. 자바에서는 각 개체마다 모니터가 하나씩 존재합니다. synchronized 메서드가 동시에 호출될 경우 해당 개체는 같은 모니터를 공유하여 하나의 스레드만 메서드를 사용할 수 있도록 제어합니다. 또한, `synchronized` 키워드를 활용한 방식은 객체를 따로 선언하지 않고 사용하므로 암묵적인 락(intrinsic lock)이라고 부릅니다. 

모든 암시적 모니터락(implicit monitors)은 [Reentrant](https://en.wikipedia.org/wiki/Reentrancy_(computing)) 특성이 있습니다. Reentrant란 다시 말해서 현재 스레드와 Lock이 결속되어있다는 뜻입니다. 스레드는 코드를 호출할 때마다 같은 락을 안전하게(데드락 없이) 가져올 수 있습니다. 또한, synchronized메서드는 같은 객체에 존재하는 다른 synchronized 메서드를 부를 수 있습니다.

## Lock

`synchronized` 키워드를 사용하는 대신에 Concurrency API에서 지원하는 다양한 `Lock` 인터페이스를 활용할 수 있습니다. Lock 인터페이스에서는 상황별로 잘 쪼개진 메서드들을 제공합니다. (암묵적인 모니터락보다는 비용이 많이 든다고 볼 수 있습니다.)

표준 JDK에 정의된 Lock들에 대해서 다음 섹션부터 자세하게 이야기해 나가도록 하겠습니다.

### ReentrantLock

`ReentrantLock`은 [상호 배제](https://ko.wikipedia.org/wiki/상호_배제)(mutual exclusion, MUTEX)를 활용한 `Lock` 구현체입니다. `synchronized` 키워드를 사용한 방법과 같지만, 객체이기 때문에 확장할 수 있다는 차이점이 존재합니다. `ReentrantLock`이라는 이름에서도 드러나듯이 암묵적인 모니터 락 활용과 마찬가지로 Reentrant 특성이 있습니다.

아래의 `ReentrantLock`를 활용한 예제코드는 `synchronized`를 사용한 이전 예제와 같게 동작합니다.

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

`lock()`메서드를 호출하여 락을 시작할 수 있으며,  `unlock()` 메서드를 호출하여 락을 해제할 수 있습니다. 예외가 발생할 경우를 대비하여 공유자원이 들어간 코드 블록을 `try/finally`로 감싸는 것을 기억해주시기 바랍니다. 이 메서드를 조합하면 앞서 사용했던 `synchronized` 키워드를 사용한 것과 같은 기능을 하는 코드를 작성할 수 있습니다. 만약 다른 스레드에서 이미 `lock()`을 호출하였을 경우에는 해당 스레드의 작업이 진행될 때까지 잠시 작업을 정지합니다. 오직 하나의 스레드만이 락이 사용된 코드에 엑세스가 가능하기 때문입니다.

`Lock` 인터페이스에서는 다음과 같이 다양한 메서드를 지원합니다.

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

첫 번째 executor에서 1초간 락이 걸린채로 대기하는 동안 두 번째 작업에서는 현재 락의 상태에 대해서 다양한 정보를 가져올 수 있습니다.

	Locked: true
	Held by me: false
	Lock acquired: false

`lock` 메서드와 비슷한 역할을 하는 `tryLock()` 메서드는 현재 스레드의 멈춤 없이 락을 걸 수 있는 역할을 합니다. 단, 해당 메서드를 사용한다면 먼저 호출된 결과 `boolean`값을 통해서 락이 걸려있는지 먼저 확인해본 뒤에 코드를 작성하여야 합니다. 

### ReadWriteLock

`ReadWriteLock` 인터페이스는 읽기 엑세스와 쓰기 엑세스 한 쌍의 락을 관리 기능을 가지고 있습니다.  이 인터페이스에는 공유변수에 쓰기 작업이 없으면 기본적으로 변수를 읽는 것은 안전할 것이라는 믿음이 내제되어있습니다. 쓰기 락이 걸려있지 않은 경우, 읽기 락은 여러 작업을 동시에 실행할 수 있습니다. 이 락은 쓰기 작업보다 읽기 작업이 많이 사용되는 경우 성능과 처리량이 향상될 수 있을 것입니다.

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

위의 예제에서는 쓰기락의 건 다음 1초간 일시 정지를 하며 그다음 Map에 새로운 값을 추가하도록 작성하였습니다. 이어지는 아래의 예시 코드에서는 쓰기 작업이 끝나기 전에 읽기 작업을 실행하고 1초간 정지하도록 코드를 작성하였습니다.

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

코드를 실행할 경우 두 읽기 작업의 경우 쓰기 작업이 종료될 때까지 기다리는 것을 확인할 수 있을 것입니다. 쓰기 작업이 끝난 뒤 읽기 작업은 동시에 실행되어 그 결과가 출력되는 것을 콘솔에 볼 수 있을 것입니다. 읽기 작업은 쓰기 작업이 다른 스레드에서 진행되지 않을 경우에 병렬로 사용할 수 있으므로, 두개의 읽기 작업은 동시에 진행되기 때문입니다.

### StampedLock

자바8에서는 `StampedLock`이라는 새로운 락이 추가되었습니다. 이는 위의 `ReadWriteLock`과 같은 개념을 가지고 있으나, 락의 여부인 stamp를 `long` 값으로 반환합니다. 이 stamp 값을 사용하여 현재 락의 상태와 락이 걸렸는지를 확인할 수 있습니다. 추가로 `StampedLock`은 다른 낙관적 락([optimistic lock](https://en.wikipedia.org/wiki/Optimistic_concurrency_control))이 가능하도록 지원합니다.

앞서 작성했던 예제코드를 `StampedLock`을 사용하여 변경해 보았습니다.

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

 `readLock()`메서드와 `writeLock()`메서드를 사용하여 얻은 stamp 값은 `finally`블록에서 락을 해제할 때 사용됩니다. `StampedLock`은 다른 락과는 다르게 Reentrant 특성이 있지 않다는 점을 기억하여야 합니다. 같은 스레드며 이미 락이 호출되었더라도 매번 호출될 때마다 새로운 stamp값을 반환합니다. 만약 `StampedLock`을 사용할 경우 데드락과 같은 상황에 빠지지 않도록 특별에 주의하여야 합니다.

이전의 `ReadWriteLock`과 마찬가지로 쓰기 락이 걸려있는 상태에선 락이 풀릴 때까지 읽기 작업은 대기 상태가 됩니다. 읽기 작업은 쓰기락이 걸려있지 않는다면 동시에 실행될 수 있습니다.

아래는 *낙관적 락(optimistic lock)*에 대한 예제코드입니다.

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


낙관적인 읽기 락의 stamp 값은 `tryOptimisticRead()`을 호출하여 얻을 수 있습니다. 메서드를 호출할 경우 현재 스레드가 사용 가능한지와 관계없이 stamp 값을 반환합니다. 만약 현재 쓰기 락이 걸려있는 상태라면 stamp 값은 0을 반환합니다. `lock.validate(stamp)`을 사용하여 락이 가능한지에 대해 언제나 확인할 수 있습니다.

위의 코드를 실행하면 아래와 같은 결과를 확인할 수 있습니다

	Optimistic Lock Valid: true
	Write Lock acquired
	Optimistic Lock Valid: false
	Write done
	Optimistic Lock Valid: false

락이 유효한지 여부를 락을 얻은 다음 바로 확인하고 있습니다. 일반적인 읽기 락과는 달리 낙관적 락은 쓰기 락을 얻기 위해 다른 스레드를 대기상태로 만들지 않습니다. 첫 번째 스레드의 상태를 검사한 다음 대기상태를 가지는 1초 동안 두 번째 스레드에서는 읽기락이 해제되는 것을 기다리지 않고 바로 쓰기락을 가져옵니다. 이 시점부터 읽기락은 더는 유효하지 않으며, 쓰기락이 해제되더라도 읽기락은 유효하지 않은 상태를 유지합니다.

그러므로 만약 낙관적 락과 함께 작업을 할 경우 매번 공유변수에 엑세스할 때마다 읽기 작업이 가능한지 확인해보아야 합니다.

`StampedLock`에서는 `tryConvertToWriteLock()`메서드를 제공합니다. 이 메서드를 사용하면 락을 걸고 해제하는 과정 없이 바로 읽기 락을 쓰기 락으로 변환할 수 있습니다.

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

첫 번째 작업에서 읽기 락을 얻은 다음 `count`를 콘솔에 출력합니다. 하지만 `count`의 현재 값이 0일 경우, 새로운 값 23을 할당합니다. 이러한 흐름의 코드를 작성할 때에는 다른 스레드에서 값을 엑세스할 경우 오류가 나지 않도록 읽기 락을 쓰기락으로 변환해주어야 합니다. 주의할 점은 `tryConvertToWriteLock()`을 사용할 경우 락을 얻을 때까지 대기하지 않으며 대신에 0을 현재 쓰기락을 진행할 수 없다는 의미로  반환합니다. 이러한 경우를 대비하기 위해서는 `writeLock()`을 호출하여 쓰기락을 진행할 때까지 대기하는 로직이 필요할 수도 있습니다.

## Semaphore

Concurrency API에서는 다른 락으로 [세마포어](http://www.joinc.co.kr/modules/moniwiki/wiki.php/Site/system_programing/IPC/semaphores)(Semaphore)를 지원합니다. 락이 일반적으로 유일한 접근을 가능하도록 권한을 부여한다고 하면, 세마포어는 접근을 허락하는 전체 세트를 관리할 수 있습니다. (역자주 : ReentrantLock의 경우 하나의 스레드만 접근 가능합니다. 다시 말해 1의 크기를 가진 세마포어라고도 볼 수 있을 것입니다) 세마포어는 특정파트에 대한 동시접속을 제한하는 시나리오에서 유용하게 사용될 수 있을 것입니다.

아래의 예제는 `sleep(5)`와 세마포어를 활용하여 엑세스의 숫자를 제한하는 예제입니다.

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

위의 `Executor`는 총 10개의 작업을 동시에 실행할 수 있도록 설정하였으나, 세마포어의 크기는 5로 제한하였고, 세마포어의 크기 5만큼의 엑세스만 가능합니다. 세마포어를 예외상황에서도 잘 관리할 수 있도록 `try/catch`를 사용하여 코드 블록을 묶어주는 과정이 필요합니다.

위의 코드를 실행할 경우 아래와 같은 결과를 볼 수 있습니다.

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

세마포어가 허용한 엑세스는 최대 5개로 제한하였습니다. 작업은 `sleep(5)`를 사용하였으므로 최소 5초간 실행되게 됩니다. `tryAcquire()`의 타임아웃 시간 1초를 선언되어 있습니다. 그러므로 5번째 이후로 실행된 작업의 경우 세마포어로 엑세스가 불가능합니다.

이상으로 concurrency 튜토리얼의 두 번째 파트를 마칩니다. 위의 코드를 스스로 실행해보시는 것을 강력하게 권장합니다. 또한,, 예제에 대한 샘플은 [Github](https://github.com/winterbe/java8-tutorial)에서 찾을 수 있습니다. star 주는걸 잊지 마세요.
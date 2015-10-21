# Optional을 활용하여 null 처리하기

자바에서는 메서드를 실행한 뒤 반환값으로 null을 줄 수 있다. 메서드의 실행 결과가 없을 경우에 null을 반환하는 것은 당연한 일이다.

하지만 메서드를 사용할 경우에는 null이 나올 수 도 있기 때문에 방어적으로 코드를 작성하여야 한다. 아래와 같이 null을 처리하는 로직을 사용하여야 한다.

[[MORE]]
  
    public String simpleCalc(int a, int b, Calc calc) {
        Result result = calculatorService.getResult();
        if (result == null) {
            // null 처리.
        }
        //do something	
    }



null을 제대로 처리하지 않았을 경우 코드를 실행할 경우에 `NullPointerException` 에러를 뿜어내는 것을 볼 수 있다.

위의 상황에서 사실 가장 간단한 해결책은 null을 넘기지 않는 것이다. 하지만 정말로 결과가 없을 경우처럼 null을 넘겨야 하는게 옳은 상황이라면, 자바 8에서 추가된 `Optional` 객체를 반환하도록 해보자.(자바8 이하는 구글의 Guava 라이브러리의 Optional 객체를 사용하도록 하자.)

Optional은 아래와 같이 사용할 수 있다.

	public Optional<String> findUserAddress(long userId) {
	    User user = userRepository.find(userId);
	    return Optional.ofNullable(user.getAddress());
	}
	
	public void getAddress(){
	    Optional<String> optionalAddress = findUserAddress(1234);
	    String address;
	    if(optionalAddress.isPresent())
	        addressStr = optionalAddress.get();
	    else
	        addressStr = "주소가 없습니다.";
	}


`findUserAddress`에서는 반환값으로 `Optional`을 설정하였다. 그 다음 `optioalAddress.isPresent`를 호출하여 값이 있는지 여부를 체크한뒤 값이 있으면 optioalAddress의 값을 반환한다.

Optinal에는 사실 특별한 내용이 없다. null이 될 수도 있는 상황에서 값을 쉽게 설정하고, 사용하는 입장에서도 null에 대한 대비를 쉽게 해 줄 수 있는 정도의 기능을 가진다.

기능보다 더욱 중요한것은 메서드를 사용했을 경우 null이 나올 수도 있다는 것을 드러내고, null에 대응하는 코드를 작성하도록 유도할 수 있다는 것이다. null이 가지는 모호함은 없애고, 결과 중 null이 쓰인다는것을 명시적으로 나타낼 수 있는 것이다.

Optional은 자바8에서 정식으로 지원한다. 그 이하 버전이라면 [Guava](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/base/Optional.html) 라이브러리에서도 유사한 기능의 클래스를 지원한다.

자바8에서의 Optional 클래스 사용방법을 살펴보도록 하자.

## Optional 반환값으로 사용하기.

`Optioan<T>`로 반환값을 설정할 수 있다. 제네릭으로 설정하는 부분에는 원래의 반환값을 입력하도록 하면 된다.

	public Optional<String> findUserAddress(long userId) {

## `Optional` 객체 생성

`Optional`의 생성자는 private이다. 대신 static 메서드를 제공하여 객체를 초기화 할 수 있다. `Optional`은 총 3개의 static 함수를 제공한다.

### of()

`static Optional<T> of(T value)`

- Optinal 객체에 값을 설정한다. 값이 null일 경우 `NullPointerException`을 반환한다.


### ofNullable()

`static Optional<T> ofNullable(T value)`

- Optinal 객체에 값을 설정한다. null 또한 사용할 수 있다.

### empty()

`static Optional<T> empty()`

- 빈 Optional객체를 반환한다.


## `Optional` 객체 활용

`Optional`객체를 반환받은 뒤에는 아래의 함수를 활용하여 결과 값을 조회하거나 추가 작업을 할수 있다. 자바8의 람다를 활용한 메서드가 많다.

### get()

`T get()`

- Optional객체에서 값을 꺼내온다. 값이 없을 경우에는 NoSuchElementException 가 발생한다.

### isPresent()

`boolean isPresent()`

- 값이 존재하는지 여부를 점검한다.

### ifPresent()

`void ifPresent(Consumer<? super T> consumer)`

- 값이 존재할 경우에만 Cunsumer를 실행한다.

		Optional<String> opt = Optional.of("서울시");
		if(opt.isPresent()){
			//do something
		}


### filter()

`Optional<T> filter(Predicate<? super T> predicate)`

- 값이 존재하고 predicate에 값과 동일한 경우에만 Optional을 반환한다. 값이 동일하지 않을 경우에는 빈 Optional을 반환한다.

		//서울이라는 값이 주소에 존재할 경우에 ifPresent까지 함께 동작한다.
        address.filter(addr -> addr.contains("서울"))
		     .ifPresent(System.out::println);

	
### map()

`Optional<U> map(Function<? super T,? extends U> mapper)`

- 값이 존재하는 경우, mapper를 실행하게 되며, 그 결과가 null이 아닐경우 다른 Optional로 감싼 뒤 반환한다. null일경우에는 빈 Optional을 반환한다.

        String x = "test";
        if (x != null) {
            String t = x.trim();
            if (t.length() < 1) {
                System.out.println(t);
            }
        }
        
        //Optional을 사용하여 아래와 같이 변경할 수있다.
        Optional<String> opt = Optional.of("test");
        opt.map(String::trim).
                filter(t -> t.length() > 1).
        ifPresent(System.out::println);



### flatMap()

`Optional<U> flatMap(Function<? super T,Optional<U>> mapper)`

- 값이 존재하는 경우, mapper를 실행하며, 그 결과가 null이 아닐 경우 다른 Optaionl로 변경한 뒤 반환한다.
- map과는 다르게 제공해야하는 mapper의 반환값이 Optional이어야 한다.
- //TODO 언제 사용하여야 하는지에 대해서 조금 더 알아보기

### orElse()

`T orElse(T other)`

- 만약 Optional의 값이 존재한다면 그 값을 반환한다. 그렇지 않다면, other의 값을 반환.

### orElseGet()

`T orElseGet(Supplier<? extends T> other)`

- 만약 Optional의 값이 존재한다면 그 값을 반환한다. 그렇지 않다면, other를 실행한 뒤 그 값을 반환한다.

### orElseThrow()

`<X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X extends Throwable`

- 만약 값이 존재한다면 그 값을 반환한다. 존재하지 않을 경우, exceptionSupplier에서 발생된 exception을 반환한다.

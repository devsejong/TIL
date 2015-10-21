테스트는 중요하다. 테스트 커버리지가 곧 프로그램의 품질인 시대이지만, 사실 더욱 중요한 것은 쉽게 읽을 수 있는 테스트코드를 작성하는 것이다.

테스트코드는 API를 작성한 개발자의 의도를 가장 잘 드러낸 코드이다. 새롭게 기능을 추가하거나, 버그를 수정할 경우 테스트를 먼저 읽고 접근하며, 이 테스트 코드가 읽기 쉬울수록 더욱 빠르게 코드를 수정해 나갈 수 있을 것이다.

읽기 쉬운 코드가 무엇인지에 대해서는 쉽게 정의할 수는 없다. 하지만 길이가 작을수록 읽기 쉽다는 것은 분명한 사실이다. JUnit을 활용한 테스트에서는 assertThat 구문을 활용하여 코드의 길이를 줄이면서도 읽혀지는 코드를 작성할 수 있도록 도와준다. 즉 assertThat을 잘 사용하면 개발자의 의도를 보다 명확하게 드러낼 수 있는 큰 장점을 얻어갈 수 있을 것이다.

assertThat을 사용하는 방법에 대해서 보다 자세하게 알아보도록 하자.

# assertThat 사용법

assertThat은 `assertThat(T actual, Matcher<? super T> matcher)`의 형태로 메서드를 사용하여 두 값을 비교할 수 있다. 첫번째 파라미터에는 비교대상 값을, 두번째 파라미터로는 비교로직이 담긴 Matcher가 사용된다.

아래는 대표적인 매쳐인 Matcher `is`를 사용한 코드이다. `is`는 첫번째 파라미터와 자기 자신의 파라미터가 동일한지 여부를 체크한다.


    @Test
    public void testAdd() {
        int result = calculator.add(4, 6);
        Assert.assertThat(result, CoreMatchers.is(10));
    }

result값이 10과 동일한지 여부를 체크하며, 값이 다를 경우에는 테스트실패 상태를 반환하게 된다. 위의 코드에서 보다시피 is를 사용하여 A is B와 같이 읽혀지는 코드를 작성할 수 있게 된다.

여기서 static메서드인 assertThat과 is는 아래와 같이 static import를 통해서 짧게 줄여서 간단하게 작성이 가능하다.


	import static org.hamcrest.CoreMatchers.is;
	import static org.junit.Assert.assertThat;

	public class CalculatorTest {
	    Calculator calculator = new Calculator();
	    @Test
	    public void testAdd() {
	        int result = calculator.add(4, 6);
	        assertThat(result, is(10));
	    }
	}

# JUnit기본제공 매쳐

위에서 소개한 is 이외에도 Junit에서는 상황에 맞는 다양한 매쳐를 지원하며 필요에 따라서 매쳐를 새롭게 구현하여 사용할 수 있다.

매쳐들을 순서대로 나열하여 여러 매쳐를 한번에 사용할 수 있도록 지원한다. 아래는 앞의 문자열이 Test로 시작되지 않는지 여부를 확인하는 예시이다.

	assertThat("Sample string.", is(not(startsWith("Test"))));

## Junit 기본 지원 매쳐

JUnit은 아래와 같은 매쳐를 기본적으로 제공한다. JUnit에서 제공하는 매쳐는 org.hamcrest.CoreMatchers 클래스에 선언된 메서드를 통해 사용할 수 있다.

- allOf
	- 내부에 선언된 모든 매쳐가 정상일 경우 통과한다.
	- `assertThat("myValue", allOf(startsWith("my"), containsString("Val”)))`
- anyOf
	- 내부에 선언된 매쳐중 하나 이상 통과할 경우 통과한다.
	- `assertThat("myValue", anyOf(startsWith("foo"), containsString("Val”)))`
- both
	- both A and B 형식으로 matcher를 사용할 수 있게 해 준다.
	- A, B 매쳐 둘다 통과할 경우 테스트가 성공한다.
	- `assertThat("fab", both(containsString("a")).and(containsString(“b”)))`
- either
	- either A or B 형식으로 matcher를 사용할 수 있게 해 준다.
	- A, B 매쳐 둘중 하나가 성공할 경우 테스트가 성공한다.
	- `assertThat("fan", either(containsString(“a”)).or(containsString(“b”)))`
- describedAs
	- 매쳐내부의 메시지를 변경할 수 있다.
	- `assertThat (new BigDecimal(“32123”), describedAs("a big decimal equal to %0", equalTo(myBigDecimal), myBigDecimal.toPlainString()));`
- everyItem
	- 배열이나 리스트를 순회하며 매쳐가 실행된다.
	- `assertThat(Arrays.asList("bar", "baz"), everyItem(startsWith("ba”)));`
- is
	- is는 두가지 용도로 사용할 수 있다.
	- A is B와 같이 비교값이 서로 같은지 여부를 확인할 경우
		- `assertThat("Simple Text", is("Simple Text"));`
		- 이경우 `assertThat("Simple Text", is(equalTo("Simple Text")))`와 동일하게 사용할 수 있다.
	- 다른 매쳐를 꾸며주는 용도로 사용. 매쳐에는 영향을 끼치지 않으며, 조금 더 표현력이 있도록 변경하여 준다.
		- `assertThat("Simple Text", is(not("simpleText")));`
		- 위의 경우 `is`가 빠지더라도 문제없이 작동된다. 하지만 is가 있음으로써 쉽게 읽혀지는 테스트 코드가 된다.
- isA
	- 비교되는 값이 특정 클래스일 경우 테스트가 통과된다. `is(instanceOf(SomeClass.class))`와 동일하다.
	- `assertThat(cheese, isA(Cheddar.class)) `
- anything
	- 항상 true를 반환하는 매쳐
- hasItem
	- 배열에서 매쳐가 통과하는 값이 하나 이상이 있는지 여부를 검사한다.
	- `assertThat(Arrays.asList("foo", "bar"), hasItem("bar"))`
- hasItems
	- 배열에서 매쳐리스트에 선언된 값들 모두가 하나 이상 있는지 여부를 검사한다.
	- `assertThat(Arrays.asList("foo", "bar", "baz"), hasItems("baz", "foo"))`
- equalTo
	- 두 값이 같은지 여부를 체크한다. `is`와 동일하게 사용할 수 있다.
	- `assertThat("foo", equalTo("foo"));`
- any
	- 비교값이 매쳐의 타입과 동일한지 여부를 체크한다. `instanceOf`와는 다르게 매쳐의 값은 앞서 비교값의 타입의 자식만 비교할 수 있다.
	- `assertThat(new Canoe(), instanceOf(Canoe.class));`
- instanceOf
	- 비교값이 매쳐의 타입과 동일한지 여부를 체크한다. `any`와는 다르게 매쳐의 값은 비교값과 연관없는 경우에도 사용할 수 있다.
	- assertThat(new Canoe(), instanceOf(Paddlable.class));
- not
	- `is`와 동일하게 두가지 경우로 사용할 수 있다.
	- 내부에 매쳐를 선언할 경우 내부 매쳐의결과를 뒤집는다.
		- `assertThat(cheese, is(not(equalTo(smelly))))`
	- `not`뒤에 값이 나올 경우, 같지 않을 경우 테스트가 통과한다.
		- `assertThat("Test", not("tEST"));`
- nullValue
	- 비교값이 null일경우 테스트가 통과한다.
	- `assertThat(cheese, is(nullValue())`
-  notNullValue
	- 비교값이 null이 아닐경우 테스트가 통과한다.
	- `assertThat(cheese, is(notNullValue()))`
- sameInstance
	- 비교매쳐의 값과 같은 인스턴스일 경우 테스트가 통과한다. theInstance 와 동일
	- `assertThat("Test", not(sameInstance("not Same Instance")));`
- theInstance
	- 비교매쳐의 값과 같은 인스턴스일 경우 테스트가 통과한다. sameInstance 와 동일
	- `assertThat("Test", not(sameInstance("not Same Instance")));`
- containsString
	- 특정 문자열이 있는지를 검사한다.
	- `assertThat("myStringOfNote", containsString("ring"));`
- startsWith
	- 특정 문자열로 시작하는지를 검사한다.
	- `assertThat("myStringOfNote", startsWith("my"))`
- endsWith
	- 특정 문자열로 종료되는지를 검사한다.
	- `assertThat("myStringOfNote", endsWith("Note"))`

# matcher 직접 정의하기

Junit에서는 기본적으로 다양한 매쳐들을 많이 제공한다. 하지만 5의 배수일 경우에만 테스트가 통과하여야 한다와 같이 기존의 매쳐로 쉽게 작성되지 않는 상황도 적지 않다. 이 경우 직접 매쳐를 생성해서 쉽게 테스트를 진행할 수 있다.

5의 배수인지를 확인하는 매쳐를 확장하여 아래와 같이 사용할 수 있는 매쳐를 만들어보자

	assertThat(25, is(divisorOfFive()));
    
반환되는 값이 Matcher형태인 메서드 divisorOfFive를 만들며, Integer데이터에 대해서만 사용할 수 있도록 Matcher의 추상 클래스 중 하나인 TypeSafeMatcher를 사용하도록 한다. 완성된 결과는 아래와 같다.

    import org.hamcrest.Description;
    import org.hamcrest.Matcher;
    import org.hamcrest.TypeSafeMatcher;
    import org.junit.Test;

    import static org.hamcrest.CoreMatchers.is;
    import static org.junit.Assert.assertThat;

    public class SimpleTest {
        @Test
        public void divisorOfFiveTest() {
            assertThat(67, is(divisorOfFive()));
        }

        public Matcher<Integer> divisorOfFive() {
            return new TypeSafeMatcher<Integer>() {
                Integer internalNumber;

                @Override
                protected boolean matchesSafely(Integer number) {
                    internalNumber = number;
                    return number % 5 == 0;
                }

                @Override
                public void describeTo(Description description) {
                    description.appendValue(internalNumber + "가 5의 배수여부 확인");
                }
            };
        }
    }
    
위의 테스트를 실행할 때 발생하는 에러메시지는 다음과 같다.

    java.lang.AssertionError: 
    Expected: is "67는 5의 배수가 아닙니다."
         but: was <67>

# 요약

- assertThat과 Matcher의 조합으로 다양한 형태의 비교로직을 만들 수 있다.
- Junit에서 다양한 Matcher를 제공한다.
- 필요에 따라 매쳐는 직접 정의할 수 있다.


## 참고

- [https://code.google.com/p/hamcrest/wiki/Tutorial]
	- hamcrest 튜톨리얼 문서
- [http://junit.org/apidocs/org/hamcrest/CoreMatchers.html]
	- Junit CoreMatcher공식 문서

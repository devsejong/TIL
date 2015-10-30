# 자바 팁


## iterator를 for문에서 사용하기

아래와 같이 사용하여 iterator를 익숙한 for문을 사용하여 처리할 수 있다.

    for (Iterator<E> iter = list.iterator(); iter.hasNext(); ) {
        E element = iter.next();
        // 1 - can call methods of element
        // 2 - can use iter.remove() to remove the current element from the list
    
        // ...
    }
## 비동기 프로그래밍

### Caller, Callee
- 함수가 다른함수를 호출하는 상황
- Caller : 호출하는 함수
- Callee : 호출당하는 함수

### 함수형 인터페이스
- 함수형 인터페이스는 호출한 Thread에서 실행
- java8부터 도입한 함수형 프로그래밍
- 1개의 추상메서드를 갖고 있는 인터페이스
- 함수를 1급객체로 사용할 수 있음
  - 함수를 변수에 할당
  - 인자로 전달
  - 반환값으로 사용
- 함수형 인터페이스를 구현한 익명클래스를 람다식으로 구현 가능
```java
public static void main(String[] args) {
    var consumer = getConsumer();
    consumer.accept(3);

    var consumerAsLambda = getConsumerAsLambda();
    consumerAsLambda.accept(4);

    handleConsumer(integer -> {
        log.info("handleConsumer");
    });
}

public static Consumer<Integer> getConsumer() {
    // 익명클래스
    Consumer<Integer> returnValue = new Consumer<>() {
        @Override
        public void accept(Integer integer) {
            log.info("value in interface: " + integer);
        }
    };
    // 함수를 반환
    return returnValue;
}

// getConsumer를 람다식으로 표현
public static Consumer<Integer> getConsumerAsLambda() {
    return integer -> {
        log.info("value in lambda: " + integer);
    };
}

// 함수를 인자로 전달
public static void handleConsumer(Consumer<Integer> consumer) {
    consumer.accept(5);
} 
```

### 함수호출관점에서의 동기, 비동기
- A모델
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelA.png)

- B모델
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelB.png)

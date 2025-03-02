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

### 함수 호출 관점에서의 동기, 비동기
- 동기
  - Caller는 Callee의 결과에 관심이 있다
  - Caller는 결과를 이용해 action을 수행
- A모델(동기)
  - main은 getResult의 결과에 관심이 있음
  - main은 결과를 이용해 다음 코드 실행
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelA.png)

- 비동기
  - Caller는 calle의 결과에 관심이 없다
  - callee는 결과를 이용해서 callback을 수행한다
- B모델(비동기)
  - main은 getResult 결과에 관심이 없음
  - getResult는 결과를 이용해서 함수형 인터페이스 실행
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelB.png)

### 함수 호출 관점에서의 Blocking, Non-Blocking
- Blocking
  - caller는 callee가 완료될때까지 대기
  - 제어권을 calle가 가지고있음
  - caller와 다른 별도의 thread가 필요하지 않음
- Non Blocking
  - caller는 callee를 기다지지 않고 본인의 일을 수행
  - 제어권을 caller가 가지고 있음
  - caller와 별도의 callee thread가 필요

![img](https://github.com/kps990515/Webflux/blob/main/resources/modelAB.png)
- A(동기 Blocking),B모델(비동기 Blocking)
  - callee를 호출한 후, callee가 완료될때까지 caller가 아무것도 할수없음
  - 제어권이 callee한테 있음

- C모델(동기 Non-Blocking)
  - getResult를 호출한 후, getResult가 완료되지않아도 
  - main은 본인의 일을 할 수 있음
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelC.png)

- D모델(비동기 Non-blocking)
  - main은 getResult의 결과에 관심이 없음(비동기)
  - getResult를 호출한 후, getResult가 완료되지 않아도 main의 일을 수행(Non Blocking)
![img](https://github.com/kps990515/Webflux/blob/main/resources/modelD.png)
![img](https://github.com/kps990515/Webflux/blob/main/resources/resultmodelD.png)

|              | 동기                                                              | 비동기                                                         |
|--------------|-------------------------------------------------------------------|----------------------------------------------------------------|
| **Blocking** | caller는 아무 것도 할 수 없는 상태가 된다. 결과를 얻은 후 직접 처리한다. | caller는 아무 것도 할 수 없는 상태가 된다. 결과는 callee가 처리한다. |
| **Non-blocking** | caller는 자기 할 일을 할 수 있다. 결과를 얻은 후 직접 처리한다.         | caller는 자기 할 일을 할 수 있다. 결과는 callee가 처리한다.        |

### IO관점에서의 Blocking / Non Blocking
- Blocking의 종류
  - CPU-bound blocking : 오랜시간 일을 한다
  - IO-bound blocking: 오랜시간 대기를 한다
  - CPU-bound blocking
    - thread가 대부분의 시간 CPU 점유
    - 연산이 많은 경우 추가적인 코어를 투입
  - IO-bound blocking
    - thread가 대부분의 시간을 대기
    - 파일 읽기/쓰기, network 요청 처리, 요청 전달 등
    - IO-bound non-blocking 가능 하다

- Blocking의 전파
  - 하나의 함수에서 여러 함수를 호출하 기도 하고, 함수 호출은 중첩적으로 발생
  - callee는 caller가 되고 다시 다른 callee를 호출
  - blocking한 함수를 하나라도 호출한 다면 caller는 blocking이 된다
  - 함수가 non-blocking하려면 모든 함수가 non-blocking이어야 한다
  - 따라서 I/O bound blocking 또한 발생하면 안된다

![img](https://github.com/kps990515/Webflux/blob/main/resources/syncModel.png)
- 두 I/O 모델의 공통점
  - Application이 Kernel의 결과에 관심이 있다
  - caller는 callee의 결과에 관심이 있다 -> 동기
- 차이점
  - A모델(Blocking)
    - Kernel이 응답을 돌려주기 전까지 Application은 아무것도 하지 않는다
    - caller는 callee가 완료될때까지 대기 -> Blocking
  - B모델(Non-blocking)
    - Kernel이 응답을 돌려주기 전에 Application은 다른 일을 수행한다
    - caller는 callee를 기다리지 않고 본 인의 일을 한다 -> Non-blocking

![img](https://github.com/kps990515/Webflux/blob/main/resources/nonBlocking.png)
- 공통점(Non-blocking)
  - Kernel이 Application의 실행을 막지 않는다
  - caller는 callee를 기다리지 않고 본인의 일을 한다 -> Non-blocking
- 차이점
  - B모델(Sync)
    - Application은 Kernel의 결과에 관심이 있다
    - caller는 callee의 결과에 관심이 있다 -> 동기
  - C모델(Async)
    - Thread1은 Kernel의 결과에 관심이 없다
    - Kernel은 작업을 완료한 후 Thread2에게 결과를 전달한다
    - caller는 callee의 결과에 관심이 없다 -> 비동기

|              | 동기                                                                                                                  | 비동기                                                                                                 |
|--------------|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| **Blocking** | Application은 Kernel이 I/O 작업을 완료할 때까지 기다린다. 그 후 결과를 직접 이용해서 이후 본인의 일을 수행한다.           | X                                                                                                      |
| **Non-blocking** | Application은 Kernel에 주기적으로 I/O 작업이 완료되었는지 확인한다. 즉 간접한 방식으로 일을 할 수 있고 작업이 완료되면 그때 본인의 일을 수행한다. | Application은 Kernel에 I/O 작업 요청을 보내고 본인의 일을 한다. 작업이 완료되면 Kernel은 signal을 보내거나 callback을 호출한다. |

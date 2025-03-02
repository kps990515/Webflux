## Reactive Streams

### Reactive manifesto
- 소프트웨어 아키텍쳐에 대한 선언문
- Reactive system의 특성을 강조하고 구축에 필요한 가이드라인 제공
- 4가지의 핵심 가치를 제시
  - Responsive(응답성)
    - 요구사항
      - 문제를 신속하게 탐지하고 효과적으로 대처
      - 신속하고 일관성 있는 응답 시간 제공
      - 신뢰할 수 있는 상한선을 설정하여 일관된 서비스 품질을 제공
    - 결과
      - 가능한 한 즉각적으로 응답
      - 사용자의 편의성과 유용성의 기초
      - 오류 처리를 단순화
  - Resilient(복원력)
    - 요구사항
      - 복제, 봉쇄, 격리, 위임에 의해 실현
      - 봉쇄: 장애는 각각의 구성 요소에 포함 
      - 격리: 구성 요소들은 서로 분리 
      - 위임: 복구 프로세스는 다른(외부의) 구성 요소에 위임 
      - 복제: 필요한 경우 복제를 통해 고가용성이 보장
    - 결과
      - 장애에 직면하더라도 응답성을 유지
      - 시스템이 부분적으로 고장이 나더라도, 전체 시스템을 위험하게 하지 않고 복구 할 수 있도록 보장
      - 구성 요소의 클라이언트는 장애를 처리하는데에 압박을 받지 않습니다
  - Elastic(유연성)
    - 요구사항
      - 경쟁하는 지점이나 중앙 집중적인 병목 현상이 존재하지 않도록 설계
      - 구성 요소를 샤딩하거나 복제하여 입력을 분산
      - 실시간 성능을 측정하는 도구를 제공
      - 응답성 있고 예측 가능한 규모 확장 알고리즘을 지원
    - 결과
      - 작업량이 변화하더라도 응답성을 유지
      - 입력 속도의 변화에 따라 이러한 입력에 할당된 자원을 증가시키거나 감소
      - 상품 및 소프트웨어 플랫폼에 비용 효율이 높은 방식으로 유연성 을 제공
  - Message Driven(메시지 기반)
    - 요구사항
      - 비동기 메시지 전달에 의존
      - 명시적인 메시지 전달
      - 위치 투명 메시징을 통신 수단으로 사용
      - 논블로킹 통신
    - 결과
      - 비동기 통신: 구성 요소는 서로 비동기적으로 메시지를 주고 받으며, 독립적인 실행을 보장한다.
      - 메시지 큐: 메시지 큐를 생성하고 배압을 적용하여 부하를 관리하고 흐름을 제어한다
      - 복원력: 구성 요소 사이에 경계를 형성하여 직접적인 장애의 전파를 막고 장애를 메시지로 지정해서 위치와 상관 없이 동일하게 장애를 관리한다.
      - 탄력성: 구성 요소 사이에 경계를 형성하여 각각의 구성 요소를 독립적으로 확장 가능하게 만들고, 자원을 더 쉽게 추가하거나 제거한다.

### 정리
- 핵심 가치: 가능한 한 즉각적으로 응답
- 첫 번째 형태: 장애에 직면하더라도 응답성을 유지
- 두 번째 형태: 작업량이 변화하더라도 응답성을 유지
- 방법: 비동기 non-blocking 기반의 메시지 큐를 사용해서 통신

### Reactive programming
- 일반적인 서비스
  - 구성 요소 혹은 객체는 다른 객체를 직접 호출하고 데이터를 받는다
  - 이 과정에서 경계가 무너지고 구성 요소의 독립적인 실행이 보장되지 않으며 복원력, 유연성 모두 위협을 받게 된다
- 동기 Stream
  - callee는 caller에게 응답이 아닌 stream을 제공
  - callee는 각각의 값을 stream을 통해서 전달
  - caller는 해당 stream을 collect하여 이를 처리
  - 결론 : caller와 callee는 동기적으로 동작, stream이 메시지 큐의 역할을 하지만, 부하를 관리할 수 없다
- 비동기 future
  - callee는 caller에게 응답이 아닌 future를 제공
  - callee는 각각의 값을 future를 통해서 전달
  - caller는 해당 future를 chaining 하여 이를 처리
  - 결론 : caller와 callee는 비동기적으로 동작, future는 메시지 큐의 역할을 할 수 없고, 부하를 관리할 수 없다. 배압도 적용할 수 없다
- Reactive stream
  - callee는 caller에게 응답이 아닌 publisher를 제공한다
  - callee는 각각의 값을 publisher를 통해서 전달
  - caller는 해당 publisher를 subscribe하거나 다른 caller에게 전달
  - caller는 subscriber를 등록하여 backpressure를 조절하여 처리 가능한만큼 전달 받는다
  - 결론 
    - 비동기적: callee는 publisher를 반환하고 caller는 subscriber를 등록
    - publisher는 메시지 큐를 생성해서 부하를 관리하고 흐름을 제어한다. back-pressure를 조절할 수 있는 수단을 제공
- Reactive programming
  - 비동기 데이터 stream을 사용하는 패러다임
  - 모든 것이 이벤트로 구성되고 이벤트를 통해서 전파
    - 데이터의 전달, 에러, 완료 까지 모두 이벤트로 취급
  - Reactive manifesto 만족

   
### message-driven vs event-driven
- message-driven
  - message: event, command, query 등 다양한 형태를 수용
  - message-driven은 event, command, query등 다양한 형태의 메시지를 비동기적으로, 가능하다면 배압을 적용해서, 전달하는 형태에 집중
- event-driven
  - message의 형태를 event로 제한
  - completion, error 심지어 값 까지도 event의 형태로 전달

### Reactive streams
![img](https://github.com/kps990515/Webflux/blob/main/resources/ReactiveStream.png)
- Publisher
  - subscribe 함수 제공해서 publisher에 다수의 subscriber 등록 지원
  - subscription을 포함하고 Subscriber가 추가되면 subscription 제공
- Subscriber
  - subscribe하는 시점에 publisher로부터 subscription을 받을 수 있는 인자 제공
  - onNext, onError, onComplete를 통해서 값이나 이벤트를 받을 수 있다
  - onNext는 여러 번, onError나 onComplete는 딱 한 번만 호출된다
- Subscription
  - back-pressure를 조절할 수 있는 request 함수
  - Publisher가 onNext를 통해서 값을 전달하는것을 취소할 수 있는 cancel 함수

- Publisher, Subscriber 연동하기
```java
  public static void main(String[] args) throws InterruptedException {
      // Publisher 생성 (1~7까지의 정수를 발행)
      Flow.Publisher<Result> publisher = new FixedIntPublisher();
      // Subscriber 생성: 한 번에 1개의 데이터를 요청하도록 설정
      Flow.Subscriber<Result> subscriber = new RequestNSubscriber<>(1);
      // Publisher에 Subscriber 등록(구독 시작)
      publisher.subscribe(subscriber);
      // 비동기 작업이 완료될 때까지 잠시 대기 (실제 애플리케이션에서는 적절한 종료 로직 필요)
      Thread.sleep(100);
  }
```
- FixedIntPublisher
  - Flow.Publisher를 구현
  - 고정된 숫자의 integer를 전달하는 publisher
  - iterator를 생성해서 subscription을 생성하고 subscriber에게 전달
  - requestCount를 세기 위해서 Result 객체사용
```java
public class FixedIntPublisher implements Flow.Publisher<FixedIntPublisher.Result> {

    // ──────────────────────────────
    // 1. 데이터 클래스 (발행할 결과 값)
    // ──────────────────────────────
    @Data
    public static class Result {
        // 발행되는 정수 값
        private final Integer value;
        // 해당 값이 발행된 시점의 요청 횟수
        private final Integer requestCount;
    }

    // ──────────────────────────────
    // 2. Publisher 인터페이스 구현
    // ──────────────────────────────
    @Override
    public void subscribe(Flow.Subscriber<? super Result> subscriber) {
        // 1~7까지의 정수를 포함하는 리스트를 동기화된 리스트로 생성
        List<Integer> numbers = Collections.synchronizedList(new ArrayList<>(List.of(1, 2, 3, 4, 5, 6, 7)));
        // 리스트에 대한 iterator 생성
        Iterator<Integer> iterator = numbers.iterator();
        // IntSubscription 객체 생성: Publisher와 Subscriber 간의 데이터 발행을 관리하는 구독 객체
        IntSubscription subscription = new IntSubscription(subscriber, iterator);
        // Subscriber에게 구독이 시작되었음을 알림 (onSubscribe 호출)
        subscriber.onSubscribe(subscription);
    }
```
- IntSubscription
  - Flow.Subscription을 구현
  - subscriber의 onNext와 subscription의 request가 동기적으로 동작하면 안되기 때문에 executor를 이용해서 별도의 쓰레드에서 실행
  - 요청 횟수를 count에 저장하고 결과에 함께 전달
  - 더 이상 iterator에 값이 없으면, onComplete 호출
  ```java
    // ──────────────────────────────
    // 3. 구독(Subscription) 관리 클래스
    // ──────────────────────────────
    // Publisher와 Subscriber 사이의 데이터 요청 및 발행을 제어하는 역할을 함.
    static class IntSubscription implements Flow.Subscription {
        // 데이터를 전달할 Subscriber
        private final Flow.Subscriber<? super Result> subscriber;
        // 발행할 숫자들의 iterator
        private final Iterator<Integer> numbers;
        // 비동기 작업을 처리할 단일 스레드 Executor
        private final ExecutorService executor = Executors.newSingleThreadExecutor();
        // 요청 횟수를 관리하는 AtomicInteger
        private final AtomicInteger count = new AtomicInteger(1);
        // 완료 여부를 관리하는 AtomicBoolean
        private final AtomicBoolean isCompleted = new AtomicBoolean(false);

        // 생성자: Subscriber와 숫자 iterator를 받아 초기화
        public IntSubscription(Flow.Subscriber<? super Result> subscriber, Iterator<Integer> numbers) {
            this.subscriber = subscriber;
            this.numbers = numbers;
        }

        // Subscriber로부터 데이터 요청(request)이 들어오면 호출됨
        @Override
        public void request(long n) {
            // Executor를 사용해 비동기로 n개의 데이터를 발행하는 작업을 수행
            executor.submit(() -> {
                for (int i = 0; i < n; i++) {
                    // 아직 발행할 데이터가 남아있다면
                    if (numbers.hasNext()) {
                        int number = numbers.next();
                        // 발행한 데이터는 리스트에서 제거 (중복 발행 방지)
                        numbers.remove();
                        // 새로운 Result 객체를 생성해 Subscriber에게 전달
                        subscriber.onNext(new Result(number, count.get()));
                    } else {
                        // 더 이상 발행할 데이터가 없으면 완료 처리
                        boolean isChanged = isCompleted.compareAndSet(false, true);
                        if (isChanged) {
                            // Executor 종료 및 Subscriber에게 완료 알림
                            executor.shutdown();
                            subscriber.onComplete();
                            isCompleted.set(true);
                        }
                        break;
                    }
                }
                // 요청 처리가 끝날 때마다 요청 횟수를 증가시킴
                count.incrementAndGet();
            });
        }

        // 구독 취소 시 호출되는 메서드
        @Override
        public void cancel() {
            // 취소 시 Subscriber에게 완료 알림 전송
            subscriber.onComplete();
        }
    }
  ```
- RequestNSubscriber
  - Flow.Subscriber를 구현
  - 최초 연결시 1개를 고정적으로 요청
  - onNext에서 count를 세고 n번째 onNext마다 request
  ```java
    // ──────────────────────────────
    // 4. Subscriber 구현 클래스
    // ──────────────────────────────
    // 데이터를 n개씩 요청하며, 데이터를 받을 때마다 로그를 남기고 추가 요청하는 Subscriber
    public static class RequestNSubscriber<T> implements Flow.Subscriber<T> {
        // 한 번에 요청할 데이터 개수
        private final Integer n;
        // 구독 객체를 저장하기 위한 변수
        private Flow.Subscription subscription;
        // 수신한 데이터 개수를 카운트
        private int count = 0;

        // 생성자: 요청 단위를 받아 저장
        public RequestNSubscriber(Integer n) {
            this.n = n;
        }

        // 구독이 시작될 때 호출됨 (Publisher가 onSubscribe 호출)
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            this.subscription = subscription;
            // 초기 데이터 요청: 1개의 데이터를 먼저 요청
            this.subscription.request(1);
        }

        // 데이터가 도착할 때마다 호출됨
        @Override
        public void onNext(T item) {
            log.info("item: {}", item);
            // 수신한 데이터 개수를 증가시킴
            count++;
            // 만약 count가 요청 단위(n)의 배수이면 추가로 n개의 데이터를 요청
            if (count % n == 0) {
                log.info("send request");
                this.subscription.request(n);
            }
        }

        // 에러 발생 시 호출됨
        @Override
        public void onError(Throwable throwable) {
            log.error("error: {}", throwable.getMessage());
        }

        // 모든 데이터 전송이 완료되면 호출됨
        @Override
        public void onComplete() {
            log.info("complete");
        }
    }
  ```
  - n이 1일때
    - 1개 처리하고 1개 요청을 complete할 때까지 반복
    - requestCount가 1개씩 증가
  - n이 3일때
    - 1개씩 처리한 후 3개를 요청
    ```java
    55:31 [pool-2-thread-1] - item: value=1, requestCount=1
    55:31 [pool-2-thread-1] - send request
    55:31 [pool-2-thread-1] - item: value=2, requestCount=2
    55:31 [pool-2-thread-1] - item: value=3, requestCount=2
    55:31 [pool-2-thread-1] - item: value=4, requestCount=2
    55:31 [pool-2-thread-1] - send request
    55:31 [pool-2-thread-1] - item: value=5, requestCount=3
    55:31 [pool-2-thread-1] - item: value=6, requestCount=3
    55:31 [pool-2-thread-1] - item: value=7, requestCount=3
    55:31 [pool-2-thread-1] - send request
    55:31 [pool-2-thread-1] - complete
    ``` 
    
- Hot Publisher
  - subscriber가 없더라도 데이터를 생성하고 stream에 push하는 publisher
  - 트위터 게시글 읽기, 공유 리소스 변화 등
  - 여러 subscriber에게 동일한 데이터 전달
- Cold Publisher
  - subscribe가 시작되는 순간부터 데이터를 생성하고 전송
  - 파일 읽기, 웹 API 요청 등
  - subscriber에 따라 독립적인 데이터 스트림 제공


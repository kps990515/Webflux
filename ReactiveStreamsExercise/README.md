## Reactive streams 실습
- ColdPublisher 구조에 대해서 파악한다
- HotPublisher를 구현한다

- Subscriber
```java
public class SimpleNamedSubscriber<T> implements Flow.Subscriber<T> {
    private Flow.Subscription subscription;
    private final String name;

    public SimpleNamedSubscriber(String name) {
        this.name = name;
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        this.subscription.request(1);
        log.info("onSubscribe");
    }

    @Override
    public void onNext(T item) {
        log.info("name: {}, onNext: {}", name, item);
        this.subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        log.error("onError: {}", throwable.getMessage());
    }

    @Override
    public void onComplete() {
        log.info("onComplete");
    }

    public void cancel() {
        log.info("cancel");
        this.subscription.cancel();
    }
}
```

- ColdPublisher Main
```java
public class SimpleColdPublisherMain {
    @SneakyThrows
    public static void main(String[] args) {
        // create publisher
        var publisher = new SimpleColdPublisher();

        // create subscriber1
        var subscriber = new SimpleNamedSubscriber<Integer>("subscriber1");
        publisher.subscribe(subscriber);

        Thread.sleep(5000);

        // create subscriber2
        var subscriber2 = new SimpleNamedSubscriber<Integer>("subscriber2");
        publisher.subscribe(subscriber2);
    }
}
```
- Cold Publisher
```java
// Flow API를 사용하여 정수를 발행하는 "Cold Publisher"를 구현한 클래스
public class SimpleColdPublisher implements Flow.Publisher<Integer> {

  // Subscriber가 구독할 때 호출되는 메서드
  @Override
  public void subscribe(Flow.Subscriber<? super Integer> subscriber) {
    // 1부터 9까지의 정수를 포함하는 리스트를 생성하고, 동기화된 리스트의 iterator를 생성
    var iterator = Collections.synchronizedList(
            IntStream.range(1, 10).boxed().collect(Collectors.toList())
    ).iterator();

    // iterator와 subscriber를 사용하여 구독(subscription) 객체 생성
    var subscription = new SimpleColdSubscription(iterator, subscriber);

    // Subscriber에게 구독이 시작되었음을 알리기 위해 onSubscribe() 호출
    subscriber.onSubscribe(subscription);
  }

  // Flow.Subscription 인터페이스를 구현한 내부 클래스
  // 구독을 관리하며 데이터를 발행하는 역할을 수행함
  @RequiredArgsConstructor // Lombok 어노테이션: final 필드를 위한 생성자 자동 생성
  public class SimpleColdSubscription implements Flow.Subscription {

    // 데이터를 순차적으로 제공할 iterator (1~9의 정수를 담고 있음)
    private final Iterator<Integer> iterator;

    // 데이터를 전달받을 Subscriber
    private final Flow.Subscriber<? super Integer> subscriber;

    // 비동기 작업 처리를 위한 단일 스레드 ExecutorService
    private final ExecutorService executorService = Executors.newSingleThreadExecutor();

    // Subscriber가 데이터를 요청(request)할 때 호출됨
    @Override
    public void request(long n) {
      // 요청된 n개의 데이터를 비동기적으로 발행하기 위해 작업 제출
      executorService.submit(() -> {
        for (int i = 0; i < n; i++) {
          if (iterator.hasNext()) {
            var number = iterator.next();
            iterator.remove();
            // Subscriber에게 데이터를 전달 (onNext 호출)
            subscriber.onNext(number);
          } else {
            // 더 이상 발행할 데이터가 없으면, 구독 완료를 알림
            subscriber.onComplete();
            // ExecutorService 종료 (비동기 작업 종료)
            executorService.shutdown();
            break;
          }
        }
      });
    }

    // 구독이 취소되었을 때 호출됨
    @Override
    public void cancel() {
      // 취소 시에도 구독이 완료되었음을 Subscriber에게 알림
      subscriber.onComplete();
    }
  }
}
```

- Hot Publisher Main
```java
public class SimpleHotPublisherMain {
    @SneakyThrows
    public static void main(String[] args) {
        // prepare publisher
        var publisher = new SimpleHotPublisher();

        // prepare subscriber1
        var subscriber = new SimpleNamedSubscriber<>("subscriber1");
        publisher.subscribe(subscriber);

        // cancel after 5s
        Thread.sleep(5000);
        subscriber.cancel();

        // prepare subscriber2,3
        var subscriber2 = new SimpleNamedSubscriber<>("subscriber2");
        var subscriber3 = new SimpleNamedSubscriber<>("subscriber3");
        publisher.subscribe(subscriber2);
        publisher.subscribe(subscriber3);

        // cancel after 5s
        Thread.sleep(5000);
        subscriber2.cancel();
        subscriber3.cancel();


        Thread.sleep(1000);

        var subscriber4 = new SimpleNamedSubscriber<>("subscriber4");
        publisher.subscribe(subscriber4);

        // cancel after 5s
        Thread.sleep(5000);
        subscriber4.cancel();

        // shutdown publisher
        publisher.shutdown();
    }
}
```

- Hot Publisher
```java
// Flow API를 사용하여 "Hot Publisher"를 구현한 클래스
// Hot Publisher는 구독 시점에 상관없이 지속적으로 데이터를 발행하는 Publisher입니다.
public class SimpleHotPublisher implements Flow.Publisher<Integer> {

    // Publisher 내부에서 실행할 단일 스레드 ExecutorService
    private final ExecutorService publisherExecutor = Executors.newSingleThreadExecutor();
    // Publisher에서 실행되는 비동기 작업(Future 객체)
    private final Future<Void> task;
    private List<Integer> numbers = new ArrayList<>();
    
    // 구독(Subscription)들을 관리하는 리스트
    private List<SimpleHotSubscription> subscriptions = new ArrayList<>();

    // 생성자: Hot Publisher를 위한 지속적으로 데이터를 발행하는 작업을 시작
    public SimpleHotPublisher() {
        // 초기 값으로 1을 리스트에 추가
        numbers.add(1);
        
        // publisherExecutor를 이용하여 비동기 작업 실행
        // 이 작업은 무한 루프를 돌면서 새로운 정수를 계속 추가하고, 구독자들에게 알림(wakeup) 호출
        task = publisherExecutor.submit(() -> {
            for (int i = 2; !Thread.interrupted(); i++) {
                // 새로운 정수를 numbers 리스트에 추가
                numbers.add(i);
                
                // 모든 구독자(subscription)에 대해 새 데이터가 추가되었음을 알림(wakeup 호출)
                subscriptions.forEach(SimpleHotSubscription::wakeup);
                
                // 100밀리초 대기 (데이터 발행 간의 간격)
                Thread.sleep(100);
            }
            return null;
        });
    }

    // Publisher 종료 메서드: 작업 취소 및 ExecutorService 종료
    public void shutdown() {
        this.task.cancel(true);
        publisherExecutor.shutdown();
    }

    // Subscriber가 구독을 요청하면 호출되는 메서드
    @Override
    public void subscribe(Flow.Subscriber<? super Integer> subscriber) {
        // 새로운 구독 객체 생성
        var subscription = new SimpleHotSubscription(subscriber);
        // Subscriber에게 구독이 시작되었음을 알리기 위해 onSubscribe() 호출
        subscriber.onSubscribe(subscription);
        // 생성된 구독 객체를 관리 리스트에 추가
        subscriptions.add(subscription);
    }

    // ──────────────────────────────
    // 내부 클래스: SimpleHotSubscription
    // Flow.Subscription 인터페이스를 구현하여 구독자와의 데이터 전달을 관리함
    // ──────────────────────────────
    private class SimpleHotSubscription implements Flow.Subscription {
        // 현재 구독자가 읽은 데이터의 인덱스(offset)
        private int offset;
        
        // 구독자가 요청한 데이터의 총 개수를 반영하는 인덱스(requiredOffset)
        private int requiredOffset;
        
        // 데이터를 전달받을 Subscriber
        private final Flow.Subscriber<? super Integer> subscriber;
        
        // 구독자 전용 ExecutorService로, 구독자가 데이터를 받아 처리하는 작업을 비동기로 실행
        private final ExecutorService subscriptionExecutorService = Executors.newSingleThreadExecutor();

        // 생성자: 구독자가 구독을 시작할 때 호출됨
        // 현재까지 발행된 데이터의 마지막 인덱스를 기준으로 offset과 requiredOffset을 초기화
        public SimpleHotSubscription(Flow.Subscriber<? super Integer> subscriber) {
            int lastElementIndex = numbers.size() - 1; // 초기에는 0 (리스트에 1이 들어있으므로)
            this.offset = lastElementIndex;
            this.requiredOffset = lastElementIndex;
            this.subscriber = subscriber;
        }

        // Subscriber가 추가 데이터를 요청할 때 호출되는 메서드
        @Override
        public void request(long n) {
            // 요청된 개수만큼 requiredOffset 증가 (요청한 만큼의 데이터가 있어야 함)
            requiredOffset += n;
            // 가능한 한 많이 데이터를 전달하도록 시도
            onNextWhilePossible();
        }

        // Publisher가 새로운 데이터 발행 시, 구독자에게 데이터를 전달하기 위해 호출하는 메서드
        public void wakeup() {
            onNextWhilePossible();
        }

        // 구독이 취소되었을 때 호출되는 메서드
        @Override
        public void cancel() {
            // 구독이 취소되었음을 Subscriber에게 알리기 위해 onComplete 호출
            this.subscriber.onComplete();
            // 현재 구독 객체가 subscriptions 리스트에 존재하면 제거
            if (subscriptions.contains(this)) {
                subscriptions.remove(this);
            }
            // 구독자 전용 ExecutorService 종료
            subscriptionExecutorService.shutdown();
        }

        // 구독자가 요청한 만큼의 데이터를 전달하는 메서드
        // 구독자 전용 ExecutorService에서 비동기로 실행
        private void onNextWhilePossible() {
            subscriptionExecutorService.submit(() -> {
                // offset이 requiredOffset보다 작고, 아직 numbers 리스트에 데이터가 남아있다면
                while (offset < requiredOffset && offset < numbers.size()) {
                    // 현재 offset 위치의 데이터를 가져옴
                    var item = numbers.get(offset);
                    // Subscriber에게 데이터를 전달
                    subscriber.onNext(item);
                    // 다음 데이터로 이동
                    offset++;
                }
            });
        }
    }
}

```
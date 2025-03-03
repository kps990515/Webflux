## CompletableFuture
- java8에서 도입
- 비동기 프로그래밍 지원
- 람다, Method Reference 지원

### Method Reference
- ::연산자를 이용해 함수에 대한 참조를 간결하게 표현
- method reference: 
- static method reference
- instance method reference
- constructor method reference
```java
@RequiredArgsConstructor
public static class Person {
  @Getter
  private final String name;
  public Boolean compareTo(Person o) {
    return o.name.compareTo(name) > 0;
  }
}
  public static void print(String name) {
    System.out.println(name);
  }
  public static void main(String[] args) {
    var target = new Person("f");
    Consumer<String> staticPrint = MethodReferenceExample::print;
    Stream.of("a", "b", "g", "h")
            .map(Person::new) // constructor reference
            .filter(target::compareTo) // method reference
            .map(Person::getName) // instance method reference
            .forEach(staticPrint); // static method reference
  } 
```

### CompletableFuture 클래스
- Future + CompletionStage
- Future
  - 비동기적인 작업을 수행
  - 해당 작업이 완료되면 결과를 반환하는 인터페이스
- CompletionStage
  - 비동기적인 작업을 수행
  - 해당 작업이 완료되면 결과를 처리하거나
  - 다른 CompletionStage를 연결 하는 인터페이스


- Future: 단순한 비동기 작업 결과 표현. 블로킹 호출을 사용하며, 체이닝이나 콜백 지원이 없음.
- CompletionStage: 비동기 작업을 논블로킹 방식으로 체이닝할 수 있는 인터페이스.
- CompletableFuture: 두 인터페이스의 장점을 모두 갖춘 구현체로, 비동기 작업 결과의 조회 및 복합 작업 구성에 모두 활용 가능.

### Future 인터페이스
- ExecutorService
  - 쓰레드 풀을 이용하여 비동기적으로 작업을 실행하고 관리
  - 별도의 쓰레드를 생성하고 관리하지 않아도 되므로, 코드를 간결하게 유지 가능
  - 쓰레드 풀을 이용하여 자원을 효율적으로 관리
  ```java
  public interface ExecutorService extends Executor {
    void execute(Runnable command);
    <T> Future<T> submit(Callable<T> task);
    void shutdown();
  } 
  ```
  - execute: Runnable 인터페이스를 구현한 작업을 쓰레드 풀에서 비동기적으로 실행
  - submit: Callable 인터페이스를 구현한 작업을 쓰레드 풀에서 비동기적으로 실행하고, 해당 작업의 결과를 Future<T> 객체로 반환
  - shutdown: ExecutorService를 종료. 더 이상 task를 받지 않는다

#### Executors - ExecutorService 생성
- newSingleThreadExecutor: 단일 쓰레드로 구성된 스레드 풀을 생성. 한 번에 하나의 작업만 실행
- newFixedThreadPool: 고정된 크기의 쓰레드 풀을 생성. 크기는 인자로 주어진 n과 동일
- newCachedThreadPool: 사용 가능한 쓰레드가 없다면 새로 생성해서 작업을 처리하고, 있다면 재사용. 쓰레드가 일정 시간 사용되지 않으면 회수
- newScheduledThreadPool: 스케줄링 기능을 갖춘 고정 크기의 쓰레드 풀을 생성. 주기적이거나 지연이 발생하는 작업을 실행
- newWorkStealingPool: work steal 알고리즘을 사용하는 ForkJoinPool을 생성

#### 예제
```java
// 새로운 쓰레드를 생성하여 1을 반환
public static Future<Integer> getFuture() {
   var executor = Executors.newSingleThreadExecutor();
   
   try {
     return executor.submit(() > {
     return 1;
     });
   } finally {
   executor.shutdown();
   }
}

// 새로운 쓰레드를 생성하고 1초 대기 후 1을 반환
public static Future<Integer> getFutureCompleteAfter1s() {
   var executor = Executors.newSingleThreadExecutor();
 
   try {
     return executor.submit(() > {
     Thread.sleep(1000);
     return 1;
     });
   } finally {
   executor.shutdown();
   }
}
```

```java
public interface Future<V> {
   boolean cancel(boolean mayInterruptIfRunning);
   boolean isCancelled();
   boolean isDone();
   V get() throws InterruptedException, ExecutionException;
   V get(long timeout, TimeUnit unit)
   throws InterruptedException, ExecutionException, TimeoutException;
}
```
- Future : isDone(), isCancelled()
  - future의 상태를 반환
  - isDone: task가 완료되었다면, 원인과 상관없이 true 반환
  - isCancelled: task가 명시적으로 취소된 경우, true 반환
- Future : get()
  - 결과를 구할 때까지 thread가 계속 block
  - future에서 무한 루프나 오랜 시간이 걸린다면 thread가 blocking 상태 유지
```java
Future future = FutureHelper.getFuture();
assert !future.isDone();
assert !future.isCancelled();

var result = future.get();
assert result.equals(1);
assert future.isDone();
assert !future.isCancelled();
```
- Future : get(long timeout, TimeUnit unit)
  - 결과를 구할 때까지 timeout동안 thread가 block
  - timeout이 넘어가도 응답이 반환되지 않으면 TimeoutException 발생
```java
Future future = FutureHelper.getFutureCompleteAfter1s();
var result = future.get(1500, TimeUnit.MILLISECONDS);
assert result.equals(1);

Future futureToTimeout = FutureHelper.getFutureCompleteAfter1s();
Exception exception = null;
try {
    futureToTimeout.get(500, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
    exception = e;
}
assert exception != null;
```

- Future : cancel(boolean mayInterruptIfRunning)
  - future의 작업 실행을 취소
  - 취소할 수 없는 상황이라면 false를 반환
  - mayInterruptIfRunning가 false라면 시작하지 않은 작업에 대해서만 취소
  ```java
  Future future = FutureHelper.getFuture();
  var successToCancel = future.cancel(true);
  assert future.isCancelled();
  assert future.isDone();
  assert successToCancel;
  
  successToCancel = future.cancel(true);
  assert future.isCancelled();
  assert future.isDone();
  assert !successToCancel;
  ```
- Future 인터페이스의 한계
  - cancel을 제외하고 외부에서 future를 컨트롤 할 수 없다
  - 반환된 결과를 get() 해서 접근하기 때문에 비동기 처리가 어렵다
  - 완료되거나 에러가 발생했는지 구분하기 어렵다

### CompletionStage 인터페이스
```java
public interface CompletionStage<T> {
 public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);
 public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);
 
 public CompletionStage<Void> thenAccept(Consumer<? super T> action);
 public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
 
 public CompletionStage<Void> thenRun(Runnable action);
 public CompletionStage<Void> thenRunAsync(Runnable action);
 
 public <U> CompletionStage<U> thenCompose(Function<? super T, ? extends CompletionStage<U > fn);
 public <U> CompletionStage<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U > fn);
 
 public CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);
}
```

- CompletionStage 연산자 조합
  - 50개에 가까운 연산자들을 활용하여 비동기 task들을 실행하고 값을 변형하는 등 chaining 을 이용한 조합 가능
  - 에러를 처리하기 위한 콜백 제공
  ```java
  Helper.completionStage()
         .thenApplyAsync(value > {
           log.info("thenApplyAsync: {}", value);
           return value + 1;
         }).thenAccept(value > {
           log.info("thenAccept: {}", value);
         }).thenRunAsync(() > {
           log.info("thenRun");
         }).exceptionally(e > {
           log.info("exceptionally: {}", e.getMessage());
           return null;
         });
  Thread.sleep(100);
  ```
   
- ForkJoinPool - thread pool
  - CompletableFuture는 내부적으로 비동기 함수들을 실행하기 위해 ForkJoinPool을 사용
  - ForkJoinPool의 기본 size = 할당된 cpu 코어 - 1
  - 데몬 쓰레드: main 쓰레드가 종료되면 즉각적으로 종료
- ForkJoinPool - fork & join
  - Task를 fork를 통해서 subtask로 나누고
  - Thread pool에서 steal work 알고리즘을 이용해서 균등하게 처리
  - join을 통해서 결과를 생성
  

- CompletionStage 연산자
  - 함수형 인터페이스: Function, Consumer, Supplier, Runnable
    - thenAccept(Consumer action): Consumer - accept 메소드
      - thenAcceptAsync 지원
      - Consumer를 파라미터로 받는다
      - 이전 task로부터 값을 받지만 값을 넘기지 않는다
      - 다음 task에게 null이 전달된다
      - 값을 받아서 action만 수행하는 경우 유용
      - thenAccept vs thenAcceptAsync
        - ![img](https://github.com/kps990515/Webflux/blob/main/resources/thenAsync.png)
        - done 상태에서 thenAccept는 caller(main)의 쓰레드에서 실행
        - done 상태가 아니면 callee(forkJoinPool)의 쓰레드에서 실행
        - done 상태에서 thenAcceptAsync는 별도의 ForkJoinPool에서 실행
        ```java
        log.info("start main");
        CompletionStage<Integer> stage = Helper.finishedStage();
        stage.thenAccept(i > {
          log.info("{} in thenAccept", i);
        }).thenAccept(i > {
          log.info("{} in thenAccept2", i);
        });
        log.info("after thenAccept");
        Thread.sleep(100);
        // 메인쓰레드에서 실행
        // [main] - start main
        // [ForkJoinPool.commonPool-worker-19] - return in future
        // [main] - 1 in thenAccept
        // [main] - null in thenAccept2
        // [main] - after thenAccept
        ```
        ```java
        log.info("start main");
        CompletionStage<Integer> stage = Helper.finishedStage();
        stage.thenAcceptAync(i > {
            log.info("{} in thenAccept", i);
        }).thenAccept(i > {
            log.info("{} in thenAccept2", i);
        });
        log.info("after thenAccept");
        Thread.sleep(100);
        // ForkJoinPool에서 실행
        // [main] - start main
        // [ForkJoinPool.commonPool-worker-19] - return in future
        // [main] - after thenAccept
        // [ForkJoinPool.commonPool-worker-19] - 1 in thenAccept
        // [ForkJoinPool.commonPool-worker-5] - null in thenAccept2
        ```
      - then*Async의 쓰레드풀 변경
        - 모든 then*Async 연산자는 executor를 추가 인자로 받는다
        - 이를 통해서 다른 쓰레드풀로 task를 실행할 수 있다
  - thenApply(Function fn): Function - apply 메소드
    - Function을 파라미터로 받는다
    - 이전 task로부터 T 타입의 값을 받아서 가공하고 U 타입의 값을 반환한다
    - 다음 task에게 반환했던 값이 전달된다
    - 값을 변형해서 전달해야 하는 경우 유용
    - ![img](https://github.com/kps990515/Webflux/blob/main/resources/thenApplyAsync.png)

  
  - thenCompose(Function fn): Function - compose 메소드 (추상 메소드 x)
    - Function을 파라미터로 받는다
    - 이전 task로부터 T 타입의 값을 받아서 가공하고 U 타입의 CompletionStage를 반환한다
    - 반환한 CompletionStage가 done 상태가 되면 값을 다음 task에 전달한다
    - 다른 future를 반환해야하는 경우 유용
    - ![img](https://github.com/kps990515/Webflux/blob/main/resources/thenComposeAsync.png)
  - thenRun(Runnable action): Runnable - run 메소드
    - Runnable을 파라미터로 받는다
    - 이전 task로부터 값을 받지 않고 값을 반환하지 않는다
    - 다음 task에게 null이 전달된다
    - future가 완료되었다는 이벤트를 기록할 때 유용
    - ![img](https://github.com/kps990515/Webflux/blob/main/resources/thenRunAsync.png)
  - exceptionally
    - Function을 파라미터로 받는다
    - 이전 task에서 발생한 exception을 받아서 처리하고 값을 반환한다
    - 다음 task에게 반환된 값을 전달한다
    - future 파이프에서 발생한 에러를 처리할때 유용
    - ![img](https://github.com/kps990515/Webflux/blob/main/resources/exceptionally.png)

### CompletableFuture
- 기본적으로 모두다 별개의 쓰레드로 실행
```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
   public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) { … }
   public static CompletableFuture<Void> runAsync(Runnable runnable) { … }
  
   public boolean complete(T value) { … }
   public boolean isCompletedExceptionally() { … }
  
   public static CompletableFuture<Void> allOf(CompletableFuture<?> . cfs) { … }
   public static CompletableFuture<Object> anyOf(CompletableFuture<?> . cfs) { … }
}
```
- supplyAsync
  - Supplier를 제공하여 CompletableFuture를 생성 가능
  - Supplier의 반환값이 CompletableFuture의 결과로
- runAsync
  - Runnable를 제공하여 CompletableFuture를 생성할 수 있다
  - 값을 반환하지 않는다
  - 다음 task에 null이 전달된다
![img](https://github.com/kps990515/Webflux/blob/main/resources/supplyrunAsync.png)
- complete
  - CompletableFuture가 완료되지 않았다면 주어진 값으로 채운다
  - complete에 의해서 상태가 바뀌었다면 true, 아니라면 false를 반환한다
  ```java
  CompletableFuture<Integer> future = new CompletableFuture >();
  assert !future.isDone();
  
  var triggered = future.complete(1);
  assert future.isDone();
  assert triggered;
  assert future.get() = 1;
  
  triggered = future.complete(2);
  assert future.isDone();
  assert !triggered;
  assert future.get() = 1;
  ```
![img](https://github.com/kps990515/Webflux/blob/main/resources/CompletableFutureStatus.png)
- isCompletedExceptionally 
  - Exception에 의해서 complete 되었는지 확인 할 수 있다
  ```java
  var futureWithException = CompletableFuture.supplyAsync(() > {
    return 1 / 0;
  });
  Thread.sleep(100);
  assert futureWithException.isDone();
  assert futureWithException.isCompletedExceptionally();
  ```
- allOf
  - 여러 completableFuture를 모아서 하나의 completableFuture로 변환할 수 있다
  - 모든 completableFuture가 완료되면 상태가 done으로 변경
  - Void를 반환하므로 각각의 값에 get으로 접근해야 한다
  - 병렬적으로 실행되기에 가장 늦는것과 시간이 비슷
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/allof.png)
- anyOf
  - 여러 completableFuture를 모아서 하나의 completableFuture로 변환할 수 있다
  - 주어진 future 중 하나라도 완료되면 상태가 done으로 변경
  - 제일 먼저 done 상태가 되는 future의 값을 반환
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/anyof.png)
      
  
#### CompletableFuture의 한계
  - 지연 로딩 기능을 제공하지 않는다
    - CompletableFuture를 반환하는 함수를 호출시 즉시 작업이 실행된다
  - 지속적으로 생성되는 데이터를 처리하기 어렵다
    - CompletableFuture에서 데이터를 반환하고 나면 다시 다른 값을 전달하기 어렵다




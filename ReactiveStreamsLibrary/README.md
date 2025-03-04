## Reactive Streams Library
| 항목                         | Project Reactor                                                | RxJava                                                           | Mutiny                                              |
|----------------------------|---------------------------------------------------------------|------------------------------------------------------------------|-----------------------------------------------------|
| **개요 및 배경**             | Spring 생태계와 긴밀한 통합, Pivotal/VMware 주도                    | 다양한 플랫폼 지원, 특히 Android 등에서 인기 있는 Rx 확장판                   | 최신 Quarkus/MicroProfile에 최적화, 간결한 API 제공          |
| **API 및 데이터 타입**        | Mono (0개 또는 1개), Flux (0개 이상)                              | Observable/Flowable (0개 이상), Single, Maybe, Completable            | Uni (최대 1개), Multi (0개 이상)                      |
| **Reactive Streams & Backpressure** | Reactive Streams 규격 준수, 효과적인 backpressure 지원                | Reactive Streams 준수, 다양한 연산자와 함께 backpressure 지원             | Reactive Streams 준수, 단순하고 직관적인 backpressure 처리        |
| **통합 및 생태계**           | Spring WebFlux, Spring Data Reactive 등과 강력하게 통합됨            | 다양한 프레임워크 및 플랫폼과 호환, 특히 Android 커뮤니티에서 오랜 검증됨     | Quarkus 및 MicroProfile 기반 프로젝트에 최적화됨             |
| **사용성 및 학습 곡선**       | 풍부한 기능과 연산자 제공으로 초기 학습이 다소 복잡할 수 있음             | 다양한 연산자로 복잡한 데이터 흐름 제어 가능, 학습 곡선은 중간 정도           | API가 간결하여 비교적 학습 곡선이 낮고 빠른 접근성이 장점          |
| **성능 및 확장성**           | 고성능 및 안정적인 확장성 제공, Spring 기반 프로젝트에 적합             | 다양한 사용 사례에 검증된 성능, 확장성도 우수함                          | 최신 경량 반응형 프로그래밍을 지원하며, 빠른 개발 및 유지보수가 용이함  |
| **결론**                    | Spring 기반 애플리케이션에 최적, 안정적이고 성숙한 생태계 지원          | 다양한 환경 지원과 풍부한 기능 제공, 복잡한 흐름 제어에 강점               | 최신 반응형 프로그래밍 패러다임에 맞춰 간결하고 직관적인 API 제공      |

### Project reactor
- Spring reactor에서 사용
- Mono와 Flux publisher 제공
- Publisher -> CorePublisher -> Mono, Flux
  

#### Flux
  - 0..n개의 item을 전달
  - 에러가 발생하면 error signal 전달하고 종료
  - 모든 item을 전달했다면 complete signal 전달하고 종료
  - backPressure 지원
  

  - Main쓰레드 실행 
    - subscribe()가 호출되면 Publisher가 내부적으로 onSubscribe()를 자동으로 호출
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/flux.png)
  - SubscribeOn(별도 쓰레드에서 실행)
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxSubscribeOn.png)
  - Subscribe하지 않으면 아무일도 일어나지 않는다
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxSubscribe.png)
  - BackPressure
    - getItems : Flux.fromIterable(List.of(1,2,3,4,5))를 통해 1부터 5까지의 값을 발행하는 Flux 객체를 생성
    - request() : BackPressure 제어
      - Reactive Streams 규격의 Subscription 인터페이스에 정의된 메서드
      - 구독자(Subscriber)가 자신이 처리할 수 있는 아이템의 개수를 퍼블리셔(Publisher)에게 알리는 역할
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxBackPressure.png)
  - Error : 에러가 발생시 바로 중단
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxError.png)
  - Complete : complete 실행시 바로 중단
  ![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxComplete.png)

#### Mono
- 1개의 item만 전달하기 때문에 next 하나만 실행하면 complete가 보장
- 혹은 전달하지 않고 complete를 하면 값이 없다는 것을 의미
- 하나의 값이 있거나 없다
```java
@Slf4j
public class MonoSimpleExample {
   @SneakyThrows
   public static void main(String[] args) {
     log.info("start main");
     getItems()
        .subscribe(new SimpleSubscriber<>(Integer.MAX_VALUE));
     log.info("end main");
     Thread.sleep(1000);
   }
   private static Mono<Integer> getItems() {
     return Mono.create(monoSink -> {
        monoSink.success(1);
     });
   }
}
```

#### Mono와 Flux
- Mono
  - Mono<T>: Optional<T> 
  - 없거나 혹은 하나의 값
  - Mono<Void>로 특정 사건이 완료되는 시점을 가리킬 수도 있다
- Flux
  - Flux<T>: List<T>
  - 무한하거나 유한한 여러 개의 값
![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxToMono1.png)
![img](https://github.com/kps990515/Webflux/blob/main/resources/fluxToMono2.png)
![img](https://github.com/kps990515/Webflux/blob/main/resources/monoToFlux1.png)
![img](https://github.com/kps990515/Webflux/blob/main/resources/monoToFlux2.png)

### RXJava
- Flowable, Observable, Single, Maybe, Completable, publisher 제공
- Flowable
  - 0..n개의 item을 전달
  - 에러가 발생하면 error signal 전달하고 종료
  - 모든 item을 전달했다면 complete signal 전달하고 종료
  - backPressure 지원
  - Reactor의 Flux와 유사
- Observable
  - 0..n개의 item을 전달
  - 에러가 발생하면 error signal 전달하고 종료
  - 모든 item을 전달했다면 complete signal 전달하고 종료
  - backPressure 지원 X
  
| 항목                           | Flowable                                         | Observable                                         |
|--------------------------------|--------------------------------------------------|----------------------------------------------------|
| **기반**                      | Pull 기반                                        | Push 기반                                          |
| **데이터 전달 방식**           | Subscriber가 request()로 요청한 만큼 아이템 전달      | Subscriber가 처리할 수 없더라도 아이템 전달             |
| **Reactive manifesto 준수**    | 메시지 드리븐을 모두 준수                         | 메시지 드리븐을 일부만 준수                         |
| **onSubscribe 전달 객체**      | Subscription 전달                                | Disposable 전달                                    |

- Single
  - 1개의 item을 전달 후 바로 onComplete signal 전달
  - 1개의 item이 없다면 onError signal 전달
  - 에러가 발생했다면 onError signal 전달
- Maybe
  - 1개의 item을 전달 후 바로 onComplete signal 전달
  - 1개의 item이 없어도 onComplete signal 전달 가능
  - 에러가 발생했다면 onError signal 전달
  - Reactor의 Mono와 유사
- Completable
  - onComplete 혹은 onError signal만 전달
  - 값이 아닌 사건을 전달

| 타입         | 취소 지원 및 특징                | 발행 아이템 개수 | onSubscribe           | onNext                                   | onComplete                              | onError               |
|--------------|------------------------------|---------------|-----------------------|------------------------------------------|-----------------------------------------|-----------------------|
| **Flowable**     | BackPressure, cancelation 지원   | 0..n         | 언제든지 호출 가능        | 언제든지 호출 가능                         | 언제든지 호출 가능                        | 언제든지 호출 가능       |
| **Observable**   | cancelation 지원                | 0..n         | 언제든지 호출 가능        | 언제든지 호출 가능                         | 언제든지 호출 가능                        | 언제든지 호출 가능       |
| **Single**       | cancelation 지원                | 1            | 언제든지 호출 가능        | 1회 호출                                  | onNext 호출 후 바로 호출                   | 언제든지 호출 가능       |
| **Maybe**        | cancelation 지원                | 0..1         | 언제든지 호출 가능        | (아이템 존재 시 onNext 호출 후 바로)        | onNext 호출 후 바로, 혹은 단독 호출         | 언제든지 호출 가능       |
| **Completable**  | cancelation 지원                | 0            | 언제든지 호출 가능        | 해당 없음                                | 언제든지 호출 가능                        | 언제든지 호출 가능       |


### Mutiny
- Hibernate reactive에서 비동기 라이브러리로 제공
- Multi, Uni publisher 제공
- Multi
  - 0..n개의 item을 전달
  - 에러가 발생하면 error signal 전달하고 종료
  - 모든 item을 전달했다면 complete signal 전달하고 종료
  - backPressure 지원
  - Reactor의 flux와 유사
- Uni
  - 0..1개의 item을 전달
  - 에러가 발생하면 error signal 전달하고 종료
  - 모든 item을 전달했다면 complete signal 전달하고 종료
  - Reactor의 Mono와 유사
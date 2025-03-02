## CompletableFuture 실습
- 동기로 되어있는 코드를 CompletableFuture로 대체한다
- 동시성을 사용해서 효율적인 코드로 변경한다
   

### 동기식 코드
```java
// 각 Repository는 sleep 1초를 줌
@RequiredArgsConstructor
public class UserBlockingService {
    private final UserRepository userRepository;
    private final ArticleRepository articleRepository;
    private final ImageRepository imageRepository;
    private final FollowRepository followRepository;

    public Optional<User> getUserById(String id) {
        return userRepository.findById(id)
                .map(user -> {
                    var image = imageRepository.findById(user.getProfileImageId())
                            .map(imageEntity -> {
                                return new Image(imageEntity.getId(), imageEntity.getName(), imageEntity.getUrl());
                            });

                    var articles = articleRepository.findAllByUserId(user.getId())
                            .stream().map(articleEntity ->
                                    new Article(articleEntity.getId(), articleEntity.getTitle(), articleEntity.getContent()))
                            .collect(Collectors.toList());

                    var followCount = followRepository.countByUserId(user.getId());

                    return new User(
                            user.getId(),
                            user.getName(),
                            user.getAge(),
                            image,
                            articles,
                            followCount
                    );
                });
    }
}
```

### Completable Future(비동기) 사용
- 차이점
  - 호출방식
    - 동기식 
      - userRepository.findById(...) → 결과 반환 후
      - imageRepository.findById(...) → 결과 반환 후
      - articleRepository.findAllByUserId(...) 순서로 차례차례 호출
      - 메인 스레드는 대기(Blocking)하고, 모든 결과를 얻은 뒤 최종적으로 User 객체를 생성
    - 비동기 
      - userRepository.findById(...)가 CompletableFuture로 반환되고, 이후 thenComposeAsync(), thenApplyAsync() 등을 통해 연속적인 비동기 작업 체인을 구성
      - 호출 시점에 바로 결과를 기다리지 않고, 나중에 모든 Future가 완료되었을 때 최종 User 객체를 생성
  - 결과를 기다리는 방식
    - 동기 코드: 각 리포지토리 메서드를 호출하면 즉시 결과를 반환받을 때까지 기다린 후 다음 로직을 진행합니다.
    - 비동기 코드
      - 여러 작업(이미지 조회, 게시글 조회, 팔로우 수 조회 등)을 동시에 요청해놓고(CompletableFuture.allOf(...))
      - 나중에 모든 Future가 완료된 시점에 한꺼번에 결과를 취합합니다. 그 사이에 메인 스레드는 다른 일을 할 수 있습니다.
  - 실행흐름
    - 동기 코드: 한 메서드가 끝나야 다음 메서드가 실행되는 순차적 흐름입니다.
    - 비동기 코드: 여러 Future가 병렬적 혹은 비동기적으로 동작합니다
      - thenAcceptAsync, thenRunAsync, thenApplyAsync 등으로 후속 작업을 등록
      - 각 Future가 완료되는 시점에 콜백이 실행
- Repository 예제코드
```java
@Slf4j
public class ArticleFutureRepository {
    private static List<ArticleEntity> articleEntities;

    public ArticleFutureRepository() {
        articleEntities = List.of(
                new ArticleEntity("1", "소식1", "내용1", "1234"),
                new ArticleEntity("2", "소식2", "내용2", "1234"),
                new ArticleEntity("3", "소식3", "내용3", "10000")
        );
    }

    @SneakyThrows
    // CompletableFuture 반환
    public CompletableFuture<List<ArticleEntity>> findAllByUserId(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            log.info("ArticleRepository.findAllByUserId: {}", userId);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return articleEntities.stream()
                    .filter(articleEntity -> articleEntity.getUserId().equals(userId))
                    .collect(Collectors.toList());
        });
    }
}
```
- thenComposeAsync: 이전 CompletableFuture의 결과를 받아서 새로 만든 비동기 작업에 전달할 수 있습니다.
- thenApplyAsync: 이전 결과를 받아서 다른 값으로 변환(매핑)하는 용도로 사용됩니다.
- CompletableFuture.allOf(...): 여러 CompletableFuture들을 동시에 기다린 후, 모든 작업이 끝나면 후속 작업을 진행합니다.
```java
@Slf4j
@RequiredArgsConstructor
public class UserFutureService {
    private final UserFutureRepository userRepository;
    private final ArticleFutureRepository articleRepository;
    private final ImageFutureRepository imageRepository;
    private final FollowFutureRepository followRepository;

    @SneakyThrows
    public CompletableFuture<Optional<User>> getUserById(String id) {
      // userRepository.findById(id)는 CompletableFuture<Optional<UserEntity>>를 반환
      // thenComposeAsync()를 사용하면, 결과(Optional<UserEntity>)를 받아 다음 비동기 작업(getUser 메서드)으로 이어갈 수 있음
        return userRepository.findById(id)
                .thenComposeAsync(this::getUser);
    }

    @SneakyThrows
    private CompletableFuture<Optional<User>> getUser(Optional<UserEntity> userEntityOptional) {
        if (userEntityOptional.isEmpty()) {
            return CompletableFuture.completedFuture(Optional.empty());
        }

        var userEntity = userEntityOptional.get();

      // (1) 이미지 조회를 비동기로 수행
      // thenApplyAsync: 이전 단계(imageRepository.findById(...))의 결과를 받아서 새로운 결과로 변환
        var imageFuture = imageRepository.findById(userEntity.getProfileImageId())
                .thenApplyAsync(imageEntityOptional ->
                        imageEntityOptional.map(imageEntity ->
                            new Image(imageEntity.getId(), imageEntity.getName(), imageEntity.getUrl())
                        )
                );

      // (2) 게시글 조회를 비동기로 수행
      // thenApplyAsync: 게시글 엔티티 리스트를 받아서 Article 객체 리스트로 변환
        var articlesFuture = articleRepository.findAllByUserId(userEntity.getId())
                .thenApplyAsync(articleEntities ->
                        articleEntities.stream()
                                .map(articleEntity ->
                                    new Article(articleEntity.getId(), articleEntity.getTitle(), articleEntity.getContent())
                                )
                        .collect(Collectors.toList())
                );

      // (3) 팔로우 수 조회를 비동기로 수행
        var followCountFuture = followRepository.countByUserId(userEntity.getId());

      // CompletableFuture.allOf(...)를 통해 (1), (2), (3) 세 가지 비동기 작업이 모두 끝날 때까지 대기
        return CompletableFuture.allOf(imageFuture, articlesFuture, followCountFuture)
                // thenAcceptAsync: allOf의 결과를 받고, 특정 작업(로그 남기기)을 수행
                .thenAcceptAsync(v -> {
                    log.info("Three futures are completed");
                })
                // thenRunAsync: 이전 결과를 사용하지 않고 추가 작업(로그 남기기)을 수행
                .thenRunAsync(() -> {
                    log.info("Three futures are also completed");
                })
                // thenApplyAsync: 이전 단계의 결과(v는 Void)를 사용하지 않고, 최종적으로 User 객체를 생성하여 반환
                .thenApplyAsync(v -> {
                    try {
                        var image = imageFuture.get();
                        var articles = articlesFuture.get();
                        var followCount = followCountFuture.get();

                        return Optional.of(
                                new User(
                                        userEntity.getId(),
                                        userEntity.getName(),
                                        userEntity.getAge(),
                                        image,
                                        articles,
                                        followCount
                                )
                        );
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                });
    }
}
```
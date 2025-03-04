## Java NIO 실습

### JavaIOClient
```java
@Slf4j
public class JavaIOClient {
    @SneakyThrows
    public static void main(String[] args) {
        log.info("start main");
        try (Socket socket = new Socket()) {
            // 서버에 연결
            socket.connect(new InetSocketAddress("localhost", 8080));

            OutputStream out = socket.getOutputStream();
            String requestBody = "This is client";
            out.write(requestBody.getBytes());
            out.flush();

            // 서버에서 응답온 후
            InputStream in = socket.getInputStream();
            byte[] responseBytes = new byte[1024];
            in.read(responseBytes);
            log.info("result: {}", new String(responseBytes).trim());
        }
        log.info("end main");
    }
}
```

### JavaIOSingleServer
- 한번 송수신 하고 종료
```java
@Slf4j
public class JavaIOSingleServer {
    @SneakyThrows
    public static void main(String[] args) {
        log.info("start main");
        try (ServerSocket serverSocket = new ServerSocket()) {
            // 서버소켓 바인드
            serverSocket.bind(new InetSocketAddress("localhost", 8080));
            // 클라이언트 연결대기
            Socket clientSocket = serverSocket.accept();

            byte[] requestBytes = new byte[1024];
            // client 메시지 수신
            InputStream in = clientSocket.getInputStream();
            in.read(requestBytes);
            log.info("request: {}", new String(requestBytes).trim());

            // client에 메시지 전달
            OutputStream out = clientSocket.getOutputStream();
            String response = "This is server";
            out.write(response.getBytes());
            out.flush();
        }
        log.info("end main");
    }
}
```

### JavaIOServer 
- 계속 열어놓는 서버
```java
@Slf4j
public class JavaIOServer {
    @SneakyThrows
    public static void main(String[] args) {
        log.info("start main");
        try (ServerSocket serverSocket = new ServerSocket()) {
            // 서버소켓 바인드
            serverSocket.bind(new InetSocketAddress("localhost", 8080));

            // 서버 계속 오픈
            while (true) {
                // 클라이언트 소켓 연결 대기
                Socket clientSocket = serverSocket.accept();

                byte[] requestBytes = new byte[1024];
                // client 메시지 수신
                InputStream in = clientSocket.getInputStream();
                in.read(requestBytes);
                log.info("request: {}", new String(requestBytes).trim());

                // client에 메시지 전달
                OutputStream out = clientSocket.getOutputStream();
                String response = "This is server";
                out.write(response.getBytes());
                out.flush();
            }
        }
    }
}
```

### JavaNIOBlockingServer 
```java
@Slf4j
public class JavaNIOBlockingServer {
    @SneakyThrows
    public static void main(String[] args) {
        log.info("start main");
        // ServerSocketChannel 필요
        try (ServerSocketChannel serverSocket = ServerSocketChannel.open()) {
            serverSocket.bind(new InetSocketAddress("localhost", 8080));

            while (true) {
                // Client연결할때까지 Blocking 발생
                SocketChannel clientSocket = serverSocket.accept();
                // DirectByteBuffer 사용
                // Client 메시지 수신
                ByteBuffer requestByteBuffer = ByteBuffer.allocateDirect(1024);
                clientSocket.read(requestByteBuffer);
                requestByteBuffer.flip();
                String requestBody = StandardCharsets.UTF_8.decode(requestByteBuffer).toString();
                log.info("request: {}", requestBody);

                // client에게 메시지 송신
                ByteBuffer responeByteBuffer = ByteBuffer.wrap("This is server".getBytes());
                clientSocket.write(responeByteBuffer);
                clientSocket.close();
            }
        }
    }
}
```

### JavaNIONonBlockingServer 
- 하나의 연결이 길어질 경우 Blocking과 비슷하게 됨
- 모두를 Main Thread에서 처리하기 때문
```java
@Slf4j
public class JavaNIONonBlockingServer {
    @SneakyThrows
    public static void main(String[] args) {
        log.info("start main");
        try (ServerSocketChannel serverSocket = ServerSocketChannel.open()) {
            serverSocket.bind(new InetSocketAddress("localhost", 8080));
            // serverSocket 객체를 non-blocking 모드로 설정
            serverSocket.configureBlocking(false);

            while (true) {
                SocketChannel clientSocket = serverSocket.accept();
                // client 연결이 없는 경우 계속 sleep
                if (clientSocket == null) {
                    Thread.sleep(100);
                    continue;
                }
                // client 연결생성된 경우 handleRequest 호출
                handleRequest(clientSocket);
            }
        }
    }

    @SneakyThrows
    private static void handleRequest(SocketChannel clientSocket) {
        ByteBuffer requestByteBuffer = ByteBuffer.allocateDirect(1024);
        // client에서 수신한 데이터가 없는 경우 대기
        while (clientSocket.read(requestByteBuffer) == 0) {
            log.info("Reading...");
        }
        // client 메시지 수신
        requestByteBuffer.flip();
        String requestBody = StandardCharsets.UTF_8.decode(requestByteBuffer).toString();
        log.info("request: {}", requestBody);

        Thread.sleep(10);

        // client에 메시지 송신
        ByteBuffer responeByteBuffer = ByteBuffer.wrap("This is server".getBytes());
        clientSocket.write(responeByteBuffer);
        clientSocket.close();
    }
}
```

### JavaIOMultiClient
- ExecutorService를 통한 비동기 처리 : 50개의 스레드
- CompletableFuture를 이용한 비동기 작업 : 1000개의 비동기 작업을 생성
```java
public class JavaIOMultiClient {
  // 고정 스레드 풀을 생성, 최대 50개의 스레드 사용
  private static ExecutorService executorService = Executors.newFixedThreadPool(50);

  @SneakyThrows
  public static void main(String[] args) {
    log.info("start main");

    // CompletableFuture를 저장할 리스트 생성
    List<CompletableFuture<Void>> futures = new ArrayList<>();

    long start = System.currentTimeMillis();

    // 1000개의 클라이언트 요청을 비동기로 실행
    for (int i = 0; i < 1000; i++) {
      // CompletableFuture를 사용하여 비동기 작업 실행
      var future = CompletableFuture.runAsync(() -> {
        try (Socket socket = new Socket()) {
          socket.connect(new InetSocketAddress("localhost", 8080));

          // 서버로 메시지 전송
          OutputStream out = socket.getOutputStream();
          String requestBody = "This is client";
          out.write(requestBody.getBytes());
          out.flush();

          // 서버로부터 메시지 수신
          InputStream in = socket.getInputStream();
          byte[] responseBytes = new byte[1024];
          in.read(responseBytes);
          log.info("result: {}", new String(responseBytes).trim());
        } catch (Exception e) {
        }
      }, executorService);

      // 생성한 CompletableFuture를 리스트에 추가
      futures.add(future);
    }

    // 모든 비동기 작업이 완료될 때까지 대기
    // new CompletableFuture[0]와 같이 길이가 0인 배열을 전달하면, Java는 내부적으로 리스트 크기에 맞는 새로운 배열을 생성하여 반환
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    executorService.shutdown();
    log.info("end main");
    long end = System.currentTimeMillis();
    log.info("duration: {}", (end - start) / 1000.0);
  }
}
```

### JavaNIONonBlockingMultiServer 
- configureBlocking(false)를 통해 non-blocking 모드로 동작하도록 설정
- 클라이언트 연결이 있으면 CompletableFuture.runAsync()를 이용하여 별도의 비동기 작업으로 요청을 처리
  - 각 클라이언트 요청을 독립적인 스레드에서 처리함으로써 서버의 응답성을 향상
  - Executors를 명시적으로 지정해줘도 되고
  - 명시적으로 지정안하면 ForkJoin 쓰레드 풀 사용
- 요청 처리가 끝나면 소켓을 닫고, 처리된 요청 건수를 AtomicInteger로 기록
```java
@Slf4j
public class JavaNIONonBlockingMultiServer {
  // 클라이언트 요청 처리 건수를 안전하게 증가시키기 위한 AtomicInteger
  private static AtomicInteger count = new AtomicInteger(0);
  // 쓰레드 풀 생성
  private static ExecutorService executorService = Executors.newFixedThreadPool(50);

  @SneakyThrows
  public static void main(String[] args) {
    log.info("start main");

    try (ServerSocketChannel serverSocket = ServerSocketChannel.open()) {
      serverSocket.bind(new InetSocketAddress("localhost", 8080));
      // non-blocking 모드로 설정: accept() 호출 시 클라이언트 연결이 없으면 null 반환
      serverSocket.configureBlocking(false);

      // 무한 루프를 통해 클라이언트 연결 감지 및 처리
      while (true) {
        // 클라이언트 연결 시도 (non-blocking이므로 연결이 없으면 즉시 null 반환)
        SocketChannel clientSocket = serverSocket.accept();
        // 연결된 클라이언트가 없으면
        if (clientSocket == null) {
          Thread.sleep(100);
          continue;
        }

        // 클라이언트 연결이 있으면 비동기로 요청을 처리
        // Executor를 명시하지 않으면, 기본적으로 ForkJoinPool 스레드를 사용
        CompletableFuture.runAsync(() -> {
          handleRequest(clientSocket);
        }, executorService);
      }
    }
  }

  @SneakyThrows
  private static void handleRequest(SocketChannel clientSocket) {
    ByteBuffer requestByteBuffer = ByteBuffer.allocateDirect(1024);
    while (clientSocket.read(requestByteBuffer) == 0) {
      log.info("Reading...");
    }

    // 클라이언트 메시지 수신
    requestByteBuffer.flip();
    String requestBody = StandardCharsets.UTF_8.decode(requestByteBuffer).toString();
    log.info("request: {}", requestBody);

    // 클라이언트 요청 처리 중 추가 지연(예제에서는 10ms)
    Thread.sleep(10);

    // 클라이언트에게 응답 메시지 전송
    ByteBuffer responeByteBuffer = ByteBuffer.wrap("This is server".getBytes());
    clientSocket.write(responeByteBuffer);
    clientSocket.close();
    // 처리된 요청 건수를 증가시키고 로그에 출력
    int c = count.incrementAndGet();
    log.info("count: {}", c);
  }
}

```
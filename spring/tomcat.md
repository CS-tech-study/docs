## Tomcat
자바 프로그램으로, JVM에서 동작하는 하나의 프로세스
- WAS를 보면 크게 Servlet Container와 Spring Container로 나눠지는데, Servlet Container가 Tomcat에 해당
- 소켓 프로그래밍을 해보면…
  - 서버 측에서는 listen 메서드로 요청을 기다리고
  - accept 메서드로  요청을 수락해 데이터 패킷 획득
- Spring Container가 처리할 수 있도록 HttpServletRequest, HttpServletResponse 생성
- 요청을 처리할 수 있는 Servlet을 찾아 **Servlet의 service 메서드**를 호출하면 비즈니스 로직 수행 시작하는데 이런 공통적인 부분을 Tomcat이 처리

<br>

## Tomcat 아키텍처
![](/spring/img/tomcat-architecture.png)
출처: https://velog.io/@hyunjae-lee/Tomcat-2-%EA%B5%AC%EC%A1%B0

- 하나의 톰캣 서버에는 여러 service 존재
- 각 서비스에는 1개의 엔진과 여러 connector로 구성
- 1개의 엔진에는 여러 host가 존재 가능 (각 Host는 가상 호스트를 나타내며, URL에 매핑)
- 하나의 Host에는 여러 Context 존재
<br>

Connector는 TCP port에서 요청을 listen 하면서 특정 engine으로 보내주는 역할
- Connector 종류: HTTP/1.1, HTTP/2
- Connector 동작 방식: BIO, NIO, NIO2

<br>

## BIO
- blocking
- 하나의 스레드와 하나의 커넥션이 매핑
- maxConnections = maxThreads = 200
  - maxConnections: 동시 처리 가능한 최대 커넥션 수
  - maxThreads: 요청을 처리하기 위한 최대 스레드 수
- *유휴 스레드 없으면 output queue에서 대기…?*

<br>

## NIO
- multiplexing
- 하나의 스레드가 여러 connection 담당
- 커넥션에 대한 데이터 오면 worker thread 할당해주는 방식
- maxConnections = 10,000, maxThreads = 200
  - maxConnections: 동시 처리 가능한 최대 커넥션 수
  - maxThreads: 요청을 처리하기 위한 최대 스레드 수
- *최대 커넥션 수를 넘기면 output queue에서 대기…?*
- 채널을 통해 데이터 들어와도 내부에서 워커 스레드 없으면 block…?

### Acceptor
- 싱글 스레드
- while(!stopCalled) 문을 돌며 소켓을 listen
- 소켓 accept 하면 NioSocketWrapper 객체로 캡슐화
- NioSocketWrapper 객체를 PollerEvent로 캡슐화

### poller
- PollerEventQueue(SynchronizedQueue 클래스의 Object[] 배열)에 PollerEvent 객체 push
- 이후 데이터 요청 오면 Object 배열에서 해당 PollerEvent 객체 null 처리

### worker
- PollerEvent 객체를 SynchronizedQueue에 추가한 후 데이터 오면 worker thread 할당
- 워커 스레드를 스레드 풀로 관리 (Dedicated Thread pool)
  - Dedicated Thread pool: Connector 마다 개별적인 스레드 풀
  - Shared Thread pool: 엔진에 할당된 여러 Connector가 공유

```yaml
server:
  tomcat:
    threads:
      max: 2 # 생성할 수 있는 thread의 총 개수
      min-spare: 2 # 항상 활성화 되어있는(idle) thread의 개수
    max-connections: 8192 # 수립가능한 connection의 총 개수
    accept-count: 1 # 작업큐의 사이즈
```

위와 같이 작성하면 2개만 동작하고 1개의 요청만 대기할 수 있다고 기대한다.
- 즉, 5개의 작업이 들어오면 2개만 처리되고
- 1개만 대기하고 남은 2개는 버려진다고 기대되지만 그렇지 않다.

3초동안 대기하는 메서드에 5번 요청을 보내도
- 5개의 요청이 2개씩 순차적으로 잘 처리되는데 이는 NIO connector 때문
- 2개의 요청이 처리되고 1개의 요청은 대기하고 나머지 요청은 버려지는 상상을 했다면 이는 BIO connector 방식
- BIO connector는 connection이 닫힐 때까지 하나의 thread는 특정 connection에 계속 할당되어있기 때문
- tomcat 9부터는 NIO connector 을 사용하므로 모든 요청을 처리할 수 있다.

<br>

## Engine
Engine은 정의된 connector로 들어온 요청을 하위 host에게 전달해주는 역할
- 1개의 엔진에는 여러 Host가 존재 가능
- 여기서 말하는 여러 Host는 완벽히 독립적인 애플리케이션
- 여러 Host가 하나의 Tomcat을 공유하니 전체적인 리소스를 최소한으로 사용 가능
- but, tomcat 200개 스레드를 여러 서비스가 함께 사용하니 NIO로 동작한다 해도 요청량엔 이점이 아닐 것이라 생각

<br>

## Tomcat NIO vs Netty

Tomcat NIO: Async + Blocking
- 채널 등록하고, 워커 스레드 할당받으면 비즈니스 로직 시작
- 비즈니스 로직에서 DB I/O 만나면 block
- Poller가 여러 소켓 채널 관리
  - Poller 기반의 멀티스레드 구조 (연결된 여러 요청에 대해 워커 스레드가 할당되어 처리되므로)

Netty: Async + Non-blocking
- JS의 Microtask Queue, Macrotask Queue 개념과 유사한 비동기 처리 제공
- 이벤트 루프와 리스너 패턴으로 I/O 처리
  - 이벤트 루프: 단일 스레드가 여러 I/O 작업 처리
- Mono, Flux 처리를 위한 Async? non-blocking
  - Mono, Flux로 응답 결과 비동기로 받음 (Future, CompletableFuture 와 유사)

<br>

## Timeout

handshaking이 발생했다면
- TCP 연결 성공 (이미 서로 간의 연결 상태 확인)
- 읽기, 쓰기 버퍼가 생성되어 서로 간의 데이터 전송 준비가 완료된 상태
- 응답 올 수 있음
- 타임아웃으로 간주하면 4-way handshake
  - 연결을 종료한다는 의사 전달
  - Active close 소켓 기준으로 마지막 time_wait 상태가 끝나야만 읽기 및 쓰기 버퍼 close 할 것이라 예상
  - Active close 상태 과정: established -> fin_wait_1 & 2 -> time_wait -> close
  - Passive close 상태 과정: established -> close_wait -> last_ack -> close
- half close로 쓰기 버퍼만 닫고, 읽기 버퍼는 그대로 열어 상대 데이터 받을 수 있을 것 같은데.. 버릴 것 같다고 예상

handshaking이 발생하지 않았다면
- TCP 연결 자체가 안된 것
- 읽기, 쓰기 버퍼가 생성되지 않은 것
- 그러므로 응답을 받을 수 없음

<br>

## OSI 7

TCP 소켓: transport(4) layer에서 동작
- L3: IP 주소로 목적지 네트워크 도달
- L4: TCP(전송 보장 O) / UDP (전송 보장 X)

HTTP 프로토콜: application(7) layer에서 동작
- FTP, SMTP, HTTP….
- 즉 HTTP 는 TCP 소켓을 통해 동작 가능

Servlet Container: L4
- TCP 소켓과 관련된 부분은 Tomcat(Servlet Container가 처리)
- 즉 Tomcat이 TCP 소켓을 통해 HTTP 통신 가능하게 함

Spring Container: L7
- Spring Container가 요청을 처리할 수 있게 HttpServletRequest, Response로 변경


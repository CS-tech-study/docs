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

> 모바일 환경에서 https://www.naver.com 을 입력하면 자동으로 https://m.naver.com 로 리다이렉션<br>
> naver.com 을 입력하면 모바일 환경은 https://m.naver.com 으로, 웹은 https://www.naver.com 으로 리다이렉션

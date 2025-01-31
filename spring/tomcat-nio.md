
## Tomcat 역할

Tomcat은 Apache 재단에서 관리되며, Java 표준 인터페이스인 servlet을 지원하기 위한 미들웨어이다.
WAS룰 Servlet Container와 Spring Container로 나눌 수 있는데, Servlet Container가 Tomcat 이다.
다음과 같은 공통부분을 Tomcat이 처리한다.
- 소켓 처리: listen 메서드로 요청 기다리고, accept 메서드로 요청 수락해 데이터 패킷 획득
- 변환: Spring Container가 처리할 수 있도록 HttpServletRequest, HttpServletResponse 생성
- 비즈니스 로직 수행: 요청을 처리할 수 있는 servlet을 찾아 servlet의 service 메서드를 호출하면 비즈니스 로직을 수행

<br>

## Tomcat 요청 처리

첫 요청에 별도의 설정이 없다면, 기본적으로 CPU 코어 수만큼의 스레드 생성해 스레드 풀에 넣어둔다.

1. 커넥션 객체를 작업 큐에 push
2. 스레드 풀에서 사용할 수 있는 스레드를 가져와 작업 큐에 넣어둔 요청 처리
3. 요청 처리 후 작업 큐가 비었다면 스레드는 다시 스레드 풀로 반납 (이떄 초기 설정보다 많은 수의 스레드가 스레드 풀에 존재하면 제거)

<br>

## Connector

Tomcat으로 요청이 들어오는 첫 번째 문이 Connector
- Connector는 TCP 포트의 요청을 listen하면서 애플리케이션으로 들어올 수 있게 함 (BIO, NIO 방식)
- Connector는 소켓 연결을 listen하면서 들어오는 패킷을 HttpServletRequest, Response 객체로 변환해 servlet container로 요청을 위임하는 역할 담당 (이후 핸들러 매핑 전략으로 알맞은 핸들러를 찾아 service를 호출하는 것은 Dispatcher Servlet 담당)
- HTTP를 비롯한 다양한 애플리케이션 레이어의 프로토콜을 처리할 수 있도록 개발되었으며, 각 프로토콜에 해당하는 connector 존재 (HTTP/1.1 Connector, HTTP/2 Connector)

<br>

## BIO 
- blocking
- 스레드와 커넥션이 일대일 관계 (thread per connection)
- max-connections = max-threads = 200

유휴 스레드 부족 상황
- blocking 방식으로 동작하기 때문에 스레드 낭비 발생
- 유휴 스레드 부족 상황은 스레드 풀 조절로 단순히 해결할 수 있는 문제가 아님
- 실행 중인 스레드가 많을수록 컨텍스트 스위칭 오버헤드 때문에 전체 처리 성능 감소 및 GC 비용 증가

### thread per connection
- 하나의 TCP 연결에 대해 하나의 스레드 할당
- 응답 후에도 스레드가 스레드 풀로 반납되는 것이 아니라, 연결이 유지되는 동안 스레드 대기

### thread per request
- HTTP 요청마다 스레드 할당
- 응답 후 스레드 풀로 스레드 반납
- Spring MVC에서 사용되는 방식

<br>

## NIO (New I/O)

- multiplexing
- 완전한 non-blocking 아님
- poller와 커넥션(채널)이 일대다 관계
  - poller는 max connections 까지 연결을 수락하고, 작업 큐 사이즈와 관계 없이 캐싱 기능을 활용하므로 하나의 스레드에서 여러 커넥션 유지 가능
- 커녁션에 대한 데이터 요청 처리 발생하면 worker 스레드 할당 (multiplexing)
- 커넥션과 worker 스레드는 일대일 관계 (blocking)
- workder 스레드는 스레드 풀에서 관리

### NIO 동작 방식
selector
- 싱글 스레드로 동작
- 여러 채널을 모니터링하고, 데이터가 준비된 채널을 감지해 이벤트 기반으로 처리 (multiplexing)

Channel
- 소켓이 오면 poller에 채널 생성
- poller에 여러 채널 생성 가능
- 클라이언트는 채널을 통해 데이터 통신 수행하므로 [ByteBuffer 필요](https://github.com/CS-tech-study/docs/blob/sangho/java/tomcat_netty.md#bytebuffer)
- 채널에 대한 데이터 처리 요청이 오면 워커 스레드 할당
- 응답 후 워커 스레드 반납 (thread per request)
- 하지만 데이터 처리 중 blocking I/O가 발생하면 워커 스레드가 block 되므로 NIO를 완전한 non-blocking이라 할 수 없으며, 이를 해결하기 위해 Netty 사용

### BIO와의 차이
- thread per connection으로 응답 후에도 스레드를 반납하지 못하는 BIO 와 다르게
- 응답 후 워커 스레드를 스레드 풀로 반납해 여러 채널이 효율적으로 사용하게 되어, 적은 스레드로 BIO 방식보다 많은 요청 처리 가능
- Tomcat 9 부터 NIO 방식이 default 됨

<br>

## Tomcat NIO vs Netty

Tomcat NIO
- poller가 여러 소켓(커넥션) 관리 (poller 기반 멀티스레드 구조)
- 채널 등록하고, worker 스레드 할당받으면 비즈니스 로직 시작
- 비즈니스 로직에서 blocking 요소 만나면 Tomcat worker 스레드 block

Netty
- 이벤트 루프와 리스너 패턴으로 I/O 처리

<br>

## Tomcat 설계
Spring Boot로 개발하면 임베디드된 Tomcat 사용
- 즉, 일반적으로 톰캣과 애플리케이션 서버가 일대일 관계
- 톰캣과 애플리케이션 서버가 일대다 관계도 가능
- 일대다 관계는 독립적인 여러 애플리케이션이 하나의 Tomcat을 공유하는, 한정적인 worker 스레드를 여러 애플리케이션 서버가 함께 사용하니 NIO로 동작한다 해도 그다지 이점이 아닐 것이라 예상
- 실제로 일대다 관계로 사용하는 경우가 많지 않음

<br>

## 요청 Timeout 처리

B 서버로 요청을 보낸 후 A 서버에서 timeout 처리했을 경우 해당 패킷은 어떻게 되는가

A 서버에서 Timeout 처리했다면...
- 3-way handshaking 이후라면 FIN 패킷 보내 4-way handshaking 작업 시작 (요청 종료 선언)
  - half close로 데이터 받을 수 있지만, 버릴 것으로 예상
- 3-way handshaking 전이라면 FIN 패킷도 보낼 필요 없음 (or RST 패킷으로 응답)

B 서버에서 해당 패킷을 받았지만...
- A 서버에서 timeout 처리로 RST 패킷을 응답받았다면 해당 패킷은 처리하지 않고 버리기
- RST 패킷: TCP 연결을 즉시 종료하거나 재설정하는데 사용, 4-way handshaking 과정 없이 연결 즉시 종료

정리
- 3-way handshaking 이후 timeout: FIN 패킷으로 연결 종료, 읽기 버퍼로 데이터 받아도 사용하지 않음
- 3-way handshaking 이전 timeout: 패킷 버리기
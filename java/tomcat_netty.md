## Tomcat의 구조
---

<img src="https://linux.systemv.pe.kr/wp-content/uploads/2014/12/TomcatArchitecture.png">

>출처: https://wisdom-and-record.tistory.com/140
- Server: tomcat의 최상위 인터페이스로 전체 컨테이너를 표현한다.
- Service: Server 안에 존재하며 Connector와 Engine을 연결해준다.
- Engine: 실제 요청을 처리하는 역할을 담당한다. Connector를 통해 요청을 받아 이를 처리한 후 응답을 보낸다.
- Host: 네트워크 이름을 나타낸다.
- Connector: 클라이언트와의 커뮤니케이션을 담당한다(이름 그대로 Connection을 처리한다).
- Context: Web application을 표현한다.

## 톰캣 connector의 역할
---

톰캣 OS가 제공한 소켓 인터페이스를 활용하여 **HTTP 요청을 이해하고 처리**하는 데 필요한 작업을 수행

1. **소켓 연결 수락 및 관리**
- OS 레벨에서 전달된 소켓 연결을 수락
-  BIO 또는 NIO와 같은 커넥터 구현 방식에 따라 소켓 처리 방식이 다름

2. **HTTP 요청 파싱**
- 패킷 데이터를 바탕으로 **HttpServletRequest 객체를 생성**
>패킷 내용을 HttpServletRequest 객체로 만드는 이유는 HTTP 요청 정보를 자바 기반의 서블릿이 처리할 수 있도록 하기 위해

3. **서블릿 컨테이너의 dispatcherServlet로 위임**
- dispatcherServlet은 요청을 처리할 수 있는 핸들러(컨트롤러)를 찾아 요청 처리를 위임

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbZwzYh%2FbtrAIOjh0Rt%2FWFnoDMZXdOIa0rlgFmbzjk%2Fimg.png">

>즉, 소켓 연결 요청을 수락(TCP 커넥션 생성)하고 서블릿 컨테이너의 dispatcherServlet에 위임하기 위해 HttpServletRequest 객체를 생성하는거 까지가 톰캣 Connector의 역할


## 네트워크 I/O
---
**프로토콜, ip주소, 포트번호로 이루어진 "소켓"이란 파일에 IO를 하는 것을 의미**

## 소켓(Socket)
---
- 클라이언트와 서버 컴퓨터의 각 프로세스가 통신으로 데이터을 주고받기 위해 사용하는 커넥션 엔드포인트
- 소켓도 리눅스 프로그램 입장에선 **결국 하나의 파일**, 따라서 **파일 디스크립터를 사용하여 접근**
> 파일 디스크립터: 프로세스가 파일에 접근할 때 사용하는 정수값, 프로세스마다 파일 디스크립터 테이블이 존재
> 
> 리눅스의 `socket()` 함수는 소켓(파일)을 생성하고 이의 파일 디스크립터를 반환

<img width="329" alt="socket" src="https://github.com/user-attachments/assets/91f3be25-c6b7-43d0-a12e-2485734c0c5c" />

 생성한 소켓 디스크립터를 반환
 
리눅스 소켓 인터페이스에서 TCP 커넥션 생성 과정

<img width="433" alt="listenfd_connfd" src="https://github.com/user-attachments/assets/f8434386-9652-49fb-a0fa-4e46117d8e2b" />

**clientfd와 connfd의 연결이 TCP 커넥션**

## BIO Connector (스레드 풀 사용)
---

```java
public class IoThreadPoolBlockingServerApplication {

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        while (true) {
            // blocking call (새로운 클라이언트가 접속할 때까지 blocking 된다.)
            Socket socket = serverSocket.accept();

            // 스레드 풀 개수인 3개이상 소켓 연결이 안된다.
            threadPool.submit(() -> handleRequest(socket));
        }
    }

    private static void handleRequest(Socket socket) {
        try (InputStream in = socket.getInputStream();
             OutputStream out = socket.getOutputStream()) {
            int data;

            // read() 메서드는 데이터를 읽을 때까지 blocking 된다. 만약 소켓이 닫혀 더이상 읽을 게 없다면 -1을 리턴한다.
            while ((data = in.read()) != -1) {
                data = Character.isLetter(data) ? toUpperCase(data) : data;
                out.write(data);
            }
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private static int toUpperCase(int data) {
        return Character.toUpperCase(data);
    }
}
```

TCP 커넥션이 생성되면 이를 위한 하나의 스레드를 즉시 할당해주는 방식
즉, **커넥션과 스레드가 1 대 1로 대응**
>소켓(connfd)의 input stream에 데이터가 들어오는 것과 상관없이 **커넥션이 생성되면 일단 스레드를 할당**, 클라이언트가 커넥션을 닫아야(clientfd를 닫아야) 서버의 input stream에서 -1(EOF)을 받아 스레드가 종료되어 스레드 풀에 반환될 수 있음

### 단점
---
- 블로킹 방식으로 동작하기 때문에 스레드의 낭비가 발생
>블로킹 I/O 방식에서는 스레드가 데이터 입출력을 기다리는 동안 **블로킹 상태**로 대기
>여기서 말하는 스레드의 낭비는 **블로킹 상태의 스레드들이 CPU 자원을 사용하는건 아니지만, 엄연히 스레드 풀의 스레드를 차지하기 때문에 유휴 스레드의 부족을 야기한다는 의미**
>(논블로킹 I/O 방식은 스레드가 입출력 대기 시간 동안에도 다른 작업을 수행할 수 있음)

이는 단순히 스레드 풀의 사이즈를 늘린다고 해결되는 문제가 아님. 실행중인 스레드들이 많으면 많을 수록 컨텍스트 스위칭 오버헤드때문에 전체 처리 성능은 감소, GC 비용도 더 커진다고 함
(이러한 문제 때문에 톰캣의 기본 maxThreads 값이 200)

## 블로킹 I/O vs 논블로킹 I/O
---
kernel -> socket으로 생각
<img src="https://velog.velcdn.com/images/tae6919/post/36750390-fd67-4ea7-a1bd-b1fc91517ca1/image.png">
<img src="https://velog.velcdn.com/images/tae6919/post/edd96872-e29c-4270-a962-2f4784ef0b81/image.png">

>출처: https://velog.io/@tae6919/block-IO-vs-non-block-IO

간단하게 설명하면 블로킹 I/O 방식은 read 시스템 콜 호출 시 소켓에 데이터가 들어올 때 까지 스레드를 블로킹 상태로 두는거고, 논블로킹 I/O 방식은 read 시스템 콜 호출 시 소켓에 데이터가 없어도 그냥 없다고 알려주고 스레드를 블로킹하지 않음

여기서 들 수 있는 의문점이 스레드는 결국 소켓으로부터 데이터를 읽어와서 HttpServletRequest 객체로 변환하기 위해 read 시스템 콜을 호출한건데, 소켓에 데이터가 없다는 결과만 받으면 무슨 의미가 있는가이다. 

I/O 작업이 완료된 상태에서 read 시스템 콜을 호출해야 데이터를 읽어올 수 있는데, 이때 I/O 작업의 완료 여부를 확인하기 위한 방법으로 2가지가 있다.

1. I/O 작업의 완료 여부를 모르니까 그냥 반복적으로 시스템 콜을 호출해서 완료되었는지 확인
> **폴링 방식**,딱봐도 비효율적

2. I/O 작업이 완료되면 이를 이벤트로 발행하도록 하고, 이 이벤트는 select() 같은 시스템 콜을 통해 감지하도록 함
>**멀티플렉싱 방식**

<img width="555" alt="스크린샷 2025-01-16 오전 1 31 50" src="https://github.com/user-attachments/assets/7d0553e4-b37e-406a-b17f-388a8a8a1249" />


>출처: https://jiwondev.tistory.com/118

이러한 멀티플렉싱 방식은 우리가 지금 확인하고 있는 **소켓 프로그래밍**에서 특히 유용하다.
하나의 서버가 여러 클라이언트의 요청(소켓 I/O)를 처리해야 하는데,
앞서 확인한 BIO 커넥터 방식의 단점(스레드의 낭비)때문에 논블로킹 IO 방식을 선택한 상황이다.
이때 1번 폴링 방식을 사용하면 특정 단일 스레드(메인 스레드)가 서버의 모든 소켓을 순회하며 read 시스템 콜을 호출하여 읽어드릴 내용이 있는지 확인해야 한다. 등록된 소켓이 100개가 있다면 매번 100번의 read 시스템 콜을 호출해야 하는 것이다. 이걸 반복해야 하니...논블로킹의 의미가 없다.

반면 2번 멀티플렉싱 방식을 사용한다면 하나의 스레드에서 **select() 시스템 콜 호출 한번으로 등록된 모든 소켓에 읽어드릴 내용이 있는지 확인할 수 있으므로 폴링 방식보다 훨씬 효율적이다.**
>**논블로킹 I/O 방식을 도입했기 때문에 메인 스레드가 select()를 통해 여러 소켓의 I/O 이벤트 수신이 가능한거다.** 논블로킹이 아니었으면 I/O 작업을 블로킹 상태로 기다리느라 select()가 실행 중이지 않으므로 이벤트 수신도 불가능하다.

 번외: **non blocking이 async와 같은 개념인가? 엄밀히 말하면 NO**

async는 **작업의 순서가 상관없기 때문에 다른 스레드의 작업 결과를 아예 신경쓰지 않는것**이고, non blocking은 단순히 다른 스레드의 **작업 결과와 상관없이 일단 리턴하여 스레드가 블로킹 상태에 들어가도록 하지 않는 것**이다.

>async 방식이 다른 스레드의 작업 결과를 신경쓰지 않는다는 의미는 다른 스레드의 결과값을 받아도 이를 반드시 즉시 처리하는게 아니라 **언젠가** 처리한다. 또는 아예 결과값을 반환받지 않고 대신 **콜백**을 넘겨줘서 **결과 처리까지 다른 스레드에게 위임**한다.

non blocking과 async는 자주 같이 사용되는(**특히 async 방식일 경우 non blocking을 하지 않을 이유가 없으므로**) 개념이어서 둘을 구분하는 데 있어 약간의 모호함이 있지만 개념 자체는 일단 다르다.

## NIO Connector

### 핵심 개념
---
- 채널
bio 커넥터는 소켓을 stream 방식으로 I/O 처리, nio 커넥터는 소켓을 channel 방식으로 I/O 처리

|        | Stream            | Channel |
| ------ | ----------------- | ------- |
| 통신 방향  | 단방향               | 양방향     |
| 개수     | 2개 필요(입력, 출력 한개씩) | 1개 필요   |
| 버퍼 사용  | X                 | O       |
| I/O 방식 | 블로킹               | 논블로킹    |

Stream은 버퍼가 따로 없기 때문에 블로킹 I/O 방식으로 동작할 수 밖에 없다.
>버퍼가 중간에 있으면 스레드의 상태는 신경 쓸 필요 없이 버퍼에만 계속 I/O 작업을 수행하면 됨 (느슨한 결합)

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fd2OFcI%2FbtrA0YrQGHb%2FhAkyrlNOXO0f1iguUKlS0k%2Fimg.png">

>출처: https://jh-labs.tistory.com/353


- selector

Poller라는 싱글 스레드가 사용하는 멀티플렉서, 이를 통해 각 채널에서 **I/O 이벤트**가 발생하면 이를 감지하고, **worker thread pool**에서 유휴 스레드를 가져와 해당 요청을 처리하도록 위임

### BIO Connector보다 요청을 더 많이 처리할 수 있는 이유
---

- BIO 커넥터가 일단 커넥션이 생성되면 스레드를 할당해서 언제 소켓에 데이터가 들어올 지도 모른 채 read 시스템 콜로 인해 스레드가 블로킹 상태로 대기하던 것과 달리, **실제로 소켓에 데이터가 들어오면 그때 worker 스레드를 할당**

- BIO 커넥터의 경우 클라이언트 측에서 커넥션을 닫기 전까진 스레드가 반환될 수 없던 것과 달리, **NIO 커넥터의 경우 응답을 반환하면 즉시 worker 스레드 풀로 반환**
>이는 해당 커넥션으로 요청이 또 들어와도(소켓에 I/O 이벤트가 또 발생해도) selector에서 이를 감지해서 다시 worker 스레드를 할당해주면 그만이기 때문

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FAhhDH%2FbtrB1nknTIl%2FG5gdNqNUWg564bfD8pPAG1%2Fimg.jpg">

**Acceptor**는 클라이언트의 연결 요청을 수락하여 SocketChannel 객체를 생성하고, 이를 **NioChannel**로 래핑한 후, **PollerEvent**로 캡슐화하여 **이벤트 큐**에 추가

Poller**는 **Selector**를 사용하여 이벤트 큐에 등록된 채널들을 감시하며, 각 채널에서 **I/O 이벤트**가 발생하면 이를 감지. 이벤트가 발생하면, **워커 스레드 풀**에서 유휴 스레드를 가져와 해당 요청을 처리하도록 위임**

그러나 이 방식도 결국 worker 스레드는 블로킹 방식으로 동작하기 때문에 bio 커넥터에 비해 적은 스레드로 요청을 처리할 수 있는거지 결국 한계가 있음. 그래서 나온게 netty

### `ByteBuffer`
---
JVM은 자바 애플리케이션을 실행하기 위해 운영 체제로부터 할당받은 물리 메모리 상에 자신만의 가상 메모리 영역을 구성

반면 소켓에서 stream이 들어오면 OS 커널은 이 데이터를 커널 버퍼에 write하는데, java 코드 상에서 이 커널 버퍼에 직접 접근할 수 있는 방법이 없었기에, 힙 메모리와 네이티브 메모리 간의 데이터 복사가 필요했음

문제점
- kernel 버퍼에서 JVM 내부 버퍼로 복사할 때 CPU 소비.
- 복사한 JVM 버퍼내 데이터 사용후 GC가 수거해야함으로써 CPU 소비.
- 복사가 진행되는 동안 I/O 요청한 JVM Thread는 Blocking된 상태.

이러한 문제점으로 인해 JDK 1.4부터 jvm 내부 버퍼에 데이터를 복사하지 않고 커널 버퍼에 바로 접근할 수 있도록 `ByteBuffer` 클래스를 지원했고, **NIO 방식이 이 `ByteBuffer` 클래스를 사용해서 소켓 I/O를 수행**(Direct Buffer)

## 톰캣 스레드 설정
---

<img src="https://hudi.blog/static/1c44d267d9665e1c335e6bc48844c2db/ac7a9/tomcat.png">

>출처: https://hudi.blog/tomcat-tuning-exercise/
- maxConnections: 커넥션 수 (연결된 리슨 소켓 수)
- maxThreads: 워커 스레드 수 
- acceptCount: maxConnections가 다 찼을 때 대기열
acceptCount 마저 초과하면 서버에서 요청을 거부하거나 클라이언트 측에서 타임아웃 발생으로 결국 요청 실패

## Netty
---
### EventLoop
---

<img src="https://mark-kim.blog/static/a97d7fd85aa6e08b6a68fc5fe37a8206/f4853/netty_eventloop.webp">

>출처: https://mark-kim.blog/netty_deepdive_1/

Tomcat은 Poller 스레드가 Selector를 통해 이벤트를 감지하고, Worker Thread에 작업을 위임하는 구조

네티도 마찬가지로 eventLoop 스레드가 selector를 통해 I/O 이벤트를 수신하고 해당 이벤트의 처리(버퍼로부터 패킷 읽어와서 request 객체 변환)를 channelPipeline로 전달

### ChannelPipeline
---
<img src="https://mark-kim.blog/static/abecb1308e0d961f815ae56431885141/64296/channelhandler_inbound_outbound.webp">

ChannelPipeline은 Inbound 이벤트(input)와 Outbound 이벤트(output)들을 처리하거나 가로채서 무언가를 실행할 수 있는 ChannelHandler의 연결
>intercepting filter pattern (spring security의 filterChain과 같은 원리)

`Socket.read()` 를 통해 inbound 이벤트가 발생하면 일련의 inbound handler 필터 체인이 적용됨, `Socket.write()`로 outbound 이벤트가 발생할 때도 마찬가지로 outbound handler 필터 체인이 적용

이때 특정 channelHandler의 로직에 블로킹이 들어가면 eventLoop 스레드가 블로킹되므로 무조건 논블로킹 방식으로 처리해야 한다.
>필요할 경우 특정 비즈니스 로직 작업을 별도의 스레드에게 위임하고 비동기 방식으로 결과를 처리해야 한다. 이때 사용되는게 task queue로 일단 별도의 스레드에게 위임해야 할 작업을 task queue에 넣어 놓는것 같다..

*ChannelPipeline은 메시지가 각각의 Handler를 거칠 때 마다 Handler에게 바인딩된 EventExecutor 쓰레드와 현재 메시지 전송을 요청하고 있는 쓰레드가 동일한지 체크한다. 만약 서로 다른 쓰레드라면(Handler의 EventExecutor가 아니라면) 메시지를 Task Queue에 삽입하고 그대로 실행을 반환한다. Queue에 쌓인 메시지는 이후에 EventExecutor에 의해 비동기적으로 처리되게 된다.*
>출처: https://velog.io/@joosing/netty-tx-message-flow

### EventLoopGroup
---

<img src="https://mark-kim.blog/static/71103248690914ca41be70a9110555ac/dd4c6/boss_child_event_group.webp">
Boss Thread Group, Worker Thread Group 2개의 eventLoop 그룹으로 구성

- boss thread group: 리슨 소켓을 통해 연결 요청을 수락하고, 커넥션이 수립되면 worker 이벤트 그룹으로 전달
- worker thread group: channelPipeline을 사용하여 I/O 이벤트를 처리(버퍼 read/write)

 새로운 연결(connect)보단 채널에 대한 read/write에 대한 이벤트가 훨씬 많이 발생하기 때문에 Worker 그룹의 EventLoop 수를 더 많이 설정한다고 한다.

### 그래서 톰캣과의 차이점이?
---

네티도 톰캣과 마찬가지로 자바 NIO 방식을 사용, 따라서 둘 다 채널 기반의 논블로킹 I/O 방식을 사용한다는 설명은 맞음
그러면 네티가 톰캣과 차이점이 뭐냐? 이게 굉장히 헷갈렸다.

결론부터 말하면 톰캣은 소켓 I/O는 논블로킹 방식으로 동작하지만 
>그래서 단일 poller 스레드가 selector로 여러 소켓 I/O 이벤트 수신을 할 수 있었던 것

소켓 I/O로 패킷 데이터(버퍼에 저장된)를 처리하는 워커 스레드는 블로킹 방식으로 작동함. 따라서 **워커 스레드는 결국 thread-per-request 구조가 될 수 밖에 없음**

반면, 네티는 channelPipeline을 도입하여 패킷 데이터가 이 필터 체인을 지나는동안 논블로킹 I/O 방식으로 처리, 비동기 방식으로 작업을 위임해야 할 경우 task queue에 일단 넣는다
> 결국 이 task queue에 등록된 작업을 언젠가 eventLoop 스레드가 처리..?

--- 
## 참고
https://mark-kim.blog/understanding-non-blocking-io-and-nio/

https://devoong2.tistory.com/entry/Tomcat-Tomcat-Thread-Pool-%EC%84%A4%EC%A0%95-%EC%A0%95%EB%A6%AC-%EB%B0%8F-%ED%85%8C%EC%8A%A4%ED%8A%B8#NIO-Non%--Blocking%--I%-FO-%--Connector

https://velog.io/@joosing/netty-tx-message-flow

https://mark-kim.blog/netty_deepdive_1/

https://sungjk.github.io/2023/08/25/netty-channel.html

https://colevelup.tistory.com/39

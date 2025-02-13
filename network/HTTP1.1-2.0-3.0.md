## HTTP 1.1 / 2.0 / 3.0

HTTP 1.0 -> HTTP 1.1의 가장 큰 변화
- persistant connection: 요청을 보내고 응답을 받으면 TCP connection을 끊는게 아니라 특정 시간동안 유지, 해당 세션으로 계속 요청과 응답을 주고받을 수 있음
>매번 새로운 요청마다 3 way handshake 과정을 수행하지 않아도 됨

번외: TCP는 UDP와 달리 3 way handshake가 필요한 이유

<img width="284" alt="3wayhandshake" src="https://github.com/user-attachments/assets/dc229709-c7dd-4a5d-8547-a6202d48b9c9" />

일단 양측이 서로 데이터를 주고받을 준비가 되었다는 걸 알리는 과정이기도 하고,

이때 양측이 서로의 **초기 sequence number를 교환**

초기 sequence number는 0-2^31-1 사이의 랜덤한 번호를 배정하기 때문에 전송측과 수신측이 이를 3 way handshake 과정에서 서로 교환해야 알 수 있음
>랜덤한 번호를 선택하는 이유는 이전 커넥션의 패킷과 쉽게 구분하기 위함도 있고, 보안을 위해서
>https://en.wikipedia.org/wiki/TCP_sequence_prediction_attack

- pipelining: 요청 -> 응답을 serial하게 할 필요 없이 응답을 기다리지 않고 계속 요청을 보낼 수 있음
<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/0*o4Rj35BFDHRtHHz1.png">

> 그러나 응답을 순서에 관계없이 받을 수 있는 완전한 비동기 방식은 아님(non-blocking synchronous 방식)
> 
> 기본적으로 각 응답이 어느 요청에 대응하는지 명확히 구분하기 위해(응답 순서가 뒤바뀌면 수신측 입장에선 각 응답이 어느 요청에 대한 응답인지 헷갈리므로)
>
> 먼저 들어온 요청에 대한 응답이 먼저 나가는 FIFO 방식으로 구현됨, 따라서 특정 요청에 대한 처리가 완료되어도 이전 요청에 대한 응답이 완료되지 않으면 응답할 수 없는 HOL(Head of Line) blocking 문제가 있음

HTTP 2.0

- 하나의 커넥션에 여러 스트림
> 이때 각 스트림은 HTTP 1.1과 달리 독립적으로 서로의 순서를 지킬 필요가 없음
> 메시지(요청 혹은 응답의 단위)를 프레임(헤더 혹은 데이터)이라는 더 작은 단위로 분할하여 각 프레임을 서로 다른 스트림으로 전송할 수 있음

<img src="https://mark-kim.blog/static/c0378d43c56fe6482d421edd37cd2553/b10c1/http_1_1_vs_http_2.webp">

스트림이라는게 어떤 물리적인 통로 개념으로 생각해서 잘 이해가 안되었었는데, 그런게 아니라 **그냥 요청-응답을 매핑하는 매핑 번호(스트림 ID)를 의미**하는 것

스트림 ID가 이 응답이 어떤 요청에 대한 응답인지 알려주기 때문에 전송하는 입장에선 꼭 들어온 요청 순서대로 응답을 보낼 필요 없이 처리가 끝나는 대로 순서에 상관 없이 보낼 수 있게 됨

그러나 이는 어디까지나 애플리케이션 계층에서 HOL blocking 문제를 해결한 것이지 전송 계층의 TCP 특성에 따른 세그먼트의 순서보장을 위한 HOL blocking 문제는 해결된 것이 아님 

HTTP 3.0
- TCP 가 아닌 UDP 채택
- QUIC 프로토콜
>앞서 2.0은 애플리케이션 계층에서의 HOL blocking 문제는 해결했지만 TCP 자체의 순서보장에 따른 HOL blocking 문제는 해결하지 못하였다고 했음, 이를 해결하기 위해 TCP 대신 UDP를 채택
>(TCP는 패킷을 무조건 순서대로 처리, 중간에 패킷이 유실될 시 다시 보내야 함, 이로 인해 병목현상 발생)

<img src="https://velog.velcdn.com/images%2Fshroad1802%2Fpost%2F9a9d652d-cbbe-4752-835c-5e3c04dd4fc6%2Fimage.png">

TCP는 순서 보장을 위해 **시퀀스 번호**를 사용하는데 만약 중간에 패킷이 손실되면, 수신 측은 연속된 시퀀스 번호에 해당하는 데이터만 상위 애플리케이션에 전달하고 ACK 번호로 해당 유실된 패킷의 시퀀스 번호를 요청하므로
손실된 패킷 이후의 시퀀스 번호를 갖는 패킷들은 애플리케이션 계층에 전달되지 못함

반면 QUIC 프로토콜은 UDP 위에서 동작하지만 TCP와 유사한 패킷 번호 체계를 구현하고 있고  **패킷 번호와 스트림 ID를 활용해** 손실된 부분만 선택적으로 재전송 요청을 하고 다른 스트림에 속한(다른 요청-응답) 패킷은 애플리케이션 계층으로 올려 보냄 

- connection ID 라는 개념을 사용하여 IP 주소가 변경되어도 커넥션을 새로 수립할 필요 없이 기존 커넥션을 사용


## Non blocking IO, Blocking IO, CPU bound, IO bound

https://www.youtube.com/watch?v=qnVKEwjG_gM

cpu bound 프로그램에서 적절한 스레드 수는 
cpu 수(코어 수) + 1 정도가 적당
> CPU-bound 작업의 경우, 실제로 코어에서 연산이 집중되므로 너무 많은 스레드를 생성하면 오히려 스레드 간 전환(컨텍스트 스위칭) 오버헤드 때문에 전체 처리 성능이 떨어질 수 있습니다. 즉, CPU-bound 작업은 코어 수에 맞춰 최소한의 스레드를 사용하는 것이 효율적

<img width="902" alt="스크린샷 2025-02-12 오전 1 12 14" src="https://github.com/user-attachments/assets/020fd3ca-5b6c-458d-8653-08eb0d7dcc44" />

>I/O-bound 작업에서는 네트워크 응답이나 디스크 I/O처럼 실제 CPU 연산이 거의 없는 작업이 주를 이루므로, 스레드가 I/O 작업을 기다리는 동안 BLOCKED 상태가 되어 CPU를 점유하지 않는다. 블로킹 상태에 있는 스레드는 스케줄러의 실행 대상에서 제외되기 때문에, 컨텍스트 스위칭 오버헤드가 상대적으로 작다.

즉, I/O-bound 작업에서는 많은 스레드를 사용해도 각 스레드가 실제로 CPU를 사용하지 않고, I/O가 완료되는 신호를 기다리기 때문에, 전체 처리 성능이 심각하게 저하되지는 않는다.

따라서 IO bound 작업은 딱 정해져있는 스레드 수는 없지만 스레드 풀의 사이즈를 적절히 설정하는게 중요
(너무 많아도 안됨)

일반적으로 스레드 수가 많아지는 것을 경계해야 하는 이유가 결국 코어 수는 한정되어 있기 때문
Non-blocking event driven 방식은 더 적은 스레드 수로 요청을 처리할 수 있기 때문에 컨텍스트 스위칭 비용이 적고, 이는 전체 처리량의 증가로 이어짐

<img width="947" alt="스크린샷 2025-02-12 오전 1 33 06" src="https://github.com/user-attachments/assets/b06da5f0-3d3f-4b98-9f47-d445fb2a6fa2" />


https://www.youtube.com/watch?v=EJNBLD3X2yg&t=1s

blocking I/O라면, 다중 클라이언트 요청들을 동시에 처리해주기 위해선 스레드를 늘리는 수 밖에 없다
(스레드를 늘리지 않으면 각 요청에 대한 작업이 끝날때까지 뒤에 도착한 요청들은 대기해야 하는, 줄서기가 됨)




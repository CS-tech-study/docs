## blocking, non-blocking, Sync, Async
- Sync와 Async는 호출되는 함수의 작업 완료 여부를 누가 신경쓰는가에 따라 구분하고
- Blocking과 Non-blocking은 호출되는 함수가 바로 리턴하는가를 기준으로 구분
<br>

- Sync + Blocking : ```read()```, ```write()```, Servlet 기반의 Spring MVC
- Sync + Non-blocking : polling 작업으로 계속해서 상태 확인
- Async + Blocking : I/O Multiplexing
- Async + Non-blocking : Spring WebFlux, Kafka

<br>

## naver.com 접속 과정
Encapsulation -> Decapsulation 과정으로 서비스 서버에 도착
### Decapsulation
- L2 -> L3 -> L4 순서로 헤더 확인 및 제거 후 상위 계층으로 패킷 전달
- 현재 계층에서 정의한 정보 + 상위 프로토콜 지시자의 정보를 헤더에 추가
- 예를 들어 L2: 출발지, 목적지 MAC 주소 (L2 사용) + EtherType 확인(IPv4, IPv6, ARP… L3 전달을 위한)
- L3: 출발지, 목적지 IP 주소 (L3 사용) + Protocol 필드 확인 (TCP/UDP, ICMP… L4 전달을 위한)
- L4: Seq, Ack flag (L4 사용, UDP는 제외) + 포트 번호로 해당 애플리케이션 (HTTP, SMTP…) 결정

<br>

## HTTP
HTTP/1.0
- 하나의 TCP 연결당 하나의 HTTP 요청/응답만 처리 -> network latency 발생

HTTP/1.1
- 하나의 TCP 연결당 여러 HTTP 요청/응답 가능
- 1.1 pipelining으로 클라이언트는 여러 요청 한 번에 보낼 수 있지만, 서버는 한 파일이 종료되면 다음 파일 전송
- 즉, 순차 응답으로 HOL(Head Of Line) Blocking 발생

HTTP/2
- multiplexing 방식으로 하나의 TCP 연결당 여러 스트림 존재
- 응답 기다리지 않고 다음 패킷 보내 HOL(Head Of Line) Blocking 문제 일부 해결
- TCP 기반으로 한 스트림에서 패킷 재전송 시 같은 연결의 모든 스트림 중단 
- 즉, HOL blocking 문제 완벽히 해결 불가
- 네이버: HTTP/1.1 + 2

HTTP/3
- QUIC(Quick UDP Internet Connections) 프로토콜로 각 스트림 개별적으로 패킷 손실 감지해 HOL blocking 문제 해결
- 같은 커넥션에 여러 스트림 존재해도, HTTP/2와 달리 독립적으로 관리되어 전체 스트림이 blocking 되지 않음
- UDP 기반이므로 3-way handshaking 과정 없음
- UDP 기반이지만, 패킷 손실 감지
- 크롬: HTTP/3

<br>

## 흐름 제어, 혼잡 제어
Low level
- Sliding window: 수신자 측 제어
- nagle 알고리즘 ON 설정: 같은 용량 -> 적은 패킷으로 전송하기 위한 목적
- 패킷 TTL 설정으로 무한 루프 방지
- ECN(Explicit Congestion Notification) flag 사용: 혼잡 감지 시 송신자에게 전송 속도 줄이도록 유도

High level
- Circuit Breaker 설정 (요청 일시적으로 차단해 불필요한 호출 제거)
- Rate limiting 설정 (API 호출 제한)
- Retry & Exponential Backoff (재시도 간격 지수적으로 늘려가기)
- Load Balancer 제작
- NIC 추가하여 인터넷 회선 분리 (인프라)

### 네트워크 호출 최소화 설계

- MSA 환경에서 각 마이크로서비스 간 통신 시 API Gateway를 거쳐 가지 않고, 다이렉트로 접근 (노드 1개 제거)
- 여러 마이크로서비스에서 사용되는 데이터를 API Gateway에서 패킷 헤더에 추가해 공용 사용
- 예를 들어 인증된 사용자 정보 -> 파라미터가 아닌 request 헤더에 추가해 전달 (유지보수)

<br>

## CORS 정책
- SOP (Same Origin Policy) : 같은 origin끼리 리소스 공유
- CORS (Cross Origin Resource Sharing) : 지정한 다른 origin도 리소스 공유 가능

React 와 AWS S3 관계에서 CORS 정책 문제 경험 -> AllowedMethods(POST), AllowedOrigins(http://localhost:3000) 을 설정해 문제 해결

<br>

## API Gateway 다운 시 대처 방안

트래픽의 진입점인 API Gateway 다운 시 추가 트래픽을 어떻게 해겷할 것인가
- 서버 이중화 (active-standby 구조, main 서버 다운 시 standby 로 트래픽 넘기기) 
- BFF (Backend for Frontend) 패턴: 클라이언트에 맞는 게이트웨이 분리 (트래픽 분리)

<br>

## 인터넷 회선 분리

standby 서버, replica 서버로의 동기화도 결국 네트워크 비용
- 복제 관련 트래픽을 별도의 인터넷 회선으로 분리해 클라이언트 트래픽과 분리
- IP 주소에서 호스트 주소뿐만 아니라 네트워크 주소도 다르게

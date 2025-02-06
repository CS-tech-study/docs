**Bandwidth**

**1.**     **채널의 주파수 도메인의 사이즈를 의미(Hz)** (아날로그 신호, 무선통신에서의 대역폭)

**2.**     **Maximum bit rate**를 의미(bps) (디지털 통신에서의 대역폭)

대역폭이 넓을 수록 해당 채널을 통해 전송할 수 있는 데이터의 양이 증가하여 디지털 통신에서의 대역폭(bps)이 증가

**Throughput :** 실제 data transfer rate
> Bandwidth가 물리적으로 가능한 최대 데이터 전송 속도라면, throughput은 실제로 데이터를 전송할 때는 데이터 뿐만 아니라 parity bit등 다른 요소들을 포함해서 전송하므로 실제 데이터 전송 속도는 bandwidth보다 작음(Throughput < bandwidth)

## OSI 7 Layer

네트워크 기능을 계층 구조로 모델링
계층 구조로 모델링 시 네트워크 장애 발생 시 어떤 레벨(구간)에서 발생했는지 보다 명확하게 파악 가능하다는 장점을 가짐

## 1계층, 물리 계층

- 전기적, 기계적, 기능적인 특성을 이용하여 데이터를 전송
- 데이터는 0과 1의 비트열로 표현되며, 이 계층에서는 단순히 데이터를 전달하는 역할
>line coding(인코더를 통해 디지털 데이터를 디지털 신호를 전송), block coding

<img width="373" alt="line_coding" src="https://github.com/user-attachments/assets/ad2ac85e-53f2-48c9-917d-ddee07578908" />


- 주요 장비: 케이블, 리피터, 허브 등

## 2계층, 데이터 링크 계층

- 1계층(물리 계층)을 통해 송수신되는 정보의 **오류 제어, 흐름 제어**를 담당
>오류 검출, 수정을 위해선 물리 계층에서 사용된 block coding이 다시 사용됨
>
>parity check, hamming code, checksum, CRC
>
>ARQ: 프레임에 오류 또는 손실 발생 시 재전송
>
>흐름 제어 방식 중 **sliding window** 

핵심은 **물리 주소인 MAC 주소를 사용하여 LAN 내에서 통신하는 계층**이 데이터 링크 계층

>MAC 주소: 네트워크 인터페이스 카드(NIC, 랜카드)와 같은 네트워크 장치에 할당된 고유 식별자로 제조 과정에서 장치에 영구적으로 설정, 각 네트워크 장치는 전세계적으로 고유한 MAC 주소를 가짐 

- **ARP**(Address Resolve Protocol) -> 네트워크 계층(ip주소를 알아야 하므로 데이터 링크 계층 X)
  
2계층에선 MAC 주소를 사용해서 통신하기 때문에 목적지를 제대로 찾아갈 수 있도록 LAN 내에서 **IP 주소를 MAC 주소로 변환**하는 프로토콜
>IP 주소는 논리 주소로 장치가 다른 네트워크로 이동하거나 네트워크 설정이 변경되면 달라질 수 있지만, MAC 주소는 물리 주소로 고정

이 LAN 내에 사용되는 통신 프로토콜이 **이더넷**

- 데이터 전송 단위: **프레임**
>상위 계층에서 받은 데이터를 이더넷 헤더와 트레일러로 감싸는 **캡슐화** 과정을 통해 생성

- 주요 장비: 브릿지, 스위치

### 허브 vs 스위치

허브: MAC 주소를 관리하지 않기 때문에 연결된 모든 네트워크 기기에 패킷을 전달해야 함(broadcast), 한번에 한방향으로만 데이터 전송이 가능(half-duplex)

스위치: 자신의 포트에 연결된 MAC 주소를 관리하기 때문에 목적지에만 정확히 전달(unicast), 동시에 양방향으로 데이터 전송 가능(full-duplex)

스위치에 MAC 주소가 캐싱이 안돼있을 경우 정확히 어디에 보내야할지 알 수가 없으므로 일단 ARP 요청을 브로드캐스트 주소로 설정하여 네트워크의 모든 장치에 전송, 모든 기기 중 요청된 ip 주소가 본인에 해당하는 장치만 ARP 응답, 스위치는 이 ARP 응답을 통해 MAC 주소를 캐싱하고 이때 원래 보낼려했던 패킷을 전송

<img width="290" alt="arp" src="https://github.com/user-attachments/assets/bccdc307-be9f-4754-afd1-40a28a1cc202" />


>스위치의 가격이 많이 내려가면서 허브는 이제 잘 사용하지 않는다고 함(100 Gbps 이상의 초고속 전송에서는 패킷을 확인하는 오버헤드때문에 허브를 사용한다고 하기도 함..)
>
https://aws-hyoh.tistory.com/75

https://brunch.co.kr/@swimjiy/49


## 3계층, 네트워크 계층

- 데이터를 목적지까지 가장 안전하고 빠르게 전달하는 **라우팅** 기능 제공
- 데이터 전송 단위: **패킷**
- 주요 장비: 라우터, L3 스위치(L2 스위치에 라우팅 기능을 장착(IP 주소 사용)
- **IP 계층**: 네트워크의 주소 (IP 주소)를 정의하고, **IP 패킷의 전달 및 라우팅을 담당하는 계층**

IP 프로토콜의 특징: 신뢰성(오류 제어, 흐름 제어) 기능이 없음(best effort service), 따라서 신뢰성 확보를 위해선 IP 계층 위의 전송 계층에 의존해야 함
>IP 프로토콜은 비연결 지향이기 때문에 각 데이터그램이 목적지까지 서로 다른 경로를 거칠 수 있음. 이말은 **전송 순서와는 상관 없이 수신지 도착 순서가 달라질 수 있음**

IP Hourglass Model

![ip hourglass](https://github.com/user-attachments/assets/e0e634a9-220d-4f45-817d-49a7d3d29133)


>**모든 라우터는 IP 프로토콜을 구현해야함**(transport layer, data link layer, physical layer는 각각 다른 프로토콜을 사용할 수 있지만 network layer는 반드시 IP 프로토콜을 구현해야 함

### IP 주소 

<img width="187" alt="ip주소2" src="https://github.com/user-attachments/assets/60b323e5-cc0f-4375-b86a-33de117e707c">

**network address + host address**로 구성

큰 기관들은 호스트가 많이 필요하므로 class A 할당, 작은 기관들은 B 또는 C 할당

- 서브넷
클래스에 따라 네트워크와 호스트를 구분하던 것에서 좀 더 세분화하여 네트워크 사이즈를 조절 가능
>CIDR(클래스 없는 도메인 간 라우팅 기법)

**x.y.z.t/n**으로 표현 (n은 서브넷 마스크의 길이(prefix length))
> 205.16.37.39/28 **이면 호스트가 4비트이므로 16개, 따라서 205.16.37.32~47까지 같은 서브넷**

**서브넷팅의 의미**

- 필요한 범위의 IP 주소만 사용하여 비용 절감
- 브로드캐스트 도메인을 줄여 불필요한 트래픽을 줄임
- 같은 서브넷에 속한 호스트끼리는 라우터를 거치지 않고 통신

## 라우팅 알고리즘

- Distance Vector(벨만포드 알고리즘)
- Link State(다익스트라 알고리즘)
>Distance Vector 알고리즘은 자신과 인접한 이웃 라우터와 테이블 정보를 교환하지만, Link State 알고리즘은 네트워크에 속한 모든 라우터와 테이블 정보를 교환 

우리가 집에서 사용하는 공유기가 L3 스위치로 라우터(3계층), 스위치(2계층), wifi 액세스 포인트 역할을 하는 장비

## **Intra-domain (AS, Autonomous Systems)**

- **동일한 특정 라우팅 정책**을 갖는 네트워크 그룹으로 하나의 관리자(ISP)를 가짐
- 각 AS는 unique한 AS number(ASN)를 가짐

<img width="275" alt="AS" src="https://github.com/user-attachments/assets/d2802730-030e-4a68-aead-85386ab7463b" />


R1, R2, R3, R4가 **게이트웨이 라우터**(다른 AS와 link를 가지는 라우터)로 **BGP(border gateway protocols)** 라는 라우팅 프로토콜을 사용
>특정 AS에 존재하는 특정 IP 주소를 다른 AS들에게 소문내서 서로를 연결시키는 과정

이 게이트웨이 라우터가 NAT(Network Address Translation)을 통해 사설망을 만들 수 있음
>퍼블릭 IP 절약, 프라이빗 IP주소는 외부로 노출되지 않으므로 보안상 이점

VPN: 퍼블릭 네트워크를 통해 분리된 프라이빗 네트워크들을 안전하게 연결하는 기술

## LAN vs WAN

LAN 또는 WAN은 **라우터**를 통해 연결됨

<img width="642" alt="스크린샷 2025-01-30 오후 11 39 55" src="https://github.com/user-attachments/assets/bbfa2c40-78ff-41b4-bdbe-74e6901fd4ba" />

호스트 또는 라우터를 노드라고 부르고, 노드와 노드 사이의 네트워크(LAN 또는 WAN)을 링크라고 부름

LAN과 WAN을 나누는 기준이 모호하긴 한데...일단 네트워크의 범위가 크냐 작냐 이런건 너무 기준없는 말인거같아서 찾아봤는데
**LAN**: MAC 주소에 기초해서 작동하는 네트워크, 브로드캐스트 주소가 적용되는 범위

>ARP 요청이 브로드캐스트 주소를 사용하므로 ARP가 닿는 범위도 같은 말
>브로드캐스트 주소: 서브넷의 마지막 IP 주소

근데 헷갈리는게...그러면 논리냐 물리냐를 떠나서 범위 자체는 LAN = 서브넷 이 맞지 않나..싶은데

**WAN**: IP 주소에 기초해서 작동하는 네트워크

<img width="960" alt="스크린샷 2025-02-06 오전 1 47 56" src="https://github.com/user-attachments/assets/83a2d656-02d8-4a7b-ac2c-212170785edf" />


>출처: https://youtu.be/N8pE-vDsJ38?si=bpo6tItvEtDB2YRy

내 입장에서 LAN이 아니어서 라우터를 거쳐야 하는 네트워크는 전부 WAN?
## 4계층, 전송 계층

네트워크 계층은 IP 주소에 따라 패킷을 전달할 뿐 이 패킷이 어떤 프로세스로 가야하는지는 모른다
>포트 번호를 통해 특정 프로세스로 전달
>즉, 포트 번호는 프로세스를 구분하는 식별자
>21(FTP), 22(SSH), 23(TELNET), 25(SMTP), 80(HTTP), 443(HTTPS)

데이터 전송 단위: **세그먼트**

소켓: 커널에 구현된 TCP/IP 프로토콜을 애플리케이션 레벨에서 접근할 수 있도록 추상화한 파일

### UDP
- **Connection-less :** 별도의 연결 없이 통신 수행
- **Unreliable :** error control X, ordering X
- **기능 :** port를 통한 프로세스 구분 (multiplex/demultiplex), checksum을 통한 기초적인 에러 검출
- **Multimedia streaming**에서 사용
- **장점 :** 구현이 쉬움, 처리 속도가 빠름

### TCP
- **Connection-oriented** : 통신 전 논리적 연결이 이루어짐
- **Reliable :** 메시지들이 순서대로, 에러 없이 수신됨
- **Stream delivery service**
기능
- **Flow control :** 수신자의 입장에서 바이트가 너무 빨리 들어오면 안되니까 전송 속도를 조절
- **Congestion control :** 네트워크가 혼잡하면(라우터의 버퍼가 한계) 전송 속도를 조절
- **Error control :** 에러가 발생한 segment는 재전송

IP 계층 위에서 3 way handshake를 통해 TCP 연결을 설정하고, 데이터의 신뢰성, 순서 보장, 흐름 제어 및 오류 검출/복구와 같은 기능을 제공(TCP 연결에서 소켓이라는 파일을 사용하는 이유)

**2계층의 오류 제어와 차이점**: Data link layer에서의 error control의 경우 하나의 링크(LAN 혹은 WAN) 사이에서 에러 없이 전송되도록 하는것이고, 라우터나 호스트에서의 패킷 손실의 경우는 해당되지 않음
>신뢰성있는 프로세스 간 통신을 위해선 전송 계층의 오류 제어가 필요

<img width="213" alt="프로세스버퍼" src="https://github.com/user-attachments/assets/9dc74c42-ab6c-4437-afe8-6a8555d6489b" />

애플리케이션 프로세스가 send() 함수를 호출하면, 데이터는 **운영체제의 송신 버퍼**에 저장, 송신 버퍼가 있기 때문에 프로세스는 비동기적으로 데이터를 전송할 수 있음
>애플리케이션 프로세스는 send() 호출 후 즉시 다음 작업을 수행할 수 있으며, 데이터의 실제 전송은 운영체제가 관리

### Flow Control

수신측은 송신측에게 ACK을 보낼 때 헤더의 window size 필드에 자신의 수신 버퍼에 공간이 얼마나 남았는지 알려줘서 송신측은 이를 통해 전송 속도를 조절

### Congestion control

라우터의 버퍼의 input 속도가 output 속도보다 빠르면 버퍼가 채워지면서 delay가 증가하다가 결국 버퍼가 overflow되고 패킷 드랍 발생
>슬라이딩 윈도우의 사이즈가 라우터 버퍼의 load를 결정하므로 이를 조절해서 최대의 throughput을 얻는게 목표, 이때 라우터는 본인의 버퍼가 얼마나 찼는지 피드백을 줄 수 없으므로 그냥 congestion이 발생할 때까지 윈도우 사이즈를 키우다가 congestion이 발생하면 그때 윈도우 사이즈를 줄이는 방식
### TCP는 IP 프로토콜 기반인데 어떻게 메시지 순서를 보장하는가?

**TCP sequence number :** 각 TCP segment는 sequence number를 가짐
Segment에 들어있는 데이터 중 **첫번째 바이트의 byte number**

>수신 측은 이 sequence number를 기반으로 수신한 세그먼트를 올바른 순서로 재조립

**Acknowledgement number**: 수신자가 segment를 수신하면 TCP-ACK을 전송자에게 보내는데 이 ACK 메시지에 acknowledgement number를 추가해서 보냄

**수신자가 중간에 누락없이 마지막으로 받은 바이트의 byte number + 1**

 -> 즉, **next expected byte**
 
 <img width="315" alt="acknumber" src="https://github.com/user-attachments/assets/67340598-2948-4d00-b05f-484f4876c7aa" />

 
 >송신 측 입장에선 이 ACK number를 확인하여 다음에 전송해야 할 sequence number를 알고 보낼 수 있음
 >ACK number는 이 번호 이전까지는 중간에 누락없이 수신자가 받았다는 의미이므로 송신 측에선 무조건 이 ACK 번호부터 바이트를 보내면 됨

근데 위 경우 104부터 다시 보내면 필연적으로 106, 107, 110은 중복되어 받게 되는데..아마 아래 사진의 버퍼에 저런식으로 간격을 두고 바이트가 들어가야 할 자리가 정해져있기 때문에 상관없을거같다...
 
<img src="https://velog.velcdn.com/images%2Fshroad1802%2Fpost%2F9a9d652d-cbbe-4752-835c-5e3c04dd4fc6%2Fimage.png">

>출처: https://velog.io/@shroad1802/TCP%EC%9D%98-%EC%98%A4%EB%A5%98%EC%A0%9C%EC%96%B4#%EC%A4%91%EB%B3%B5-%EC%84%B8%EA%B7%B8%EB%A8%BC%ED%8A%B8

송신 측에서 설정한 타임아웃까지 ACK이 오지 않으면 세그먼트가 손실된 것으로 판단하고 재전송
송신측에서 타임아웃 시간이 만료되기 전에 동일한 ACK number를 3번 받으면 그냥 세그먼트가 손실된 것으로 판단하고 타임아웃이 되지 않아도 바로 재전송(빠른 재전송)

>카프카의 멱등성 프로듀서에서 사용되는 sequence number랑 굉장히 유사한 개념인거같음...
>궁금한게 그럼 카프카의 프로토콜도 결국 TCP/IP 기반인데 왜 굳이 애플리케이션 레벨에서 한번 더 sequence number를 사용하는걸까..?



---
## 참고
https://m.blog.naver.com/lyshyn/221300886868
https://brunch.co.kr/@swimjiy/49
https://aws-hyoh.tistory.com/75
https://velog.io/@shroad1802/TCP%EC%9D%98-%EC%98%A4%EB%A5%98%EC%A0%9C%EC%96%B4#%EC%A4%91%EB%B3%B5-%EC%84%B8%EA%B7%B8%EB%A8%BC%ED%8A%B8

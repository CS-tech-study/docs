scale out한 분산 서버 환경에서 트래픽을 균등하게 분산시키기 위해 필요

로드 밸런서가 없다면...
- 기껏 scale out하여 서버를 여러대 두어도 특정 서버에만 트래픽이 몰리고 다른 서버들은 놀고 있으면 단일 서버 환경이랑 똑같다
- 특정 서버가 다운되었을 때 해당 서버로 요청을 보낸 클라이언트는 장애 상황을 겪게 된다

따라서 로드 밸런서의 목적은 다음과 같다.
- 서버 과부하 방지(특정 서버에 트래픽이 몰리는 것을 방지)
- 가용성 확보(특정 서버가 다운되어도 다른 서버로 트래픽을 라우팅)

## 로드 밸런싱 알고리즘 종류


- Round Robin
- Least Connection
- IP Hash
- 등등..

## 로드 밸런서의 작동 원리..?

클라이언트가 서버에 서비스를 요청할 때 서버에 직접 TCP connection을 맺는 것이 아니라 로드밸런서를 통해 connection이 맺어진다. 

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FGdtg3%2FbtrC1iDQiOD%2FRPakGslhkJo3Cgvyc6Ba70%2Fimg.png">

지난 시간에 web server가 was 앞단에 위치하는 리버스 프록시를 설명하면서 DMZ 영역에 리버스 프록시(로드 밸런서)를 위치시키고 서비스들을 내부망에 위치시키는 망 분리를 하는 경우에 대해 설명했는데, 
이때 내부망-> DMZ 영역 접근은 가능한데 반대로 DMZ 영역 -> 내부망은 접근할 수 없다고 설명했음

근데 로드 밸런서가 요청을 내부망의 특정 서비스에 라우팅해줄려면 DMZ -> 내부망으로 접근해야 하므로 이 경우 방화벽 설정을 통해 최소한의 포트만 허용하여 DMZ에서 내부망으로의 특정 트래픽을 허용하도록 구성해야 한다.

> 내부망은 사설 IP를 사용하므로 NAT 과정이 필요

## L4 로드 밸런서 vs L7 로드 밸런서

L4 계층 부터 포트 번호를 통해 어떤 프로세스로 가야하는지 알 수 있기 때문에 L4 로드 밸런서와 L7 로드 밸런서가 많이 사용된다
> 보충설명: Network layer는 IP 주소에 따라 패킷을 전달할 뿐 이 패킷이
>  어떤 프로세스로 가야하는지는 모름, **이는 transport layer의 port를 통해 특정 프로세스로 전달**

|           | L4 로드밸런서        | L7 로드밸런서                               |
| --------- | --------------- | -------------------------------------- |
| 로드 밸런싱 기준 | 클라이언트의 IP주소, 포트 | + url, 패킷의 내용(http 헤더), payload, 쿠키 등  |

한마디로 정리하면 L7 로드밸런서는 L4 로드밸런서의 ip주소, 포트 정보 뿐만 아니라 L7에 해당하는 다양한 정보를 추가적으로 활용할 수 있기 때문에 더 세밀한 트래픽 분산이 가능하다 

## 로드 밸런서의 scale out

단일 로드 밸런서를 사용하면 이 로드 밸런서에 장애 발생 시 전체 서비스로 장애가 확산될것이므로, 로드 밸런서도 스케일 아웃이 필요
>AWS ALB, NLB의 경우 가용 영역 당 하나의 로드 밸런서를 위치 시키는 걸로 알고 있음

DNS에 여러 로드 밸런서들의 ip 주소를 등록하여 클라이언트 입장에선 로드 밸런서의 ip 주소를 알 필요 없이 DNS만 알고 있으면 됨

## 의문점

일단 제가 api gateway 관련 지식이 부족해서 헷갈리는거 같긴한데...

-  API Gateway가 있는 경우 따로 로드 밸런서를 둘 필요가 없는건가?

API gateway의 경우 L7 로드밸런서의 기능 외에도 rate limiting(서버에서 특정 임계치까지만 클라이언트의 요청을 허용(throttling)을 할 수 있다고 하는데..

기능 범위의 측면에서 L4 로드밸런서 < L7 로드밸런서 < API gateway 인거 같은데

일단 gpt 피셜로는 아래와 같이 설명함

• **L4 LB**는 네트워크 레벨(TCP/UDP)에서의 **단순/고성능 분산**이 목적이고,

• **L7 LB**는 HTTP 트래픽의 **헤더/URL 기반 분산, SSL 종료** 등이 가능하며,

• **API Gateway**는 L7 수준 라우팅 뿐 아니라, **인증, 권한, 트래픽 제어, API Key 관리, 로깅/모니터링** 등 **‘API 관리 전반’을 담당**한다.


<img width="982" alt="스크린샷 2025-01-31 오후 6 35 11" src="https://github.com/user-attachments/assets/c4498830-b7a8-4be7-8220-81358b1bcfc6" />



gpt 피셜로는 실무에서 다음과 같은 조합이 있다고 하는데..

1. **L4 LB + API Gateway**

- L4 LB로 **다수의 Gateway 인스턴스**에 트래픽을 **단순 분산**하고, Gateway에서 **L7 로직(라우팅+인증**을 수행.
- 클라우드 환경에서 **NLB** → (Auto Scaling된) **API Gateway** or **Microservice** 식.

2. **L7 LB + API Gateway**

- L7 LB가 먼저 **HTTPS 종단** 및 **도메인/서브도메인 분기** 등을 수행, 일부 트래픽(정적 리소스 등)은 LB에서 직접 처리하고,
- **API 요청**만 API Gateway로 **프록시**해서 고급 기능(인증/보안/모니터링) 적용.
- 예: AWS ALB에서 정적 파일은 S3, API는 Gateway로.

3. **(단독) L7 LB**

- 작은 규모나 단순 웹 서비스에서는 굳이 API Gateway를 쓰지 않고, **L7 LB**의 경로 기반 라우팅만으로 마이크로서비스를 노출하기도 한다.

4. **(단독) API Gateway**

- Spring Cloud Gateway, Kong, AWS API Gateway 등 자체적으로 L7 기능이 충실하다면, 로드 밸런서를 **L4**로 두고 API Gateway로 고급 라우팅+보안+관리까지 처리.

<img src="https://deploy.equinix.com/media/images/owKg-apigatewaylbcombined.png">

https://deploy.equinix.com/blog/how-api-gateways-differ-from-load-balancers-and-which-to-use/


## 참고
---
https://etloveguitar.tistory.com/136

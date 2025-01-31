## 로드밸런싱

부하 분산을 위해 애플리케이션 서버를 여러 노드에서 운영
- 스케일 아웃을 통해 서버 처리 능력 향상뿐만 아니라 장애에도 대처
- 서버 부하분산을 위한 방법으로 DNS Round Robin, 로드밸런서, GSLB 등이 존재
- 여기서 로드밸런서는 자신이 모든 부하를 받고, 실제 여러 서버로 부하 분산하는 방식

<br>

## L4, L7
L4, L7
- L4 로드밸런서: IP 주소와 포트 번호 정보로 부하 분산
- L7 로드밸런서: URL, 헤더 등 애플리케이션 레벨 정보로 부하 분산

모놀리식 환경
- 분산 서버로 운영하는 경우 IP 주소 동일, 포트 번호 다름 (로컬 개발 환경)
- 앞단에 로드 밸런서 필요 (L4)

MSA 환경 
- 앞단에 Spring Cloud Gateway 필요
- G/W가 uri 기반으로 처리할 노드 선택
- 그러므로 Spring Cloud Gateway는 API G/W로 L7 역할 수행

AWS
- AWS NLB(Network L/B)가 L4 로드밸런서
- AWS ALB(Application L/B)가 L7 로드밸런서

*Dispatcher Servlet*
- 애플리케이션 내부에서 HTTP 요청을 처리할 수 있는 컨트롤러로 라우팅
- 그러면 uri 기반으로 컨트롤러 선택하니 L7 역할 수행…?

*AWS EC2*
- EC2 배포 시 다음과 같이 인바운드, 아웃바운드 설정
- tcp 8080 / ssh 22 / http 80 / https 443
- 해당 포트로의 트래픽 허용하는 것이지, 별도의 서버가 생성되는 것이 아님

<br>

## NAT
현재 IP 부족으로 IPv4 -> IPv6 주소체계로 넘어가고 있는데
- 이 문제를 해결하기 위한 다른 방법으로 public IP, private IP 구분해 사용
- NAT를 통해 여러 Private IP를 하나의 Public IP로 매핑해 주소 부족 문제 해결

분산 서버라도 클라이언트는 해당 서버가 단일 서버인지, 분산 서버인지 알 수 없음
- 클라이언트는 DNS를 통해 제공된 public IP로 접근하면
- L4 스위치에서 여러 분산 서버의 private IP 중 하나로 변경해 전달 (부하 분산)
- L4 스위치 기능: NAT -> 로드밸런싱

<br>

## Round Robin 알고리즘
- Spring Cloud G/W, L/B는 Round Robin 방식으로 부하 분산

Least Connection 알고리즘
- 현재 세션이 가장 적게 연결된 서버로 요청 보내는 방식
- 가장 짧은 대기열로 보내지만, 해당 서버가 빠르게 처리해준다는 보장은 없음
- 실제로 토스 증권에서도 지연 시간 증가로 인해 least connection 알고리즘에서 round robin으로 돌아온 사례
- https://www.youtube.com/watch?v=WKYE-QtzO6g&t=1s

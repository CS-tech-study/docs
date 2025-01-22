# WS & WAS

![그림1](/spring/images/ws-was.png)

> 출처: [https://velog.io/@kwontae1313/WS와-WAS](https://velog.io/@kwontae1313/WS%EC%99%80-WAS)
> 

## WS (Web Server)

### 하는 일

- 정적인 리소스(이미지, HTML 문서..)를 클라이언트에게 전달
- 동적인 요청이 들어올 경우 WAS에게 위임

## WAS (Web Application Server)

### 하는 일

- 동적인 요청에 대해 DB와 질의 응답, 비즈니스로직 수행
- 정적인 리소스에 대한 응답도 처리할 수 있음 (WS의 역할 수행 가능)

### 그럼 왜 분리?

1. 유지 보수성 증가
    - 둘은 서로가 어떻게 구현되는지, 어떤 동작을 하는지 알아야할 필요가 없습니다.
    - 각각의 동작에 특화된 유지보수 작업을 수행할 수 있습니다.
2. 리버스 프록시
    - 웹 서버가 클라이언트의 요청을 먼저 수신하고 필터링한 후, 필요한 요청만 WAS로 전달함으로써 직접적인 접근을 차단합니다.
    - WAS와 내부 시스템을 바깥쪽에서는 알 수 없게 보호하고, 보안 위협을 최소화할 수 있습니다.
3. 부하 분산 (로드 밸런싱)
    - 여러 대의 WAS를 구성하여 로드밸런서를 통해 요청을 분산 시킴으로써 시스템의 확장성과 안정성을 높일 수 있습니다.

## 로드 밸런싱을 위해 여러 WAS를 올린다면?

![그림2](/spring/images/load_balancing.png)

> 출처: https://taetaetae.github.io/2019/08/04/apache-load-balancing/
> 

로드 밸런싱을 할 수 있다는 건 알겠는데 과연 이런 문제는 없을까요?

### 1. **세션 관리**

- 세션 정보가 한 WAS에만 저장되면, 요청이 다른 WAS로 전달될 때 세션이 유지되지 않을 수 있지 않을까요?

⇒ 

- **Sticky Session:** 특정 세션의 요청을 처음 처리한 서버로만 전송
- **세션 공유:** Redis와 같은 중앙 저장소에 저장

### 2. **데이터 일관성**

- 동시에 한가지 데이터에 대해 여러 WAS에서 요청을 보내거나 제대로 반영되지 않을 수 있지 않을까요?

⇒ 

- 모든 WAS가 동일한 데이터베이스(DB)와 연결
- 트랜잭션을 적절하게 묶어주고, 락을 걸어서 충돌 방지

---

## 우리 스터디에서 이 주제에 대해 말하게 된 계기 - 둘은 분리?

우리 스터디에서 이 주제를 선정하게 된 시작이 이것이라고 생각합니다.

그렇다면 둘은 완전히 분리된 두 프로그램일까요?

### AJP 프로토콜

- WS(Apache)와 WAS간 연결 요청을 위임할 수 있는 바이너리 프로토콜.
- 주로 8009포트를 이용

### **Ghostcat**

- AJP Request를 처리할 때 WAS에서 org.apache.coyote.AjpProcessor로 데이터를 처리하는데 이때 검증 없이 요청을 처리한다.
- 영향을 받는 버전(Apache Tomcat)
    - 9.0.30 이하
    - 8.5.50 이하
    - 7.0.99 이하

#### ⇒ 이런 프로토콜을 사용한다는 것은 분리되어 동작하는 프로세스임을 방증

---

### 참고자료

https://velog.io/@dkwktm45/WSWeb-Server-vs-WASWeb-Application-Server

[https://velog.io/@kwontae1313/WS와-WAS](https://velog.io/@kwontae1313/WS%EC%99%80-WAS)

https://taetaetae.github.io/2019/08/04/apache-load-balancing/

https://noobnim.tistory.com/26

https://secuhh.tistory.com/24
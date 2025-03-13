## ACID

- Atomicity 원자성
  - 트랜잭션 내 모든 연산이 원자적으로 처리되어 모두 커밋 또는 모두 취소되는 것을 보장
  - 단일 환경: @Transactional (proxy 기반 AOP로 동작)
  - 분산 환경: 한 요청에 대한 여러 트랜잭션으로 분리 (2PC, saga 패턴)
- Consistency 일관성
  - 데이터베이스의 무결성 제약
  - ‘1부터 10까지의 값만 가질 수 있다’ 같은 약속
  - 비동기식으로 복제되는 시스템, ACID, CAP 정리 등 여러 상황에서 일관성의 의미가 조금씩 다름
- Isolation 격리성
  - 각 트랜잭션은 데이터베이스에서 유일하게 실행되는 트랜잭션인 것처럼 동작할 수 있음을 의미 -> 여러 동시성 문제 방지
  - MySQL은 MVCC 기법으로 쓰기 작업과 읽기 작업이 서로 막지 않고 처리 가능
  - MVCC: 락을 사용하지 않고 특정 트랜잭션에게 UNDO 영역에 존재하는 여러 버전 중 일관된 버전만 제공
- Durability 지속성
  - 트랜잭션이 커밋되면 그 결과는 영구적으로 반영되어야 함
  - 하지만 현실에서 완벽한 지속성 보장은 어려움 (손실 가능성 존재)

<br>

## 트랜잭션 격리 수준

- Level 0: Read Uncommitted
  - 커밋되지 않은 데이터 읽기 가능
  - 커밋되지 않은 데이터로 새로운 데이터 생성 시 발생할 수 있는 문제까지 생각한다면 해당 격리 수준은 위험
- Level 1: Read committed
  - 커밋된 데이터만 read
  - 읽기 작업은 undo 영역에 있는 커밋된 데이터 제공
  - MySQL 해당 격리 수준에서 gap, next key lock 적용되지 않아 phantom read 현상 발생할 수 있음
  - 가끔 읽기 작업을 read committed 격리 수준으로 내려 진행하시는 분들이 계셨는데, 개인적인 생각으로 읽기 작업엔 락이 필요하지 않아 격리 수준을 내릴 필요가 없다고 생각...
- Level 2: Repeatable Read
  - MySQL에서는 MVCC 메커니즘을 통해 트랜잭션은 같은 버전의 데이터를 제공해 비반복 읽기 현상을 방지
  - MVCC 기법은 읽기, 쓰기 작업이 서로 막지 않아 빠른 처리 가능
  - MySQL 해당 격리 수준부터 gap, next key lock 적용되어 Phantom read 현상을 방지하나, 쓰기 쿼리나 락을 사용한 읽기 쿼리를 잘못 작성하면 많은 부분이 잠길 수 있음
    - Phantom read: 한 트랜잭션에서 여러번 조회 시 새로운 데이터가 추가되거나 제거된 상태
    - Write Skew: 서로 다른 행에 대해 독립적인 변경 발생 -> 이 과정에서 정합성이 깨지는 문제, Phantom read 현상 때문에 쓰기 스큐 발생할 수 있음
- Level 3: Serializable
  - 공유 자원에 대한 트랜잭션 순차 처리해 일관성 제공
  - 읽기 작업도 S-Lock을 획득하기 위해 대기 -> 동시성 저하
  - MySQL에서는 next key lock으로 phantom read 현상을 해결하고 있어, 해당 격리 수준을 사용할 일이 거의 없지 않을까 추측

<br>

## Index 설정

- index 설정으로 full scan을 막을 수 있음
- Index 설정 시 B+ Tree 생성되므로 여러 조건 고려
  - Cardinality와 Selectivity가 높은 요소 (유니크한 PK, LOGIN ID...)
  - Cardinality: 유니크한 개수
  - Selectivity: 데이터 집합에서 특정 값을 잘 골라낼 수 있는 지표
    - Selectivity = Cardinality / Total Number Of Records
    - 1에 가까울수록 인덱스 성능 높아짐
- B+ Tree는 정렬된 상태이므로 index를 자주 변경되는 요소로 설정하면 성능 저하로 이어짐

### Clustered Index
- MySQL Innodb 스토리지 엔진은 PK가 Clustered Index로 자동 설정
- clustered index는 리프 페이지에 실제 데이터 저장

### Non-clustered index

- MySQL InnoDB 스토리지 엔진의 경우 리프 페이지에 실제 데이터의 PK 값 저장
- PK 확인 후 PK를 가지고 Clustered Index을 한 번 더 타 리프 노드 도달에 데이터 접근
- 인덱스 값만으로 쿼리를 해결할 수 있다면 Clustered Index를 타지 않음 (covering index)
- 만약 Clustered Index 검색이 필요한 상황에서 Non-clustered Index를 Range Scan 했다고 해서 Clustered Index에서도 Range Scan을 한다는 보장 없음 
- 즉, 각 레코드마다 랜덤 I/O 발생할 수 있어 그만큼 어떤 값을 인덱스로 설정할 것인지도 중요

### 복합키

A, B, C 순서로 복합키 생성
- where A = ? and B = ? and C = ? -> 인덱스 사용
- where A = ? and B = ? -> 인덱스 사용
- where A = ? -> 인덱스 사용
- where A = ? and C = ? -> A 부분만 인덱스 스캔할 가능성 있음
- where B = ? and C = ? -> full 스캔 가능성 높음
- where A = ? and C = ? and B = ? -> full 스캔 가능성 높음

### B+ Tree

- B+ Tree는 리프 노드가 연결된 구조라 Range Scan 가능
- 리프 노드가 단방향으로 연결된 구조
- 보통 정렬 기준은 오름차순
- 오름차순 Range Scan은 빠름 (forward index scan)
- 반면 내림차순 Range scan은 상대적으로 속도 느림 (backward index scan)
- 그렇다고 마지막 부분을 가져오는 상황에서 Limit 사용해 forward index scan을 유도하면 더 많은 부분을 스캔할 가능성이 있어 차라리 DESC를 사용해 backward index scan이 낫다고 개인적으로 생각...

### 데이터 일관성, 동기화

- 세선 A: select ... where PK for update 진행, X, REC_NOT_GAP 락 획득
- 세션 B: update ... where secondary index 시도
  - index에 대한 X, REC_NOT_GAP 획득
  - 그러나 PK에 대한 X, REC_NOT_GAP 락 획득 대기 (세션 A 작업이 커밋될 때까지 대기)

***

### 비동기 Replication (Real MySQL 8.0 2권 참고)

- replica 서버에는 바이너리 로그를 소스 서버에게 요청 
  - 완전한 push도, 단순한 pull 도 아님 (gpt 내용, 공식 문서에서 push, pull 내용 찾지 못함)
  - 소스 서버는 replica가 연결을 먼저 요청해야만 binary log 보내기 시작 (완전한 push X)
  - replica가 주기적으로 요청하는 것이 아니라 끊기지 않는 한 지속적인 로그 스트리밍을 받음 (streaming pull)
- 기본 복제 방식은 단방향 비동기로 소스 서버는 레플리카에 변경 이벤트가 잘 전달 및 적용되었는지 파악하지 않음
  - 이는 소스 서버와 레플리카 서버의 장애가 전파되지 않다는 장점이지만,
  - 데이터 일관성이 충분히 깨질 가능성 있음
<br>

### 복제 과정
- 레플리카 서버가 연결되면 소스 서버는 바이너리 로그 덤프 스레드 생성해 바이너리 로그 전송
- 레플리카 서버의 Replication IO 스레드가 바이러니 로그를 받아 릴레이 로그에 기록
- 레플리카 서버의 Replication SQL 스레드가 릴레이 로그를 읽고 디스크에 기록
<br>

### 반동기 복제
- 변경 이벤트를 레플리카 서버가 릴레이 로그에 기록 후 소스 서버로 응답
- 소스 서버는 이 응답을 받으면 트랜잭션을 커밋하고 클라이언트에게 결과 반환
- 완전 비동기 방식보다는 이벤트 전달을 보장할 수 있지만, 속도 느려짐
- 또한, 변경 이벤트가 전달되었음을 보장하는 것이지, 레플리카 서버의 데이터 파일에 적용되었음을 보장하는 것이 아님
  - ~~이처럼 데이터 일관성을 유지하는 것은 어려운 문제 (예를 들어 다중 DB에서 Phantom 현상을 어떻게 해결할 것인가) 라고 생각했지만,~~
  - 공식 문서에서 '반동기 복제는 커밋이 성공적으로 반환되면 데이터가 최소 두 곳에 존재해 데이터 무결성을 제공한다'라고 작성된 것을 보면 릴레이 로그가 데이터 파일에 적용되는 것을 내부에서 알아서 보장하는 듯
  - https://dev.mysql.com/doc/refman/8.0/en/replication-semisync.html
<br>

### 원격 복제
- 소스 서버 미국, 복제 서버 한국 & 일본 (온프레미스 환경이라 가정)
- 원격 복제도 기본 설정으로 가능
- 대신 latency 발생할 수 있음 (최종적 일관성)
- 인터넷망 사용료 발생
  - 리플리케이션 구조를 최적화하지 않으면 2-3년 전부터 논란이 되었던 CP-ISP 간 망 사용료 문제로 이어지지 않을까... 개인적인 생각
  - 상위 ISP가 같은 리전에서는 계층형 복제 구조를 적용하거나
  - MySQL에 바이러니 로그 압축 기능이 있던데 사용하거나 (정확한 해결책인지 모르겠음)
<br>

### 리더 선출 및 쓰기 폐기
- 리더에서 발생한 쓰기가 replica로 동기화되지 못하고 리더 다운
- 다운된 리더의 최신 데이터 변경 사항을 가진 replica가 새로운 리더
- 이때 반영하지 못한 쓰기를 단순 폐기하면 고객 만족도 당연히 떨어짐
- 또한 다운된 이전 리더가 활성화되어 다시 리플리케이션 구조에 추가되었을 때 충돌 발생할 수 있음
  - 이전 리더에서 PK 15, 16 생성 / 현재 리더 PK 14 까지 동기화한 상태에서 리더 승격 후 새로운 15, 16 생성
  - https://github.blog/2012-09-14-github-availability-this-week/
## ACID

- Atomicity 원자성
  - 트랜잭션 내 모든 연산이 원자적으로 처리되어 모두 커밋 또는 모두 취소되는 것을 보장
  - 단일 환경: @Transactional (proxy 기반 AOP로 동작)
  - 분산 환경: 한 요청에 대한 여러 트랜잭션으로 분리 (2PC, saga 패턴)
- Consistency 일관성
  - 데이터베이스의 무결성 유지
  - A -> B 이체 시 A 잔고는 줄고, B 잔고는 늘어야 함
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
  - MySQL 해당 격리 수준부터 gap, next key lock 적용되어 쓰기 쿼리나 락을 사용한 읽기 쿼리를 잘못 작성하면 많은 부분이 잠길 수 있음
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
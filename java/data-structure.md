## BBST

Balanced Binary Search Tree
- 트리의 높이를 제한해 O(logN) 시간 복잡도 보장
- 정렬 유지 (왼쪽 자식 < 부모 < 오른쪽 자식)
  - Heap은 부모-자식 간의 정렬 관계만 유지
- red-black tree가 BBST의 한 종류

### 구현체

- c++: set, map, multiset, multimap
- java: TreeSet, TreeMap

<br>

## Linked List

- 리스트 내 모든 요소 포인터로 연결
- 동적 배열 (여러 노드가 메모리 내 연속적이지 않아도 됨)
- double linked list 지만, ArrayList 처럼 특정 지점에 바로 접근 불가
  - ArrayList 특정 지점 조회 O(1)
  - Linked List O(n)

<br>

## ArrayList

- java: ArrayList, cpp: vector
- 동적 배열
- 연속된 메모리 공간 유지
- resize 발생 시 새로운 배열 할당하고 기존 배열 복사 O(n)
- 조회 시간 복잡도 O(1)

<br>

## ConcurrentHashMap
- 동기화된 HashMap 이면서, synchronized 를 사용하지 않아 Hashtable보다 성능 좋음
- 버킷 기반 동기화
- 버킷 단위로 락 획득 == 버킷 단위로 병렬 처리 가능
- 버킷에는 여러 Node 존재, 해시값이 같은 노드뿐만 아니라 해시값이 달라도 같은 버킷에 존재할 수 있음.. 는 것 같음
- 버킷의 여러 노드는 Linked List로 관리하다가 8(TREEIFY_THRESHOLD)개를 넘어가면 트리 구조로 변경
- CAS 연산과 모니터 락(일부 로직)으로 동기화 제공 
- synchronized 키워드는 ConcurrentHashMap만 사용하는 듯
  - Concurrent... LinkedDeque, LinkedQueue, SkipListMap, SkipListSet 코드에 synchronized 키워드 없음

<br>

## Hash

hash 기반 자료구조는 해시 충돌 해소 필요
- HashMap, HashSet, ConcurrentHashMap... 등 모두 정렬되지 않음
- 충돌 발생하지 않으면 조회 O(1), 충돌 발생하면 조회 O(logN) ~ O(N)

### Hash Collision

Separate Chaining
- 하나의 위치에 여러 항목이 저장될 수 있는 방법
- 일반적으로 LinkedList 인데.... java 8 부터 상황에 따라 Red-Black Tree로 변경된 듯 (자바 버전 확실하지 않음)
- 그래서 충돌 발생 시 조회 시간 복잡도가 최악의 경우 O(N) 이었는데, O(logN)로 개선
<br>

Open Addressing
- 분리 연결법은 추가 공간 필요
- 개방 주소법은 추가 공간 없이 주어진 테이블에서 무조건 해결하는 방식
- 충돌 발생 시 선형 탐사법, 제곱 탐사법, 이중 해싱으로 해결
<br>

선형 탐사법
- 한 칸씩 옮겨가면서 빈 버킷 찾기
- 1차 군집 현상에서 성능 저하될 수 있음
- 1차 군집 현상: 특정 영역에 키가 몰려있는 경우
<br>

제곱 탐사법
- h(x + i^2) mod m
- 1차 군집 현상이 발생해도 그 영역을 빠르게 벗어날 수 있음
- 그러나 2차 군집 현상에서 성능 저하
- 2차 군집 현상: 여러 원소가 동일한 초기 해시값을 가지면 같은 순서로 조사
- 개발할 때도 2차 군집 현상을 고려해 어떤 기준으로 어떻게 재시도할 것인지 고려해야 함
- 그렇지 않으면 재시도로 인한 네트워크 혼잡도 증가할 수 있음
<br>

이중 해싱
- 충돌 발생 시 보조 해시 함수 사용
- (h(x) + i*f(x)) mod m
- 추가 해시를 사용해 다른 방식에 비해 많은 연산량 요구

<br>

### Bloom & cuckoo Filter

- https://redis.io/docs/latest/develop/data-types/probabilistic/bloom-filter/
- https://brilliant.org/wiki/cuckoo-filter/
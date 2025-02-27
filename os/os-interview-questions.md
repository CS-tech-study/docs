## 프로세스, 스레드 생성

프로세스 생성
- 부모 프로세스에서 fork() 호출하면 자식 프로세스 생성
- PCB 할당
- 부모 프로세스의 코드 영역을 제외한 모든 메모리 영역 Copy-on-Write 기법
  - CoW 기법: 처음부터 메모리 영역을 복사하는 것이 아니라, 페이지 테이블 공유하다가 수정 시 복사본 생성
- 코드 영역은 공유 (코드 영역 수정되지 않음)

스레드 생성
- TCB & 개별 스택 영역 할당
- 나머지 영역인 코드, 힙 영역 공유

<br>

## 스레드 상태

- waiting: 다른 스레드의 시그널을 기다리는 상태 (wait(), join() 호출)
- timed_waiting: 주어진 시간동안 대기 (sleep(), wait(timeout), join(timeout) 호출)
- blocked: 사용하고자 하는 객체의 락 해제를 대기 (synchronized 블록 진입, ReentrantLock tryLock)
- suspend
  - 보류 상태는 프로세스 단위에 해당, 스레드 단위에 적용 불가
  - suspend 되었다는 것은 프로세스가 Swap 영역으로 방출되어 실행 불가능한 상태
  - 프로세스가 swap 영역으로 방출되면 모든 스레드도 함께 swap
  - swap 영역 방출은 page 단위로 발생하므로 스레드마다 suspend 상태 적용 불가

<br>

## Page Table
page, page frame 단위를 줄이면 내부 단편화를 더 줄일 수 있지만, 페이지 테이블의 부담 증가
- 한 프로세스의 여러 page가 연속된 page frame에 할당된다는 보장 없음. page table로 위치 관리
- 프로세스마다 페이지 테이블 소유
- 페이지 테이블도 페이지 프레임에 할당 (PTE(Page table entry)는 연속적)

페이지 테이블에는 각 페이지에 대한 프레임 시작 주소 (base address) 저장
- 논리 주소는 페이지 번호와 오프셋으로 구성
- 페이지 번호는 페이지 테이블 인덱스
- 오프셋은 프레임 내 상대적 위치
- 논리 주소의 페이지 번호를 가지고 페이지 테이블에서 프레임의 시작 주소 찾기
- 프레임의 시작 주소에서 offset 만큼 떨어진 곳이 실제 데이터가 있는 물리 주소

주소 변환은 MMU 로 처리되며
- TLB(Translation Lookaside Buffer)를 통해 PTE 캐싱 제공
- 자주 사용되는 PTE 캐싱하고, 다시 접근 시 캐시에서 물리 주소 제공해 주소 변환 과정 생략

<br>

## Paged Segmentation

- 내부 단편화: 고정 분할에서 생기는 낭비 (Linux 기준 4kb 미만 발생)
- 외부 단편화: 동적 분할에서 생기는 낭비
- paging 기법: 프로세스는 Page(4KB) 단위, 메인 메모리는 Page Frame(4KB) 단위로 분할하며 외부 단편화는 막고 내부 단편화는 4KB 미만으로 발생
- Segment: **연관된 기능**을 수행하는 하나의 모듈 프로그램을 다루며 Code, Data, Stack, Heap, 서브루틴, 프로시저, 함수 등을 Segment라고 함
  - 정적 세그먼트: Code, Data(+BSS)는 컴파일 시 사이즈가 정해지며 변경이 불가능
  - 동적 세그먼트: Stack, Heap은 Runtime 과정에서 메모리 할당이 이루어짐
- 현재 사용하는 방식이 **Paged Segmentation**으로 **먼저 Segmentation을 수행하고 각 Segment 별 Paging 수행**

<br>

## 데드락

- 서로가 락을 해제해주길 기대하는 교착 상태
- MySQL 데드락 발생 시 자동 감지해 롤백 처리
- Application 단에서 dao.CannotAcquireLockException로 핸들링
- 해결책으로 ordered locking, 트랜잭션 분리
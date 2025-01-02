## 유저 모드, 커널 모드

프로그램으로부터 OS 자원을 보호하기 위해 모드 분리

### 모드 전환되는 시점
interrupt, system call 발생하면 유저 모드에서 커널 모드로 전환
- 주어진 time quantum (time slice) 모두 사용
- interrupt: page fault
- system call: ```open()```, ```read()```, ```fork()``` 호출

### 유저 모드 -> 커널 모드 동작 방식
- 현재 상태 TCB에 저장 (running -> waiting)
- Interrupt vector table에서 해당 시스템 콜에 대한 처리 루틴 주소(system call table) 찾고 (or Interrupt Descriptor table)
- system call handler를 통해 system call table의 특정 위치의 함수 수행
- **시스템 콜 핸들러가 모든 작업 수행하면 TCB에 저장된 정보를 레지스터에 복원해 유저 모드로 전환** (waiting -> running)

<br>

## 문맥 전환 작업

### 커널 스레드 기준
- TCB에 CPU 레지스터 저장
- **메모리 전환이 발생하지 않아** 프로세스 컨텍스트 스위칭보다 빠름

### 프로세스 기준
- PCB에 CPU 레지스터 값 저장
- memory context: 가상 주소를 물리 주소로 변환하기 위해 필요한 정보
  - page table를 PCB에 저장
  - TLB(Translation Lookaside Buffer) 초기화
- System context: HW, memory 제외한 나머지 부분

<br>

## KLT(1) : ULT(1)
- **JVM은 KLT와 ULT가 일대일 구조**
- 기존 자바 스레드는 KLT를 wrapping 한 것이므로 무한 생산 불가능
- 스레드 스케줄링 OS가 결정
- ULT block 되면 KLT도 block
- **ULT에 인터럽트 발생하면 KLT 문맥 전환 발생 (이때 유저 모드 -> 커널 모드 전환)**

### jdk 21 virtual thread
- 커널 스레드와 플랫폼 스레드(ULT)가 일대일 관계
- 플랫폼 스레드와 가상 스레드가 일대다 관계
- 가상 스레드 스케줄링 JVM이 결정
- 플랫폼 스레드에 여러 가상 스레드를 mount/unmount 하지만, 커널 스레드의 block으로 이어지지 않고 다른 가상 스레드를 mount 하여 계속 작업 진행

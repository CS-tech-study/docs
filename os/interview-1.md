## 가상 주소 공간이 물리 주소 공간보다 클 수 있는 이유

OS는 메모리 관리를 용이하게 하기 위해 메모리를 page 단위로 관리, **page directory라는 자료구조에 가상 page number가 물리 page number와 매핑**이 되어있어 가상 주소를 물리 주소로 변환

<img width="561" alt="스크린샷 2025-02-20 오전 12 24 57" src="https://github.com/user-attachments/assets/d3de0238-e445-4400-8d48-d120defe5240" />


이때, 가상 주소가 메모리 주소만 가리킬 수 있는게 아니라 디스크의 swap space를 가리킬 수도 있음
이러한 **page directory는 kernel space에 저장**되어 있음

## **segmentation**

주소 공간 전체를 하나의 연속적인 부분으로 메모리에 할당하지 말고, **code, stack, heap segment**를 나누어서 각 segment별로 base, bound register를 가지도록 함
>base register: 프로세스의 주소 공간이 물리 메모리에서 시작하는 위치를 저장하는 레지스터
>bound register: 프로세스의 주소 공간 크기 또는 유효한 메모리 범위를 저장하는 레지스터

**각 segment가 다른 물리적 메모리에 위치하여 메모리 낭비를 줄임**

**각 segment는 물리적 메모리에 개별적으로 위치할 수 있고, 커지고 작아질 수 있으며, 개별적으로 read/write/execute protection bit를 가질 수 있음**

## paging

segmentation은 프로세스의 주소 공간을 좀 더 잘게 쪼갠 것일 뿐 각 세그먼트는 결국 연속적이어야 하므로 주소 공간을 아예 고정된 단위로 잘게 나눈 방식이 paging

장점

-  **유연성 :** 프로세스는 실제로 어떻게 주소 공간을 사용하는 지 알 필요가 없음(그냥 고정된 사이즈의 청크기 때문에 stack인지 heap인지 등 알필요 없음, 주소 공간을 효과적으로 추상화

- **단순성 :** 가상 메모리와 물리 메모리를 둘 다 동일한 크기의 블록으로 나누어 관리하기 때문에 가상 메모리의 페이지를 물리 메모리의 프레임에 **그대로 대응**


가상 주소를 물리 주소로 변환할 때 사용하는게 페이지 테이블

<img width="317" alt="스크린샷 2025-02-20 오전 1 46 23" src="https://github.com/user-attachments/assets/4432440a-59ca-41cc-88f9-99d62cd4811f" />

<img width="517" alt="스크린샷 2025-02-20 오전 1 54 15" src="https://github.com/user-attachments/assets/01fd54c8-99b3-4d05-b606-ccd634a9bc3a" />


present bit가 0이면 페이지가 물리 메모리에 없다는 뜻으로 page fault 발생시킴
>물리 메모리에 공간이 있으면 프레임을 로드하고, 없으면 page replacement 알고리즘에 따라 물리 메모리를 디스크로 내리고 로드

주소 변환을 위해선 페이지 테이블 참조가 반드시 선행되어야 하므로 이 페이지 테이블 참조에 따른 성능 저하를 해소하기 위해 존재하는게 TLB(**translation lookaside buffer**)
>메모리에 위치하는 페이지 테이블과 달리 MMU에 캐싱
>모든 가상 메모리 참조는 **우선 TLB를 확인하여** 캐시 히트인 경우 이를 사용, 없다면 페이지 테이블 참조 후 TLB 업데이트

## 페이지 테이블의 문제점

page table을 유지하는 것만으로 너무 많은 메모리를 사용

>페이지의 크기가 4KB라고 하면 2^32/2^12 = 2^20, page table entry 하나당 4바이트를 사용한다고 가정하면 2^20 * 2^2 = 2^22, **page table**만으로 4MB**를 사용

**Page size를 키우면 page table의 entry 수가 줄으므로 테이블 사이즈는 줄일 수 있지만** 이렇게 하면 작은 메모리가 필요한 경우에도 큰 페이지를 할당하게 되어 낭비가 큰 **internal fragmentation 문제**가 발생

 ## **Multi-level page table**

- 페이지 테이블을 계층적으로 구성하여 메모리 사용을 효율화
- 페이지 테이블을 여러 개의 페이지 크기 단위로 분할하고, 각 단위를 **Page Directory**를 통해 관리

일단 page table을 일정 단위로 자르고 만약 해당 단위의 모든 페이지가 invalid하다면 그냥 page table을 위한 page를 할당하지 않음


<img width="531" alt="스크린샷 2025-02-20 오전 2 09 15" src="https://github.com/user-attachments/assets/702faadf-0593-4200-9c15-82216eb2c280" />


왼쪽의 linear page table의 경우 page가 할당되지 않아도(매핑된 물리 프레임이 없어도) page table entry들을 위한 메모리를 할당했어야 했는데 multi-level page table의 경우 page table 별로 만약 valid한 page가 없다면 page table을 위한 page를 아예 할당하지 않음(대신 page directory에서 invalid로 표시)

- **Page table을 위한 page 하나당 하나의 page directory entry를 가짐**
-  **PDE는 PTE와 마찬가지로 valid bit과 page frame number를 가짐**
- **PDE가 valid하다는건 page table의 entry 중 적어도 하나는 valid하다는 것을 의미(매핑된 물리 프레임이 있다는 의미)**
## 제어 흐름 Control Flow 

프로그램이 명령어나 함수 호출을 순차적으로 실행할 때 따르는 **일반적인 실행 경로**

**1)** **CPU가 메모리(ex. Volatile DRAM)에서 데이터를 긁어 Data Bus를 통해 가져온다. (Fetch)**  
**2) 그리고 이 데이터를 CPU의 Instruction Register 저장공간에 저장한다.**  
**3) CPU의 Program Counter는 '그 다음'의 명령이 담긴 주소를 가리키도록 조정된다. (PC++)**  
**4) CPU는 레지스터에 든 명령을 Decode하고 Execute한다. (Decode & Execute)**  
**5)** 하드웨어적으로 **Clock이 뛰면서 이 과정이 계속해서 반복**된다.

https://velog.io/@junttang/SP-1.1-%EC%98%88%EC%99%B8-%EC%B2%98%EB%A6%AC-%ED%9D%90%EB%A6%84-%EC%98%88%EC%99%B8

>위 1~5까지의 과정을 control flow라고 함

## 예외 처리 흐름(Exceptional control flow)

위 control flow 흐름이 아닌 **예외 상황**(에러나 비정상적인 조건)이 발생했을 때 실행되는 경로

**Low level mechanism**

- **Exception :** 시스템 이벤트에 의해 제어 흐름이 바뀌는 상황, 하드웨어와 OS 소프트웨어를 사용하여 구현됨

**Higher level mechanism**

- **Process context switch :** OS 소프트웨어와 하드웨어 타이머로 구현
- **Signals :** OS 소프트웨어에 의해 구현
- **Nonlocal jumps :** setjmp(), longjmp()  
  

**Exception**

특정 이벤트로 인해 컨트롤이 OS 커널로 넘겨지는 행위
> 0으로 나누는 경우, 산술 오버플로우, page fault, IO request complete, ctrl + c 등


Exception Table

- exception table의 주소는 특수한 CPU 레지스터인 exception table base register에 저장
- 하드웨어가 exception을 트리거 하면 나머지 일은 소프트웨어인 exception handler에서 수행
- 각 이벤트는 exception table의 인덱스 역할을 하는 exception number가 존재
> exception k가 발생하면 index k로 가서 이에 저장되어 있는 핸들러 k가 호출**됨
- 핸들러의 실행은 커널 모드에서 실행됨, 즉 user mode -> kernel mode 전환이 발생
- 핸들러 호출 후 원래 코드 흐름으로 돌아와야 하니까 함수 호출처럼 리턴 주소를 스택에 push하고 핸들러 호출 + 원래 코드 흐름으로 복귀 시 필요한 정보들도 스택에 push -> 핸들러 종료 후 이를 스택에서 pop하면서 user mode로 복귀

<img width="569" alt="스크린샷 2025-02-20 오전 12 44 23" src="https://github.com/user-attachments/assets/a186f27d-0075-4598-912a-7878f541cf4f" />


예외 흐름이 외부에 의해 발생 vs 프로그램 내부 명령어의 실행 결과로 발생으로 
비동기적(인터럽트), 동기적(trap, fault, abort)가 나뉨

- Interrupt: 명령어로 처리되는게 아닌 **외부 하드웨어 이벤트**로 인해 발생
>외부 이벤트에 의해 발생되므로 **프로그램의 실행 흐름과 무관하게 언제든지 발생**할 수 있기 때문에 비동기적이라고 하는데, 아마 프로그램의 순차적 흐름과 별개로 예상할 수 없는 타이밍에 실행될 수 있다는 의미에서 비동기라고 하는거같음...

<img width="422" alt="스크린샷 2025-02-20 오전 12 55 58" src="https://github.com/user-attachments/assets/41d0ebce-375f-42b7-9528-eee6a760e428" />


CPU 내부의 인터럽트 pin이 high(외부 하드웨어가 전기적 신호를 사용해서)가 되면 현재 명령어 수행을 마치고 인터럽트 핸들러를 실행

>timer interrupt(컨텍스트 스위칭), IO interrupt(키보드, 네트워크 패킷 도착, disk에서 데이터 도착 등)

- trap: **System call**을 통해 사용자에게 제어된 커널 서비스 접근을 제공

- fault: 핸들러가 조치를 취할 수 있을 만한 에러 상황 발생 시 발생, fault가 발생하면 프로세서는 제어권을 fault handler에게 넘기고 핸들러가 **에러 상황을 고칠 수 있으면 제어권을 fault를 발생시킨 명령어로 이동하여 재실행, 고칠 수 없다면 fault를 발생시킨 프로그램을 종료시키는 abort 수행**
>page fault exception, protection faults (i.e., segmentation fault), floating point exceptions (i.e., divide by zero)

trap, fault는 **프로그램 내부 명령어의 실행 결과로 인해** 발생하는 예외 흐름으로 프로그램의 순차적 흐름에 의한 예상 가능한 상황이므로 (프로세서 입장에선 언젠간 일어날 일이 일어난 것일 뿐) 동기적



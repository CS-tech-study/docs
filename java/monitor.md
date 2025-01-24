# Monitor

monitor란 공유 자원을 내부적으로 숨기고 공유 자원에 접근하기 위한 인터페이스만 제공해 내부에서 알아서 동시성 이슈를 차단
- synchronized 키워드를 사용하면 JVM이 알아서 해당 객체에 Monitor 메커니즘으로 동시성을 제어
- 자바 객체의 mark word를 사용해 동기화 상태 관리 https://stackoverflow.com/questions/26357186/what-is-in-java-object-header
- 단일 JVM 내에서만 보장
- blocking 방식

## synchronized 구현 예시

```java
public class Account {

   private int balance = 0;  // 공유 자원
   public synchronized void withdraw(int money) {  // 프로시저
	balance -= money;
   }
   public synchronized void deposit(int money) {   // 프로시저
	balance += money;
   }
}
```
- 공유 자원에 접근하기 위한 여러 프로시저가 존재해도
- 특정 프로시저가 동작 중이라면 다른 프로시저에 접근 불가 -> blocking & 동시성 문제 차단 
- 스레드 상태 block 전환 시 사용자 모드 -> 커널 모드 전환 비용 발생 [링크 추가 예정]
<br>

- **blocking 방식인 synchronized 키워드는 실무에서도 거의 사용되지 않는다**고 피드백받음
- java.util.concurrent.atomic 패키지로 대체!

<br>

## Entry set vs Wait set

Entry Set
 - 모니터의 lock을 획득하기 위한 스레드가 대기하는 자료구조
 - lock이 반납될 때까지 entry set에서 대기하다가 -> 락이 반납되면 entry set에서 대기하던 스레드 중 하나가 락을 획득하고 모니터(임계 영역)에 진입

Wait set
- 모니터의 조건 변수와 함께 사용되는 공간으로, 특정 조건에 대한 스레드가 대기
- 모니터의 condition이 해당 wait set의 조건을 만족하면, 해당 wait set으로 신호를 보내 대기 중인 스레드 중 하나를 깨워 entry set으로 이동

<br>

## Condition Variables

- ```await()```, ```signal()```, ```signalAll()``` -> Condition 인터페이스 메서드
- ```wait()```, ```notify()```, ```notifyAll()``` -> 더 낮은 수준 (Object class)
- 

<br>

## Mesa vs Hoare Style Monitor

Monitor 방식은 Mesa, Hoare 스타일이 있다.

Mesa style (Signal and Continue)
- Lock을 잡고 있으면서 ```notify()``` 호출한 스레드는 ```notify()``` 호출 동시에 Lock을 반납하지 않음
- wait set에서 ```notify()``` 호출로 깨어난 스레드는 모니터에 바로 접근하는 것이 아니라, entry set으로 이동해 다시 대기 (sleep 상태 유지)
- Runnable 상태로 바로 전환하지 않고, sleep 상태를 유지하는 이유는 해당 스레드가 CPU를 선점했지만, Lock이 아직 해제되지 않았다면 결국 다시 대기해야 함
- ```notify()``` 호출로 깨어난 스레드는 모니터로 바로 접근하는 것이 아니므로 condition이 깨질 수 있음 (sleep 상태 유지하다가 -> runnable 전환)
- JVM이 사용하는 방식 (+ 대부분의 OS도 이 방식 사용)

Hoare style (Signal and Wait)
- 조건별 wait set, ```wait()```, ```notify(```) 존재
- ```notify()``` 호출로 깨어난 스레드는 모니터에 바로 진입 -> 공평성 문제 발생

> Mesa style == Signal and Continue... Hoare style == Signal and Wait ...?
> - 완벽하게 같다고 할 순 없다...
> - Mesa/Hoare는 설계 철학과 기본적인 동작 규칙 등의 방법론적인 의미
> - Signal and Continue/Wait는 특정 조건에서의 실행 흐름을 다루는 구현 세부 사항

### Spurious wakeup

Spurious wakeup 현상이 발생할 수 있기 때문에 임계 영역에 들어가기 전 다시 한 번 상태 확인을 한다.
- Spurious wakeup이란 signal을 하지 않았음에도 스레드가 깨어나는 현상 (or 한 번의 signal로 여러 스레드가 깨어나는 현상)
- 실제로 스레드 스케줄러도 하드웨어, 소프트웨어의 비정상적 동작으로 일시적인 블랙아웃을 겪을 수 있다.
- 스케줄러의 블랙아웃동안 대기 중인 스레드에게 전달할 신호롤 놓치면 스레드가 영원히 대기 상태에 빠질 수 있고, 이를 방지하기 위해 스케줄러는 대기 중인 모든 스레드를 깨우는 방법을 사용

<br>

## 메모리 가시성

synchronized 키워드, java.util.concurrent.atomic 패키지를 사용해 메모리 가시성 보장 가능
- 메모리 가시성이란 멀티스레드 환경에서 한 스레드가 변경한 데이터를 다른 스레드에서 즉시 볼 수 있는지에 관한 개념

### synchronized 키워드 
- synchronized 블록에 진입하기 전 스레드의 로컬 뷰는 메인 메모리와 동기화하여 로컬 뷰에 저장된 변숫값이 메인 메모리의 최신 값인지 확인
- synchronized 블록을 빠져나오기 전 로컬 뷰에서 변경된 변숫값을 메인 메모리에 반영

### java.util.concurrent.atomic 패키지
- 내부에서 CAS 알고리즘과 volatile 변수를 사용해 메모리 가시성 보장
- Lock을 사용하지 않아 lock-free or non-blocking 동기화라고도 함
- incrementAndGet, decrementAndGet 메서드를 사용하면 ```compareAndSetInt()``` 를 통해 메인 메모리에 있는 값을 변경

```java
@IntrinsicCandidate
public final native boolean compareAndSetInt(Object o, long offset,
                                            int expected,
                                            int x);
```
- native 키워드를 사용해 CPU 캐시가 아닌 메인 메모리 값을 직접 변경
- 하드웨어 수준의 atomic 연산을 보장하니 lost update 같은 문제 차단

### volatile
- volatile 변수도 ```compareAndSetInt()``` 처럼 값을 메인 메모리에서 가져오지만, 
- 로우 레벨의 여러 명령어가 재정렬 없이 실행하는 것이지 로우 레벨의 여러 명령어가 원자적으로 처리됨을 보장하지 않음
- volatile 변수만 사용한다고 동시성 문제를 해결할 수 없음
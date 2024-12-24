# JVM Monitor

## **모니터란?**

공유 자원을 내부적으로 숨기고 공유 자원에 접근하기 위한 인터페이스만 제공함으로써 자원을 보호하고 프로세스 간에 동기화를 시킨다. 이때 이 인터페이스를 모니터라고 한다.

### 기능

- 임계구역으로 지정된 변수나 자원에 접근하고자 하는 프로세스는 직접 P()나 V()를 사용하지 않게 모니터에 작업 요청을 한다.
- 모니터는 **조건 변수**와 **임계구역 관리**를 통해 프로세스 간의 동기화를 보장한다.

> #### EntrySet
> 모니터의 Lock을 획득하기 위해 대기 중인 스레드를 모아 놓은 자료구조.
> 어떤 스레드가 Lock을 사용 중일 때, 나머지 스레드는 EntrySet에 대기한다.
> EntrySet에 있는 스레드들은 Lock이 반납될 때까지 기다리며, 락이 반납되면 스레드 중 하나가 선택되어 락을 획득하고 임계 영역으로 진입한다.

> #### WaitSet
> 모니터의 조건 변수와 함께 사용되며, 스레드가 특정 조건이 만족될 때까지 대기하는 자료구조.
> 스레드는 WaitSet에 들어가 대기할 때 Lock을 해제한다.
> 다른 스레드가 조건을 만족시켜 신호를 보내면, 스레드는 EntrySet으로 이동해 Lock을 재획득하려 시도한다.


## 조건 변수

스레드 간 협력을 가능하게 하여 공동의 목표를 달성하고 데이터의 일관성과 안전성을 보장하는 동기화 메커니즘. 

### Java에서의 조건 변수

`Condition` 인터페이스의 메서드인 `await()`, `signal()`, `signalAll()`을 사용하여 구현되며, 낮은 수준에서는 `Object` 클래스의 `wait()`, `notify()`, `notifyAll()` 메서드로도 조건 변수의 동작을 흉내낼 수 있다.
- 특정 조건이 만족되지 않을 때 = 스레드는 `await()` 메서드를 호출하여 조건 변수의 대기 큐에 들어가 대기한다.
- 다른 스레드가 조건을 만족시키면 = `signal()` 를 호출하여 대기 중인 스레드 중 하나를 깨우거나, `signalAll()` 을 호출하여 모든 대기 중인 스레드를 깨운다.

### Signal and Wait & Signal and Continue

`notify()` 메서드를 실행한 후, 하나의 모니터를 두고 어떤 스레드가 먼저 소유할 것인가에 따라 두 가지 조건 변수 방식으로 나눌 수 있다. 

- Signal and Wait
    1. 현재 모니터를 소유하고 있는 스레드가 `wait()`을 실행 → 모니터 내부에서 스레드는 자신을 일시 중단하고 Lock을 해제한 후 WaitSet에 들어간다.
    2. 깨우는 스레드가 `notify()` 또는 `notifyAll()` 명령을 실행 → Wait Set에 있는 대기 스레드 중 하나 또는 모든 스레드를 깨운다.
    3. 깨우는 스레드는 자신의 작업을 완료한 후 Lock을 해제 → 대기 중인 스레드가 Lock을 획득한 후 모든 작업을 마치고 Lock을 해제한다. 이 시점에서 깨운 스레드가 Lock을 획득하여 작업을 이어나간다.
- Signal and Continue
    1. 현재 모니터를 소유하고 있는 스레드가 `wait()`을 실행 → 모니터 내부에서 스레드는 자신을 일시 중단하고 Lock을 해제한 후 WaitSet에 들어간다.
    2. 깨우는 스레드가 `notify()` 또는 `notifyAll()` 명령을 실행 → WaitSet에 있는 대기 스레드 중 하나 또는 모든 스레드를 깨운다. 이때 깨어난 스레드들은 EntrySet으로 이동
    3. 깨우는 스레드는 Lock을 계속 유지하면서 모든 작업을 완료한 후 Lock을 해제 → Lock이 해제되면 EntrySet에 대기하고 있는 스레드들 중 하나가 Lock을 획득하여 작업을 이어나간다.
    
    자바에서는 이 조건 변수 형식을 사용한다.
    

---

# Mesa vs. Hoare

운영 체제에서 모니터를 구현하는 두 가지 방식.

threadA가 threadB를 깨웠을 때, A가 계속 실행되어야 할까? 아니면 A가 멈추고 B가 실행되어야 할까?
전자가 mesa style semantic이고, 후자가 hoare style semantic.

## Mesa

- 깨운 스레드(A)가 락과 CPU를 계속 소유하며, 깨어난 스레드(B)는 준비 큐에 삽입된다.
- 대기 중이던 스레드는 깨어난 후에도 다른 스레드가 락을 획득하고 항목을 제거할 수 있으므로 다시 대기해야 할 수도 있다.
- 큐와 상태 확인 비용으로 인해 메모리 사용 증가
- B가 바로 깨어나지 않기 때문에, 대기하는 동안 다른 thread에 의해 condition이 깨질 가능성이 존재한다.
    
    ⇒ 깨어났을 때 다시 한 번 condition을 체크해야 하기 때문에 구현 시에 `while`을 사용
    
- JVM도 이 방식을 사용 (대부분의 OS도)*

```java
while (queue.isEmpty()) {
    condition.await();
}
```

## Hoare

- 깨어난 스레드(B)에게 락과 CPU를 모두 주고, 깨운 스레드(A)는 sleep한다.
- 대기 중이던 스레드는 큐에 항목이 추가된 직후 즉시 실행된다.
- 간결한 관리로 메모리 사용이 적다.
- B가 바로 일어난다
    
    ⇒ 다시 condition을 체크할 필요가 없어 구현 시에 `if`만 사용
    

```java
if (!queue.isEmpty()) {
    // 조건이 만족되었으므로 바로 작업 실행
}
```

### *왜 Mesa 방식을 사용할까? = **spurious wakeup**

signal을 하지 않았음에도 불구하고 thread가 깨어나버리는 현상을 말한다.

만약 `if`를 써버리는 바람에 멋대로 깨어난 thread가 진짜 멋대로 데이터에 접근할 위험이 있다.

### 잠깐!

이거 Hoare = Signal and Wait, Mesa = Signal and Continue 아닌가?

(엄밀히는 다르지만) 맞다. 

Mesa/Hoare는 설계 철학과 기본적인 동작 규칙 등의 방법론적인 의미, Signal and Wait/Continue는 특정 조건에서의 실행 흐름을 다루는 구현 세부사항이라고 할 수 있다.

---

참고자료

https://devdebin.tistory.com/335

https://samuel-sorial.hashnode.dev/mesa-vs-hoare-semantics

[https://rntlqvnf.github.io/lecture notes/os-4th-1/](https://rntlqvnf.github.io/lecture%20notes/os-4th-1/)

https://velog.io/@richpin/OS-Condition-Variable
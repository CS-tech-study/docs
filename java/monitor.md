## sychronized 

Synchronized 키워드를 사용하면 JVM이 알아서 해당 객체에 Monitor 메커니즘으로 동시성을 제어한다.
- 자바 객체의 mark word를 사용해 동기화 상태 관리 https://stackoverflow.com/questions/26357186/what-is-in-java-object-header
- 하지만 단일 JVM 내에서만 보장
- blocking 방식


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
위와 같이 synchronized 키워드를 추가하면 JVM이 알아서 동기화한다.
- 공유 자원에 접근하는 메서드를 프로시저라고 하는데
- 여러 프로시저가 존재해도 특정 프로시저가 동작 중이라면 다른 프로시저에는 접근이 불가능해 동시성 문제를 해결한다.

특정 프로시저가 동작 중이라면 다른 프로시저에는 접근이 불가능해 이를 blocking 동기화라고도 한다.
- Blocking 동기화는 결국 성능 문제로 이어진다.
- Lock을 가진 스레드가 반납할 때까지 모든 스레드가 block -> run 상태로 변경해 Lock을 획득하는 과정에서 비용 발생
- *스레드 상태 변화가 왜 큰 비용? 유저-커널 모드 전환 발행*

이는 CAS 연산으로 non-blocking 동기화를 제공하는 java.util.concurrent.atomic 패키지를 사용해 해결할 수 있다.

<br>

## package java.util.concurrent.atomic

java.util.concurrent.atomic 패키지는 update 연산을 atomic하게 처리하는데 Lock을 사용하지 않고 CAS 알고리즘을 사용한다. 그래서 lock-free 또는 non-blocking 동기화라고 한다.

```java
AtomicInteger count = new AtomicInteger(0);

Thread producer = new Thread(() -> {
   for(int i = 0; i < 1000000; i++) count.incrementAndGet();
});

Thread consumer = new Thread(() -> {
   for(int i = 0; i < 1000000; i++) count.decrementAndGet();
});

producer.start();
consumer.start();

producer.join();
consumer.join();

System.out.println("COUNT: " + count.get());
```

위 코드를 테스트해보면 count는 0 인데, AtomicInteger가 내부에서 CAS 알고리즘을 사용하고 있기 때문
- 기존 값과 변경할 값을 같이 전달
- 기존 값이 현재 메모리에 있는 값과 같으면 변경할 값을 반영하고 true 리턴
- 그렇지 않다면 false 리턴

incrementAndGet, decrementAndGet 메서드를 사용하면 compareAndSetInt()를 통해 메인 메모리에 있는 값을 변경한다.

```java
@IntrinsicCandidate
public final native boolean compareAndSetInt(Object o, long offset,
                                            int expected,
                                            int x);
```

native 키워드를 사용해 CPU 캐시가 아닌 메인 메모리 값을 직접 변경하며, 하드웨어 수준의 atomic 연산을 보장하니 lost update 같은 문제를 방지한다. 

그럼 메인 메모리 값을 변경하니 'volatile 키워드만 추가하면 되지 않나'라고 생각할 수 있는데 volatile 변수를 사용한 다음 코드를 테스트해보면 실행할 때마다 다른 값으로 출력된다.

```java
static volatile int count = 0;

public static void main(String[] args) throws InterruptedException {
   
    Thread producer = new Thread(() -> {
        for(int i = 0; i < 1000000; i++) count++;
    });
    Thread consumer = new Thread(() -> {
        for(int i = 0; i < 1000000; i++) count--;
    });
```
volatile 변수는 값을 메인 메모리에서 가져오고 로우 레벨의 여러 명령어가 재정렬 없이 실행되는 것이지, 로우 레벨의 여러 명령어가 원자적으로 처리됨을 보장하지 않는다.

#### 명령어 재정렬

```text
- 명령어 1: a에 3 더하기
- 명령어 2: a에 6 곱하기
- 명령어 3: b에 2 더하기
```
- 명령어 1, 2는 재정렬할 수 없지만, 명령어 3은 어느 시점에 실행되어도 결과가 달라지지 않음
- 그러므로 명령어 1과 명령어 3은 서로 다른 명령어 처리 유닛으로 처리해 프로그램이 지정한 순서와 다르게 처리할 수 있음
- volatile 변수를 사용하면 프로그램이 지정한 순서대로 처리 (기억상 lock addl 명령어가 붙어 이전 작업이 순서대로 모두 처리되었음을 보장한다... 고 배웠는데 관련 링크 없음)

그래서 결론은 volatile 변수만 사용한다고 동시성 문제를 해결할 수 없으며, java.util.concurrent.atomic 패키지에서 제공하는 AtomicInteger, AtomicBoolean 등 여러 클래스도 volatile 변수와 CAS 연산을 함께 사용해 동시성 문제를 방지한다.

<br>

## Mesa Style Monitor

JVM에서는 Monitor 방식으로 동기화를 제공하는데 Mesa 스타일과, Hoare 스타일 모니터로 나눌 수 있다.

Entry set에서 대기 중인 단 1개의 스레드만 모니터에 접근할 수 있다.
- 모니터에 진입한 스레드는 notify() 를 호출해 wait set에 대기하는 스레드 깨움
- 이때 signal을 받은 스레드가 Lock을 바로 획득하는 것이 아니라 Entry set으로 이동
- Lock을 반납하면 Entry set에 대기 중인 스레드 중 하나를 깨워 작업 시작
- but, wait set 1개

여기서 고민해볼 점은 'Entry set에서 깨운 스레드가 공유 자원을 사용할 수 있는 상태'인가이다. condition을 만족해 깨웠지만, 모니터에 바로 진입하는 것이 아니라서 condition이 깨질 수 있다.

<br>

## Hoare Style Monitor

Mesa Style Monitor에서는 condition이 깨진 의미 없는 스레드가 공유 자원에 접근하는 과정에서 쓸데없는 비용이 발생한다는 문제가 발생한다. 이는 Hoare Style Monitor 방식으로 해결할 수 있다.

Hoare Style Monitor는 조건별 wait set, wait(), notify() 이 존재한다.
- Wait Set에서 깬 스레드가 Entry Set으로 이동하지 않고 Lock을 바로 획득하는 방식
- condition을 만족한 스레드를 깨워 Lock 권한을 바로 주니 while 문을 condition을 확인할 필요없음

하지만 공평성 문제로 Mesa style monitor 방식을 더 많이 사용한다.

<br>

## Explicit Lock (java.util.concurrent.locks)
Mesa Style Monitor를 사용하면 의미 없는 스레드가 깨어날 가능성이 있어 Hoare Style Monitor로 해결했다. 하지만 Hoare Style Monitor는 특정 Wait Set이 Lock을 독점하는 문제가 발생해 공평성 문제가 발생할 수 있는데 이는 java.util.concurrent.locks 패키지를 사용해 해결한다.
- 조건을 충족한 스레드를 깨워 Entry Set으로 보내므로, 의미 없는 스레드를 깨우는 Mesa Style Monitor 단점 해결
- 깨운 스레드에게 바로 Lock을 주는 것이 아니라 Entry Set에 있는 스레드 중 1개에게 Lock 권한을 전달해 Hoare 스타일 모니터 방식보다 공평하게 Lock 획득할 가능성 있음

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;


public class Account {

   private final Lock lock = new ReentrantLock();
   private final Condition producerCondition = lock.newCondition();
   private final Condition consumerCondition = lock.newCondition();
   private static final int MAX_COUNT = 3;
   private int count = 0;


   public void produce() throws InterruptedException {
       lock.lock();

       try {
           while (count == MAX_COUNT) {
               producerCondition.await();
           }
           count++;
           consumerCondition.signal();
       }finally {
           lock.unlock();
       }
   }
```
- 생산자는 자원이 가득 차 있을 때 producer 전용 condition.await를 호출해 wait set에 대기
- 소비자 스레드가 producer 전용 condition.signal을 호출해주면 대기 중이던 producer를 깨워 entry set으로 보내고 producer가 자원 생산
- 자원이 채워졌으니 consumer 전용 condition.signal을 호출해 consumer를 깨워 entry set으로 보냄



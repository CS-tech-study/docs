# Redisson

- 사용 목적: 분산 환경에서 Lock을 획득해 작업을 제어하기 위해 사용

### 간단한 흐름
- getLock으로 Redisson Lock 사용을 위한 클라이언트 객체를 만들고
- tryLock으로 RLock 획득을 시도하는데
- 만약 획득에 실패하면 wait time 동안 Redisson Lock 획득을 retry

아래 내용은 RLock 획득하지 못하고, Retry 하는 과정 설명

<br>

## PublishSubscribe.subscribe

- Redisson Lock 획득 실패 & Wait Time 유효 -> LockPubSub.subscribe 호출
- LockPubSub이 PublishSubscribe를 상속하고 있어서 PublishSubscribe.subscribe() 호출

```java
public class PublishSubscribeService {

    public CompletableFuture<E> subscribe(String entryName, String channelName) {
        AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
        CompletableFuture<E> newPromise = new CompletableFuture<>();

        semaphore.acquire().thenAccept(c -> {
            if (newPromise.isDone()) {
                semaphore.release();
                return;
            }

            E entry = entries.get(entryName);
            if (entry != null) {
                entry.acquire();
                semaphore.release();
                entry.getPromise().whenComplete((r, e) -> {
                    if (e != null) {
                        newPromise.completeExceptionally(e);
                        return;
                    }
                    newPromise.complete(r);
                });
                return;
            }

            E value = createEntry(newPromise);
            value.acquire();
```

- AsyncSemaphore **GET (생성 아님)**
- Redisson Lock Entry **생성 or GET**

<br>

## AsyncSemaphore GET

```java
package org.redisson.misc;

public class AsyncSemaphore {

    private final AtomicInteger counter;
    private final Queue<CompletableFuture<Void>> listeners = new ConcurrentLinkedQueue<>();

```
- org.redisson.misc 패키지에서 제공하는 AsyncSemaphore의 대기 작업은 ConcurrentLinkedQueue에서 관리
- java.util.concurrent 패키지에서 제공하는 Semaphore는 AbstractQueuedSynchronizer 기반의 Node 큐에서 관리

<br>

```java
public class PublishSubscribeService {

    private final AsyncSemaphore[] locks = new AsyncSemaphore[50];

    public PublishSubscribeService(ConnectionManager connectionManager) {
        super();
        this.connectionManager = connectionManager;
        this.config = connectionManager.getServiceManager().getConfig();
        for (int i = 0; i < locks.length; i++) {
            locks[i] = new AsyncSemaphore(1);
        }
    }

    AsyncSemaphore getSemaphore(ChannelName channelName) {
        return locks[Math.abs(channelName.hashCode() % locks.length)];
    }
```
- PublishSubscribeService에 의해 AsyncSemaphore 50개 생성되어 있고
- PublishSubscribe.subscribe()에서는 이미 생성된 AsyncSemaphore 중 Redisson Lock Hash code 기반으로 AsyncSeamphore를 가져옴

<br>

### 50개로 고정된 이유 (추측)
- AsyncSemaphore 자체가 '같은 Redisson Lock 획득을 위한 대기 공간' 보다는 '**Redis 서버에 요청을 보내기 위한 대기 공간**'에 초점이 맞춰져 있다고 추측
- 즉, '같은 Redisson Lock 획득을 위한 대기 공간'은 Redisson Lock Entry에서 관리되고 Redisson Lock Entry에서 poll 된 작업이 AsyncSemaphore의 listener에 추가되어 Redis 서버로 요청을 보내는 구조라고 추측
- 현재 Redisson Lock Entry에서 poll 된 작업을 AsyncSemaphore의 listener 추가하는 코드를 찾지 못함

<br>

### Semaphore 구현을 추가로 사용하는 이유 (추측)
- 대기 작업을 ConcurrentLinkedQueue에서 관리하는데 왜 Semaphore(permits = 1)를 추가로 사용할까 고민해봤는데
- Redis 서버의 요청량을 제한하기 위함이라고 추측

<br>

## RedissonLockEntry GET or 생성

PublishSubscribe.subscribe() 에서 GET or 생성하는데 그 차이를 다음과 같이 추측
- GET: 이미 같은 Redisson Lock을 획득하려는 대기 작업 존재 (이후 추가)
- 생성: 그렇지 않은 경우

```java
public class RedissonLockEntry implements PubSubEntry<RedissonLockEntry> {

    private volatile int counter;

    private final Semaphore latch;
    private final CompletableFuture<RedissonLockEntry> promise;
    private final ConcurrentLinkedQueue<Runnable> listeners = new ConcurrentLinkedQueue<Runnable>();

    public RedissonLockEntry(CompletableFuture<RedissonLockEntry> promise) {
        super();
        this.latch = new Semaphore(0);
        this.promise = promise;
    }

    public int acquired() {
        return counter;
    }

    public void acquire() {
        counter++;
    }

    public int release() {
        return --counter;
    }
```

- 같은 Redisson Lock 획득에 실패한 요청은 Runnable로 생성되어 ConcurrentLinkedQueue에 대기하는 것으로 추측


### Redisson Lock Entry - poll / semaphore.acquire(), release() 호출 지점
```java
public class LockPubSub extends PublishSubscribe<RedissonLockEntry> {
    
    @Override
    protected void onMessage(RedissonLockEntry value, Long message) {
        if (message.equals(UNLOCK_MESSAGE)) {
            value.tryRunListener();

            value.getLatch().release();
        } else if (message.equals(READ_UNLOCK_MESSAGE)) {
            value.tryRunAllListeners();

            value.getLatch().release(value.getLatch().getQueueLength());
        }
    }

}
```

```java
public class RedissonLockEntry implements PubSubEntry<RedissonLockEntry> {

    private volatile int counter;

    private final Semaphore latch;
    private final CompletableFuture<RedissonLockEntry> promise;
    private final ConcurrentLinkedQueue<Runnable> listeners = new ConcurrentLinkedQueue<Runnable>();

   public void tryRunListener() {
        Runnable runnableToExecute = listeners.poll();
        if (runnableToExecute != null) {
            runnableToExecute.run();
        }
    }
```
- Redis 서버로부터 응답이 오면 RedissonLockEntry.listener에 대기하고 있는 Runnable poll 해 작업 시작
- 이후 **RedissonLockEntry.release()**
- 그럼 **RedissonLockEntry.acquire()** 는 어디서 하는가 -> PublishSubscribe.subscribe에서 (아래 코드)

```java
public CompletableFuture<E> subscribe(String entryName, String channelName) {
    ...

    semaphore.acquire().thenAccept(c -> {
        ...

        E entry = entries.get(entryName);
        if (entry != null) {
            entry.acquire();
            semaphore.release();
            entry.getPromise().whenComplete((r, e) -> {
                if (e != null) {
                    newPromise.completeExceptionally(e);
                    return;
                }
                newPromise.complete(r);
            });
            return;
        }

        E value = createEntry(newPromise);
        value.acquire();
```

### Permits = 0 설정 이유


```java
public class RedissonLockEntry implements PubSubEntry<RedissonLockEntry> {
    private final Semaphore latch;

    public RedissonLockEntry(CompletableFuture<RedissonLockEntry> promise) {
        super();
        this.latch = new Semaphore(0);
    }
```
그럼 왜 RedissonLockEntry에서 사용하는 Semaphore는 permits = 0 으로 설정한 이유?
- 글쎄...

<br>

## 결론: Redisson tryLock 호출한 스레드 관리

- 같은 Redisson Lock을 획득 시도하는 작업은 RedissonLockEntry에서 관리되고
- 이후 Redis 서버로부터 응답이 오면 AsyncSemaphore로 옮겨지고
- 순서를 기다리다가 Redis 서버로 RLock 재시도 요청 보냄

<br>

## Named Lock 대신 Redisson 사용 이유

- MySQL Named Lock은 잘 사용되지 않는다고 알고 있음

### Runnable, CompletableFuture

- Redisson Lock 획득 과정에서 Runnable, CompletableFuture 사용
- 즉, 요청한 스레드는 Lock 획득 전까지 다른 작업 가능
- 하지만 Named Lock을 사용하려면 Named Lock을 획득할 때까지 blocking

<br>

- JVM memory 사용률이나 스레드 상태를 측정해보고 싶었지만,
- Micrometer에서 이에 근접한 측정 요소를 찾지 못함


### 트랜잭션 로직 길이

```
Hibernate: SELECT GET_LOCK(?, ?)
Hibernate: select i1_0.id,i1_0.stock from item i1_0 where i1_0.id=? for update
Hibernate: update item set stock=? where id=?
Hibernate: SELECT RELEASE_LOCK(?)
```
- Named Lock을 획득하기 위해 일단 DB Connection을 잡아야 한다는 점
- 한정적인 DB connection을 잘 사용하려면 트랜잭션 로직 자체가 길면 안 되는데 Named Lock을 사용해 트랜잭션 로직 길어짐
    - 실무에서는 트랜잭션 로직을 줄이기가 쉽지 않아서 Named Lock 대신 Redisson을 사용해 트랜잭션 로직을 조금이라도 줄이지 않을까 추측

<br>

### 실행 속도 측정

![](/redis/img/lock-duration-results.png)

- Redisson Lock 사용 로직이 가장 느림 (원인 추측: Redis 서버 접근이 추가로 발생)
- Named Lock을 잡고 Pessimistic Lock을 잡는 로직이 가장 빠름
- 예상과 다른 결과가 나와... 측정 방법을 바꿔볼까 생각 중...

<br>

## Redisson 사용 시 주의 사항

- Spin Lock 현상이 발생해 동시성 문제 발생 가능성 있음
    - Spin Lock: 2대 이상의 서버가 자신이 락 주인이라 착각해 DB로 쿼리를 날리는 상황
- 이를 위해 Redisson 사용하고 DB Lock을 추가로 잡아야 함
- [토스 증권](https://www.youtube.com/watch?v=UOWy6zdsD-c): Optimistic Lock 추가 사용
    - Optismistic Lock 롤백 작업을 애플리케이션에 직접 구현해야 하지만, 앞단에서 Redisson Lock을 잡고 들어오기 때문에 Rollback 되는 작업이 많지 않을 것이라 예상

<br>

## 해결 못한 부분

- [ ] 아래 내용을 해결하지 못해 **결론 내용이 추측인 상태**
- [ ] RedissonLockEntry에서 poll 된 작업을 AsyncSemaphore의 listener 추가하는 코드를 찾지 못함
- [ ] RedissonLockEntry.promise(CompletableFuture<RedissonLockEntry>)에 추가되는 코드 찾지 못함
- [ ] AsyncSemaphore.listenrs(CompletableFuture<Void>)에 추가되는 코드 찾지 못함
- [ ] AsyncSemaphore는 Redis 서버로 보내는 요청을 조절하기 위해 permits = 1 로 설정했다면 RedissonLockEntry에서 사용하는 Semaphore는 왜 permits = 0 으로 설정한 이유

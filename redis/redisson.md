```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {  
        long time = unit.toMillis(waitTime);  
        long current = System.currentTimeMillis();  
        long threadId = Thread.currentThread().getId();  
        Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);  
        // lock acquired  
        if (ttl == null) {  
            return true;  
        }  
          
        time -= System.currentTimeMillis() - current;  
        if (time <= 0) {  
            acquireFailed(waitTime, unit, threadId);  
            return false;  
        }  
          
        current = System.currentTimeMillis();  
        CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);  
        try {  
            subscribeFuture.get(time, TimeUnit.MILLISECONDS);  
        } catch (TimeoutException e) {  
            if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(  
                    "Unable to acquire subscription lock after " + time + "ms. " +  
                            "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {  
                subscribeFuture.whenComplete((res, ex) -> {  
                    if (ex == null) {  
                        unsubscribe(res, threadId);  
                    }  
                });  
            }  
            acquireFailed(waitTime, unit, threadId);  
            return false;  
        } catch (ExecutionException e) {  
            acquireFailed(waitTime, unit, threadId);  
            return false;  
        }  
  
        try {  
            time -= System.currentTimeMillis() - current;  
            if (time <= 0) {  
                acquireFailed(waitTime, unit, threadId);  
                return false;  
            }  
          
            while (true) {  
                long currentTime = System.currentTimeMillis();  
                ttl = tryAcquire(waitTime, leaseTime, unit, threadId);  
                // lock acquired  
                if (ttl == null) {  
                    return true;  
                }  
  
                time -= System.currentTimeMillis() - currentTime;  
                if (time <= 0) {  
                    acquireFailed(waitTime, unit, threadId);  
                    return false;  
                }  
  
                // waiting for message  
                currentTime = System.currentTimeMillis();  
                if (ttl >= 0 && ttl < time) {  
                    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);  
                } else {  
                    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);  
                }  
  
                time -= System.currentTimeMillis() - currentTime;  
                if (time <= 0) {  
                    acquireFailed(waitTime, unit, threadId);  
                    return false;  
                }  
            }  
        } finally {  
            unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);  
        }  
//        return get(tryLockAsync(waitTime, leaseTime, unit));  
    }
```

## subscribe 메소드(PublishSubscribe.java) 
```java PublishSubscribe.java
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
  
        E oldValue = entries.putIfAbsent(entryName, value);  
        if (oldValue != null) {  
            oldValue.acquire();  
            semaphore.release();  
            oldValue.getPromise().whenComplete((r, e) -> {  
                if (e != null) {  
                    newPromise.completeExceptionally(e);  
                    return;  
                }  
                newPromise.complete(r);  
            });  
            return;  
        }  
  
        RedisPubSubListener<Object> listener = createListener(channelName, value);  
        CompletableFuture<PubSubConnectionEntry> s = service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, listener);  
        newPromise.whenComplete((r, e) -> {  
            if (e != null) {  
                s.completeExceptionally(e);  
            }  
        });  
        s.whenComplete((r, e) -> {  
            if (e != null) {  
                entries.remove(entryName);  
                value.getPromise().completeExceptionally(e);  
                return;  
            }  
            value.getPromise().complete(value); // 리스너로 pubsub 채널 구독이 완료되면 RedissonLockEntry의 cf객체
        });  
  
    });  
  
    return newPromise;  
}
```

```java
// AsyncSemaphore acquire 메소드
public CompletableFuture<Void> acquire() {  
    CompletableFuture<Void> future = new CompletableFuture<>();  
    listeners.add(future);  
    tryForkAndRun();  
    return future;  
}
```
일반적인 세마포어의 acquire는 락을 획득할 때까지 블로킹되는것과 달리
AsyncSemaphore의 acquire는 블로킹하는 대신 listerners (ConcurrentLinkedQueue인 리스너 큐)에 CompletableFuture를 추가하고 `tryForkAndRun`에서 listeners에서 하나 poll해서 완료시키고 코드 진행 이어감
>AsyncSemaphore의 acquire 메소드는 큐 뒤에 CompletableFuture 요소 하나 추가하고 맨 앞에 하나 빼서 완료시키는 FIFO 구조
>근데 처음에 큐가 비어있는 상황에서 하나 넣고 하나 빼는거면 어떤 의도로 이런 구조를 한거지...순차적으로 콜백을 등록하게 하기 위해서인가

```java
private final Queue<CompletableFuture<Void>> listeners = new ConcurrentLinkedQueue<>();
```
결과값이 Void인데 이는 이 CompletableFuture가 먼가 특별한 로직의 결과값을 받는게 아니라 단순히 해당 CompletableFuture를 complete()를 통해 완료시키면 subscribe()메소드에서 semaphore.acquire().thenAccept(..)을 통해 이후에 실행되어야 할 로직이 실행되도록 하기 위해서이기 때문

```java
// AsyncSemaphore.java
private void tryRun() {  
    while (true) {  
        ...
  
            boolean complete;  
            if (executorService != null) {  
                stackSize.incrementAndGet();  
                complete = future.complete(null);  
                stackSize.decrementAndGet();  
            } else {  
                complete = future.complete(null);  
            }  
            if (complete) {  
                return;  
            }  
        }  
  
        ... 
    }  
}
```
listeners에서 poll 된(차례가 된) CF(CompletableFuture, 앞으로 길어서 CF라고 하겠음) 객체를 complete시켜서 subscribe() 메소드에서 등록한 이후 로직을 수행하도록 함

`subscribe()` 메소드는 일단 AsyncSemaphore 획득 후 무엇을 할지 콜백으로 정의하고 있는데
`newPromise` -> 아직 정확히 뭔지는 모르겠는데 subscribe() 메소드의 최종 리턴값, 아마 구독 관련 결과 아닐까 추측
```java
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
```
newPromise가 완료되면(이미 RedissonLockEntry를 결과값으로 전달하면) AsyncSemaphore를 release하고 리턴, 그렇지 않다면 
key를 entryName(lock key), value를 RedissonLockEntry로 하는 hashmap에서 RedissonLockEntry를꺼내오는데 **entry(RedissonLockEntry)를 acquire()하면 바로  AsyncSemaphore를 release하는걸 봐선 AsyncSemaphore는 entry.acquire()의 race condition을 예방하기 위해 존재하는 것으로 추측됨**

entry.acquire() 의 경우 단순히 RedissonLockEntry의 counter++ 하는건데 이 counter가 무슨 의미인거지 생각을 해봤는데 entry에 리스너로 등록하는 콜백의 수(혹은 entry를 사용하는 스레드의 수)를 의미하는 것으로 추측
암튼 이 entry의 promise 객체(CF 객체)에 결과값 r(RedissonLockEntry 객체)이 전달되면 콜백으로 subscribe 메소드의 리턴값 `newPromise`를 완료시키고 리턴
(entry.promise 완료 -> newPromise 완료)
>코드에 진행을 읽어보면 **RedissonLockEntry 객체가 이미 존재한다면 후에 나올 별도의 Pub/Sub 구독 작업을 하지 않고 리턴하는데...그러면 하나의 RedissonLockEntry 객체를 여러 스레드가 공유하는거 아닐까 추측**

>부가 설명: completeExceptionally()와 complete()는 둘 다 전달받은 인자를 작업 결과로 하여 completableFuture의 작업 결과를 필요로 하는 체이닝된 메소드(ex.thenApply)에게 전달하는 역할인데 completeExceptionally는 예외를 전달, complete는 정상적으로 처리된 작업 결과를 전달

```java
	E value = createEntry(newPromise);  
    value.acquire();  
  
    E oldValue = entries.putIfAbsent(entryName, value);  
    if (oldValue != null) {  
        oldValue.acquire();  
        semaphore.release();  
        oldValue.getPromise().whenComplete((r, e) -> {  
            if (e != null) {  
                newPromise.completeExceptionally(e);  
                return;  
            }  
            newPromise.complete(r);  
        });  
        return;  
    }  
  
    RedisPubSubListener<Object> listener = createListener(channelName, value);  
    CompletableFuture<PubSubConnectionEntry> s = service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, listener);  
    newPromise.whenComplete((r, e) -> {  
        if (e != null) {  
            s.completeExceptionally(e);  
        }  
    });  
    s.whenComplete((r, e) -> {  
        if (e != null) {  
            entries.remove(entryName);  
            value.getPromise().completeExceptionally(e);  
            return;  
        }  
        if (!value.getPromise().complete(value)) {  
            if (value.getPromise().isCompletedExceptionally()) {  
                entries.remove(entryName);  
            }  
        }  
    });  
```
- hashmap에 entryName에 해당하는 RedissonLockEntry가 없었다면 createEntry() 메소드를 통해 새로 생성
- putIfAbsent() 메소드는 hashmap에 해당 key-value가 없었다면 넣어주고 null을 리턴, 이미 있다면 있는 key-value를 리턴
>종합하면 RedissonLockEntry를 새로 생성한 상황이라면 entries에 아직 넣어지지 않았을 것이므로 이를 넣어주고, Redis 채널에 대한 Pub/Sub Listener 를 생성하고 RedissonLockEntry가 redisson_lock__channel:{lock key} 이란 채널을 구독하도록 함
>(oldValue가 null이 아니면 이미 해당 RedissonLockEntry가 있다는 뜻이고 이러한 Pub/Sub 작업도 이미 했을 것이므로 구독 작업을 진행하지 않음)

subscribe 내 비동기 결과 메소드 체이닝 흐름은 다음과 같다.

semaphore ->
subscribeNoTimeout(entry 새로 만들때만) -> entry.promise->
newPromise

>subscribeNoTimeout를 실패하면 entry를 그냥 삭제하는 것과 createListener에서 인자로 RedissonLockEntry를 전달하는 것을 보아 **RedissonLockEntry 객체가 사실상 채널을 구독하는 형태이지 않나 추측**

> 그리고 **구독 작업이 완료되면 (`s.whenComplete` 부분) RedissonLockEntry의 promise(CF 객체)를 완료시키는걸 보아 promise 객체는 구독 작업이 완료될때까지 tryLock 코드 진행을 잠시 멈추게 하기 위해 존재하는 것으로 일단 추측..** 

``` java
// PublishSubscribe.java
RedisPubSubListener<Object> listener = createListener(channelName, value);

PublishSubscribe.this.onMessage(value, (Long) message);

// LockPubSub.java
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

// RedissonLockEntry.java
public void tryRunListener() {  
    Runnable runnableToExecute = listeners.poll();  
    if (runnableToExecute != null) {  
        runnableToExecute.run();  
    }  
}
```

이 Runnable에 어떤 로직이 담겨있는지 잘 모르겠음...디버깅해봐도 listeners 사이즈가 0인데(등록된 Runnable이 없는데)

**결국 subscribe 메소드의 목적은 구독 작업이 완료된 RedissonLockEntry를 결과값으로 전달하는 것**

다시 RedissonLock.java의 tryLock 메소드로 돌아가서

```java
CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);  
try {  
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);  
}
```
subscribe 메소드 실행 후 바로 뒤에 try문 내에서 이 CF의 결과값을 get()메소드를 통해 기다리게 되는데 마냥 기다리는게 아니라 get()에 남은 시간(time) 만큼 타임아웃을 줘서 이 시간 내에 subscribe()의 CF에 결과값이 넘어오지 않으면 TimeoutException을 던지게 함 (애초에 분산 락을 획득하는 모든 로직이 결국 waitTime 내에 이루어져야 하므로)

```java
// waiting for message  
currentTime = System.currentTimeMillis();
if (ttl >= 0 && ttl < time) {  
    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);  
} else {  
    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);  
}
```

```java
public RedissonLockEntry(CompletableFuture<RedissonLockEntry> promise) {  
    super();  
    this.latch = new Semaphore(0);  
    this.promise = promise;  
}
```
RedissonLockEntry의 경우 latch의 permits를 0으로 시작하므로 
getLatch(RedissonLockEntry의 semaphore)를 acquire 하기 위해선 어디선가 release해줘야 하는데,
이는 onMessage 메소드에서 구독한 채널로부터 메시지를 받았을 때 release 한다. (그래서 주석에 waiting for message라고 한듯)

while문으로 계속 spin하는게 아니라 세마포어를 얻지 못하면 BLOCKED 상태로 CPU 자원을 다른 스레드에게 넘기기 때문에, 락 획득 시도를 계속 반복하는 스핀 락보다 더 효율적
그리고 sleep하다가 일어나는 시점도 구독한 채널에서 메시지를 받아서 semaphore를 release하면 그때 세마포어 획득 시도

여러 스레드가 하나의 RedissonLockEntry를 같이 사용하면 이 RedissonLockEntry도 세마포어를 통해 동시성 처리를 해줘야 하므로 latch가 있는거 같고
getLatch()를 통해 내 차례가 될때 까지 BLOCKED 되다가 구독한 채널로부터 메시지가 와서 세마포어가 release되면 일어나서 락 획득을 시도해서 성공하면 이제 다시 while 문 처음으로 돌라가서 tryAcquire(실제로 분산락을 잡는 로직)을 수행

---
## 결론

결론적으로 결국엔 세마포어로 분산 락 획득에 동시성을 제어하는건데 
분산 락 자체는 레디스 서버가 관리하는 거니까 분산 락의 획득 가능 여부는 레디스 서버가 알고있는거기 때문에 분산 락이 release되면 이를 채널에 메시지를 보낸다.
이를 구독하고 있던 RedissonLockEntry가 메시지를 통해 분산 락을 획득할 수 있다는 것을 알게 되고 세마포어를 release해서 대기중인 스레드가 분산 락을 획득 시도하도록 한다.

# Redisson

### Redisson 정의

Java 기반의 고성능 비동기 & 락프리 *Redis 클라이언트 라이브러리*

### RedissonClient 코드 Github에서 확인

https://github.com/redisson/redisson/blob/master/redisson/src/main/java/org/redisson/api/RedissonClient.java

### 주요 기능 및 메서드

1. 자료구조 API
    - RMap, RSet, RList, RQueue, RDeque 등 Redis의 데이터 구조를 Java에서 사용할 수 있는 API를 제공
    - RTimeSeries, RGeo, RStream 등 Redis의 기능을 Java API로 활용
2. 분산 동기화 도구
    - Redisson은 다양한 동기화 메커니즘을 제공합니다:
        - RLock, RReadWriteLock: 분산 락 구현
        - RSemaphore, RCountDownLatch: 분산 세마포어 및 래치
3. 캐싱 및 TTL 지원
    - RMapCache, RSetCache와 같은 메서드는 캐시 데이터를 저장하고, TTL 기반 데이터 자동 삭제를 지원
4. 메시징 기능
    - Pub/Sub 모델을 기반으로 하는 메시징 기능을 지원:
        - RTopic, RShardedTopic, RReliableTopic을 통해 메시지 브로커처럼 Redis를 활용 가능
5. 배치 작업 및 트랜잭션
    - 배치 작업(RBatch)과 트랜잭션(RTransaction)을 통해 여러 Redis 명령을 효율적으로 실행 가능
6. 분산 서비스
    - 원격 서비스 호출(RRemoteService), 작업 스케줄링(RScheduledExecutorService) 등을 지원해 분산 시스템을 손쉽게 구축

### 장점

1. 간단한 Redis 활용:
    - Java에서 Redis의 고급 기능(분산 락, 스트림 등)을 간단한 메서드 호출로 구현 가능
2. Lock에 타임아웃 지정:
    - Lock에 TTL을 지정해 무한정 대기상태로 빠질 수 있는 위험 차단
3. pub/sub방식: 
    - 스핀락을 사용하지 않으므로 요청 처리를 위한 오버헤드 발생 감소

---

# RLock tryLock

### RLock 코드 Github에서 확인

- RLock
    
    https://github.com/redisson/redisson/blob/master/redisson/src/main/java/org/redisson/api/RLock.java
    
- RedissonLock (RLock 구현체)
    
    https://github.com/redisson/redisson/blob/master/redisson/src/main/java/org/redisson/RedissonLock.java
    

### RLock 인터페이스 분석

RLock은 Redis 기반으로 설계된 Redisson의 분산 락 인터페이스입니다.

세마포어 기반의 동기화 및 비동기 작업을 관리하는 CompletableFuture를 효과적으로 사용할 수 있는 환경을 제공합니다.

### tryLock 메소드

- 공식 문서 내 메소드의 설명 (한글 번역)

> 설정된 **leaseTime**으로 락을 획득하려고 시도합니다. 필요하다면, 락이 사용 가능해질 때까지 설정된 **waitTime** 동안 대기합니다. 락은 설정된 **leaseTime** 간격 이후에 자동으로 해제됩니다.
> 
> - **매개변수**:
>     - **waitTime**: 락을 획득하기 위해 대기할 최대 시간입니다.
>     - **leaseTime**: 락을 유지할 시간입니다.
>     - **unit**: 시간 단위입니다.
> - **반환값**:
>     - 락을 성공적으로 획득하면 true를 반환하며, 락이 이미 설정된 경우 false를 반환합니다.
> - **예외**:
>     - **InterruptedException**: 스레드가 인터럽트될 경우 발생합니다.

### 구현체 (RedissonLock) 분석

#### 1. Lock 획득 시도

```java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    // 남은 시간을 밀리초로 저장
    long time = unit.toMillis(waitTime);
    // 현재 시간을 밀리초로 저장
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    
    // *lock 획득 시도 
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // ttl == null -> lock 획득 -> return true
    if (ttl == null) {
        return true;
    }
...
```

- lock 획득 시도 (설명으로 코드 대체)
    - tryAcquire(Long) -> tryAcquireAsync0(RFuture<Long>) -> tryAcquireAsync(RFuture<Long>)
    - **RFuture는 CompletableFutureWrapper의 인터페이스 (하단에서 따로 설명)**
    
    ```java
    private RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
      // 1. RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(...)
    		  // a. 키가 존재하지 않는지 확인.
    		  // b. 키가 존재하고, threadId 가 해당 키의 필드로 존재하는지 확인.
    		  // => a || b : null을 리턴
      // 2. CompletionStage<Long> s = handleNoSync(threadId, ttlRemainingFuture)
    		  // 락 획득 작업의 결과를 처리하는 메서드, 비동기적으로 수행 가능.
      // 3. ttlRemainingFuture = new CompletableFutureWrapper<>(s);
    		  // a. ttlRemaining == null -> lock 획득 
    			  // return ttlRemaining => null 리턴
    		 // b. new CompletableFutureWrapper<>(f);
    			 // => CompletionStage<Long> 타입의 f를 CompletableFutureWrapper로 감싸서 리턴
    }
    ```
    

#### 2. Redis Pub/Sub 채널 구독

```java
...
// ttl이 존재할 때
    time -= System.currentTimeMillis() - current;
    // 남은 대기 시간 만료
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
...
```

```java
...
	// 현재시간 갱신
    current = System.currentTimeMillis();
    // *threadId로 채널을 구독
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    try {
        subscribeFuture.get(time, TimeUnit.MILLISECONDS);
    } // 대기하는 동안 대기 시간이 지난 경우
    catch (TimeoutException e) {
        if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
                "Unable to acquire subscription lock after " + time + "ms. " +
                        "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
            subscribeFuture.whenComplete((res, ex) -> {
		        // 구독 취소
                if (ex == null) {
                    unsubscribe(res, threadId);
                }
            });
        }
        // 획득 실패
        acquireFailed(waitTime, unit, threadId);
        return false;
    } // CompletableFuture에서 예외가 발생한 경우
    catch (ExecutionException e) {
        LOGGER.error(e.getMessage(), e);
        // 획득 실패
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
...
```

- threadId로 채널을 구독 (설명으로 코드 대체)
    
    ```java
    public CompletableFuture<E> subscribe(String entryName, String channelName) {
        // 1. 비동기 세마포어 (semaphore) 획득
        // 2. CompletableFuture<E> newPromise 생성 (구독의 결과를 나타내는 비동기 객체) 
        
    		// 3. 세마포어 획득 
    				// a. 작업(newPromise) 완료 -> 작업 수행 
    				// b. 작업이 완료되지 않음 -> 기존 entry 확인
    						// I. 기존 entry 존재 
    								// 해당 entry 점유 
    								// 세마포어 해제 후, 해당 entry 결과를 newPromise에 연결 
    						// II. 기존 entry 존재하지 않음
    					      // 새로운 entry 생성 & entries에 추가 
    							      // 이미 다른 쓰레드에서 동일한 entry 사용 시 -> 새로 만든 entry 무시
    
    		// 4. Redis Pub/Sub Listener 생성 & Redis 채널 구독
        // 5. newPromise 와 CompletableFuture 에 대해 콜백함수를 선언
    	    // + CompletableFuture 에 RedissonLockEntry 을 담아 리턴
    }
    ```
    

#### 3. 락 획득 재시도

```java
...
    try {
		    // 대기 시간 만료 체크
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
    
        while (true) {
		        // 현재시간 갱신
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
            // 락 획득 완료
            if (ttl == null) {
                return true;
            }
            
			// 대기 시간 만료 체크
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }

		    // 현재시간 갱신
            currentTime = System.currentTimeMillis();
            // *메시지 대기
            if (ttl >= 0 && ttl < time) {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }

			// 대기 시간 만료 체크
            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    } // Redis 채널 구독 취소
    finally {
        unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);
    }
//  return get(tryLockAsync(waitTime, leaseTime, unit));
}
```

- 메시지 대기
    
    commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(long timeout, TimeUnit unit);
    

---

## CompletableFuture

### CompletableFuture

- CompletableFuture는 Java의 비동기 프로그래밍을 위한 표준 API로, Java 8에서 도입되었습니다.

---

### RFuture

- Redisson에서 제공하는 비동기 작업 결과 표현 객체입니다.
- Java의 표준 비동기 API인 CompletableFuture와 비슷한 역할을 하며, 작업 완료 후 결과를 처리하는 thenApply, thenAccept 등의 비동기 작업 체인을 구성할 수 있습니다.

---

### 왜 비동기 처리를 위한 도구를 사용하는가?

1. Redis의 비동기 네트워크 요청
    - Redis는 네트워크 통신 기반의 데이터 저장소입니다. 이 과정에서 I/O 작업이 비동기적으로 처리되면 애플리케이션의 성능을 극대화할 수 있습니다.
    - RFuture는 Redis 요청의 결과를 비동기적으로 받아오는 데 적합합니다.
2. 효율적인 리소스 관리
    - 비동기 작업은 스레드를 차단하지 않으므로, 더 많은 작업을 동시에 처리할 수 있습니다.
    - 예를 들어, 락 획득과 같은 Redis 작업을 비동기로 처리하면 다른 작업이 블로킹되지 않습니다.
3. 편리한 비동기 작업 체인
    - thenApply, thenAccept, exceptionally와 같은 체인 호출로 작업 완료 후 로직을 처리할 수 있어 코드가 간결하고 유지보수가 쉬워집니다.

---

### 참고 자료

[https://devoong2.tistory.com/entry/Spring-Redisson-라이브러리를-이용한-Distribute-Lock-동시성-처리-1](https://devoong2.tistory.com/entry/Spring-Redisson-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-Distribute-Lock-%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%B2%98%EB%A6%AC-1)

[https://devoong2.tistory.com/entry/Spring-Redisson-TryLock-동작-과정?category=1088625](https://devoong2.tistory.com/entry/Spring-Redisson-TryLock-%EB%8F%99%EC%9E%91-%EA%B3%BC%EC%A0%95?category=1088625)

[https://velog.io/@dooh97/Redisson-작동원리-알아보기](https://velog.io/@dooh97/Redisson-%EC%9E%91%EB%8F%99%EC%9B%90%EB%A6%AC-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0)
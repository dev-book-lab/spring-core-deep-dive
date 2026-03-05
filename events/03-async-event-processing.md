# 비동기 이벤트 처리 @Async — 스레드 분리 원리와 스레드 풀 설정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Async`와 결합한 `@EventListener`에서 스레드가 분리되는 정확한 지점은?
- `SimpleApplicationEventMulticaster`에 `Executor`를 설정하는 방식과 `@Async`를 결합하는 방식의 차이는?
- 비동기 리스너에서 발생한 예외는 어떻게 처리되는가?
- 비동기 이벤트 처리 시 트랜잭션 컨텍스트는 공유되는가?
- `@Async` 비동기 이벤트와 `@TransactionalEventListener`를 함께 사용할 때 주의해야 할 점은?

---

## 🔍 왜 이게 존재하는가

### 문제: 이벤트 리스너가 느리면 발행자도 블로킹된다

```java
// 동기 처리 — 발행자가 리스너 완료를 기다림
@EventListener
public void sendConfirmationEmail(OrderPlacedEvent event) {
    emailClient.send(...);  // 외부 API 호출 — 200ms 소요
    // 이 동안 OrderService.placeOrder()가 블로킹됨!
}

// 문제:
// OrderService가 이메일 전송 완료를 기다릴 이유 없음
// 이메일 전송 실패가 주문 트랜잭션을 롤백해서는 안 됨
// 이메일은 비핵심 처리 → 별도 스레드에서 처리해야 함
```

```
비동기 이벤트 처리가 필요한 상황:
  이메일/알림 발송 (외부 API, 느림)
  감사 로그 기록 (DB 쓰기, 독립 트랜잭션)
  통계 집계 (배치성, 즉시 필요 없음)
  캐시 갱신 (지연 허용 가능)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Async 하나면 모든 이벤트가 비동기가 된다

```java
// ❌ 잘못된 이해:
// "SimpleApplicationEventMulticaster에 Executor 설정 → 전체 비동기"
// "@Async 하나로 해당 리스너만 비동기"

// ✅ 실제 차이:

// 방법 1: Multicaster에 Executor 설정
// → 모든 리스너가 Executor를 통해 실행 (일괄 비동기)
// → @EventListener, ApplicationListener<T> 모두 영향
// → 동기/비동기 혼용 불가

// 방법 2: @Async + @EventListener
// → 해당 리스너 메서드만 비동기
// → 다른 리스너는 여전히 동기
// → 세밀한 제어 가능, 권장 방식
```

### Before: 비동기 이벤트 리스너에서 예외가 발생하면 롤백된다

```java
// ❌ 잘못된 이해:
@Async
@EventListener
public void onOrderPlaced(OrderPlacedEvent e) {
    throw new RuntimeException("이메일 발송 실패");
    // "이게 OrderService 트랜잭션을 롤백시킨다"
}

// ✅ 실제:
// @Async 리스너는 별도 스레드에서 실행
// → 발행 스레드(OrderService)와 완전히 분리
// → 예외가 발행 스레드로 전파되지 않음
// → OrderService 트랜잭션에 영향 없음
// → AsyncUncaughtExceptionHandler로 별도 처리 필요
```

---

## ✨ 올바른 이해와 사용

### After: 두 가지 비동기 방식 구조 비교

```
방법 1: SimpleApplicationEventMulticaster + Executor
  publishEvent()
    → multicastEvent()
      → for each listener:
          executor.execute(() -> invokeListener(listener, event))
          → 모든 리스너가 별도 스레드

  @Bean("applicationEventMulticaster")
  public ApplicationEventMulticaster eventMulticaster() {
      SimpleApplicationEventMulticaster m = new SimpleApplicationEventMulticaster();
      m.setTaskExecutor(taskExecutor);
      return m;
  }

방법 2: @Async + @EventListener (권장)
  publishEvent()
    → multicastEvent()
      → invokeListener(adapter, event)  ← 발행 스레드
        → ApplicationListenerMethodAdapter.onApplicationEvent()
          → AsyncAnnotationBeanPostProcessor가 생성한 프록시 메서드 호출
            → TaskExecutor.execute(() -> 실제 리스너 메서드)  ← 새 스레드

  @Async
  @EventListener
  public void onOrderPlaced(OrderPlacedEvent e) { ... }
  // @EnableAsync 설정 필요
```

---

## 🔬 내부 동작 원리

### 1. @Async + @EventListener — 스레드 분리 지점

```java
// 1단계: @EnableAsync 선언
@Configuration
@EnableAsync  // AsyncAnnotationBeanPostProcessor 등록
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-async-");
        executor.initialize();
        return executor;
    }
}

// 2단계: 리스너에 @Async 선언
@Component
public class OrderEventListener {

    @Async               // 이 메서드를 비동기로 실행
    @EventListener
    public void sendEmail(OrderPlacedEvent event) {
        // 별도 스레드(event-async-1)에서 실행
        emailService.send(event.getOrder().getEmail());
    }

    @EventListener       // @Async 없음 → 동기 실행
    public void auditSync(OrderPlacedEvent event) {
        // 발행 스레드에서 실행
    }
}
```

```
스레드 분리 과정:

OrderService.placeOrder() [main thread]
  → publisher.publishEvent(event)
    → multicastEvent()
      → invokeListener(sendEmailAdapter, event) [main thread]
        → OrderEventListener 프록시.sendEmail(event) [main thread]
          → AsyncExecutionInterceptor.invoke() [main thread]
            → taskExecutor.submit(실제 sendEmail) [event-async-1 thread]
            → 즉시 반환! [main thread 계속 진행]
      → invokeListener(auditAdapter, event) [main thread]
        → OrderEventListener.auditSync(event) [main thread, 동기]
```

### 2. SimpleApplicationEventMulticaster + Executor

```java
// 전체 비동기 방식
@Bean(name = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();

    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(3);
    executor.setMaxPoolSize(10);
    executor.setThreadNamePrefix("event-");
    executor.initialize();
    multicaster.setTaskExecutor(executor);

    // 예외 핸들러 설정
    multicaster.setErrorHandler(t ->
        log.error("이벤트 처리 중 예외", t));

    return multicaster;
}
```

```java
// SimpleApplicationEventMulticaster.multicastEvent() 내부
public void multicastEvent(ApplicationEvent event, ResolvableType eventType) {
    Executor executor = getTaskExecutor();

    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null && supportsAsyncExecution(listener)) {
            // Executor가 있으면 비동기 실행
            executor.execute(() -> invokeListener(listener, event));
        } else {
            // 없으면 동기 실행
            invokeListener(listener, event);
        }
    }
}

// supportsAsyncExecution():
// → ApplicationListenerMethodAdapter의 @Async 여부 확인
// → 또는 단순히 executor != null이면 모두 비동기 (설정에 따라)
```

### 3. 비동기 예외 처리

```java
// @Async 메서드의 예외 처리
// → AsyncUncaughtExceptionHandler 구현 필요

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new SimpleAsyncUncaughtExceptionHandler();
        // 기본: 로그만 출력

        // 또는 커스텀 핸들러
        // return (ex, method, params) -> {
        //     log.error("비동기 이벤트 처리 실패: method={}", method.getName(), ex);
        //     alertService.notifyFailure(ex);
        // };
    }
}
```

```
@Async 예외 처리 흐름:

비동기 스레드에서 예외 발생
  → AsyncExecutionAspectSupport.handleError()
    → void 반환 타입: AsyncUncaughtExceptionHandler.handleUncaughtException()
    → Future/CompletableFuture 반환 타입: Future에 예외 저장 (get() 시 전파)

@EventListener + @Async는 보통 void 반환
  → AsyncUncaughtExceptionHandler가 처리
  → 발행 스레드로 예외 전파 안 됨
  → 명시적 예외 처리 코드 필수
```

### 4. 트랜잭션 컨텍스트 분리

```java
@Service
@Transactional
public class OrderService {
    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order));
        // 이 시점: 트랜잭션 아직 커밋 안 됨
    }
}

@Component
public class OrderEventListener {

    // ❌ 비동기 리스너에서 동일 트랜잭션 불가
    @Async
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        // 새 스레드 → 새 트랜잭션 컨텍스트
        // OrderService 트랜잭션의 DB 변경이 아직 커밋되지 않을 수 있음!
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        // → 커밋 전 조회 → 데이터 없거나 이전 상태
    }

    // ✅ 커밋 후 처리: @TransactionalEventListener 사용 (05 문서)
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlacedAfterCommit(OrderPlacedEvent event) {
        // 트랜잭션 커밋 완료 후 비동기 실행
        // → DB에 주문 데이터 확정된 상태
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        emailService.send(order);  // 안전
    }
}
```

### 5. @Async 스레드 풀 선택

```java
// 이벤트 처리 전용 스레드 풀 분리
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }

    @Bean("auditExecutor")
    public Executor auditExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setThreadNamePrefix("audit-");
        executor.initialize();
        return executor;
    }
}

@Component
public class OrderEventListener {

    @Async("emailExecutor")    // 이메일 전용 스레드 풀
    @EventListener
    public void sendEmail(OrderPlacedEvent event) { ... }

    @Async("auditExecutor")    // 감사 전용 스레드 풀
    @EventListener
    public void auditOrder(OrderPlacedEvent event) { ... }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 스레드 이름 확인

```java
@Async
@EventListener
public void asyncListener(OrderPlacedEvent event) {
    System.out.println("리스너 스레드: " + Thread.currentThread().getName());
    // → "event-async-1" (스레드 풀 이름 접두사)
}

@EventListener
public void syncListener(OrderPlacedEvent event) {
    System.out.println("리스너 스레드: " + Thread.currentThread().getName());
    // → "main" (발행 스레드)
}
```

### 실험 2: 비동기 예외 처리 확인

```java
@Async
@EventListener
public void failingListener(OrderPlacedEvent event) {
    throw new RuntimeException("의도적 실패");
}

// 발행자 측
publisher.publishEvent(new OrderPlacedEvent(order));
// → 예외 전파 없음, 정상 진행
// → AsyncUncaughtExceptionHandler 로그에서만 확인 가능
```

### 실험 3: 트랜잭션 격리 확인

```java
@Async
@EventListener
public void checkTransaction(OrderPlacedEvent event) {
    boolean hasTransaction = TransactionSynchronizationManager.isActualTransactionActive();
    System.out.println("트랜잭션 활성: " + hasTransaction);
    // → false (새 스레드 = 새 트랜잭션 컨텍스트, 기존 트랜잭션 공유 안 됨)
}
```

---

## 🤔 트레이드오프

```
@Async + @EventListener (권장):
  장점  리스너별 비동기/동기 혼용 가능
        전용 스레드 풀 지정 가능 (@Async("poolName"))
        동기 리스너 영향 없음
  단점  @EnableAsync 설정 필요
        Self-Invocation 주의 (프록시 우회)

Multicaster Executor 설정:
  장점  설정 한 곳에서 전체 제어
        @EnableAsync 불필요
  단점  모든 리스너가 비동기 (동기/비동기 혼용 어려움)
        리스너별 예외 처리 복잡

스레드 풀 크기:
  너무 작음  이벤트 큐 적체, 처리 지연
  너무 큼    과도한 메모리 사용, 컨텍스트 스위칭 증가
  권장: 이벤트 처리 시간과 빈도를 기반으로 산정
        I/O 중심: 스레드 수 = CPU 코어 수 × (1 + 대기 시간/처리 시간)

비동기 + 트랜잭션:
  @Async → 트랜잭션 분리 (발행자 트랜잭션과 무관)
  커밋 후 처리 필요 → @TransactionalEventListener(AFTER_COMMIT) + @Async 조합
  비동기 리스너 내 트랜잭션 필요 → @Transactional 별도 선언
```

---

## 📌 핵심 정리

```
@Async + @EventListener 스레드 분리 지점
  multicastEvent() → invokeListener() [발행 스레드]
  → 프록시.method() → AsyncExecutionInterceptor
    → taskExecutor.submit(실제 메서드) → 즉시 반환 [발행 스레드 계속]
    실제 메서드 [별도 스레드]에서 실행

두 가지 비동기 방식
  Multicaster Executor: 전체 리스너 일괄 비동기
  @Async: 특정 리스너만 비동기 (권장, 세밀 제어)

비동기 예외 처리
  void 반환 → AsyncUncaughtExceptionHandler
  Future 반환 → Future.get() 시 예외 전파
  발행 스레드로는 전파 안 됨 → 명시적 핸들러 필수

트랜잭션 격리
  @Async = 새 스레드 = 새 트랜잭션 컨텍스트
  발행자 트랜잭션 공유 불가
  커밋 후 처리 → @TransactionalEventListener(AFTER_COMMIT) + @Async 조합
```

---

## 🤔 생각해볼 문제

**Q1.** `@Async` 리스너에서 새로운 이벤트를 `publisher.publishEvent()`로 발행하면 어떤 스레드에서 처리되는가?

**Q2.** `SimpleApplicationEventMulticaster`에 `Executor`를 설정했을 때 `ApplicationListener<T>` 구현체와 `@EventListener` 어댑터 모두 비동기로 실행되는가?

**Q3.** 비동기 이벤트 리스너에서 `@Transactional`을 선언하면 발행자의 트랜잭션과 어떻게 동작하는가?

> 💡 **해설**
>
> **Q1.** `@Async` 리스너 내부에서 발행한 이벤트는 해당 비동기 스레드에서 `publishEvent()`가 호출된다. 새로운 이벤트의 리스너들은 `multicastEvent()`를 통해 처리되는데, 다시 `@Async`인 리스너는 또 다른 별도 스레드로 분기되고, 동기 리스너는 현재 비동기 스레드에서 실행된다. 즉 연쇄 이벤트의 비동기 리스너는 원래 발행 스레드가 아닌 비동기 스레드를 기준으로 분기된다.
>
> **Q2.** `supportsAsyncExecution(listener)`의 판별에 따라 다르다. 단순히 `Executor != null`만으로 모두 비동기 처리하도록 구현하면 모든 리스너가 비동기가 된다. 그러나 Spring의 기본 구현에서는 `ApplicationListenerMethodAdapter`가 `@Async`를 가지고 있지 않아도 `Executor`가 설정된 경우 비동기로 실행될 수 있다. `ApplicationListener<T>` 구현체도 동일하게 `Executor`를 통해 실행된다. 이 점이 "전체 비동기"가 되는 이유이며, 동기/비동기 혼용이 어려운 단점이다.
>
> **Q3.** `@Async` 리스너는 새 스레드에서 실행되므로 발행자의 `ThreadLocal` 기반 트랜잭션 컨텍스트가 전파되지 않는다. 비동기 리스너에 `@Transactional`을 선언하면 리스너 메서드 실행 시 새 트랜잭션이 시작된다. 이 새 트랜잭션은 발행자의 트랜잭션과 독립적이므로 발행자 트랜잭션이 롤백되어도 리스너 트랜잭션에 영향을 주지 않고, 반대로 리스너 트랜잭션이 롤백되어도 발행자 트랜잭션에 영향을 주지 않는다. 발행자 트랜잭션 커밋 후 실행되어야 한다면 `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` + `@Transactional`을 함께 사용한다.

---

<div align="center">

**[⬅️ 이전: @EventListener vs ApplicationListener](./02-eventlistener-vs-applicationlistener.md)** | **[다음: 이벤트 실행 순서 보장 ➡️](./04-event-ordering.md)**

</div>

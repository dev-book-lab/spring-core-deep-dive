# 트랜잭션 바운드 이벤트 — @TransactionalEventListener의 Phase 동작 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@TransactionalEventListener`가 트랜잭션 단계에 바인딩되는 정확한 메커니즘은?
- `TransactionSynchronization`이 등록되고 호출되는 시점은?
- `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`, `BEFORE_COMMIT` 각 phase의 실행 시점 차이는?
- 트랜잭션이 없는 상태에서 `@TransactionalEventListener`는 어떻게 동작하는가?
- `@Async` + `@TransactionalEventListener` 조합이 필요한 이유와 주의점은?

---

## 🔍 왜 이게 존재하는가

### 문제: 트랜잭션 커밋 전에 이벤트 처리를 시작하면 데이터 일관성이 깨진다

```java
@Service
@Transactional
public class OrderService {
    public void placeOrder(Order order) {
        orderRepository.save(order);   // DB에 쓰기 (아직 커밋 안 됨)
        publisher.publishEvent(new OrderPlacedEvent(order.getId()));
        // 문제: 이 시점에 이벤트 리스너가 실행됨
        // → 리스너에서 orderRepository.findById(order.getId()) 조회 시
        //   아직 커밋되지 않은 데이터 → 없거나 이전 상태
        //   (다른 트랜잭션에서는 이 데이터가 안 보임)
    }
}

@EventListener  // 일반 이벤트 리스너
public void onOrderPlaced(OrderPlacedEvent event) {
    Order order = orderRepository.findById(event.getOrderId())
        .orElseThrow();  // 데이터 없음! 트랜잭션 아직 커밋 전
    emailService.send(order);
}
```

```
@TransactionalEventListener가 해결하는 것:
  이벤트 발행은 트랜잭션 중에
  리스너 실행은 트랜잭션 완료(커밋/롤백) 후에
  → 커밋 후 DB 데이터 확정된 상태에서 안전하게 처리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @TransactionalEventListener 리스너도 같은 트랜잭션에서 실행된다

```java
// ❌ 잘못된 이해
@TransactionalEventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    // "AFTER_COMMIT이지만 같은 트랜잭션 안"
    orderRepository.updateStatus(event.getOrderId(), "CONFIRMED");
    // 이 변경이 원래 트랜잭션에 포함된다고 생각
}
```

```
✅ 실제:
  AFTER_COMMIT 단계에서 호출되는 시점은 원래 트랜잭션이 커밋 완료 후
  → 원래 트랜잭션 컨텍스트는 이미 닫혀 있음
  → 리스너 내부의 DB 작업은 트랜잭션 없이 실행됨 (또는 새 트랜잭션 필요)

  트랜잭션 필요 시:
  @TransactionalEventListener
  @Transactional(propagation = Propagation.REQUIRES_NEW)  // 새 트랜잭션 시작
  public void onOrderPlaced(OrderPlacedEvent event) { ... }
```

### Before: 트랜잭션 없이 publishEvent()해도 @TransactionalEventListener가 실행된다

```java
// ❌ 잘못된 이해:
// "트랜잭션 없이 발행해도 AFTER_COMMIT으로 실행된다"

// ✅ 실제:
// 트랜잭션 없는 컨텍스트에서 publishEvent()
// → TransactionSynchronization 등록 불가 (활성 트랜잭션 없음)
// → fallbackExecution = false (기본) → 리스너 실행 안 됨 (무시)
// → fallbackExecution = true → 즉시 실행 (일반 @EventListener처럼)
```

---

## ✨ 올바른 이해와 사용

### After: TransactionSynchronization 등록 → 콜백 호출 전체 흐름

```
@Transactional 메서드 실행 중
  → publisher.publishEvent(event)
    → ApplicationListenerMethodTransactionalAdapter.onApplicationEvent()
      → 현재 활성 트랜잭션 있는가? (TransactionSynchronizationManager 확인)
        Yes → TransactionSynchronizationManager.registerSynchronization()
              EventTransactionSynchronization 등록
              (이벤트 저장, phase 정보 보관)
              → 즉시 반환 (리스너 실행 안 함!)
        No  → fallbackExecution 설정에 따라
              false → 무시 (리스너 실행 안 함)
              true  → 즉시 실행
  ↓
@Transactional 메서드 완료 → 트랜잭션 커밋 진행
  TransactionSynchronizationUtils.triggerBeforeCommit()
    → phase == BEFORE_COMMIT인 리스너 실행
  ↓
  커밋 실행 (실제 DB flush + commit)
  ↓
  TransactionSynchronizationUtils.triggerAfterCommit()
    → phase == AFTER_COMMIT인 리스너 실행
  TransactionSynchronizationUtils.triggerAfterCompletion(STATUS_COMMITTED)
    → phase == AFTER_COMPLETION인 리스너 실행
  ↓
롤백 시:
  TransactionSynchronizationUtils.triggerAfterCompletion(STATUS_ROLLED_BACK)
    → phase == AFTER_ROLLBACK인 리스너 실행
    → phase == AFTER_COMPLETION인 리스너 실행
```

---

## 🔬 내부 동작 원리

### 1. @TransactionalEventListener — 내부 구조

```java
// @TransactionalEventListener = @EventListener + 트랜잭션 phase 바인딩
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@EventListener
public @interface TransactionalEventListener {

    TransactionPhase phase() default TransactionPhase.AFTER_COMMIT;
    // BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK, AFTER_COMPLETION

    boolean fallbackExecution() default false;
    // true: 트랜잭션 없이 발행 시 즉시 실행
    // false: 트랜잭션 없으면 무시 (기본)

    // @EventListener의 속성 상속
    Class<?>[] classes() default {};
    String condition() default "";
}
```

```java
// TransactionPhase 각 단계 의미
public enum TransactionPhase {
    BEFORE_COMMIT,
    // 커밋 직전 — 같은 트랜잭션에서 DB 작업 가능 (아직 커밋 전)
    // 여기서 예외 발생 → 트랜잭션 롤백 가능

    AFTER_COMMIT,
    // 커밋 완료 후 — DB 변경 확정 상태
    // 가장 일반적으로 사용 (이메일, 알림 등)
    // 여기서 예외 발생 → 이미 커밋됨, 롤백 불가

    AFTER_ROLLBACK,
    // 롤백 후 — 보상 처리, 실패 알림
    // 롤백 시에만 실행 (커밋 시 실행 안 됨)

    AFTER_COMPLETION
    // 커밋 또는 롤백 완료 후 — 항상 실행
    // 리소스 정리, 감사 로그 등
}
```

### 2. ApplicationListenerMethodTransactionalAdapter — 이벤트 수신 처리

```java
// TransactionalEventListenerFactory가 생성하는 어댑터
public class ApplicationListenerMethodTransactionalAdapter
        extends ApplicationListenerMethodAdapter {

    private final TransactionPhase phase;
    private final boolean fallbackExecution;

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // 활성 트랜잭션이 있으면 동기화 등록
        if (TransactionSynchronizationManager.isSynchronizationActive()
                && TransactionSynchronizationManager.isActualTransactionActive()) {

            TransactionSynchronization synchronization =
                new TransactionalApplicationListenerSynchronization(event, this, ...);
            TransactionSynchronizationManager.registerSynchronization(synchronization);
            // → 이벤트를 큐에 보관, 실제 실행은 트랜잭션 완료 시

        } else if (this.fallbackExecution) {
            // 트랜잭션 없고 fallbackExecution=true → 즉시 실행
            processEvent(event);

        } else {
            // 트랜잭션 없고 fallbackExecution=false → 무시
            if (logger.isDebugEnabled()) {
                logger.debug("No transaction is active - skipping " + event);
            }
        }
    }
}
```

### 3. TransactionalApplicationListenerSynchronization — 실행 트리거

```java
class TransactionalApplicationListenerSynchronization
        implements TransactionSynchronization {

    private final ApplicationEvent event;
    private final ApplicationListenerMethodTransactionalAdapter listener;

    @Override
    public void beforeCommit(boolean readOnly) {
        if (this.listener.getTransactionPhase() == TransactionPhase.BEFORE_COMMIT) {
            processEventWithErrorHandling();
        }
    }

    @Override
    public void afterCommit() {
        if (this.listener.getTransactionPhase() == TransactionPhase.AFTER_COMMIT) {
            processEventWithErrorHandling();
        }
    }

    @Override
    public void afterCompletion(int status) {
        TransactionPhase phase = this.listener.getTransactionPhase();

        if (phase == TransactionPhase.AFTER_COMPLETION) {
            processEventWithErrorHandling();
        }
        if (phase == TransactionPhase.AFTER_ROLLBACK
                && status == STATUS_ROLLED_BACK) {
            processEventWithErrorHandling();
        }
    }

    private void processEventWithErrorHandling() {
        try {
            this.listener.processEvent(this.event);
        } catch (Exception ex) {
            handleException(ex);
        }
    }
}
```

### 4. BEFORE_COMMIT — 같은 트랜잭션에서 추가 처리

```java
// BEFORE_COMMIT: 커밋 직전 → 같은 트랜잭션 컨텍스트 접근 가능
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void beforeCommit(OrderPlacedEvent event) {
    // 이 시점은 아직 커밋 전 → 같은 트랜잭션 안
    // → DB 추가 작업이 원래 트랜잭션에 포함됨

    AuditLog log = new AuditLog(event.getOrderId(), "ORDER_PLACED");
    auditRepository.save(log);  // 원래 트랜잭션에 포함됨

    // 예외 발생 시 → 원래 트랜잭션 롤백 가능
}
```

### 5. AFTER_COMMIT + @Async — 가장 안전한 패턴

```java
// 커밋 후 비동기 처리: AFTER_COMMIT + @Async
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void afterCommitAsync(OrderPlacedEvent event) {
    // 1. 커밋 완료 후 실행 → DB 데이터 확정
    // 2. 별도 스레드 (@Async) → 발행자 비블로킹
    // 3. 새 트랜잭션 (REQUIRES_NEW) → 독립적 DB 작업

    Order order = orderRepository.findById(event.getOrderId())
        .orElseThrow();  // 커밋 후이므로 안전하게 조회
    emailService.send(order.getEmail());
}
```

```
주의사항:
  @Async 없이 AFTER_COMMIT:
    afterCommit() 콜백이 원래 트랜잭션 완료 스레드에서 실행
    → 동기 실행 → 발행자 블로킹
    → 리스너 예외가 트랜잭션 완료 흐름에 영향 가능

  @Async + AFTER_COMMIT:
    별도 스레드에서 커밋 후 실행
    → 발행자 즉시 반환
    → 리스너 예외 분리 (AsyncUncaughtExceptionHandler)
    → 단, 새 트랜잭션(@Transactional) 선언 필요
```

### 6. fallbackExecution 활용

```java
// 테스트 환경 — 트랜잭션 없이 이벤트 테스트
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT,
    fallbackExecution = true  // 트랜잭션 없을 때도 즉시 실행
)
public void onOrderPlaced(OrderPlacedEvent event) {
    sendEmail(event);
}

// 프로덕션: @Transactional 메서드에서 publishEvent() → AFTER_COMMIT 후 실행
// 단위 테스트: 트랜잭션 없이 publishEvent() → fallbackExecution=true → 즉시 실행
// → 테스트 용이성 향상 (하지만 프로덕션 동작과 차이 주의)
```

---

## 💻 실험으로 확인하기

### 실험 1: AFTER_COMMIT phase 동작 확인

```java
@Service
@Transactional
public class OrderService {
    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order.getId()));
        System.out.println("트랜잭션 메서드 완료 전");
    }
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void afterCommit(OrderPlacedEvent e) {
    System.out.println("AFTER_COMMIT 리스너 실행");
    Order order = orderRepository.findById(e.getOrderId()).orElseThrow();
    System.out.println("주문 조회 성공: " + order.getId());
}

// 출력:
// "트랜잭션 메서드 완료 전"
// (커밋 실행)
// "AFTER_COMMIT 리스너 실행"
// "주문 조회 성공: 1"  ← 커밋 후이므로 안전하게 조회
```

### 실험 2: 트랜잭션 없이 발행 (fallbackExecution=false)

```java
// @Transactional 없는 메서드에서 발행
public void publishWithoutTransaction() {
    publisher.publishEvent(new OrderPlacedEvent(1L));
    System.out.println("이벤트 발행 완료");
}

@TransactionalEventListener  // fallbackExecution=false (기본)
public void listener(OrderPlacedEvent e) {
    System.out.println("리스너 실행");
}

publishWithoutTransaction();
// 출력: "이벤트 발행 완료"  (리스너 실행 안 됨 — 무시)
```

### 실험 3: BEFORE_COMMIT에서 예외 → 롤백

```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void beforeCommit(OrderPlacedEvent e) {
    throw new RuntimeException("검증 실패");
    // → 트랜잭션 롤백 발생
}

// @Transactional 메서드에서 publishEvent() 후 커밋 시도
// → BEFORE_COMMIT 리스너 예외 → 트랜잭션 롤백
// → orderRepository.save()도 롤백됨
```

---

## 🤔 트레이드오프

```
BEFORE_COMMIT:
  장점  같은 트랜잭션 → 예외 시 롤백 가능, 원자성 보장
  단점  느린 처리가 커밋을 지연, 예외 처리 복잡
  사용  커밋 전 추가 검증, 감사 로그 (동일 트랜잭션 포함 필요)

AFTER_COMMIT:
  장점  DB 확정 후 처리 → 데이터 일관성 보장
  단점  원래 트랜잭션 롤백 불가 (이미 커밋됨)
        예외 발생 시 보상 트랜잭션 필요
  사용  이메일, 외부 API 호출, 캐시 갱신 (가장 일반적)

AFTER_ROLLBACK:
  사용  롤백 후 보상 처리, 실패 알림, 잔여 처리 정리

AFTER_COMPLETION:
  사용  커밋/롤백 무관 리소스 정리, 감사 로그 (항상 기록)

fallbackExecution:
  false (기본): 트랜잭션 없으면 무시 → 일관성 보장
  true: 트랜잭션 없어도 실행 → 테스트 용이, 프로덕션 동작 차이 주의

@Async + @TransactionalEventListener:
  AFTER_COMMIT + @Async: 커밋 후 비동기 → 이메일 등 비핵심 처리 권장
  BEFORE_COMMIT + @Async: 의미 없음 (비동기라 커밋 전 실행 보장 불가)
```

---

## 📌 핵심 정리

```
동작 핵심
  onApplicationEvent() 호출 시:
    활성 트랜잭션 있음 → TransactionSynchronization 등록 (이벤트 큐잉)
    없음 + fallbackExecution=false → 무시
    없음 + fallbackExecution=true → 즉시 실행

TransactionSynchronization 콜백 순서
  beforeCommit()   → BEFORE_COMMIT 리스너
  [실제 커밋]
  afterCommit()    → AFTER_COMMIT 리스너
  afterCompletion(STATUS_COMMITTED) → AFTER_COMPLETION 리스너
  afterCompletion(STATUS_ROLLED_BACK) → AFTER_ROLLBACK + AFTER_COMPLETION

Phase별 트랜잭션 상태
  BEFORE_COMMIT  원래 트랜잭션 활성 → DB 작업 포함 가능, 예외 시 롤백
  AFTER_COMMIT   트랜잭션 닫힘 → 새 트랜잭션(@Transactional REQUIRES_NEW) 필요
  AFTER_ROLLBACK 트랜잭션 닫힘 → 보상 처리용
  AFTER_COMPLETION 항상 실행 → 리소스 정리용

권장 패턴
  커밋 후 비동기 처리:
  @Async + @TransactionalEventListener(AFTER_COMMIT) + @Transactional(REQUIRES_NEW)
```

---

## 🤔 생각해볼 문제

**Q1.** `AFTER_COMMIT` 리스너에서 `@Transactional` 없이 `orderRepository.save()`를 호출하면 어떻게 동작하는가?

**Q2.** `AFTER_COMMIT` 리스너에서 예외가 발생하면 원래 트랜잭션이 롤백되는가?

**Q3.** 하나의 트랜잭션에서 같은 이벤트를 여러 번 발행하면 리스너가 여러 번 실행되는가?

> 💡 **해설**
>
> **Q1.** `AFTER_COMMIT` 시점은 원래 트랜잭션이 커밋 완료된 후로, 트랜잭션 컨텍스트가 정리된 상태다. `@Transactional` 없이 `save()`를 호출하면 `SimpleJpaRepository.save()`가 트랜잭션 없이 실행된다. JPA의 경우 트랜잭션 없이는 flush가 안 되어 `TransactionRequiredException`이 발생할 수 있다. 또는 `@Transactional(propagation = SUPPORTS)`가 적용된 경우 트랜잭션 없이 실행되어 예상과 다른 동작이 발생한다. `AFTER_COMMIT` 리스너에서 DB 작업이 필요하면 반드시 `@Transactional(propagation = REQUIRES_NEW)`를 선언해 새 트랜잭션을 시작해야 한다.
>
> **Q2.** 원래 트랜잭션은 이미 커밋 완료 상태이므로 롤백되지 않는다. `AFTER_COMMIT` 콜백에서 발생한 예외는 `AbstractPlatformTransactionManager`의 커밋 완료 흐름에서 처리되는데, 기본적으로 예외가 로그에 기록되고 무시되거나 `afterCompletion()` 처리로 넘어간다. 원래 트랜잭션을 롤백하려면 `BEFORE_COMMIT` 단계를 사용해야 한다. 이 차이가 `BEFORE_COMMIT`과 `AFTER_COMMIT`을 선택할 때 핵심 기준이 된다.
>
> **Q3.** 이벤트 발행마다 `TransactionSynchronizationManager.registerSynchronization()`이 호출되어 각각의 `EventTransactionSynchronization`이 등록된다. 동기화 목록에 같은 이벤트 타입의 동기화가 여러 개 쌓이고, 커밋 시 모두 실행된다. 즉 이벤트를 3번 발행하면 `AFTER_COMMIT` 리스너도 3번 실행된다. 중복 처리를 막으려면 이벤트에 식별자를 포함시키고 리스너에서 멱등성 처리를 구현해야 한다.

---

<div align="center">

**[⬅️ 이전: 이벤트 실행 순서 보장](./04-event-ordering.md)** | **[홈으로 🏠](../README.md)**

</div>

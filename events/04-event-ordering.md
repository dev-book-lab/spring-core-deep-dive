# 이벤트 실행 순서 보장 — @Order와 순서가 깨지는 경우

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Order` / `Ordered`가 리스너 실행 순서에 영향을 미치는 정확한 메커니즘은?
- `AnnotationAwareOrderComparator`가 리스너를 정렬하는 시점은?
- 같은 `@Order` 값을 가진 리스너들의 순서는 보장되는가?
- 비동기 리스너에서는 `@Order`가 의미 없는 이유는?
- `SmartApplicationListener`와 일반 `ApplicationListener`의 순서 처리 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 여러 리스너가 같은 이벤트를 처리할 때 순서가 중요한 경우가 있다

```java
// 주문 확정 이벤트 리스너 3개
// 1. 재고 차감 먼저
// 2. 결제 처리
// 3. 이메일 발송 (앞 두 단계 성공 후)

// @Order 없으면 실행 순서 비결정적
@EventListener void deductInventory(OrderPlacedEvent e) { ... }
@EventListener void processPayment(OrderPlacedEvent e) { ... }
@EventListener void sendEmail(OrderPlacedEvent e) { ... }
```

```
@Order가 해결하는 것:
  숫자가 낮을수록 먼저 실행 (Ordered.HIGHEST_PRECEDENCE = Integer.MIN_VALUE)
  → 선행 처리 완료 후 후속 처리 보장
  → 예외 발생 시 남은 리스너 실행 중단 (동기 환경)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Order가 없으면 등록 순서대로 실행된다

```
❌ 잘못된 이해:
  "@Order가 없으면 Bean 등록 순서 = 실행 순서"

✅ 실제:
  @Order 없는 리스너는 AnnotationAwareOrderComparator에서
  Ordered.LOWEST_PRECEDENCE(Integer.MAX_VALUE)로 취급
  → 같은 값이면 내부 정렬(stable sort)에 의존
  → JVM/클래스패스 로딩 순서에 따라 달라질 수 있음
  → 프로덕션 환경에서 순서 의존 로직에 @Order 없으면 위험
```

### Before: 비동기 리스너에서 @Order로 순서를 제어할 수 있다

```java
// ❌ 잘못된 이해
@Order(1)
@Async
@EventListener
public void firstListener(MyEvent e) { ... }

@Order(2)
@Async
@EventListener
public void secondListener(MyEvent e) { ... }
// "firstListener가 완료된 후 secondListener 실행"
```

```
✅ 실제:
  @Async 리스너는 별도 스레드에서 실행
  → "실행 제출 순서"는 @Order로 제어 가능 (어떤 리스너를 먼저 submit)
  → 하지만 "완료 순서"는 스레드 스케줄러 의존
  → Order(1) 리스너가 먼저 submit되지만 늦게 끝날 수 있음
  → 비동기에서 완료 순서 보장 불가
```

---

## ✨ 올바른 이해와 사용

### After: 정렬 적용 위치와 동작 범위

```
정렬 적용 위치:
  AbstractApplicationEventMulticaster.retrieveApplicationListeners()
  → 리스너 탐색 완료 후 AnnotationAwareOrderComparator.sort(allListeners)
  → 결과를 retrieverCache에 저장 (캐시된 순서 재사용)

정렬 기준 (AnnotationAwareOrderComparator):
  1. Ordered 인터페이스 구현 → getOrder() 값
  2. @Order / @Priority 어노테이션 → value()
  3. 없으면 Ordered.LOWEST_PRECEDENCE (Integer.MAX_VALUE)
  → 값이 작을수록 먼저 실행

보장 범위:
  동기 실행 → @Order로 실행 순서 보장
  비동기 실행 → submit 순서만 보장, 완료 순서 미보장
```

---

## 🔬 내부 동작 원리

### 1. AnnotationAwareOrderComparator — 정렬 메커니즘

```java
// AbstractApplicationEventMulticaster.retrieveApplicationListeners()
private Collection<ApplicationListener<?>> retrieveApplicationListeners(...) {
    List<ApplicationListener<?>> allListeners = new ArrayList<>();

    // 리스너 수집 (생략)...

    // 정렬 적용
    AnnotationAwareOrderComparator.sort(allListeners);
    // → @Order, @Priority, Ordered 인터페이스 순서대로 정렬

    return allListeners;
}
```

```java
// AnnotationAwareOrderComparator.getOrder() — 우선순위 결정
protected Integer findOrder(Object obj) {
    // 1. Ordered 인터페이스 직접 구현
    if (obj instanceof Ordered ordered) {
        return ordered.getOrder();
    }

    // 2. @Order 어노테이션 탐색 (클래스 레벨)
    Order order = AnnotationUtils.findAnnotation(obj.getClass(), Order.class);
    if (order != null) return order.value();

    // 3. ApplicationListenerMethodAdapter → 메서드 레벨 @Order
    if (obj instanceof ApplicationListenerMethodAdapter alma) {
        return alma.getOrder();
        // → @EventListener 메서드에 붙은 @Order 값
    }

    // 4. 없으면 LOWEST_PRECEDENCE
    return Ordered.LOWEST_PRECEDENCE;
}
```

### 2. @EventListener + @Order 적용 방식

```java
// 방법 1: 메서드 레벨 @Order
@Component
public class OrderEventListener {

    @Order(1)
    @EventListener
    public void step1(OrderPlacedEvent e) {
        System.out.println("1단계: 재고 차감");
    }

    @Order(2)
    @EventListener
    public void step2(OrderPlacedEvent e) {
        System.out.println("2단계: 결제 처리");
    }

    @Order(3)
    @EventListener
    public void step3(OrderPlacedEvent e) {
        System.out.println("3단계: 이메일 발송");
    }
}
// 출력: 1단계 → 2단계 → 3단계 (보장)
```

```java
// 방법 2: 클래스 레벨 @Order (ApplicationListener<T> 구현 시)
@Order(1)
@Component
public class InventoryListener implements ApplicationListener<OrderPlacedEvent> {
    @Override
    public void onApplicationEvent(OrderPlacedEvent event) {
        System.out.println("재고 차감");
    }
}

@Order(2)
@Component
public class PaymentListener implements ApplicationListener<OrderPlacedEvent> {
    @Override
    public void onApplicationEvent(OrderPlacedEvent event) {
        System.out.println("결제 처리");
    }
}
```

### 3. ApplicationListenerMethodAdapter.getOrder()

```java
// ApplicationListenerMethodAdapter의 order 결정
private static int resolveOrder(Method method) {
    // 메서드 레벨 @Order 확인
    Order ann = AnnotatedElementUtils.findMergedAnnotation(method, Order.class);
    return (ann != null ? ann.value() : Ordered.LOWEST_PRECEDENCE);
}

// AnnotationAwareOrderComparator에서 이 값 사용
```

### 4. SmartApplicationListener — 더 세밀한 제어

```java
// SmartApplicationListener = ApplicationListener + 이벤트/소스 타입 필터 + 순서
public interface SmartApplicationListener extends ApplicationListener<ApplicationEvent>, Ordered {

    boolean supportsEventType(Class<? extends ApplicationEvent> eventType);
    default boolean supportsSourceType(Class<?> sourceType) { return true; }
    // getOrder() 상속 (Ordered)
}

// 구현 예
@Component
public class PrioritizedOrderListener implements SmartApplicationListener {

    @Override
    public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        return OrderPlacedEvent.class.isAssignableFrom(eventType);
    }

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // OrderPlacedEvent만 처리
    }

    @Override
    public int getOrder() {
        return 10;  // @Order(10)과 동일 효과
    }
}
```

### 5. 동일 @Order 값 → 순서 미보장

```java
@Order(1)
@EventListener
public void listenerA(OrderPlacedEvent e) { System.out.println("A"); }

@Order(1)  // A와 동일 값
@EventListener
public void listenerB(OrderPlacedEvent e) { System.out.println("B"); }

// AnnotationAwareOrderComparator는 안정 정렬(stable sort)이 아닐 수 있음
// → A, B 중 어느 것이 먼저 실행될지 보장 안 됨
// → 같은 값이 필요하다면 서로 순서 의존 없이 설계해야 함
```

### 6. 순서가 보장되지 않는 케이스

```java
// 케이스 1: @Async 비동기 리스너
@Order(1)
@Async
@EventListener
public void firstAsync(MyEvent e) {
    Thread.sleep(200);  // 느린 처리
}

@Order(2)
@Async
@EventListener
public void secondAsync(MyEvent e) {
    Thread.sleep(10);   // 빠른 처리
}
// 완료 순서: secondAsync → firstAsync (Order(1)이 늦게 끝남)
// submit 순서: firstAsync → secondAsync (Order 준수)
// 완료 순서 보장 불가

// 케이스 2: Multicaster에 Executor 설정
// → 모든 리스너가 비동기 → 완료 순서 미보장

// 케이스 3: 다른 ApplicationContext 계층에서 온 리스너
// → 부모/자식 컨텍스트의 리스너 순서는 별도 관리
// → 계층 간 @Order 비교 미보장
```

---

## 💻 실험으로 확인하기

### 실험 1: @Order 효과 확인

```java
@Component
public class OrderedListeners {

    @Order(3)
    @EventListener
    public void third(TestEvent e) { System.out.println("3번째"); }

    @Order(1)
    @EventListener
    public void first(TestEvent e) { System.out.println("1번째"); }

    @Order(2)
    @EventListener
    public void second(TestEvent e) { System.out.println("2번째"); }
}

publisher.publishEvent(new TestEvent(this));
// 출력: 1번째 → 2번째 → 3번째 (@Order 값 오름차순)
```

### 실험 2: 1번 리스너 예외 시 2, 3번 리스너 미실행

```java
@Order(1)
@EventListener
public void first(TestEvent e) {
    throw new RuntimeException("1번 실패");
}

@Order(2)
@EventListener
public void second(TestEvent e) {
    System.out.println("2번 (실행될까?)");
}

publisher.publishEvent(new TestEvent(this));
// → RuntimeException 전파 → 2번 리스너 실행 안 됨
// → ErrorHandler 없으면 발행자로 예외 전파
```

### 실험 3: Ordered.HIGHEST_PRECEDENCE로 가장 먼저 실행

```java
@Order(Ordered.HIGHEST_PRECEDENCE)  // Integer.MIN_VALUE
@EventListener
public void veryFirst(TestEvent e) {
    System.out.println("항상 가장 먼저");
}

@Order(Ordered.LOWEST_PRECEDENCE)   // Integer.MAX_VALUE
@EventListener
public void veryLast(TestEvent e) {
    System.out.println("항상 가장 나중");
}
```

---

## 🤔 트레이드오프

```
@Order 사용:
  장점  명확한 실행 순서 선언, 의도 문서화
  단점  리스너 간 묵시적 결합 (순서에 의존하는 설계)
       여러 모듈의 리스너 순서 조율 복잡

순서 의존 설계의 위험:
  리스너 A가 먼저 실행되고 리스너 B가 A의 결과를 사용하는 패턴
  → B에서 A 결과를 어떻게 받는가? 공유 상태? 이벤트 체인?
  → 이벤트 기반 설계 원칙에 위배 (느슨한 결합)
  → 순서 의존이 필요하다면 직접 의존이 더 명확할 수 있음

@Order 대안:
  1. 이벤트 체인: 1번 리스너 완료 후 새 이벤트 발행 → 2번 리스너
  2. 직접 의존: 순서가 중요하면 서비스 메서드로 명시적 호출
  3. 트랜잭션 단계별 이벤트: @TransactionalEventListener phase 활용

비동기 리스너 순서:
  submit 순서 제어(의미 없음) vs 완료 순서 제어(불가)
  → 비동기 리스너 간 순서 의존 설계 금지
  → 각 비동기 리스너는 독립적이어야 함
```

---

## 📌 핵심 정리

```
순서 결정 메커니즘
  retrieveApplicationListeners() 완료 후
  AnnotationAwareOrderComparator.sort() 적용
  → 결과 retrieverCache에 저장 (재정렬 없음)

우선순위 결정 (낮을수록 먼저 실행)
  Ordered.getOrder() > @Order > @Priority > LOWEST_PRECEDENCE(기본)
  HIGHEST_PRECEDENCE = Integer.MIN_VALUE
  LOWEST_PRECEDENCE  = Integer.MAX_VALUE

순서 보장 범위
  동기 리스너 → @Order로 완전히 보장
  비동기 리스너(@Async) → submit 순서만, 완료 순서 미보장
  같은 @Order 값 → 순서 미보장

동일 값 충돌
  같은 order 값 → 안정 정렬 미보장
  → 순서 의존 로직에서 고유한 order 값 사용 필수

SmartApplicationListener
  ApplicationListener + 이벤트/소스 필터 + Ordered 통합
  → 가장 세밀한 순서/필터 제어
```

---

## 🤔 생각해볼 문제

**Q1.** `retrieverCache`에 캐시된 순서는 리스너가 동적으로 추가되면 갱신되는가?

**Q2.** `@Order(1)` 리스너에서 새 이벤트를 발행하면 그 이벤트의 리스너들은 현재 이벤트 처리 중에 실행되는가?

**Q3.** Spring Cloud나 Spring Integration처럼 외부 메시지에서 이벤트를 발행할 때 `@Order` 순서가 보장되는가?

> 💡 **해설**
>
> **Q1.** 갱신된다. `AbstractApplicationEventMulticaster.addApplicationListener()` 또는 `removeApplicationListener()`가 호출되면 `retrieverCache.clear()`가 실행되어 캐시 전체가 무효화된다. 다음 이벤트 발행 시 `getApplicationListeners()`에서 캐시 미스가 발생하고 `retrieveApplicationListeners()`가 다시 전체 탐색과 정렬을 수행하여 새 리스너를 포함한 결과를 캐시한다. 캐시 갱신 비용은 리스너가 드물게 추가/제거된다면 무시할 수 있는 수준이다.
>
> **Q2.** 동기 이벤트의 경우 발행 스레드에서 처리가 이루어지므로, `@Order(1)` 리스너 내에서 새 이벤트를 발행하면 그 즉시 `publishEvent()` → `multicastEvent()`가 호출된다. 새 이벤트의 리스너들이 모두 실행 완료된 후에야 원래 이벤트의 `@Order(2)` 리스너로 제어가 돌아온다. 즉 이벤트 처리가 중첩(재귀적으로)으로 실행된다. 이 동작은 의도한 이벤트 체인 패턴으로도 활용 가능하지만, 깊은 중첩은 스택 오버플로우나 추적하기 어려운 흐름을 만들 수 있다.
>
> **Q3.** 보장된다. `@Order`는 `ApplicationEventMulticaster.getApplicationListeners()`에서 적용되며, 이벤트가 어디서 발행되었든 관계없이 동일하게 동작한다. Spring Cloud Stream이나 Spring Integration이 내부적으로 `ApplicationEventPublisher.publishEvent()`를 사용하면 같은 멀티캐스터를 통하므로 `@Order` 순서가 적용된다. 단 외부 메시지 처리가 별도 스레드에서 발행되는 비동기 구조라면, 동시에 여러 이벤트가 병렬 처리될 수 있어 서로 다른 이벤트 처리 간 순서는 보장되지 않는다.

---

<div align="center">

**[⬅️ 이전: 비동기 이벤트 처리 @Async](./03-async-event-processing.md)** | **[다음: 트랜잭션 바운드 이벤트 ➡️](./05-transaction-bound-events.md)**

</div>

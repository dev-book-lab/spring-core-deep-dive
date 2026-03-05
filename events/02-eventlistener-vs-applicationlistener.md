# @EventListener vs ApplicationListener — EventListenerMethodProcessor 변환 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EventListenerMethodProcessor`가 `@EventListener` 메서드를 `ApplicationListener`로 변환하는 정확한 시점은?
- `ApplicationListenerMethodAdapter`가 `@EventListener` 메서드를 래핑하는 방식은?
- `@EventListener`의 `condition` SpEL 표현식은 언제 평가되는가?
- `ApplicationListener<T>` 제네릭 타입 추론과 `@EventListener` 파라미터 타입 추론의 차이는?
- 이벤트 처리 후 반환값으로 새 이벤트를 발행하는 방식은?

---

## 🔍 왜 이게 존재하는가

### 문제: ApplicationListener 인터페이스 구현은 반복 코드가 많다

```java
// 인터페이스 기반 — 장황함
@Component
public class OrderEventListener implements ApplicationListener<OrderPlacedEvent> {

    @Override
    public void onApplicationEvent(OrderPlacedEvent event) {
        // 이벤트 처리
    }
}
// 단점: 이벤트 타입마다 별도 클래스, 단일 메서드 제약

// @EventListener — 간결함
@Component
public class OrderEventListener {

    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) { ... }

    @EventListener
    public void onOrderCancelled(OrderCancelledEvent event) { ... }
    // 하나의 클래스에 여러 이벤트 처리 가능
}
```

```
@EventListener가 추가로 제공하는 것:
  condition: SpEL 조건부 실행
  classes: 여러 이벤트 타입 동시 구독
  반환값: 새 이벤트 즉시 발행
  @Async 결합: 비동기 처리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @EventListener 메서드는 직접 호출된다

```
❌ 잘못된 이해:
  "스프링이 @EventListener 붙은 메서드를 직접 찾아서 invoke()한다"

✅ 실제:
  EventListenerMethodProcessor가 시작 시 @EventListener 메서드를 탐색
  → 각 메서드를 ApplicationListenerMethodAdapter로 래핑
  → 래핑된 ApplicationListener를 컨텍스트에 등록
  → 이벤트 발행 시 ApplicationListener.onApplicationEvent() 호출
  → 내부에서 원본 @EventListener 메서드 invoke()

  즉: @EventListener → 어댑터 패턴 → ApplicationListener 인프라 재사용
```

### Before: condition SpEL은 항상 평가된다

```java
@EventListener(condition = "#event.order.amount > 1000")
public void onLargeOrder(OrderPlacedEvent event) { ... }
```

```
❌ 잘못된 이해: "모든 OrderPlacedEvent에서 SpEL 평가"

✅ 실제:
  리스너 탐색(supportsEvent) 단계: 이벤트 타입만 체크 → 조건 미평가
  invokeListener 단계: ApplicationListenerMethodAdapter.processEvent()
    → shouldHandle() → evaluateCondition()
    → SpEL 평가: false이면 메서드 호출 건너뜀

  캐시:
  → 동일 condition 표현식은 ExpressionParser 파싱 결과 캐시
  → SpEL 평가 자체는 매 이벤트마다 실행
```

---

## ✨ 올바른 이해와 사용

### After: 변환 파이프라인 전체 구조

```
컨텍스트 refresh() → finishBeanFactoryInitialization()
  → Singleton Bean 초기화 완료
  ↓
SmartInitializingSingleton.afterSingletonsInstantiated()
  → EventListenerMethodProcessor.afterSingletonsInstantiated()
    → 모든 Bean 순회
    → processBean(beanName, targetType)
      → @EventListener 메서드 탐색 (AnnotationUtils.findAnnotation)
      → 각 메서드마다:
        EventListenerFactory.createApplicationListener()
          → ApplicationListenerMethodAdapter 생성
        context.addApplicationListener(adapter)
        → SimpleApplicationEventMulticaster에 등록
```

---

## 🔬 내부 동작 원리

### 1. EventListenerMethodProcessor — @EventListener 탐색 및 등록

```java
// EventListenerMethodProcessor implements SmartInitializingSingleton
// SmartInitializingSingleton: 모든 싱글톤 Bean 초기화 완료 후 콜백

public class EventListenerMethodProcessor
        implements SmartInitializingSingleton, ApplicationContextAware {

    @Override
    public void afterSingletonsInstantiated() {
        ConfigurableListableBeanFactory beanFactory = this.beanFactory;
        String[] beanNames = beanFactory.getBeanNamesForType(Object.class);

        for (String beanName : beanNames) {
            // CGLIB 서브클래스 → 원본 클래스로 변환
            Class<?> type = AutoProxyUtils.determineTargetClass(beanFactory, beanName);

            processBean(beanName, type);
        }
    }

    private void processBean(String beanName, Class<?> targetType) {
        // @EventListener 메서드 탐색 (캐시됨)
        Map<Method, EventListener> annotatedMethods =
            MethodIntrospector.selectMethods(targetType,
                (MethodIntrospector.MetadataLookup<EventListener>) method ->
                    AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));

        if (annotatedMethods.isEmpty()) return;

        for (Map.Entry<Method, EventListener> entry : annotatedMethods.entrySet()) {
            Method method = entry.getKey();
            EventListener eventListener = entry.getValue();

            // EventListenerFactory로 ApplicationListener 생성
            for (EventListenerFactory factory : this.eventListenerFactories) {
                if (factory.supportsMethod(method)) {
                    Method methodToUse = AopUtils.selectInvocableMethod(method,
                        this.beanFactory.getType(beanName));
                    ApplicationListener<?> listener =
                        factory.createApplicationListener(beanName, targetType, methodToUse);

                    // 초기화 (BeanFactory 주입 등)
                    if (listener instanceof ApplicationListenerMethodAdapter alma) {
                        alma.init(this.applicationContext, this.evaluator);
                    }

                    // 컨텍스트에 등록
                    this.applicationContext.addApplicationListener(listener);
                    break;
                }
            }
        }
    }
}
```

### 2. ApplicationListenerMethodAdapter — 핵심 어댑터

```java
// ApplicationListenerMethodAdapter implements GenericApplicationListener
public class ApplicationListenerMethodAdapter implements GenericApplicationListener {

    private final String beanName;
    private final Method method;
    private final Method bridgedMethod;
    private final List<ResolvableType> declaredEventTypes;  // 파라미터 타입
    private final String condition;                          // SpEL 조건
    private final int order;                                 // @Order

    public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass,
                                             Method method) {
        this.beanName = beanName;
        this.method = BridgeMethodResolver.findBridgedMethod(method);

        // @EventListener의 파라미터 타입 또는 classes 속성으로 이벤트 타입 결정
        EventListener ann = AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class);
        this.declaredEventTypes = resolveDeclaredEventTypes(method, ann);
        // → 파라미터 타입: onOrderPlaced(OrderPlacedEvent) → OrderPlacedEvent
        // → @EventListener(classes = {A.class, B.class}) → [A, B]

        this.condition = (ann != null ? ann.condition() : null);
        this.order = resolveOrder(method);
    }

    // GenericApplicationListener.supportsEventType()
    @Override
    public boolean supportsEventType(ResolvableType eventType) {
        for (ResolvableType declaredEventType : this.declaredEventTypes) {
            if (declaredEventType.isAssignableFrom(eventType)) return true;
            // PayloadApplicationEvent 처리
            if (PayloadApplicationEvent.class.isAssignableFrom(eventType.toClass())) {
                ResolvableType payloadType = eventType.as(PayloadApplicationEvent.class).getGeneric();
                if (declaredEventType.isAssignableFrom(payloadType)) return true;
            }
        }
        return false;
    }

    // 이벤트 수신 → 실제 처리
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        processEvent(event);
    }

    public void processEvent(ApplicationEvent event) {
        // 이벤트 → 메서드 인수 변환
        Object[] args = resolveArguments(event);

        // SpEL condition 평가
        if (shouldHandle(event, args)) {
            // 실제 메서드 호출
            Object result = doInvoke(args);

            // 반환값이 있으면 새 이벤트로 발행
            if (result != null) {
                handleResult(result);
            }
        }
    }
}
```

### 3. resolveArguments() — 이벤트 → 메서드 인수 변환

```java
private Object[] resolveArguments(ApplicationEvent event) {
    ResolvableType declaredEventType = getResolvableType(event);
    if (declaredEventType == null) return null;

    if (this.method.getParameterCount() == 0) {
        return new Object[0];
        // 파라미터 없는 @EventListener → 인수 없이 호출
    }

    Class<?> declaredEventClass = declaredEventType.toClass();

    if (ApplicationEvent.class.isAssignableFrom(declaredEventClass)
            || event.getClass().isAssignableFrom(declaredEventClass)) {
        // ApplicationEvent 타입 파라미터 → 이벤트 직접 전달
        return new Object[]{event};
    }

    // POJO 이벤트 (PayloadApplicationEvent) → payload 추출
    if (event instanceof PayloadApplicationEvent<?> pae) {
        return new Object[]{pae.getPayload()};
    }

    return new Object[]{event};
}
```

### 4. shouldHandle() — SpEL condition 평가

```java
private boolean shouldHandle(ApplicationEvent event, Object[] args) {
    if (this.condition == null || this.condition.isBlank()) {
        return true;  // 조건 없음 → 항상 처리
    }

    // SpEL 평가 컨텍스트 구성
    EvaluationContext evaluationContext =
        this.evaluator.createEvaluationContext(
            this.bean, this.targetClass, this.method, args, event, this.beanFactory);

    // SpEL 표현식 평가
    // "#event.order.amount > 1000" → event 루트 객체로 접근 가능
    return Boolean.TRUE.equals(
        this.evaluator.condition(this.condition, this.methodKey, evaluationContext));
}
```

```
SpEL 컨텍스트에서 사용 가능한 변수:
  #root.event    → ApplicationEvent 객체
  #root.args     → 메서드 인수 배열
  #event         → 첫 번째 파라미터 (이벤트 객체)
  #a0, #p0       → 첫 번째 파라미터 접근 방법

예시:
  @EventListener(condition = "#event.order.amount > 1000")
  @EventListener(condition = "#root.event.source != null")
```

### 5. handleResult() — 반환값으로 연쇄 이벤트 발행

```java
protected void handleResult(Object result) {
    if (result.getClass().isArray()) {
        // 배열 → 각 요소를 이벤트로 발행
        Object[] events = ObjectUtils.toObjectArray(result);
        for (Object event : events) {
            publishEvent(event);
        }
    } else if (result instanceof Collection<?> events) {
        // 컬렉션 → 각 요소를 이벤트로 발행
        for (Object event : events) {
            publishEvent(event);
        }
    } else {
        // 단일 객체 → 이벤트로 발행
        publishEvent(result);
    }
}
```

```java
// 사용 예: 이벤트 변환 / 연쇄 발행
@EventListener
public PaymentRequiredEvent onOrderPlaced(OrderPlacedEvent event) {
    // OrderPlacedEvent → PaymentRequiredEvent 변환 후 즉시 발행
    return new PaymentRequiredEvent(event.getOrder());
}

@EventListener
public List<NotificationEvent> onPaymentCompleted(PaymentCompletedEvent event) {
    // 여러 이벤트 동시 발행
    return List.of(
        new EmailNotificationEvent(event),
        new SmsNotificationEvent(event)
    );
}
```

### 6. ApplicationListener<T> vs @EventListener 타입 추론 비교

```java
// ApplicationListener<T> — 컴파일 타임 제네릭 타입 인수로 결정
@Component
public class OrderListener implements ApplicationListener<OrderPlacedEvent> {
    // GenericTypeResolver.resolveTypeArgument()로 OrderPlacedEvent 추론
    // → 컴파일된 클래스 파일의 시그니처에서 읽음
}

// @EventListener — 메서드 파라미터 타입으로 결정
@Component
public class OrderListener {
    @EventListener
    public void on(OrderPlacedEvent event) {
        // Method.getParameterTypes()[0] = OrderPlacedEvent
        // → 런타임에 메서드 리플렉션으로 읽음
    }
}

// 차이가 생기는 경우: 람다 / 익명 클래스
// ApplicationListener<OrderPlacedEvent> listener = event -> { ... };
// → 람다는 제네릭 타입 정보 소거 → 타입 추론 실패 가능
// → @EventListener 메서드는 항상 파라미터 타입 명확
```

---

## 💻 실험으로 확인하기

### 실험 1: @EventListener 등록 확인

```java
// 등록된 리스너 목록 확인
AbstractApplicationEventMulticaster multicaster =
    ctx.getBean("applicationEventMulticaster",
        AbstractApplicationEventMulticaster.class);

// defaultRetriever에서 ApplicationListenerMethodAdapter 확인
// (reflection으로 접근)
Field field = AbstractApplicationEventMulticaster.class
    .getDeclaredField("defaultRetriever");
field.setAccessible(true);
// → ApplicationListenerMethodAdapter 인스턴스 확인 가능
```

### 실험 2: condition SpEL 동작 확인

```java
@EventListener(condition = "#event.amount > 100")
public void onLargePayment(PaymentEvent event) {
    System.out.println("대금 수신: " + event.getAmount());
}

// amount = 50 → 리스너 실행 안 됨
publisher.publishEvent(new PaymentEvent(50));

// amount = 200 → 리스너 실행됨
publisher.publishEvent(new PaymentEvent(200));
```

### 실험 3: 반환값 이벤트 연쇄 발행

```java
@EventListener
public OrderConfirmedEvent onOrderPlaced(OrderPlacedEvent event) {
    System.out.println("주문 처리 → 확인 이벤트 발행");
    return new OrderConfirmedEvent(event.getOrder());
}

@EventListener
public void onOrderConfirmed(OrderConfirmedEvent event) {
    System.out.println("주문 확인 처리");
}

publisher.publishEvent(new OrderPlacedEvent(order));
// 출력: "주문 처리 → 확인 이벤트 발행" → "주문 확인 처리" (동기 연쇄)
```

---

## 🤔 트레이드오프

```
@EventListener vs ApplicationListener<T>:

  @EventListener:
    장점  간결, 한 클래스에 여러 이벤트 처리
          condition, 반환값 연쇄 발행 지원
          @Async 쉽게 결합
    단점  처리 시점: afterSingletonsInstantiated() 이후
          → refresh() 완료 전 이벤트(earlyEvents)는 @EventListener로 수신 불가
          Bean이 완전히 초기화된 후에야 등록됨

  ApplicationListener<T>:
    장점  @Component만 있으면 스캔 시 바로 등록
          컨텍스트 초기화 중 이벤트도 수신 가능
          타입 안전 (컴파일 타임 체크)
    단점  인터페이스 구현 필수, 이벤트 타입당 클래스
          condition, 연쇄 발행 기본 지원 없음

condition SpEL:
  장점  런타임 조건 간결하게 표현
  단점  컴파일 타임 오류 감지 불가, 성능 오버헤드 (매 이벤트 평가)
  대안  if 문으로 메서드 내부에서 조건 처리 (명시적, 테스트 쉬움)
```

---

## 📌 핵심 정리

```
처리 시점
  EventListenerMethodProcessor.afterSingletonsInstantiated()
  → 모든 싱글톤 Bean 초기화 완료 후
  → @EventListener 메서드 → ApplicationListenerMethodAdapter 생성 → 컨텍스트 등록

ApplicationListenerMethodAdapter 역할
  supportsEventType(): 파라미터 타입으로 이벤트 지원 여부 판별
  onApplicationEvent() → processEvent()
    resolveArguments(): ApplicationEvent → 메서드 인수 변환
    shouldHandle(): SpEL condition 평가
    doInvoke(): 원본 메서드 리플렉션 호출
    handleResult(): 반환값 → 새 이벤트 발행

@EventListener 특수 기능
  condition: SpEL 조건 (#event.xxx, #root.args 등)
  classes: 여러 이벤트 타입 동시 구독
  반환값: 단일/배열/컬렉션 → 연쇄 이벤트 발행

ApplicationListener 사용 권장 상황
  컨텍스트 초기화 중 이벤트 수신 필요
  (ContextRefreshedEvent를 아주 이른 시점에 처리해야 할 때)
```

---

## 🤔 생각해볼 문제

**Q1.** `@EventListener`가 `afterSingletonsInstantiated()` 시점에 등록되는 이유는 무엇인가? `BeanPostProcessor` 처리 시점에 등록하면 안 되는가?

**Q2.** `@EventListener` 메서드에 `@Transactional`을 붙이면 어떻게 동작하는가?

**Q3.** 같은 이벤트를 `ApplicationListener<T>` 인터페이스와 `@EventListener` 어노테이션 두 방식으로 모두 구독하면 어떤 순서로 실행되는가?

> 💡 **해설**
>
> **Q1.** `BeanPostProcessor` 처리 시점에는 아직 모든 Bean이 생성되지 않았다. `BeanPostProcessor` 자체가 처리 중이므로 일반 Bean에 대한 `postProcessAfterInitialization()`이 진행 중이다. 이 시점에 `@EventListener` 메서드를 탐색하면 순환 의존성 문제나 아직 생성되지 않은 Bean의 메서드를 처리하려는 문제가 생길 수 있다. `afterSingletonsInstantiated()`는 모든 Singleton Bean이 완전히 초기화된 후에 호출되므로, 모든 Bean에 대해 안전하게 `@EventListener` 메서드를 탐색할 수 있다.
>
> **Q2.** `@EventListener` 메서드에 `@Transactional`을 붙이면, `ApplicationListenerMethodAdapter`가 해당 Bean의 실제 인스턴스를 통해 메서드를 호출할 때 AOP 프록시를 통한 호출이 이루어진다. 이벤트 발행 스레드와 같은 트랜잭션 컨텍스트가 있다면 `REQUIRED` 전파 방식에서 해당 트랜잭션에 참여한다. 그러나 Self-Invocation 문제를 피하려면 `ApplicationListenerMethodAdapter`가 Bean의 프록시를 통해 호출해야 한다. `doInvoke()`에서 `this.bean`을 통해 호출하는데, 이때 `this.bean`이 CGLIB 프록시이면 트랜잭션이 정상 적용된다.
>
> **Q3.** `AbstractApplicationEventMulticaster.getApplicationListeners()`는 탐색 결과를 `AnnotationAwareOrderComparator`로 정렬한다. `ApplicationListener<T>`는 `@Order` 어노테이션이나 `Ordered` 인터페이스를 구현해 순서를 지정할 수 있고, `ApplicationListenerMethodAdapter`도 `@Order`로 순서를 지정할 수 있다. 명시적 순서가 없다면 등록 순서에 따라 결정되는데, `ApplicationListener<T>` 구현체는 스캔 단계에서, `@EventListener` 어댑터는 `afterSingletonsInstantiated()` 단계에서 등록되므로 일반적으로 `ApplicationListener<T>` 먼저 실행된다. 정확한 순서 보장이 필요하면 `@Order`를 명시해야 한다.

---

<div align="center">

**[⬅️ 이전: ApplicationEvent 발행-구독 메커니즘](./01-application-event-mechanism.md)** | **[다음: 비동기 이벤트 처리 @Async ➡️](./03-async-event-processing.md)**

</div>

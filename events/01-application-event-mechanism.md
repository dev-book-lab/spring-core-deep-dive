# ApplicationEvent 발행-구독 메커니즘 — ApplicationEventMulticaster 내부 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `publishEvent()`가 호출되면 내부에서 어떤 순서로 리스너를 찾아 호출하는가?
- `SimpleApplicationEventMulticaster`와 `ApplicationEventMulticaster`의 관계는?
- 리스너 탐색에서 제네릭 타입(`ApplicationEvent<T>`)은 어떻게 처리되는가?
- `retrieverCache`가 성능 최적화에 기여하는 방식은?
- 스프링 컨텍스트 생명주기 이벤트(`ContextRefreshedEvent` 등)는 어디서 발행되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean 간 직접 의존 없이 이벤트를 전달해야 한다

```java
// 직접 의존 방식 — 강한 결합
@Service
public class OrderService {
    @Autowired EmailService emailService;    // 직접 의존
    @Autowired AuditService auditService;   // 직접 의존
    @Autowired InventoryService inventory;  // 직접 의존

    public void placeOrder(Order order) {
        // 모든 후속 처리를 직접 알고 있어야 함
        emailService.sendConfirmation(order);
        auditService.record(order);
        inventory.deduct(order);
    }
}

// 이벤트 방식 — 느슨한 결합
@Service
public class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        // OrderService는 이후 처리를 모름
        publisher.publishEvent(new OrderPlacedEvent(order));
    }
}

// 각 서비스가 독립적으로 구독
@Component
public class EmailListener {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) { ... }
}
```

```
발행-구독 패턴이 제공하는 것:
  발행자 ↔ 구독자 간 직접 의존 제거
  → 새 리스너 추가 시 OrderService 수정 불필요
  → 관심사 분리, 테스트 용이
  → 동기/비동기 처리 전환 용이
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: publishEvent()는 항상 비동기로 동작한다

```
❌ 잘못된 이해:
  "publishEvent() = 이벤트를 큐에 넣고 비동기로 처리"

✅ 실제:
  기본 SimpleApplicationEventMulticaster는 동기 처리
  → publishEvent() 호출 스레드에서 모든 리스너를 순차 실행
  → 리스너 실행 완료 후 publishEvent() 반환

  비동기로 만들려면:
  1. SimpleApplicationEventMulticaster에 Executor 설정
  2. @EventListener + @Async 조합
  → 03 문서에서 상세 다룸
```

### Before: ApplicationEvent를 상속해야만 이벤트를 발행할 수 있다

```java
// ❌ 과거 방식 (Spring 4.2 이전 강제)
public class OrderPlacedEvent extends ApplicationEvent {
    public OrderPlacedEvent(Object source) { super(source); }
}

// ✅ Spring 4.2+: 임의 객체도 이벤트로 발행 가능
public class OrderPlacedEvent {  // POJO, ApplicationEvent 상속 불필요
    private final Order order;
    public OrderPlacedEvent(Order order) { this.order = order; }
}

// POJO 이벤트 발행
publisher.publishEvent(new OrderPlacedEvent(order));
// → 내부에서 PayloadApplicationEvent<OrderPlacedEvent>로 래핑
```

---

## ✨ 올바른 이해와 사용

### After: publishEvent() → 리스너 탐색 → 호출 전체 흐름

```
publishEvent(event)
  ↓ AbstractApplicationContext
  resolveEvent(event)
    → POJO면 PayloadApplicationEvent로 래핑
    → ApplicationEvent면 그대로
  ↓
  getApplicationEventMulticaster()
    → SimpleApplicationEventMulticaster (기본)
  ↓
  multicastEvent(event, eventType)
  ↓
  getApplicationListeners(event, eventType)
    → retrieverCache 조회 (캐시 히트 시 즉시 반환)
    → 캐시 미스 시 retrieveApplicationListeners()
      → 등록된 모든 리스너 순회
      → supportsEvent(listener, eventType, sourceType) 필터링
      → 결과 캐시 저장
  ↓
  각 리스너에 대해 invokeListener(listener, event)
    → 동기: listener.onApplicationEvent(event) 직접 호출
    → 비동기(Executor 설정 시): executor.execute(() → ...)
```

---

## 🔬 내부 동작 원리

### 1. AbstractApplicationContext.publishEvent()

```java
// AbstractApplicationContext (ApplicationEventPublisher 구현)
protected void publishEvent(Object event, ResolvableType eventType) {

    // POJO 이벤트 → PayloadApplicationEvent로 래핑
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent ae) {
        applicationEvent = ae;
    } else {
        applicationEvent = new PayloadApplicationEvent<>(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
        }
    }

    // 컨텍스트가 아직 refresh() 완료 전이면 이벤트를 큐에 저장
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
        return;
    }

    // 멀티캐스터에 이벤트 전달
    getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);

    // 부모 컨텍스트에도 전달 (계층 구조)
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext aac) {
            aac.publishEvent(event, eventType);
        } else {
            this.parent.publishEvent(event);
        }
    }
}
```

```
earlyApplicationEvents:
  refresh() 완료 전(multicastEvent 불가 시점)에 발행된 이벤트 보관
  → finishRefresh() 단계에서 멀티캐스터 초기화 완료 후 일괄 전달
  → ContextRefreshedEvent 직전에 모두 소진
```

### 2. SimpleApplicationEventMulticaster.multicastEvent()

```java
// SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster
public void multicastEvent(ApplicationEvent event, ResolvableType eventType) {

    ResolvableType type = (eventType != null ? eventType : ResolvableType.forInstance(event));

    // Executor 설정 여부에 따라 동기/비동기 분기
    Executor executor = getTaskExecutor();

    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null && supportsAsyncExecution(listener)) {
            // 비동기 실행
            executor.execute(() -> invokeListener(listener, event));
        } else {
            // 동기 실행 (기본)
            invokeListener(listener, event);
        }
    }
}

protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    if (errorHandler != null) {
        try {
            doInvokeListener(listener, event);
        } catch (Throwable err) {
            errorHandler.handleError(err);
        }
    } else {
        doInvokeListener(listener, event);
    }
}

@SuppressWarnings({"rawtypes", "unchecked"})
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
    listener.onApplicationEvent(event);
    // → ApplicationListenerMethodAdapter.onApplicationEvent() 호출
    //   → @EventListener 메서드 실행 (02 문서에서 상세)
}
```

### 3. getApplicationListeners() — 리스너 탐색과 캐시

```java
// AbstractApplicationEventMulticaster.getApplicationListeners()
protected Collection<ApplicationListener<?>> getApplicationListeners(
        ApplicationEvent event, ResolvableType eventType) {

    Object source = event.getSource();
    Class<?> sourceType = (source != null ? source.getClass() : null);
    ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

    // ① retrieverCache 조회 (ConcurrentHashMap)
    CachedListenerRetriever newRetriever = null;
    CachedListenerRetriever existingRetriever = this.retrieverCache.get(cacheKey);
    if (existingRetriever != null) {
        Collection<ApplicationListener<?>> result =
            existingRetriever.getApplicationListeners();
        if (result != null) return result;  // 캐시 히트!
    }

    // ② 캐시 미스 → 전체 탐색
    return retrieveApplicationListeners(eventType, sourceType, newRetriever);
}

private Collection<ApplicationListener<?>> retrieveApplicationListeners(
        ResolvableType eventType, Class<?> sourceType,
        CachedListenerRetriever retriever) {

    List<ApplicationListener<?>> allListeners = new ArrayList<>();

    // 등록된 모든 ApplicationListener 순회
    for (ApplicationListener<?> listener : this.defaultRetriever.applicationListeners) {
        if (supportsEvent(listener, eventType, sourceType)) {
            allListeners.add(listener);
        }
    }

    // BeanFactory에서 ApplicationListener Bean 이름 탐색
    for (String listenerBeanName : this.defaultRetriever.applicationListenerBeans) {
        ApplicationListener<?> listener =
            this.beanFactory.getBean(listenerBeanName, ApplicationListener.class);
        if (!allListeners.contains(listener)
                && supportsEvent(listener, eventType, sourceType)) {
            allListeners.add(listener);
        }
    }

    // @Order 기반 정렬
    AnnotationAwareOrderComparator.sort(allListeners);

    // 결과 캐시 저장
    if (retriever != null) {
        retriever.applicationListeners = new LinkedHashSet<>(allListeners);
        this.retrieverCache.put(cacheKey, retriever);
    }

    return allListeners;
}
```

```
캐시 키: ListenerCacheKey(eventType, sourceType)
  → 같은 이벤트 타입 + 소스 타입 조합은 캐시 히트
  → 최초 호출 시만 전체 탐색, 이후는 O(1)

캐시 무효화:
  리스너 추가/제거 시 retrieverCache.clear()
  컨텍스트 refresh 시 초기화
```

### 4. supportsEvent() — 리스너 필터링

```java
// 리스너가 이 이벤트를 처리할 수 있는지 확인
protected boolean supportsEvent(ApplicationListener<?> listener,
                                 ResolvableType eventType,
                                 Class<?> sourceType) {

    GenericApplicationListener smartListener =
        (listener instanceof GenericApplicationListener gal ? gal
            : new GenericApplicationListenerAdapter(listener));

    return (smartListener.supportsEventType(eventType)
        && smartListener.supportsSourceType(sourceType));
}
```

```java
// GenericApplicationListenerAdapter — 타입 추론
public boolean supportsEventType(ResolvableType eventType) {
    Class<?> typeArg = GenericTypeResolver.resolveTypeArgument(
        this.delegate.getClass(), ApplicationListener.class);
    // ApplicationListener<OrderPlacedEvent> → typeArg = OrderPlacedEvent

    if (typeArg == null || typeArg == ApplicationEvent.class) {
        return true;  // 타입 인수 없으면 모든 이벤트 수신
    }

    if (PayloadApplicationEvent.class.isAssignableFrom(eventType.resolve(Object.class))) {
        // POJO 이벤트(PayloadApplicationEvent) → payload 타입으로 비교
        ResolvableType payloadType = eventType.as(PayloadApplicationEvent.class)
                                               .getGeneric();
        return eventType.isAssignableFrom(typeArg)
            || payloadType.isAssignableFrom(typeArg);
    }

    return ResolvableType.forClass(typeArg).isAssignableFrom(eventType);
}
```

### 5. 스프링 컨텍스트 생명주기 이벤트

```java
// AbstractApplicationContext.refresh()에서 발행되는 이벤트들

// 1. ContextRefreshedEvent — refresh() 완료 시
protected void finishRefresh() {
    // ...
    publishEvent(new ContextRefreshedEvent(this));
}

// 2. ContextClosedEvent — close() 시
public void close() {
    // ...
    publishEvent(new ContextClosedEvent(this));
}

// 3. ContextStartedEvent / ContextStoppedEvent — start()/stop() 시

// 기타 Spring Boot 이벤트 (SpringApplication 레벨):
// ApplicationStartingEvent
// ApplicationEnvironmentPreparedEvent
// ApplicationContextInitializedEvent
// ApplicationPreparedEvent
// ApplicationStartedEvent
// ApplicationReadyEvent
// ApplicationFailedEvent
// → SpringApplicationRunListeners를 통해 발행
```

### 6. ApplicationEventMulticaster Bean 등록 시점

```java
// AbstractApplicationContext.initApplicationEventMulticaster()
// → refresh() 의 단계 중 하나

protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();

    // 사용자가 "applicationEventMulticaster" 이름으로 Bean 등록했으면 그것 사용
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME,
                ApplicationEventMulticaster.class);
    } else {
        // 없으면 기본 SimpleApplicationEventMulticaster 생성
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME,
            this.applicationEventMulticaster);
    }
}
```

```
커스텀 멀티캐스터 등록:
@Bean(name = "applicationEventMulticaster")
public ApplicationEventMulticaster eventMulticaster() {
    SimpleApplicationEventMulticaster multicaster =
        new SimpleApplicationEventMulticaster();
    multicaster.setTaskExecutor(new ThreadPoolTaskExecutor());
    // → 비동기 이벤트 처리 활성화
    return multicaster;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 동기 이벤트 처리 확인

```java
@Component
public class OrderEventDemo {
    @Autowired ApplicationEventPublisher publisher;

    public void demo() {
        System.out.println("1. 발행 전");
        publisher.publishEvent(new OrderPlacedEvent(new Order()));
        System.out.println("3. 발행 후");  // 리스너 실행 완료 후 여기 도달
    }
}

@Component
public class EmailListener {
    @EventListener
    public void on(OrderPlacedEvent e) {
        System.out.println("2. 리스너 실행 중");
    }
}
// 출력: 1 → 2 → 3 (동기, 순차)
```

### 실험 2: POJO 이벤트와 PayloadApplicationEvent 래핑

```java
// POJO 이벤트 발행
publisher.publishEvent("Hello, Event!");

@EventListener
public void onString(String payload) {
    System.out.println("받은 메시지: " + payload);
}
// → String이 PayloadApplicationEvent<String>으로 래핑되어 전달
```

### 실험 3: 컨텍스트 생명주기 이벤트 수신

```java
@Component
public class ContextEventListener {

    @EventListener
    public void onRefreshed(ContextRefreshedEvent e) {
        System.out.println("컨텍스트 refresh 완료: " + e.getApplicationContext());
    }

    @EventListener
    public void onClosed(ContextClosedEvent e) {
        System.out.println("컨텍스트 close: " + e.getApplicationContext());
    }
}
```

---

## 🤔 트레이드오프

```
동기 이벤트 (기본):
  장점  트랜잭션 컨텍스트 공유 가능, 예외 전파 명확
  단점  리스너 실행 시간이 발행자에게 영향
  사용  빠른 처리, 트랜잭션 연동 필요 시

비동기 이벤트 (Executor 설정):
  장점  발행자 즉시 반환, 병렬 처리 가능
  단점  트랜잭션 공유 불가, 예외 처리 복잡
  사용  이메일 발송, 알림, 로그 같은 비핵심 후처리

retrieverCache:
  장점  반복 이벤트 타입에 대한 O(1) 리스너 탐색
  단점  리스너 동적 변경 시 캐시 무효화 필요
  영향  이벤트 타입이 다양할수록 캐시 효율 감소

POJO 이벤트 vs ApplicationEvent:
  POJO  단순, 스프링 의존 없음, 테스트 쉬움
  ApplicationEvent  source/timestamp 제공, 컨텍스트 접근 가능
```

---

## 📌 핵심 정리

```
발행 경로
  publishEvent() → ApplicationEventMulticaster.multicastEvent()
  POJO → PayloadApplicationEvent 자동 래핑
  earlyApplicationEvents: refresh 전 이벤트 큐잉 → 완료 후 소진

리스너 탐색
  getApplicationListeners() → retrieverCache (ConcurrentHashMap) 조회
  캐시 미스 → 전체 리스너 supportsEvent() 필터 → 정렬 → 캐시
  캐시 키: (eventType, sourceType)

동기 처리 (기본)
  invokeListener() → doInvokeListener() → listener.onApplicationEvent()
  발행 스레드에서 순차 실행 → 발행자 블로킹

비동기 처리
  SimpleApplicationEventMulticaster에 Executor 설정
  → executor.execute(() → invokeListener(...))

생명주기 이벤트 발행 위치
  ContextRefreshedEvent → finishRefresh()
  ContextClosedEvent    → doClose()
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 이벤트 타입을 구독하는 리스너가 5개 있고 3번째 리스너에서 예외가 발생하면 4, 5번 리스너는 실행되는가?

**Q2.** `publishEvent()`를 `@Transactional` 메서드 안에서 호출하면 리스너도 같은 트랜잭션 안에서 실행되는가?

**Q3.** 부모 컨텍스트와 자식 컨텍스트가 있을 때 부모에서 발행한 이벤트를 자식 리스너가 수신할 수 있는가? 반대의 경우는?

> 💡 **해설**
>
> **Q1.** 기본 설정에서는 실행되지 않는다. `doInvokeListener()`에서 예외가 발생하면 `ErrorHandler`가 설정되지 않은 경우 예외가 그대로 전파되어 `multicastEvent()`의 for 루프가 중단된다. 4, 5번 리스너는 호출되지 않는다. `ErrorHandler`를 설정하면(`SimpleApplicationEventMulticaster.setErrorHandler()`) 예외를 캐치해 처리하고 다음 리스너로 진행할 수 있다.
>
> **Q2.** 동기 이벤트의 경우 발행 스레드에서 리스너가 실행되므로, `@Transactional` 메서드 내에서 `publishEvent()`를 호출하면 리스너도 같은 트랜잭션 컨텍스트에서 실행된다. `TransactionSynchronizationManager.isActualTransactionActive()`가 `true`를 반환하고, 리스너 내에서 같은 DB 커넥션을 사용할 수 있다. 단, 리스너 내의 `@Transactional` 설정에 따라 전파 방식이 달라진다. `@TransactionalEventListener`를 사용하면 트랜잭션 커밋 후 리스너를 실행할 수 있다(05 문서 참조).
>
> **Q3.** 자식 컨텍스트에서 발행한 이벤트는 `publishEvent()` 내부에서 부모 컨텍스트의 `publishEvent()`도 호출하므로, 부모의 리스너도 이벤트를 수신한다. 반대로 부모 컨텍스트에서 발행한 이벤트는 자식 컨텍스트의 멀티캐스터로 전달되지 않는다. 부모 컨텍스트는 자식의 존재를 알지 못하기 때문이다. Spring MVC 환경에서 루트 컨텍스트(부모)의 이벤트를 웹 컨텍스트(자식) 리스너로 수신하려면 별도 구성이 필요하다.

---

<div align="center">

**[다음: @EventListener vs ApplicationListener ➡️](./02-eventlistener-vs-applicationlistener.md)**

</div>

# Optional Dependency 처리 — 없어도 되는 의존성을 다루는 네 가지 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `required=false`, `Optional<T>`, `@Nullable`, `ObjectProvider<T>` 는 어떻게 다른가?
- 각 방법이 내부적으로 처리되는 시점과 경로는 어디인가?
- Bean이 없을 때 각 방법에서 발생하는 결과는 무엇인가?
- `ObjectProvider`가 단순 `Optional`보다 유리한 상황은 언제인가?
- Prototype scope Bean을 Singleton에 안전하게 주입하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: 있어도 되고 없어도 되는 의존성

```java
@Service
public class NotificationService {
    @Autowired
    private SlackClient slackClient;  // Slack 연동이 설정된 환경에서만 존재

    public void notify(String msg) {
        slackClient.send(msg);  // SlackClient Bean 없으면 컨텍스트 시작 실패
    }
}
```

```
문제:
  SlackClient가 개발 환경에서는 없을 수 있음
  → NoSuchBeanDefinitionException → 컨텍스트 시작 실패
  → 기능이 비활성화되는 게 아니라 애플리케이션 자체가 뜨지 않음

원하는 동작:
  SlackClient 있으면 → Slack 알림 발송
  SlackClient 없으면 → 조용히 skip

해결 방법 네 가지:
  1. @Autowired(required=false)
  2. Optional<T>
  3. @Nullable
  4. ObjectProvider<T>
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: required=false면 NPE가 없다

```java
@Autowired(required = false)
private SlackClient slackClient;

public void notify(String msg) {
    slackClient.send(msg);  // Bean 없으면 NPE!
}
```

```
❌ 오해:
  required=false → 컨텍스트 시작 성공 → 이후 안전

✅ 실제:
  Bean 있으면 → 정상 주입
  Bean 없으면 → null 주입 → NPE 위험 그대로

required=false는 컨텍스트 시작 실패를 막는 것
null 처리는 여전히 개발자 책임
```

### Before: ObjectProvider는 Optional의 fancy 버전이다

```
❌ 잘못된 이해:
  "ObjectProvider도 결국 Bean 없으면 null 아닌가"

✅ 실제 차이:
  Optional<T>       → 주입 시점에 Bean 탐색, 이후 고정
  ObjectProvider<T> → getObject() 호출 시점에 Bean 탐색 (지연)
                      Prototype Bean 매번 새 인스턴스 획득 가능
                      여러 Bean을 Stream으로 처리 가능
  → 목적과 사용 시점이 근본적으로 다름
```

---

## ✨ 올바른 이해와 사용

### After: 상황에 맞는 방법 선택

```
방법 선택 기준:

required=false    가장 단순, null 체크 필수, 레거시에서 자주 보임
Optional<T>       Java 표준, null 없이 존재 여부 명시적 표현
@Nullable         의도 문서화 + IDE 정적 분석 지원
ObjectProvider<T> 지연 탐색, Prototype 지원, 다중 Bean 스트림
```

---

## 🔬 내부 동작 원리

### 1. required=false 처리

```java
// resolveDependency() — required 분기
public Object resolveDependency(DependencyDescriptor descriptor, ...) {
    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);

    if (matchingBeans.isEmpty()) {
        if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(...);  // required=true → 예외
        }
        return null;  // required=false → null 반환
    }
    // ...
}

// AutowiredFieldElement.inject()
protected void inject(Object bean, String beanName, PropertyValues pvs) {
    Object value = resolveFieldValue(field, bean, beanName);

    if (value != null) {              // null이면 field.set() 건너뜀
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);
    }
    // value == null → 필드는 null(기본값) 유지
}
```

```
required=false 흐름:
  Bean 없음 → null 반환 → field.set() 건너뜀
  → 필드 = null, 컨텍스트 시작 성공
```

### 2. Optional<T> 처리

```java
@Autowired
private Optional<SlackClient> slackClient;
```

```java
// DefaultListableBeanFactory.resolveDependency()
public Object resolveDependency(DependencyDescriptor descriptor, ...) {

    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    }
    // ...
}

private Optional<?> createOptionalDependency(DependencyDescriptor descriptor, ...) {
    // required=false로 내부 탐색
    DependencyDescriptor descriptorToUse = new NestedDependencyDescriptor(descriptor) {
        @Override public boolean isRequired() { return false; }
    };

    Object result = doResolveDependency(descriptorToUse, beanName, null, null);

    // null이면 Optional.empty(), 있으면 Optional.of(bean)
    return Optional.ofNullable(result);
}
```

```
Optional<T> 흐름:
  스프링이 Optional 타입 감지
  → required=false로 내부 탐색
  → Bean 있음: Optional.of(bean)
  → Bean 없음: Optional.empty()
  → 필드에는 항상 Optional 객체 (null 아님)

사용:
  slackClient.ifPresent(c -> c.send(msg));
  slackClient.map(SlackClient::send).orElse(defaultResult);
```

### 3. @Nullable 처리

```java
@Autowired @Nullable
private SlackClient slackClient;
```

```java
// AutowiredAnnotationBeanPostProcessor — isRequired() 판단
private boolean isRequired(FieldElement element) {
    // @Autowired(required=false) 또는 @Nullable 있으면 → not required
    return !AnnotationUtils.hasAnnotation(element.field, Nullable.class)
        && element.required;
}
```

```
@Nullable 흐름:
  required=false와 동작 동일 (Bean 없으면 null)
  차이: JSR-305 어노테이션 → IDE 정적 분석에서 null 경고 억제
        "이 null은 의도된 것"을 코드로 문서화
```

### 4. ObjectProvider<T> 처리

```java
@Autowired
private ObjectProvider<SlackClient> slackClientProvider;
```

```java
// DefaultListableBeanFactory.resolveDependency()
public Object resolveDependency(DependencyDescriptor descriptor, ...) {

    if (ObjectFactory.class == descriptor.getDependencyType()
            || ObjectProvider.class == descriptor.getDependencyType()) {
        // Bean 탐색 안 함 — Provider 객체만 반환
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    }
    // ...
}

// DependencyObjectProvider — 실제 탐색은 getObject() 호출 시
private class DependencyObjectProvider implements ObjectProvider<Object> {

    @Override
    public Object getObject() throws BeansException {
        // 이 시점에 Bean 탐색
        return doResolveDependency(this.descriptor, this.beanName, null, null);
    }

    @Override
    public Object getIfAvailable() {
        try {
            return doResolveDependency(/* required=false */);
        } catch (NoSuchBeanDefinitionException ex) {
            return null;
        }
    }

    @Override
    public Stream<Object> stream() {
        // 해당 타입 모든 Bean을 Stream으로
        return Arrays.stream(getBeanNamesForType(descriptor.getDependencyType()))
                .map(name -> getBean(name));
    }

    @Override
    public Stream<Object> orderedStream() {
        // @Order / Ordered 기준 정렬 후 Stream
        return stream().sorted(COMPARATOR_BY_ORDER);
    }
}
```

```
ObjectProvider<T> 핵심 메서드:

getObject()        Bean 없으면 NoSuchBeanDefinitionException
getIfAvailable()   Bean 없으면 null (안전)
getIfUnique()      Bean 1개면 반환, 없거나 여러 개면 null
stream()           해당 타입 모든 Bean을 Stream으로
orderedStream()    @Order 순서 적용 Stream

핵심 특징:
  주입 시점이 아닌 getObject() 호출 시점에 Bean 탐색
  → Prototype Bean: 호출마다 새 인스턴스
  → 지연 초기화 패턴 구현 가능
```

### 5. ObjectProvider로 Prototype Bean 안전하게 주입

```java
// 잘못된 방법: Singleton에 Prototype 직접 주입
@Service  // Singleton
class OrderService {
    @Autowired
    PrototypeBean prototypeBean;
    // → 컨텍스트 시작 시 한 번 주입 → 이후 매번 같은 인스턴스 (Prototype 의미 없음)
}

// 올바른 방법: ObjectProvider 사용
@Service  // Singleton
class OrderService {
    @Autowired
    ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void processOrder() {
        PrototypeBean bean = prototypeBeanProvider.getObject();
        // → 호출마다 새 PrototypeBean 인스턴스 생성
        bean.process();
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Bean 없을 때 각 방법 결과 비교

```java
// SlackClient Bean 미등록

@Service
class TestService {
    @Autowired(required=false) SlackClient a;          // → null
    @Autowired Optional<SlackClient> b;                // → Optional.empty()
    @Autowired @Nullable SlackClient c;                // → null
    @Autowired ObjectProvider<SlackClient> d;          // → Provider 객체 (null 아님)

    void check() {
        System.out.println(a);                         // null
        System.out.println(b.isPresent());             // false
        System.out.println(c);                         // null
        System.out.println(d.getIfAvailable());        // null
    }
}
```

### 실험 2: ObjectProvider로 Prototype 매번 새로 생성

```java
@Component @Scope("prototype")
class RequestContext {
    private final long id = System.nanoTime();
    public long getId() { return id; }
}

@Service
class ProcessingService {
    @Autowired ObjectProvider<RequestContext> provider;

    public void process() {
        RequestContext ctx1 = provider.getObject();
        RequestContext ctx2 = provider.getObject();
        System.out.println(ctx1 == ctx2);  // false → 매번 새 인스턴스
    }
}
```

### 실험 3: orderedStream()으로 순서 있는 다중 Bean 처리

```java
@Component @Order(3) class EmailNotifier implements Notifier {}
@Component @Order(1) class SlackNotifier implements Notifier {}
@Component @Order(2) class SmsNotifier   implements Notifier {}

@Service
class BroadcastService {
    @Autowired ObjectProvider<Notifier> notifiers;

    public void broadcast(String msg) {
        notifiers.orderedStream()
                 .forEach(n -> n.notify(msg));
        // 순서: SlackNotifier(1) → SmsNotifier(2) → EmailNotifier(3)
    }
}
```

---

## 🤔 트레이드오프

```
required=false:
  장점  가장 단순, 기존 코드 최소 변경
  단점  null 체크 누락 시 NPE 런타임 오류

Optional<T>:
  장점  null 없이 존재 여부 표현, Java 표준
  단점  Optional 래핑/언래핑 코드 증가

@Nullable:
  장점  의도 문서화, IDE 정적 분석 지원
  단점  null 체크 여전히 필요

ObjectProvider<T>:
  장점  지연 탐색, Prototype 지원, stream/orderedStream
  단점  코드 복잡도 증가

선택 가이드:
  단순 선택적 의존    → Optional<T>
  Prototype 매번 획득 → ObjectProvider<T>
  여러 Bean 순회      → ObjectProvider<T>.orderedStream()
  레거시 유지         → required=false
```

---

## 📌 핵심 정리

```
네 가지 방법 비교

방법               Bean 없을 때   탐색 시점     null 위험
required=false     null 주입      주입 시점     있음
Optional<T>        empty()        주입 시점     없음
@Nullable          null 주입      주입 시점     있음
ObjectProvider<T>  Provider 객체  getObject()  없음(getIfAvailable)

ObjectProvider 주요 메서드
  getObject()       Bean 없으면 예외
  getIfAvailable()  Bean 없으면 null
  getIfUnique()     복수 존재 시 null
  stream()          전체 Bean Stream
  orderedStream()   @Order 순서 적용 Stream

Prototype + Singleton
  ObjectProvider<T>.getObject() → 매 호출마다 새 인스턴스
```

---

## 🤔 생각해볼 문제

**Q1.** `@Autowired(required=false)`와 `@Autowired @Nullable`은 동작이 동일한가? 차이가 있다면 무엇인가?

**Q2.** `ObjectProvider<T>.getObject()`를 Singleton Bean의 매 메서드 호출마다 실행하는 것과, 처음 한 번만 `getIfAvailable()`로 캐싱하는 것의 차이는? 어떤 경우에 각각을 써야 하는가?

**Q3.** 다음 코드에서 `orderedStream()`으로 반환되는 Bean의 순서는 보장되는가? `stream()`과의 차이는?

```java
@Autowired ObjectProvider<Notifier> notifiers;
notifiers.stream().forEach(n -> n.notify(msg));
notifiers.orderedStream().forEach(n -> n.notify(msg));
```

> 💡 **해설**
>
> **Q1.** 동작은 거의 동일하다. 둘 다 Bean이 없을 때 null을 주입하고 컨텍스트 시작을 성공시킨다. 차이는 의미와 도구 지원에 있다. `required=false`는 스프링 고유 속성이고, `@Nullable`은 JSR-305 어노테이션으로 IDE와 정적 분석 도구(IntelliJ, SpotBugs)가 null 가능성을 인식해 경고와 자동완성을 제공한다. `@Nullable`은 필드뿐 아니라 파라미터·반환 타입에도 붙일 수 있어 null 계약을 더 광범위하게 문서화할 수 있다.
>
> **Q2.** Singleton Bean이면 `getIfAvailable()`로 한 번 캐싱해도 무방하다. `getObject()`를 매번 호출해도 Singleton은 1단계 캐시에서 즉시 반환되므로 성능 차이는 미미하지만 코드가 불필요하게 복잡해진다. 반면 Prototype scope Bean이라면 반드시 매번 `getObject()`를 호출해야 새 인스턴스를 얻을 수 있다. 한 번 캐싱하면 Prototype의 의미가 사라져 사실상 Singleton처럼 동작한다.
>
> **Q3.** `stream()`은 BeanDefinition 등록 순서를 따르지만 순서 보장이 명시적이지 않다. `orderedStream()`은 Bean에 붙은 `@Order` 어노테이션 또는 `Ordered` 인터페이스의 `getOrder()` 값 기준으로 오름차순 정렬해 반환하므로 순서가 보장된다. 파이프라인·필터 체인·알림 전송 순서처럼 실행 순서가 중요한 곳에서는 반드시 `orderedStream()`을 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: @Qualifier & @Primary 우선순위](./04-qualifier-primary-priority.md)** | **[다음: Constructor Binding의 장점 ➡️](./06-constructor-binding.md)**

</div>

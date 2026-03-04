# Bean Scope와 프록시 — Singleton에 Prototype을 주입할 때의 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Singleton / Prototype / Request / Session Scope는 생명주기가 어떻게 다른가?
- Singleton Bean에 Prototype Bean을 `@Autowired`로 주입하면 왜 Prototype 의미가 사라지는가?
- Scope Proxy는 이 문제를 어떻게 해결하는가? 내부적으로 어떤 객체가 주입되는가?
- `@RequestScope`, `@SessionScope`는 웹 환경에서 어떻게 동작하는가?
- Scope Proxy의 `proxyMode`에서 `TARGET_CLASS`와 `INTERFACES`는 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: 짧은 생명주기 Bean을 긴 생명주기 Bean에 주입하면

```java
@Service  // Singleton — 애플리케이션 생애 동안 1개
public class OrderService {

    @Autowired
    private ShoppingCart cart;  // Request Scope — 요청마다 새로 생성되어야 함

    public void addItem(Item item) {
        cart.add(item);  // 항상 같은 cart 인스턴스 → 모든 사용자가 공유!
    }
}
```

```
Scope 불일치 문제:
  Singleton Bean은 컨텍스트 시작 시 1개 생성, 이후 고정
  → @Autowired 주입도 딱 한 번 발생

  Prototype / Request / Session Bean은 짧은 생명주기
  → 요청마다 / 세션마다 새 인스턴스 필요

  하지만 Singleton 생성 시 주입 → 이후 항상 같은 인스턴스
  → Prototype이 사실상 Singleton처럼 동작
  → Request Scope Bean이 여러 사용자 요청 간에 공유 → 심각한 버그
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Scope만 선언하면 동작한다

```java
// ❌ 잘못된 사용
@Component
@Scope("request")
public class ShoppingCart { ... }

@Service  // Singleton
public class OrderService {
    @Autowired
    ShoppingCart cart;  // proxyMode 없음

    // → 컨텍스트 시작 시 ShoppingCart 인스턴스 하나 생성
    // → 이후 모든 요청에서 같은 인스턴스 사용
    // → Request Scope 의미 없음
}
```

```
❌ 잘못된 이해:
  "@Scope("request") 붙이면 요청마다 새 인스턴스가 주입된다"

✅ 실제:
  @Autowired 주입은 Singleton 생성 시 한 번만 발생
  → proxyMode 없이는 항상 동일 인스턴스

해결:
  @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
  → 프록시가 주입되고, 실제 요청 Bean은 프록시를 통해 조회
```

---

## ✨ 올바른 이해와 사용

### After: Scope Proxy로 생명주기 불일치를 해소

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    // ...
}

@Service
public class OrderService {
    @Autowired
    ShoppingCart cart;  // 실제론 프록시 주입

    public void addItem(Item item) {
        cart.add(item);
        // → 프록시가 현재 요청 스레드의 실제 ShoppingCart 탐색
        // → 요청마다 다른 ShoppingCart 인스턴스에 위임
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Scope 등록과 Bean 생성 책임

```java
// Scope 인터페이스 — 각 Scope 구현체가 인스턴스 생명주기 관리
public interface Scope {
    // 현재 Scope에서 Bean 획득 (없으면 objectFactory로 생성)
    Object get(String name, ObjectFactory<?> objectFactory);
    // 현재 Scope에서 Bean 제거
    Object remove(String name);
    // 소멸 콜백 등록
    void registerDestructionCallback(String name, Runnable callback);
}

// 각 Scope 구현체:
// SingletonScope  → singletonObjects Map (DefaultSingletonBeanRegistry)
// PrototypeScope  → 매번 새 인스턴스 (캐시 없음)
// RequestScope    → HttpServletRequest 속성에 저장 (RequestContextHolder)
// SessionScope    → HttpSession 속성에 저장
```

```java
// AbstractBeanFactory.doGetBean() — Scope 분기
protected <T> T doGetBean(String name, ...) {

    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

    if (mbd.isSingleton()) {
        // 1단계 캐시 탐색 → 없으면 createBean()
        sharedInstance = getSingleton(beanName, () -> createBean(...));

    } else if (mbd.isPrototype()) {
        // 캐시 없음 → 항상 createBean()
        prototypeInstance = createBean(beanName, mbd, args);

    } else {
        // 커스텀 Scope (request, session, ...)
        String scopeName = mbd.getScope();
        Scope scope = this.scopes.get(scopeName);
        Object scopedInstance = scope.get(beanName,
            () -> createBean(beanName, mbd, args));
        // → Scope.get()이 캐시 여부 결정
        //   Request Scope: 현재 Request 속성에 있으면 반환, 없으면 생성
    }
}
```

### 2. Prototype Scope 상세 — 추적하지 않음

```java
// PrototypeScope는 별도 클래스가 없음
// AbstractBeanFactory.doGetBean()의 isPrototype() 분기가 직접 처리

} else if (mbd.isPrototype()) {
    Object prototypeInstance;
    try {
        beforePrototypeCreation(beanName);   // 생성 중 표시
        prototypeInstance = createBean(beanName, mbd, args);  // 항상 새 인스턴스
    } finally {
        afterPrototypeCreation(beanName);    // 생성 중 표시 해제
    }
    return adaptBeanInstance(name, prototypeInstance, requiredType);
}
// → singletonObjects 등록 없음
// → disposableBeans 등록 없음 (소멸 콜백 없음)
```

### 3. Scope Proxy 생성 원리

```java
// @Scope(proxyMode = TARGET_CLASS) 처리
// ScopedProxyUtils.createScopedProxy()

public static BeanDefinitionHolder createScopedProxy(
        BeanDefinitionHolder definition, BeanDefinitionRegistry registry, boolean proxyTargetClass) {

    String originalBeanName = definition.getBeanName();
    BeanDefinition targetDefinition = definition.getBeanDefinition();

    // 원본 Bean을 "scopedTarget.beanName"으로 재등록
    String targetBeanName = getTargetBeanName(originalBeanName);  // "scopedTarget.shoppingCart"
    registry.registerBeanDefinition(targetBeanName, targetDefinition);

    // 프록시 BeanDefinition 생성
    RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
    proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
    // → 원본 beanName으로 프록시 등록

    return new BeanDefinitionHolder(proxyDefinition, originalBeanName);
}
```

```java
// ScopedProxyFactoryBean — 실제 프록시 생성
public class ScopedProxyFactoryBean extends ProxyConfig implements FactoryBean<Object> {

    @Override
    public Object getObject() {
        // CGLIB 또는 JDK Proxy 생성
        // TargetSource: 매 메서드 호출마다 현재 Scope에서 실제 Bean 조회
        return createScopedProxy();
    }

    // SimpleBeanTargetSource — 현재 Scope에서 Bean 탐색
    class SimpleBeanTargetSource implements TargetSource {
        @Override
        public Object getTarget() {
            // 요청마다: RequestScope.get() → HttpServletRequest에서 Bean 조회
            // 없으면: 새 인스턴스 생성 후 Request 속성에 저장
            return beanFactory.getBean(targetBeanName);
        }
    }
}
```

```
Scope Proxy 구조:

컨테이너에는 두 Bean이 등록됨:
  "shoppingCart"        → ScopedProxyFactoryBean이 만든 CGLIB 프록시
  "scopedTarget.shoppingCart" → 실제 ShoppingCart BeanDefinition

OrderService에 주입되는 것:
  → "shoppingCart" 즉 프록시

cart.add(item) 호출 시:
  → 프록시 intercept
  → beanFactory.getBean("scopedTarget.shoppingCart") 호출
  → RequestScope.get() → 현재 Request에서 인스턴스 탐색
  → 없으면 createBean() → Request 속성에 저장
  → 있으면 그것 반환
  → 실제 ShoppingCart.add(item) 위임
```

### 4. proxyMode — TARGET_CLASS vs INTERFACES

```java
// TARGET_CLASS: CGLIB 서브클래스 프록시
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart { ... }  // 인터페이스 없어도 OK

// INTERFACES: JDK 동적 프록시
@Scope(value = "prototype", proxyMode = ScopedProxyMode.INTERFACES)
public class ShoppingCartImpl implements ShoppingCart { ... }
// → ShoppingCart 인터페이스를 통해서만 주입 가능
```

```
선택 기준:
  TARGET_CLASS (CGLIB)
    인터페이스 없어도 사용 가능
    final 클래스 / final 메서드 사용 불가
    기본 생성자 필요 (CGLIB 서브클래싱)

  INTERFACES (JDK Proxy)
    인터페이스 필수
    final 제약 없음
    인터페이스에 정의된 메서드만 프록시 경유

실무: 대부분 TARGET_CLASS (인터페이스 없는 경우 많음)
```

### 5. Request / Session Scope 동작

```java
// RequestScope.get() — HttpServletRequest에 Bean 저장
public class RequestScope extends AbstractRequestAttributesScope {

    @Override
    protected int getScope() {
        return RequestAttributes.SCOPE_REQUEST;
    }
}

// AbstractRequestAttributesScope.get()
public Object get(String name, ObjectFactory<?> objectFactory) {
    RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();

    // Request 속성에서 Bean 조회
    Object scopedObject = attributes.getAttribute(name, getScope());

    if (scopedObject == null) {
        // 없으면 새로 생성
        scopedObject = objectFactory.getObject();
        // Request 속성에 저장 (요청 종료 시 자동 제거)
        attributes.setAttribute(name, scopedObject, getScope());
    }
    return scopedObject;
}
```

```
Request Scope 생명주기:
  HTTP 요청 시작 → RequestContextHolder에 RequestAttributes 저장
  첫 Bean 접근   → Request 속성에 인스턴스 생성 및 저장
  같은 요청 내   → 같은 인스턴스 재사용
  요청 종료      → Request 속성 자동 제거 → Bean 소멸 콜백 가능

Session Scope:
  HttpSession 속성에 저장
  세션 생존 기간 동안 유지
  세션 무효화 시 소멸
```

---

## 💻 실험으로 확인하기

### 실험 1: Prototype 주입 함정 확인

```java
@Component @Scope("prototype")
class PrototypeBean {
    private final long id = System.nanoTime();
    public long getId() { return id; }
}

@Service
class SingletonService {
    @Autowired PrototypeBean proto;  // proxyMode 없음

    public long getId() { return proto.getId(); }
}

// 테스트
System.out.println(service.getId());  // 1234
System.out.println(service.getId());  // 1234  ← 같은 인스턴스!
```

### 실험 2: Scope Proxy로 해결

```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
class PrototypeBean {
    private final long id = System.nanoTime();
    public long getId() { return id; }
}

// 같은 SingletonService
System.out.println(service.getId());  // 1111
System.out.println(service.getId());  // 2222  ← 매번 새 인스턴스!
```

### 실험 3: 주입된 객체가 프록시임을 확인

```java
@Autowired ShoppingCart cart;

System.out.println(cart.getClass());
// class com.example.ShoppingCart$$SpringCGLIB$$...  ← 프록시

System.out.println(cart instanceof ShoppingCart);
// true  ← 프록시는 ShoppingCart 서브클래스
```

---

## 🤔 트레이드오프

```
Scope Proxy 사용 시:
  장점  Singleton에서 짧은 생명주기 Bean을 안전하게 사용
       코드 변경 최소화 (cart.add() 그대로 사용)
  단점  매 메서드 호출마다 Scope.get() 오버헤드
       프록시 디버깅 어려움 (실제 클래스 확인 필요)
       직렬화 시 주의 (프록시 직렬화 불가)

ObjectProvider 대안:
  @Autowired ObjectProvider<PrototypeBean> provider;
  PrototypeBean bean = provider.getObject();  // 호출 시 새 인스턴스
  → 명시적, 오버헤드 없음, 하지만 API 변경 필요

Request/Session Scope 주의:
  비웹 스레드에서 접근 시 IllegalStateException
    (RequestContextHolder에 RequestAttributes 없음)
  → @Async 메서드에서 Request Scope Bean 사용 주의
  → 비동기 처리 시 필요한 값을 미리 꺼내 전달

Prototype Bean 소멸:
  소멸 콜백 없음 → GC에 의존
  외부 리소스(@PreDestroy로 close 필요)는 ObjectProvider + try-finally
```

---

## 📌 핵심 정리

```
Scope별 특징
  Singleton   컨텍스트당 1개, 소멸 콜백 O
  Prototype   getBean()마다 새 인스턴스, 소멸 콜백 X, 컨테이너 추적 X
  Request     HTTP 요청당 1개, 요청 종료 시 소멸
  Session     HTTP 세션당 1개, 세션 무효화 시 소멸

Scope 불일치 문제
  Singleton에 짧은 생명주기 Bean 주입
  → 주입 시 한 번만 해결 → 이후 동일 인스턴스
  → Prototype/Request/Session 의미 소실

Scope Proxy 해결책
  proxyMode = TARGET_CLASS / INTERFACES
  → 프록시 주입, 메서드 호출 시 Scope.get()으로 실제 Bean 탐색
  → 두 BeanDefinition 등록: 프록시(원본 이름) + 실제(scopedTarget.*)

대안
  ObjectProvider<T>.getObject() → 명시적 새 인스턴스 획득
  → Prototype 주입 문제 해결에 적합
```

---

## 🤔 생각해볼 문제

**Q1.** Prototype Bean에 `@PreDestroy`가 있을 때, 컨테이너가 소멸 콜백을 호출하지 않는 이유를 `registerDisposableBeanIfNecessary()` 로직과 연결해 설명하라.

**Q2.** `@Scope(value = "request", proxyMode = TARGET_CLASS)` Bean을 `@Async` 메서드 내에서 사용하면 어떤 문제가 생기는가?

**Q3.** Singleton Bean A가 `@Scope(value = "prototype", proxyMode = TARGET_CLASS)` Bean B를 주입받을 때, `A.getB().hashCode()`를 두 번 호출하면 같은 값인가 다른 값인가?

> 💡 **해설**
>
> **Q1.** `AbstractBeanFactory.registerDisposableBeanIfNecessary()`에서 `if (!mbd.isPrototype())` 조건으로 Prototype은 `disposableBeans`에 등록하지 않는다. Prototype은 `doGetBean()`에서 `createBean()` 후 바로 반환하며, 컨테이너는 해당 인스턴스를 어떤 Map에도 보관하지 않는다. 인스턴스를 추적하지 않으므로 종료 시점에 어떤 인스턴스가 살아있는지 알 방법이 없어 소멸 콜백 호출 자체가 불가능하다.
>
> **Q2.** `@Async` 메서드는 별도 스레드에서 실행된다. Request Scope Bean은 `RequestContextHolder.currentRequestAttributes()`로 현재 스레드의 Request 속성을 조회한다. 그런데 `@Async` 스레드에는 HTTP 요청 컨텍스트가 전파되지 않으므로 `RequestContextHolder`가 비어 있어 `IllegalStateException: No thread-bound request found`가 발생한다. 해결책은 `@Async` 메서드 호출 전 필요한 값을 꺼내 파라미터로 전달하거나, `RequestContextHolder.setRequestAttributes()`로 수동으로 전파하는 것이다.
>
> **Q3.** proxyMode가 `TARGET_CLASS`이고 Scope가 `prototype`이면, `getB()`를 호출할 때마다 프록시가 `Scope.get()`을 호출하고 Prototype이므로 매번 `createBean()`이 실행되어 새 인스턴스가 반환된다. 따라서 `A.getB()`를 두 번 호출하면 서로 다른 인스턴스가 반환되고 `hashCode()` 값도 다르다. 만약 같은 메서드 호출 내에서 `B b = A.getB(); b.hashCode(); b.hashCode();`처럼 참조를 유지한다면 같은 인스턴스이므로 같은 값이다.

---

<div align="center">

**[⬅️ 이전: Aware 인터페이스 체인](./04-aware-interfaces.md)** | **[다음: FactoryBean vs ObjectProvider ➡️](./06-factorybean-vs-objectprovider.md)**

</div>

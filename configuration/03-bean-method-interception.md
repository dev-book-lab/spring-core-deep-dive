# @Bean 메서드 호출 가로채기 — BeanMethodInterceptor의 싱글톤 보장 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `BeanMethodInterceptor.intercept()`가 호출되는 정확한 경로는?
- "현재 Bean 생성 중인가?"를 어떻게 판단해 재귀 호출을 처리하는가?
- 이미 등록된 Singleton Bean을 반환하는 경로와 최초 생성 경로의 분기는?
- `@Bean(name = "...")` 메서드 이름 vs Bean 이름 충돌은 어떻게 처리되는가?
- Scoped Bean(`@Scope("prototype")`)에 대한 처리는 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: @Bean 메서드를 직접 호출했을 때 컨테이너의 Bean을 반환해야 한다

```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    @Bean
    public OrderService orderService() {
        DataSource ds = dataSource();
        // 이 호출이 컨테이너의 DataSource를 반환해야 함
        // 그냥 실행하면 new HikariDataSource() 새 인스턴스 생성
        return new OrderService(ds);
    }
}
```

```
BeanMethodInterceptor의 역할:
  dataSource() 호출 → 가로챔
  → 컨테이너에 "dataSource" Bean이 이미 있는가?
    있음 → 컨테이너에서 꺼내 반환 (싱글톤 보장)
    없음 → 현재 이 Bean이 생성 중인가?
      생성 중 → super.dataSource() 호출 (재귀 방지)
      아님 → ctx.getBean("dataSource") → 새로 생성 후 반환
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: BeanMethodInterceptor가 모든 호출을 컨테이너로 위임한다

```
❌ 잘못된 이해:
  "dataSource() 호출 → 항상 beanFactory.getBean('dataSource')"

✅ 실제 (현재 Bean이 생성 중인 경우):
  컨테이너가 DataSource Bean을 최초 생성하는 도중
  → orderService()의 dataSource() 호출
  → 이 시점엔 DataSource가 아직 컨테이너에 없음
  → isCurrentlyInCreation("dataSource") 체크
  → 생성 중이면 → super.dataSource() 호출 (원본 실행)
  → 생성 중이 아니면 → beanFactory.getBean("dataSource")
```

### Before: @Bean 메서드 이름이 곧 Bean 이름이다

```java
// ❌ 잘못된 이해
@Bean
public DataSource myDataSource() { ... }
// → Bean 이름 = "myDataSource" 라고 생각

// ✅ 실제: @Bean(name = "dataSource") 가 있으면 그게 Bean 이름
@Bean(name = {"dataSource", "primaryDs"})
public DataSource myDataSource() { ... }
// → Bean 이름 = "dataSource" (첫 번째가 기본)
// → 메서드 이름 "myDataSource"는 Bean 이름이 아님

// BeanMethodInterceptor에서 Bean 이름 결정:
// @Bean의 name 속성 → name[0]
// name 없으면 → 메서드 이름
```

---

## ✨ 올바른 이해와 사용

### After: intercept() 의 분기 로직 전체 흐름

```
dataSource() 호출 (CGLIB 오버라이딩 메서드)
  ↓ BeanMethodInterceptor.intercept()
  ↓
  resolveBeanReference(method, beanName, ...)
  ↓
  현재 호출이 외부(스프링 컨테이너)에서 왔는가?
    isCurrentlyInvokedFactoryMethod(method)
    → true (컨테이너가 이 메서드를 직접 호출 중)
    → super.dataSource() 실행 → 실제 객체 생성
  ↓
  → false (다른 @Bean 메서드 내부에서 직접 호출)
    beanFactory.getBean(beanName)
    → 이미 있으면 기존 Bean 반환
    → 없으면 (Prototype 등) 새로 생성 후 반환
```

---

## 🔬 내부 동작 원리

### 1. BeanMethodInterceptor.intercept() — 분기의 핵심

```java
// ConfigurationClassEnhancer.BeanMethodInterceptor
private static class BeanMethodInterceptor implements MethodInterceptor, ConditionalCallback {

    @Override
    public Object intercept(Object enhancedConfigInstance, Method beanMethod,
                             Object[] beanMethodArgs, MethodProxy cglibMethodProxy)
            throws Throwable {

        // 1. $$beanFactory 필드에서 BeanFactory 꺼내기
        ConfigurableBeanFactory beanFactory =
            getBeanFactory(enhancedConfigInstance);

        // 2. 이 @Bean 메서드의 Bean 이름 결정
        String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);
        // → @Bean(name="...") 있으면 그것, 없으면 메서드 이름

        // 3. @Scope("scopedProxy") 처리 확인
        if (BeanAnnotationHelper.isScopedProxy(beanMethod)) {
            String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
            if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
                beanName = scopedBeanName;
            }
        }

        // 4. 핵심 분기: 현재 이 메서드가 컨테이너에 의해 직접 호출되고 있는가?
        if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
            // 컨테이너가 이 @Bean 메서드를 통해 Bean을 생성 중
            // → super 메서드를 그냥 실행해서 실제 객체 반환
            // → BeanFactory가 이 결과를 싱글톤 캐시에 저장할 것임
            return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
        }

        // 5. 다른 @Bean 메서드 내부에서 직접 호출된 경우
        // → 컨테이너에서 기존 Bean을 반환
        return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
    }
}
```

### 2. isCurrentlyInvokedFactoryMethod() — "컨테이너 호출 중" 판별

```java
private static boolean isCurrentlyInvokedFactoryMethod(Method method) {
    Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
    // → ThreadLocal에 저장된 "현재 컨테이너가 호출 중인 팩토리 메서드"

    return (currentlyInvoked != null
        && method.getName().equals(currentlyInvoked.getName())
        && Arrays.equals(method.getParameterTypes(), currentlyInvoked.getParameterTypes()));
}
```

```java
// SimpleInstantiationStrategy.instantiate() — @Bean 메서드로 Bean 생성 시
public Object instantiate(RootBeanDefinition bd, BeanWrapper bw, ...) {
    // @Bean 메서드로 생성하는 경우 factoryMethod를 ThreadLocal에 저장
    Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();
    try {
        currentlyInvokedFactoryMethod.set(uniqueCandidate);
        // → "현재 컨테이너가 dataSource() 호출 중" 표시

        Object result = factoryMethod.invoke(factoryBean, args);
        // → AppConfig$$SpringCGLIB$$0.dataSource() 호출
        //   → BeanMethodInterceptor.intercept() 실행
        //   → isCurrentlyInvokedFactoryMethod() = true
        //   → cglibMethodProxy.invokeSuper() → 실제 객체 생성

        return result;
    } finally {
        currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
        // ThreadLocal 복원
    }
}
```

```
ThreadLocal 흐름 정리:

컨테이너가 DataSource 생성 시작
  → SimpleInstantiationStrategy: currentlyInvoked = dataSource 메서드 저장
    → CGLIB.dataSource() 호출
      → BeanMethodInterceptor.intercept()
        → isCurrentlyInvokedFactoryMethod() = true (ThreadLocal 일치)
        → invokeSuper() → new HikariDataSource() 반환
    → 결과를 singletonObjects에 저장
  → currentlyInvoked 복원 (null)

이후 orderService() 내부에서 dataSource() 호출:
  → BeanMethodInterceptor.intercept()
    → isCurrentlyInvokedFactoryMethod() = false (ThreadLocal에 dataSource 없음)
    → resolveBeanReference() → beanFactory.getBean("dataSource") → 캐시된 Bean 반환
```

### 3. resolveBeanReference() — 컨테이너에서 Bean 조회

```java
private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
                                     ConfigurableBeanFactory beanFactory, String beanName) {
    // Bean이 현재 생성 중인가? (Circular Dependency 감지)
    boolean alreadyInCreation = beanFactory.isCurrentlyInCreation(beanName);

    try {
        if (alreadyInCreation) {
            // 순환 참조 중이면 생성 중 상태에서 잠시 해제
            beanFactory.setCurrentlyInCreation(beanName, false);
        }

        // 파라미터가 있는 @Bean 메서드: 인수 처리
        Object beanInstance = (!ObjectUtils.isEmpty(beanMethodArgs)
            ? beanFactory.getBean(beanName, beanMethodArgs)
            : beanFactory.getBean(beanName));
        // → Singleton이면 캐시에서 반환
        // → Prototype이면 새로 생성

        // 반환 타입 확인 (디버그 목적)
        if (!ClassUtils.isAssignableValue(beanMethod.getReturnType(), beanInstance)) {
            if (beanInstance.equals(null)) {
                // NullBean 처리
                beanInstance = null;
            }
        }
        return beanInstance;

    } finally {
        if (alreadyInCreation) {
            beanFactory.setCurrentlyInCreation(beanName, true);
        }
    }
}
```

### 4. Scoped Bean (@Scope("prototype")) 처리

```java
// Prototype Bean에서의 동작

@Configuration
public class AppConfig {

    @Bean
    @Scope("prototype")
    public RequestContext requestContext() {
        return new RequestContext();
    }

    @Bean
    public OrderService orderService() {
        // requestContext() 직접 호출
        // → BeanMethodInterceptor.intercept()
        // → isCurrentlyInvokedFactoryMethod() = false
        // → beanFactory.getBean("requestContext")
        //   → Prototype이므로 항상 새 인스턴스 생성
        return new OrderService(requestContext());
    }
}
```

```
Singleton vs Prototype 처리:
  Singleton: beanFactory.getBean() → singletonObjects 캐시 반환
  Prototype: beanFactory.getBean() → 매번 새 인스턴스 생성
  Request/Session: beanFactory.getBean() → 현재 스코프 인스턴스 반환

  → BeanMethodInterceptor는 컨테이너 위임만 담당
  → 실제 스코프 처리는 BeanFactory 내부 로직
```

### 5. @Bean 이름 결정 — BeanAnnotationHelper

```java
// BeanAnnotationHelper.determineBeanNameFor()
public static String determineBeanNameFor(Method beanMethod) {
    AnnotationAttributes bean =
        AnnotationConfigUtils.attributesFor(beanMethod, Bean.class);

    // @Bean(name = {"primaryDs", "dataSource"}) → "primaryDs" (첫 번째)
    List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
    if (!names.isEmpty()) {
        return names.get(0);
    }

    // name 없으면 메서드 이름
    return beanMethod.getName();
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 직접 호출 vs 컨테이너 호출 동작 비교

```java
@Configuration
public class AppConfig {
    private int count = 0;

    @Bean
    public DataSource dataSource() {
        count++;
        System.out.println("DataSource 생성 #" + count);
        return new HikariDataSource();
    }

    @Bean
    public OrderService orderService() { return new OrderService(dataSource()); }

    @Bean
    public AuditService auditService() { return new AuditService(dataSource()); }
}

// 출력:
// DataSource 생성 #1  ← 컨테이너가 dataSource Bean 최초 생성 시만
// (orderService, auditService의 dataSource() 호출은 캐시 반환)
```

### 실험 2: ThreadLocal 동작 확인 (개념적)

```java
// SimpleInstantiationStrategy를 상속해 ThreadLocal 값 출력
class DebugInstantiationStrategy extends SimpleInstantiationStrategy {
    @Override
    public Object instantiate(...) {
        System.out.println("현재 팩토리 메서드: " +
            getCurrentlyInvokedFactoryMethod());
        return super.instantiate(...);
    }
}
```

### 실험 3: Prototype @Bean 직접 호출

```java
@Configuration
public class AppConfig {

    @Bean
    @Scope("prototype")
    public Validator validator() {
        return new Validator();
    }

    @Bean
    public OrderService orderService() {
        Validator v1 = validator();  // BeanMethodInterceptor → 새 인스턴스
        Validator v2 = validator();  // BeanMethodInterceptor → 또 새 인스턴스
        System.out.println(v1 == v2);  // false
        return new OrderService(v1);
    }
}
```

---

## 🤔 트레이드오프

```
BeanMethodInterceptor ThreadLocal 방식:
  장점  스레드 안전, 재귀/순환 호출도 정확히 처리
  단점  ThreadLocal 설정/복원 오버헤드 (미미)
        디버깅 시 ThreadLocal 상태 확인 필요

직접 호출 vs 파라미터 주입:
  // 직접 호출 방식 (Full Mode만 안전)
  @Bean OrderService orderService() {
      return new OrderService(dataSource());
  }

  // 파라미터 주입 방식 (Full/Lite Mode 모두 안전)
  @Bean OrderService orderService(DataSource ds) {
      return new OrderService(ds);
  }
  → 파라미터 주입이 더 명시적이고 테스트 친화적
  → Lite Mode 전환 시에도 안전

Scoped Proxy와 조합:
  @Scope(value="request", proxyMode=ScopedProxyMode.TARGET_CLASS)
  + @Configuration Full Mode
  → BeanMethodInterceptor가 ScopedProxy Bean을 반환
  → 스코프 처리는 프록시가 담당
```

---

## 📌 핵심 정리

```
BeanMethodInterceptor.intercept() 분기

isCurrentlyInvokedFactoryMethod() = true
  컨테이너가 이 @Bean 메서드를 직접 호출 중
  → invokeSuper() → 원본 메서드 실행 → 실제 객체 생성
  → 결과를 컨테이너가 싱글톤 캐시에 저장

isCurrentlyInvokedFactoryMethod() = false
  다른 @Bean 메서드 내부에서 직접 호출
  → beanFactory.getBean(beanName)
  → Singleton: 캐시에서 반환
  → Prototype: 새 인스턴스 생성

ThreadLocal 기반 판별
  SimpleInstantiationStrategy가 @Bean 메서드 호출 전
  currentlyInvokedFactoryMethod ThreadLocal에 메서드 저장
  → BeanMethodInterceptor에서 비교로 "컨테이너 직접 호출"인지 판별

Bean 이름 결정
  @Bean(name="...") → name[0]
  없으면 → 메서드 이름

Scoped Bean 처리
  beanFactory.getBean() 위임 → 스코프 정책은 BeanFactory가 처리
  Prototype → 항상 새 인스턴스
```

---

## 🤔 생각해볼 문제

**Q1.** `@Bean` 메서드가 `private`이면 `BeanMethodInterceptor`가 동작하는가?

**Q2.** `@Configuration` 클래스 안에서 `this.dataSource()`로 호출하는 것과 `dataSource()`로 호출하는 것의 차이는?

**Q3.** 두 개의 `@Configuration` 클래스 A, B가 있고 A의 `@Bean` 메서드 안에서 B의 `@Bean` 메서드를 직접 호출하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 동작하지 않는다. CGLIB 서브클래스는 원본 클래스를 상속해 `@Bean` 메서드를 `@Override`하는 방식으로 인터셉션을 구현한다. `private` 메서드는 JVM 규칙상 서브클래스에서 오버라이딩 불가(`invokespecial`로 호출됨)이므로 CGLIB이 인터셉터 코드를 삽입할 수 없다. 따라서 `private @Bean` 메서드는 컨테이너가 등록하지도 않고 인터셉션도 되지 않는다. 스프링은 `private @Bean` 메서드를 발견하면 경고 로그를 남기고 무시한다.
>
> **Q2.** 결과는 동일하다. `this.dataSource()`이든 `dataSource()`이든 JVM 바이트코드 레벨에서는 둘 다 `invokevirtual this.dataSource()`로 컴파일된다. CGLIB 서브클래스의 `this`는 `AppConfig$$SpringCGLIB$$0` 인스턴스이고, `invokevirtual`은 동적 디스패치이므로 `AppConfig$$SpringCGLIB$$0.dataSource()` → `BeanMethodInterceptor.intercept()`로 연결된다. `super.dataSource()`처럼 `invokespecial`로 원본을 직접 호출하는 경우에만 인터셉션이 우회된다.
>
> **Q3.** `B`의 `@Bean` 메서드도 `B$$SpringCGLIB$$0` 인스턴스를 통해 오버라이딩됐지만, `A`의 코드 내에서 `bInstance.beanMethod()` 형태로 호출해야 인터셉션이 동작한다. 만약 `A` 안에 `B` 인스턴스가 주입되어 있고 그것이 CGLIB 프록시라면 `beanFactory.getBean()`으로 위임된다. 그러나 `A` 내부에서 `new B().beanMethod()`처럼 직접 새 인스턴스를 만들어 호출하면 원본 `B` 인스턴스이므로 CGLIB 인터셉션이 없고 새 객체가 생성된다. 일반적으로 다른 `@Configuration`의 `@Bean`을 참조할 때는 `@Autowired`로 주입받아 사용하는 것이 올바른 패턴이다.

---

<div align="center">

**[⬅️ 이전: CGLIB 프록시가 적용되는 이유](./02-cglib-proxy-on-configuration.md)** | **[다음: Lite Mode vs Full Mode 트레이드오프 ➡️](./04-lite-mode-vs-full-mode.md)**

</div>

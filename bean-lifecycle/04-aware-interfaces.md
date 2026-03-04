# Aware 인터페이스 체인 — 컨테이너 인프라에 접근하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`는 언제, 어떤 순서로 호출되는가?
- `invokeAwareMethods()`와 `ApplicationContextAwareProcessor` 두 경로로 나뉘는 이유는?
- Aware 인터페이스 대신 `@Autowired`로 `ApplicationContext`를 주입받으면 어떤 차이가 있는가?
- Aware 인터페이스 사용의 트레이드오프는 무엇인가?
- 커스텀 Aware 인터페이스를 만들 수 있는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean이 자신의 컨테이너 정보에 접근해야 하는 경우

```java
@Service
public class DynamicBeanLoader {
    // 런타임에 다른 Bean을 동적으로 조회해야 함
    // 자신의 Bean 이름을 알아야 로그에 남길 수 있음
    // ApplicationContext에 접근해 이벤트를 발행해야 함
}
```

```
필요한 정보:
  - 자신의 Bean 이름 → BeanNameAware
  - BeanFactory 참조 → BeanFactoryAware
  - ApplicationContext 참조 → ApplicationContextAware
  - ClassLoader → BeanClassLoaderAware
  - Environment → EnvironmentAware
  - 기타 인프라 객체

해결:
  스프링이 특정 인터페이스를 구현한 Bean을 감지
  → Bean 초기화 단계에서 해당 정보를 콜백으로 주입
  → Bean이 원할 때 사용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Aware 인터페이스가 @Autowired보다 항상 낫다

```java
// ❌ Aware 남용 — 설계 결합도 증가
@Service
public class OrderService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
    }

    public void process() {
        // ApplicationContext를 통해 Bean을 매번 동적 조회
        PaymentService ps = applicationContext.getBean(PaymentService.class);
        ps.pay();
    }
}
```

```
문제:
  ApplicationContext를 Service Layer에 끌어들임
  → 스프링 컨테이너에 강하게 결합
  → 테스트 시 ApplicationContext Mock 필요
  → DI 원칙 위반 (정적 의존성 대신 동적 조회)

대부분의 경우:
  @Autowired PaymentService paymentService; 가 더 적절
  Aware는 정말 컨테이너 인프라가 필요한 경우에만 사용
```

### Before: 모든 Aware가 같은 시점에 호출된다

```java
// ❌ 잘못된 이해
// "BeanFactoryAware와 ApplicationContextAware는 같은 시점"

// ✅ 실제:
// BeanFactoryAware   → invokeAwareMethods() (5단계, BPP Before 이전)
// ApplicationContextAware → ApplicationContextAwareProcessor (BPP Before 내부)
// → 서로 다른 시점, 다른 처리 경로
```

---

## ✨ 올바른 이해와 사용

### After: 두 처리 경로와 호출 순서를 명확히 파악

```
Aware 처리 두 경로:

경로 1: invokeAwareMethods() — BPP Before 이전
  BeanNameAware.setBeanName()
  BeanClassLoaderAware.setBeanClassLoader()
  BeanFactoryAware.setBeanFactory()
  → AbstractAutowireCapableBeanFactory가 직접 처리
  → 빠른 경로 (BPP 체인 밖)

경로 2: ApplicationContextAwareProcessor (BPP Before 내)
  EnvironmentAware.setEnvironment()
  EmbeddedValueResolverAware.setEmbeddedValueResolver()
  ResourceLoaderAware.setResourceLoader()
  ApplicationEventPublisherAware.setApplicationEventPublisher()
  MessageSourceAware.setMessageSource()
  ApplicationContextAware.setApplicationContext()
  → BPP Before 체인에서 처리

전체 순서:
  setBeanName → setBeanClassLoader → setBeanFactory
  → (BPP Before 시작)
  → setEnvironment → setResourceLoader → setApplicationContext
  → @PostConstruct
```

---

## 🔬 내부 동작 원리

### 1. invokeAwareMethods() — 경로 1

```java
// AbstractAutowireCapableBeanFactory.invokeAwareMethods()
private void invokeAwareMethods(String beanName, Object bean) {

    if (bean instanceof Aware) {

        if (bean instanceof BeanNameAware beanNameAware) {
            beanNameAware.setBeanName(beanName);
        }

        if (bean instanceof BeanClassLoaderAware beanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                beanClassLoaderAware.setBeanClassLoader(bcl);
            }
        }

        if (bean instanceof BeanFactoryAware beanFactoryAware) {
            beanFactoryAware.setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

```
경로 1 특징:
  BPP 체인을 거치지 않는 직접 instanceof 체크
  세 가지만 처리 (BeanName, BeanClassLoader, BeanFactory)
  BPP Before보다 먼저 실행 → 가장 빠른 시점에 주입
  BeanFactory는 ApplicationContext보다 상위 개념
  → BeanFactory 주입이 ApplicationContext 주입보다 항상 먼저
```

### 2. ApplicationContextAwareProcessor — 경로 2

```java
// ApplicationContextAwareProcessor.postProcessBeforeInitialization()
public Object postProcessBeforeInitialization(Object bean, String beanName) {

    if (!(bean instanceof EnvironmentAware
            || bean instanceof EmbeddedValueResolverAware
            || bean instanceof ResourceLoaderAware
            || bean instanceof ApplicationEventPublisherAware
            || bean instanceof MessageSourceAware
            || bean instanceof ApplicationContextAware
            || bean instanceof ApplicationStartupAware)) {
        return bean;  // 해당 없으면 즉시 반환 (성능 최적화)
    }

    invokeAwareInterfaces(bean);
    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    // 순서대로 호출 (조건부)
    if (bean instanceof EnvironmentAware ea) {
        ea.setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware evra) {
        evra.setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware rla) {
        rla.setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware aepa) {
        aepa.setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware msa) {
        msa.setMessageSource(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware aca) {
        aca.setApplicationContext(this.applicationContext);
    }
}
```

```
경로 2 특징:
  BPP Before 체인 내에서 실행
  ApplicationContextAwareProcessor가 가장 먼저 등록되는 BPP
  → 사용자 정의 BPP보다 먼저 실행
  ApplicationContext 자체를 여러 Aware에 주입
    (ApplicationContext는 ResourceLoader, EventPublisher 등 구현)
```

### 3. 전체 Aware 호출 순서 정리

```java
// 모든 Aware 구현 시 호출 순서 확인 예제
@Component
public class AllAwareBean implements
        BeanNameAware, BeanClassLoaderAware, BeanFactoryAware,
        EnvironmentAware, ResourceLoaderAware,
        ApplicationEventPublisherAware, ApplicationContextAware {

    @Override public void setBeanName(String name) {
        System.out.println("1. BeanNameAware");
    }
    @Override public void setBeanClassLoader(ClassLoader cl) {
        System.out.println("2. BeanClassLoaderAware");
    }
    @Override public void setBeanFactory(BeanFactory bf) {
        System.out.println("3. BeanFactoryAware");
    }
    @Override public void setEnvironment(Environment env) {
        System.out.println("4. EnvironmentAware");           // BPP Before 시작
    }
    @Override public void setResourceLoader(ResourceLoader rl) {
        System.out.println("5. ResourceLoaderAware");
    }
    @Override public void setApplicationEventPublisher(ApplicationEventPublisher pub) {
        System.out.println("6. ApplicationEventPublisherAware");
    }
    @Override public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("7. ApplicationContextAware");
    }

    @PostConstruct
    public void init() {
        System.out.println("8. @PostConstruct");
    }
}
```

```
출력:
1. BeanNameAware
2. BeanClassLoaderAware
3. BeanFactoryAware
4. EnvironmentAware
5. ResourceLoaderAware
6. ApplicationEventPublisherAware
7. ApplicationContextAware
8. @PostConstruct
```

### 4. @Autowired vs ApplicationContextAware 비교

```java
// 방법 1: ApplicationContextAware
@Service
public class ServiceA implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void doWork() {
        SomeBean bean = ctx.getBean(SomeBean.class);
    }
}

// 방법 2: @Autowired
@Service
public class ServiceB {
    @Autowired
    private ApplicationContext ctx;  // ApplicationContext를 직접 주입받을 수 있음

    public void doWork() {
        SomeBean bean = ctx.getBean(SomeBean.class);
    }
}
```

```
차이:
  ApplicationContextAware
    → 초기화 순서 보장 (BPP Before에서 setApplicationContext 호출)
    → 스프링 인터페이스 구현 → 코드 결합
    → 인프라 컴포넌트(BPP, BeanFactoryPostProcessor)에서 유용

  @Autowired ApplicationContext
    → 일반 Bean에서 더 간결
    → 동일한 ApplicationContext 주입
    → 대부분의 경우 권장

실무 선택:
  일반 @Service, @Component → @Autowired ApplicationContext
  BeanPostProcessor 구현체 → ApplicationContextAware
    (BPP 자체가 일반 Bean보다 먼저 생성 → @Autowired 시점 불안정)
```

### 5. 커스텀 Aware 인터페이스

```java
// 커스텀 Aware 정의
public interface TenantContextAware extends Aware {
    void setTenantContext(TenantContext tenantContext);
}

// 커스텀 BPP로 처리
@Component
public class TenantContextAwareProcessor implements BeanPostProcessor {

    @Autowired
    private TenantContext tenantContext;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof TenantContextAware tca) {
            tca.setTenantContext(tenantContext);
        }
        return bean;
    }
}

// 사용
@Service
public class TenantService implements TenantContextAware {
    private TenantContext tenantContext;

    @Override
    public void setTenantContext(TenantContext ctx) {
        this.tenantContext = ctx;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: BeanFactoryAware vs ApplicationContextAware 시점 차이 확인

```java
@Component
public class TimingBean implements BeanFactoryAware, ApplicationContextAware {

    @Override
    public void setBeanFactory(BeanFactory bf) {
        // ApplicationContext인지 확인
        System.out.println("BeanFactory 시점: "
            + (bf instanceof ApplicationContext ? "AppCtx" : "BeanFactory only"));
        // → "BeanFactory only"
        // BeanFactory 주입 시점에는 아직 ApplicationContext 완성 전
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("ApplicationContext 시점: " + ctx.getDisplayName());
        // → 완전히 초기화된 ApplicationContext
    }
}
```

### 실험 2: 동적 Bean 조회 필요 시 올바른 패턴

```java
// 플러그인 시스템 — 런타임에 Bean 이름으로 조회
@Service
public class PluginManager implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void executePlugin(String pluginBeanName) {
        // 런타임에 결정되는 Bean 이름 → 동적 조회 불가피
        Plugin plugin = ctx.getBean(pluginBeanName, Plugin.class);
        plugin.execute();
    }
}
```

---

## 🤔 트레이드오프

```
Aware 인터페이스 사용:
  장점  초기화 순서 명확, 인프라 접근 공식 경로
  단점  스프링 인터페이스 구현 → 코드 결합
       테스트 시 Mock 설정 복잡

@Autowired로 대체 가능 여부:
  ApplicationContext @Autowired → 대부분 대체 가능
  BeanFactory @Autowired       → 대체 가능
  BeanName → @Value("${...}")나 리플렉션보다 BeanNameAware가 간결

Aware 사용이 적절한 경우:
  BeanPostProcessor / BeanFactoryPostProcessor 구현체
    → @Autowired 시점 불안정, Aware가 안전
  인프라 컴포넌트 (ApplicationContext 계층 구현)
  커스텀 Aware 체인으로 도메인 컨텍스트 전파

사용 지양:
  일반 Service / Repository 레이어
  → 컨테이너 인프라를 비즈니스 로직에 끌어들이는 것은 설계 오염
```

---

## 📌 핵심 정리

```
두 처리 경로

경로 1: invokeAwareMethods() — BPP Before 이전
  BeanNameAware → BeanClassLoaderAware → BeanFactoryAware

경로 2: ApplicationContextAwareProcessor — BPP Before 내
  EnvironmentAware → EmbeddedValueResolverAware
  → ResourceLoaderAware → ApplicationEventPublisherAware
  → MessageSourceAware → ApplicationContextAware

전체 순서 (항상 고정)
  setBeanName → setBeanClassLoader → setBeanFactory
  → setEnvironment → setResourceLoader
  → setApplicationEventPublisher → setApplicationContext
  → @PostConstruct

@Autowired vs Aware
  일반 Bean: @Autowired ApplicationContext 권장
  BPP 구현체: ApplicationContextAware 사용

커스텀 Aware
  Aware 마커 인터페이스 상속 + BPP로 처리
```

---

## 🤔 생각해볼 문제

**Q1.** `BeanFactoryAware.setBeanFactory()`에서 주입받는 `BeanFactory`가 `ApplicationContext`인지 확인하는 방법은? 실제로 무엇이 주입되는가?

**Q2.** `BeanPostProcessor` 구현체에서 `@Autowired ApplicationContext`를 쓰면 어떤 문제가 생길 수 있는가? `ApplicationContextAware`를 써야 하는 이유는?

**Q3.** 다음 코드에서 `setApplicationContext()` 안에서 `getBean()`을 호출해도 안전한가?

```java
@Component
public class EarlyAccessBean implements ApplicationContextAware {
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        SomeService svc = ctx.getBean(SomeService.class);  // 안전한가?
        svc.init();
    }
}
```

> 💡 **해설**
>
> **Q1.** `AbstractAutowireCapableBeanFactory`는 `ConfigurableListableBeanFactory`의 구현체이고, 실제 컨테이너는 `AnnotationConfigApplicationContext`처럼 `ConfigurableApplicationContext`를 구현한다. `setBeanFactory()`에 주입되는 것은 `AbstractAutowireCapableBeanFactory.this` 즉 내부 BeanFactory 구현체다. `instanceof ApplicationContext`로 확인하면 `false`가 나온다. ApplicationContext 자체가 주입되려면 `ApplicationContextAware`를 사용해야 한다.
>
> **Q2.** BPP는 일반 Bean보다 먼저 생성된다. BPP 생성 시점에 `@Autowired ApplicationContext`를 주입받으려면 `ApplicationContext` Bean이 이미 준비돼 있어야 하는데, BPP 생성 단계에서는 `ApplicationContext`가 아직 완전히 초기화되지 않은 상태일 수 있다. 스프링이 경고 로그(`is not eligible for getting processed by all BeanPostProcessors`)를 출력하거나 주입이 불안정하게 이루어질 수 있다. `ApplicationContextAware`는 BPP 체인을 통해 안전한 시점에 주입되므로 BPP 구현체에서는 이 방식이 안전하다.
>
> **Q3.** 위험하다. `setApplicationContext()`는 `BPP Before` 단계에서 호출된다. 이 시점에 다른 Bean들이 아직 완전히 초기화되지 않았을 수 있다. `SomeService`가 아직 생성 중이거나 자신의 `@PostConstruct`를 실행하기 전일 수 있어 불완전한 상태의 Bean을 사용할 위험이 있다. 동적 Bean 조회가 필요하다면 `@PostConstruct`나 `SmartInitializingSingleton.afterSingletonsInstantiated()` 내에서 수행하는 것이 안전하다.

---

<div align="center">

**[⬅️ 이전: @PreDestroy vs DisposableBean](./03-predestroy-vs-disposablebean.md)** | **[다음: Bean Scope와 프록시 ➡️](./05-bean-scope-and-proxy.md)**

</div>

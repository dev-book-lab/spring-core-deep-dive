# Lazy Initialization 동작 원리 — @Lazy가 만드는 프록시

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Lazy`는 Bean 생성 시점을 어떻게 뒤로 미루는가?
- `@Lazy`로 주입받은 객체는 실제로 무엇인가? 진짜 Bean인가, 프록시인가?
- `DefaultListableBeanFactory`에서 Lazy 처리가 이루어지는 정확한 위치는?
- `spring.main.lazy-initialization=true`는 어떻게 전체 Bean에 적용되는가?
- `@Lazy`를 생성자 주입에서 쓰면 순환 참조가 해결되는 원리는?
- Lazy Init의 장단점과 운영 환경 주의사항은?

---

## 🔍 왜 이게 존재하는가

### 문제: 무거운 Bean이 애플리케이션 시작 시간을 길게 만든다

```java
@Service
public class ReportService {
    // 생성 시 대용량 데이터 로드, 외부 API 연결, 복잡한 초기화
    public ReportService() {
        loadLargeDataSet();        // 10초
        connectToExternalSystem(); // 5초
    }
    // 이 서비스는 하루에 한 번 배치에서만 사용됨
}
```

```
문제:
  컨텍스트 refresh() 시 모든 Non-Lazy Singleton 즉시 생성
  → ReportService: 15초 초기화
  → 실제로는 배치 실행 시(하루 1회)만 필요

해결:
  @Lazy → 첫 번째 실제 사용 시점까지 생성 지연
  → 애플리케이션 시작 시간 단축
  → 사용하지 않으면 생성 자체 안 됨
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Lazy를 붙이면 주입받는 곳에서도 @Lazy가 필요 없다

```java
@Service @Lazy
public class HeavyService { ... }

@Service
public class OrderService {
    @Autowired
    private HeavyService heavyService;  // @Lazy 없이 주입
    // → HeavyService를 주입받으려면 HeavyService Bean이 필요
    // → OrderService 생성 시 HeavyService 생성 발생 (Lazy 무효!)
}
```

```
❌ 잘못된 이해:
  "Bean 선언에 @Lazy 붙이면 주입 시 알아서 지연된다"

✅ 실제:
  Bean 선언의 @Lazy  → preInstantiateSingletons()에서 제외
  주입 지점의 @Lazy  → 프록시를 주입받아 실제 Bean 생성 지연

  OrderService가 직접 HeavyService를 주입받으면
  → OrderService 생성 시 HeavyService 필요
  → HeavyService 즉시 생성 (Lazy 효과 없음)

  해결: 주입 지점에도 @Lazy 추가
  @Autowired @Lazy HeavyService heavyService;
```

### Before: @Lazy Bean은 존재하지 않다가 생성된다

```java
// ❌ 잘못된 이해
// "@Lazy Bean은 사용 전에는 컨테이너에 없다"

// ✅ 실제:
// BeanDefinition은 refresh() 시점에 등록됨 (항상)
// 인스턴스만 지연 생성됨
// containsBean("heavyService") → true (BeanDefinition 있음)
// getBean("heavyService")     → 이 시점에 인스턴스 생성
```

---

## ✨ 올바른 이해와 사용

### After: @Lazy의 두 가지 역할을 구분한다

```
@Lazy의 두 가지 사용 위치:

1. Bean 선언에 @Lazy
   @Service @Lazy
   class HeavyService { }
   → preInstantiateSingletons() 루프에서 제외
   → 최초 getBean() 호출 시 생성

2. 주입 지점에 @Lazy
   @Autowired @Lazy HeavyService heavyService;
   → HeavyService 프록시(CGLIB)를 주입
   → heavyService.method() 호출 시 실제 HeavyService 생성
   → 생성자 주입 순환 참조 해결에 활용
```

---

## 🔬 내부 동작 원리

### 1. Bean 선언 @Lazy — preInstantiateSingletons() 제외

```java
// DefaultListableBeanFactory.preInstantiateSingletons()
public void preInstantiateSingletons() {
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

        if (!bd.isAbstract()
                && bd.isSingleton()
                && !bd.isLazyInit()) {      // ← @Lazy면 여기서 건너뜀
            if (isFactoryBean(beanName)) {
                getBean(FACTORY_BEAN_PREFIX + beanName);
            } else {
                getBean(beanName);
            }
        }
    }
    // @Lazy Bean은 이 루프에서 생성 안 됨
    // → 최초 getBean() 호출 시 doGetBean() → createBean() 실행
}
```

```
@Lazy Bean 생성 시점:

refresh() 완료 → @Lazy Bean: BeanDefinition만 존재, 인스턴스 없음

첫 getBean("heavyService"):
  getSingleton("heavyService") → null (1단계 캐시 없음)
  → createBean("heavyService") 실행 → 인스턴스 생성
  → singletonObjects.put("heavyService", instance)
  → 이후 getBean() → 캐시 반환
```

### 2. 주입 지점 @Lazy — 프록시 생성

```java
// @Autowired @Lazy 처리 — AutowiredAnnotationBeanPostProcessor
// AutowiredFieldElement.resolveFieldValue()

protected Object resolveFieldValue(Field field, Object bean, String beanName) {
    DependencyDescriptor desc = new DependencyDescriptor(field, this.required);

    // @Lazy 감지
    if (field.isAnnotationPresent(Lazy.class)) {
        // 실제 Bean 탐색 안 함 → 프록시 생성
        return buildLazyResolutionProxy(desc, beanName);
    }

    return resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
}

// buildLazyResolutionProxy()
private Object buildLazyResolutionProxy(DependencyDescriptor descriptor, String beanName) {
    // TargetSource: 실제 Bean을 지연하여 제공
    TargetSource ts = new TargetSource() {
        @Override
        public Object getTarget() {
            // 이 시점에 실제 Bean 탐색
            return beanFactory.doResolveDependency(descriptor, beanName, null, null);
        }
        @Override
        public Class<?> getTargetClass() {
            return descriptor.getDependencyType();
        }
    };

    // CGLIB 프록시 생성 (인터페이스 있으면 JDK Proxy)
    ProxyFactory pf = new ProxyFactory();
    pf.setTargetSource(ts);
    pf.addInterface(descriptor.getDependencyType());  // 인터페이스가 있으면
    return pf.getProxy(beanFactory.getBeanClassLoader());
}
```

```
주입 지점 @Lazy 흐름:

OrderService 생성 시:
  @Autowired @Lazy HeavyService heavyService
  → buildLazyResolutionProxy() 호출
  → CGLIB/JDK 프록시 객체 생성 (HeavyService 인스턴스 아님)
  → 프록시를 heavyService 필드에 주입

OrderService.someMethod() 내부에서 heavyService.doWork() 호출 시:
  프록시 intercept
  → TargetSource.getTarget() 호출
  → beanFactory.doResolveDependency() → HeavyService Bean 탐색
  → HeavyService 인스턴스 없으면 → createBean() 실행
  → 이후 프록시는 이 인스턴스에 위임
```

### 3. 순환 참조 해결에서의 @Lazy

```java
// 생성자 주입 순환 참조 — 기본적으로 해결 불가
@Service class A { A(B b) {} }  // BeanCurrentlyInCreationException
@Service class B { B(A a) {} }

// @Lazy로 해결
@Service
class A {
    private final B b;
    A(@Lazy B b) {   // B의 프록시를 주입받음
        this.b = b;
    }
}

@Service
class B {
    private final A a;
    B(A a) { this.a = a; }
}
```

```
@Lazy 순환 해결 흐름:

A 생성 시작:
  생성자 파라미터 B 필요
  → @Lazy B → 실제 B 생성 안 함
  → B의 CGLIB 프록시 생성 (즉시)
  → 프록시로 A 생성 완료

B 생성 시작 (A.b.method() 호출 시점 또는 컨텍스트 시작 후):
  생성자 파라미터 A 필요
  → A는 이미 완성 → 주입 성공
  → B 생성 완료

A에서 b.method() 호출:
  프록시 intercept → 실제 B 인스턴스 탐색 → 있으면 위임
```

### 4. spring.main.lazy-initialization — 전체 Bean Lazy 처리

```java
// LazyInitializationBeanFactoryPostProcessor
// spring.main.lazy-initialization=true 시 등록됨
public class LazyInitializationBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        for (String beanName : beanFactory.getBeanDefinitionNames()) {
            BeanDefinition bd = beanFactory.getBeanDefinition(beanName);

            // 이미 명시적으로 lazy=false인 Bean은 건드리지 않음
            if (bd.getLazyInit() == null) {
                bd.setLazyInit(true);  // 모든 Bean에 Lazy 강제 적용
            }
        }
    }
}
```

```
전체 Lazy 적용 시:
  애플리케이션 시작 시간: 크게 단축
  첫 요청 응답 시간: 증가 (많은 Bean이 이때 생성됨)
  시작 시 오류 감지: 불가 (설정 오류가 첫 사용 시점까지 숨어 있음)

권장 사용:
  개발 환경 시작 속도 개선
  운영 환경: 주의 (첫 요청 지연, 오류 조기 감지 불가)
```

---

## 💻 실험으로 확인하기

### 실험 1: Bean 선언 @Lazy 생성 시점 확인

```java
@Service @Lazy
public class HeavyService {
    public HeavyService() {
        System.out.println("[생성] HeavyService @ " + System.currentTimeMillis());
    }
}

@SpringBootApplication
public class App {
    public static void main(String[] args) {
        var ctx = SpringApplication.run(App.class, args);
        System.out.println("컨텍스트 시작 완료");     // 여기까지 HeavyService 생성 안 됨
        ctx.getBean(HeavyService.class);               // 여기서 생성
    }
}
```

```
출력:
컨텍스트 시작 완료
[생성] HeavyService @ 1700000001234   ← getBean() 호출 시 생성
```

### 실험 2: 주입 지점 @Lazy — 프록시 확인

```java
@Service
public class OrderService {
    @Autowired @Lazy
    private HeavyService heavyService;

    public void checkProxy() {
        System.out.println(heavyService.getClass());
        // com.example.HeavyService$$SpringCGLIB$$... ← 프록시
        System.out.println(heavyService instanceof HeavyService);
        // true ← 프록시는 HeavyService 서브클래스
    }
}
```

### 실험 3: 전체 Lazy 시작 시간 비교

```yaml
# application.yml
spring:
  main:
    lazy-initialization: true
```

```
시작 시간 비교 (Bean 500개 기준):
  일반:    약 8초
  Lazy:    약 2초 (첫 요청 시 나머지 생성)
```

---

## 🤔 트레이드오프

```
@Lazy Bean 선언:
  장점  무거운 초기화 지연, 사용 안 하면 생성 안 됨
  단점  첫 사용 시 지연 발생, 시작 시 설정 오류 감지 불가

주입 지점 @Lazy:
  장점  순환 참조 해결, 선택적 지연 로딩
  단점  프록시 오버헤드, 클래스 final이면 CGLIB 적용 불가

spring.main.lazy-initialization=true:
  장점  전체 시작 속도 개선 (개발 환경에 유용)
  단점  첫 요청 응답 지연
       시작 시 설정 오류, @PostConstruct 검증 모두 지연
       운영 환경에서 예상치 못한 지연 스파이크

운영 환경 권장:
  전체 Lazy 사용 자제
  시작 시간 개선이 필요하면 선택적 @Lazy 적용
  GraalVM Native Image로 시작 시간 문제 해결 고려
```

---

## 📌 핵심 정리

```
두 가지 @Lazy 역할

Bean 선언 @Lazy
  BeanDefinition.isLazyInit() = true
  preInstantiateSingletons() 루프에서 제외
  최초 getBean() 시 createBean() 실행

주입 지점 @Lazy
  buildLazyResolutionProxy()로 CGLIB/JDK 프록시 생성
  프록시 주입 → 실제 Bean은 첫 메서드 호출 시 탐색
  생성자 주입 순환 참조 해결에 활용

전체 Lazy (spring.main.lazy-initialization)
  LazyInitializationBeanFactoryPostProcessor
  모든 BeanDefinition.lazyInit = true 설정
  개발 환경 시작 속도 개선용

BeanDefinition은 항상 존재
  인스턴스만 지연 → containsBean() = true
  getBean() 호출 전까지 인스턴스 없음
```

---

## 🤔 생각해볼 문제

**Q1.** `@Service @Lazy` 클래스가 있고, 다른 Singleton Bean이 `@Autowired`(Lazy 없이)로 주입받는다면 실제로 Lazy가 동작하는가?

**Q2.** 주입 지점 `@Lazy`로 주입받은 프록시 객체에서 `equals()`나 `hashCode()`를 호출하면 어떻게 되는가?

**Q3.** `spring.main.lazy-initialization=true` 환경에서 `@EventListener`가 붙은 Bean이 있을 때 어떤 문제가 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** 실제로 Lazy가 동작하지 않는다. Bean 선언의 `@Lazy`는 `preInstantiateSingletons()` 루프에서만 제외시킨다. 그러나 `@Autowired`(Lazy 없이)로 주입받는 다른 Singleton Bean이 생성될 때 스프링이 `HeavyService`를 의존성으로 탐색하고 `getBean()`을 호출하게 되어 결국 그 시점에 `HeavyService`가 생성된다. 주입 지점에도 `@Lazy`를 붙여야만 프록시가 대신 주입되어 실제 Bean 생성이 지연된다.
>
> **Q2.** 프록시가 `equals()`, `hashCode()` 호출을 intercept하고 `TargetSource.getTarget()`을 통해 실제 Bean을 가져온 뒤 그 Bean의 메서드에 위임한다. 즉, `equals()`나 `hashCode()` 호출 자체가 실제 Bean 생성을 트리거할 수 있다. `proxy.equals(proxy)` 같은 경우도 내부적으로 실제 Bean을 가져온 뒤 비교한다. 따라서 "아직 생성 안 됐다"는 가정 하에 `equals()`를 쓰는 코드는 예상치 못한 시점에 Bean 생성을 유발할 수 있다.
>
> **Q3.** `@EventListener`가 붙은 Bean이 Lazy로 등록되면, 해당 Bean의 인스턴스가 생성되기 전까지 `EventListenerMethodProcessor`가 이벤트 리스너를 등록하지 못한다. 따라서 그 Bean이 생성되기 전에 발행된 이벤트는 수신되지 않는다. 예를 들어 `ApplicationReadyEvent`를 처리하는 `@EventListener`가 있는 Bean이 Lazy라면, `ApplicationReadyEvent`가 발행되는 시점(컨텍스트 시작 완료)에 그 Bean이 아직 생성되지 않아 이벤트를 놓칠 수 있다. 이벤트 리스너 Bean에는 `@Lazy`를 사용하지 않는 것이 안전하다.

---

<div align="center">

**[⬅️ 이전: Constructor Binding의 장점](./06-constructor-binding.md)** | **[홈으로 🏠](../README.md)**

</div>

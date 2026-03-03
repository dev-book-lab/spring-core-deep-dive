# BeanPostProcessor 체인 — @Autowired와 @Transactional이 동작하는 진짜 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- BeanPostProcessor는 무엇이고, 언제 호출되는가?
- `@Autowired`가 주입을 수행하는 정확한 시점은 어디인가?
- `@Transactional`이 프록시로 교체되는 시점은 어디인가?
- `postProcessBeforeInitialization`과 `postProcessAfterInitialization`의 차이는?
- 여러 BeanPostProcessor가 등록됐을 때 실행 순서는 어떻게 결정되는가?
- BeanPostProcessor 자체는 어떻게 등록되고, 왜 일반 Bean보다 먼저 생성되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 스프링이 Bean을 생성한 후, 어떻게 추가 처리를 끼워 넣는가

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;  // 누가, 언제 주입하는가?
}

@Service
@Transactional
public class OrderService {
    public void placeOrder() { ... }  // 누가, 언제 프록시로 바꾸는가?
}
```

```
스프링 컨테이너가 Bean을 만드는 과정:

  1. 인스턴스 생성 (new UserService())
  2. 프로퍼티 주입 (??)
  3. 초기화 콜백 (??)
  4. 완성된 Bean 반환

2번과 3번 사이사이에 무언가가 끼어들어야 한다.
  - @Autowired 필드 채우기
  - @Transactional → 프록시로 교체
  - @Validated → 유효성 검사 프록시
  - @Async → 비동기 프록시

이 "끼어들기"를 담당하는 확장 포인트가 BeanPostProcessor다.
```

스프링 자체도 `@Autowired`, `@Transactional`, `@Async` 등을 모두  
BeanPostProcessor로 구현한다. 특별한 하드코딩이 아니다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: BeanPostProcessor를 일반 Bean처럼 등록하면 작동 안 할 수 있다

```java
// ❌ 문제 상황
@Configuration
public class AppConfig {

    @Bean
    public MyBeanPostProcessor myBpp() {
        return new MyBeanPostProcessor();
    }

    @Bean
    @Scope("prototype")
    public HeavyService heavyService() {
        return new HeavyService();
    }
}

// MyBeanPostProcessor가 HeavyService보다 늦게 생성되면?
// → HeavyService는 BeanPostProcessor 처리 없이 생성됨
```

```
BeanPostProcessor 등록 타이밍 문제:

일반 Bean 생성 순서:
  @Configuration 처리
  → @Bean 메서드 순서대로 등록
  → refresh() 시 순서대로 생성

BeanPostProcessor 필요 시점:
  모든 Bean 생성 이전에 등록 완료돼야 함

→ BeanPostProcessor가 늦게 등록되면
  먼저 생성된 Bean들은 해당 BPP의 처리를 받지 못함
  스프링이 경고 로그 출력:
  "Bean 'xxx' is not eligible for getting processed by all BeanPostProcessors"
```

---

## ✨ 올바른 이해와 사용

### After: BeanPostProcessor는 일반 Bean보다 먼저, 별도 단계에서 등록된다

```java
// AbstractApplicationContext.refresh() 내부 순서

public void refresh() {
    // ...
    // 1단계: BeanFactory 준비
    prepareBeanFactory(beanFactory);

    // 2단계: BeanFactoryPostProcessor 실행 (BeanDefinition 수정)
    invokeBeanFactoryPostProcessors(beanFactory);

    // 3단계: BeanPostProcessor 등록 ← 여기서 먼저 생성/등록
    registerBeanPostProcessors(beanFactory);

    // 4단계: 메시지 소스, 이벤트 멀티캐스터 초기화
    // ...

    // 5단계: 일반 Bean Eager 생성 ← BPP 전부 등록된 후
    finishBeanFactoryInitialization(beanFactory);
}
```

```
BeanPostProcessor 등록이 3단계에서 먼저 완료됨
→ 5단계 일반 Bean 생성 시 이미 모든 BPP가 체인에 있음
→ 모든 Bean이 빠짐없이 BPP 처리를 받음
```

---

## 🔬 내부 동작 원리

### 1. BeanPostProcessor 인터페이스

```java
// spring-beans/.../BeanPostProcessor.java
public interface BeanPostProcessor {

    // Bean 초기화 콜백(@PostConstruct, afterPropertiesSet) 실행 전 호출
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;  // bean을 그대로 또는 교체해서 반환
    }

    // Bean 초기화 콜백 실행 후 호출
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;  // 프록시로 교체하는 것이 주로 여기서 발생
    }
}
```

```
두 메서드의 역할:

postProcessBeforeInitialization
  → @PostConstruct, afterPropertiesSet() 실행 전
  → @Autowired 필드 주입 (AutowiredAnnotationBeanPostProcessor)
  → @Value, @Resource 처리
  → 실제 Bean 객체에 대한 전처리

postProcessAfterInitialization
  → 초기화 완료 후
  → 프록시 교체 (AbstractAutoProxyCreator)
  → @Transactional, @Async, @Validated 등 AOP 적용
  → 반환값이 원본 Bean을 대체함
```

### 2. Bean 생성 전체 흐름에서 BPP 호출 위치

```java
// AbstractAutowireCapableBeanFactory.java (단순화)
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {

    // ── 1. 인스턴스 생성 ────────────────────────────────────────────
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();
    // new UserService() — 아직 필드는 모두 null

    // ── 2. MergedBeanDefinitionPostProcessor 처리 ───────────────────
    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
    // @Autowired, @Value 대상 필드/메서드를 미리 캐싱 (메타데이터 수집)

    // ── 3. Early Singleton 등록 (순환 참조 대응) ────────────────────
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));

    // ── 4. 프로퍼티 주입 ────────────────────────────────────────────
    populateBean(beanName, mbd, instanceWrapper);
    // InstantiationAwareBeanPostProcessor.postProcessProperties() 호출
    // → AutowiredAnnotationBeanPostProcessor가 @Autowired 필드 주입

    // ── 5. 초기화 ───────────────────────────────────────────────────
    exposedObject = initializeBean(beanName, exposedObject, mbd);

    return exposedObject;
}

protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {

    // Aware 인터페이스 콜백 (BeanNameAware, BeanFactoryAware 등)
    invokeAwareMethods(beanName, bean);

    // ── BPP Before ──────────────────────────────────────────────────
    Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    // CommonAnnotationBeanPostProcessor → @PostConstruct 실행 (여기서!)

    // 초기화 콜백
    invokeInitMethods(beanName, wrappedBean, mbd);
    // InitializingBean.afterPropertiesSet(), init-method

    // ── BPP After ───────────────────────────────────────────────────
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    // AbstractAutoProxyCreator → @Transactional 프록시로 교체

    return wrappedBean;
}
```

```
Bean 생성 단계와 BPP 호출 전체 순서:

1. createBeanInstance()        new UserService()
2. applyMergedBeanDefinition   @Autowired 메타데이터 캐싱
3. addSingletonFactory()       순환 참조용 early reference 등록
4. populateBean()              @Autowired 필드 실제 주입
   └── postProcessProperties() ← InstantiationAwareBeanPostProcessor
5. invokeAwareMethods()        BeanNameAware, BeanFactoryAware 콜백
6. BPP Before                  @PostConstruct 실행
7. invokeInitMethods()         afterPropertiesSet(), init-method
8. BPP After                   프록시 교체 (@Transactional 등)
```

### 3. 핵심 BeanPostProcessor 구현체들

```
AutowiredAnnotationBeanPostProcessor
  역할: @Autowired, @Value, @Inject 처리
  구현: InstantiationAwareBeanPostProcessor
  단계: populateBean() → postProcessProperties()

  내부 동작:
    1. @Autowired 대상 메타데이터 수집 (findAutowiringMetadata)
    2. 타입으로 Bean 후보 탐색 (DefaultListableBeanFactory)
    3. 리플렉션으로 필드/메서드에 주입
    field.setAccessible(true);
    field.set(bean, resolvedValue);

CommonAnnotationBeanPostProcessor
  역할: @PostConstruct, @PreDestroy, @Resource 처리
  단계: BPP Before → @PostConstruct 실행
        소멸 시 → @PreDestroy 실행

AbstractAutoProxyCreator (AOP 프록시 생성기)
  역할: @Transactional, @Async, @Validated 프록시 생성
  단계: BPP After → 원본 Bean을 프록시로 교체
  구현체: AnnotationAwareAspectJAutoProxyCreator

  내부 동작:
    1. Bean이 프록시 대상인지 확인 (Advisor 매칭)
    2. 대상이면 JDK Proxy 또는 CGLIB 프록시 생성
    3. 원본 Bean 대신 프록시 반환
    → 컨테이너에는 프록시가 등록됨

ApplicationContextAwareProcessor
  역할: ApplicationContextAware 인터페이스 처리
  단계: BPP Before
  내부: bean.setApplicationContext(this.applicationContext)
```

### 4. BPP 실행 순서 — Ordered와 PriorityOrdered

```java
// PostProcessorRegistrationDelegate.java (단순화)
public static void registerBeanPostProcessors(
        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

    // BPP를 세 그룹으로 분류
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();  // PriorityOrdered
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();           // Ordered
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();        // 나머지

    for (String ppName : postProcessorNames) {
        if (bpp instanceof PriorityOrdered) {
            priorityOrderedPostProcessors.add(bpp);
        } else if (bpp instanceof Ordered) {
            orderedPostProcessors.add(bpp);
        } else {
            nonOrderedPostProcessors.add(bpp);
        }
    }

    // 등록 순서: PriorityOrdered → Ordered → 나머지
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
}
```

```
BPP 실행 우선순위:

PriorityOrdered (가장 먼저)
  AutowiredAnnotationBeanPostProcessor  (order = Ordered.LOWEST_PRECEDENCE - 2)
  CommonAnnotationBeanPostProcessor     (order = Ordered.LOWEST_PRECEDENCE - 3)

Ordered (두 번째)
  사용자 정의 BPP (@Order 또는 implements Ordered)

일반 (마지막)
  AbstractAutoProxyCreator (AOP)
  → 프록시는 모든 주입이 완료된 후에 적용돼야 하므로

주의:
  같은 우선순위 내에서는 등록 순서 (선언 순서)
  숫자가 낮을수록 먼저 실행 (Integer.MIN_VALUE = 최우선)
```

### 5. 커스텀 BeanPostProcessor 작성 패턴

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor, Ordered {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(LogExecution.class)) {
            System.out.println("[BPP Before] " + beanName + " 초기화 전");
        }
        return bean;  // 원본 반환 (교체 안 함)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(LogExecution.class)) {
            // 프록시로 교체
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("[LOG] " + method.getName() + " 호출");
                    return method.invoke(bean, args);
                });
        }
        return bean;
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;  // 마지막 순서
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: BPP 호출 순서 직접 확인

```java
@Component
public class OrderTrackerBPP implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (beanName.equals("targetBean")) {
            System.out.println("[BPP Before] " + beanName);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (beanName.equals("targetBean")) {
            System.out.println("[BPP After] " + beanName
                + " | class=" + bean.getClass().getSimpleName());
        }
        return bean;
    }
}

@Service
@Transactional
public class TargetBean {

    @PostConstruct
    public void init() {
        System.out.println("[@PostConstruct] TargetBean 초기화");
    }
}
```

```
출력 순서:
[BPP Before] targetBean
[@PostConstruct] TargetBean 초기화   ← BPP Before 안에서 실행
[BPP After] targetBean | class=TargetBean$$EnhancerBySpringCGLIB  ← 프록시로 교체됨
```

### 실험 2: BPP After에서 프록시 교체 확인

```java
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

OrderService service = ctx.getBean(OrderService.class);

// @Transactional이 붙어있으면
System.out.println(service.getClass());
// class com.example.OrderService$$EnhancerBySpringCGLIB$$1a2b3c4d
// → 원본이 아닌 CGLIB 프록시

System.out.println(service instanceof OrderService);
// true → 프록시는 OrderService의 서브클래스
```

---

## 🤔 트레이드오프

```
BeanPostProcessor 남용 주의:
  모든 Bean 생성 시 호출됨 (수백 개 Bean × BPP 개수)
  → 무거운 로직 넣으면 시작 시간 크게 증가
  → 빠른 판별 후 조기 반환 패턴 권장:
    if (!bean.getClass().isAnnotationPresent(MyAnnotation.class)) return bean;

BPP 안에서 다른 Bean 주입 주의:
  BPP는 일반 Bean보다 먼저 생성됨
  → BPP 안에서 @Autowired로 일반 Bean 주입 시
  → 해당 Bean이 BPP 처리를 못 받을 수 있음
  → ApplicationContextAware로 getBean() 지연 조회 권장

postProcessAfterInitialization에서 반환값 null 주의:
  null을 반환하면 해당 Bean이 null로 등록됨
  → 이후 의존하는 Bean 전부 NPE
  → 처리 대상 아니면 반드시 원본 bean 반환
```

---

## 📌 핵심 정리

```
BeanPostProcessor 역할
  Bean 생성 전후에 끼어드는 확장 포인트
  @Autowired, @Transactional 등 스프링 기능의 실제 구현 위치

두 메서드
  Before: 초기화 콜백 전 — @PostConstruct 실행, @Autowired 주입
  After:  초기화 콜백 후 — 프록시 교체 (@Transactional, @Async)

등록 시점
  refresh() 3단계에서 일반 Bean보다 먼저 등록
  → 모든 Bean 생성 시 BPP 체인이 이미 준비됨

실행 순서
  PriorityOrdered → Ordered → 일반
  AutowiredAnnotationBeanPostProcessor (주입) 먼저
  AbstractAutoProxyCreator (프록시) 나중

핵심 구현체
  AutowiredAnnotationBeanPostProcessor → @Autowired 필드 주입
  CommonAnnotationBeanPostProcessor    → @PostConstruct, @Resource
  AbstractAutoProxyCreator             → AOP 프록시 생성

반환값
  원본 bean 또는 교체할 새 객체 (프록시)
  null 반환 절대 금지
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `MyService`의 `@Autowired` 필드가 주입되는 정확한 단계는 `doCreateBean()` 기준으로 어디인가?

```java
@Service
public class MyService {
    @Autowired
    private MyRepository myRepository;

    @PostConstruct
    public void init() {
        System.out.println(myRepository.getClass()); // null이 아님을 확인
    }
}
```

**Q2.** 다음 커스텀 BPP에서 문제점을 찾아라.

```java
@Component
public class BadBPP implements BeanPostProcessor {
    @Autowired
    private UserService userService;  // 일반 Bean 주입

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof SomeBean) {
            userService.doSomething();
        }
        return null;  // ??
    }
}
```

**Q3.** `@Transactional`이 붙은 Bean을 `ctx.getBean()`으로 꺼냈을 때 실제 클래스가 `$$EnhancerBySpringCGLIB`인 이유를 BPP 흐름 기준으로 설명하라.

> 💡 **해설**
>
> **Q1.** `@Autowired` 필드 주입은 `doCreateBean()` 4단계 `populateBean()` 안에서 `AutowiredAnnotationBeanPostProcessor.postProcessProperties()`가 호출될 때 발생한다. `@PostConstruct`는 그 이후인 6단계 `BPP Before` 안에서 `CommonAnnotationBeanPostProcessor`가 실행할 때 호출된다. 따라서 `init()` 메서드가 실행되는 시점에는 `myRepository`가 이미 주입되어 있어 null이 아니다.
>
> **Q2.** 문제 두 가지. ① `return null` — `postProcessAfterInitialization`에서 null을 반환하면 해당 Bean이 null로 컨테이너에 등록되어, 이 Bean에 의존하는 모든 곳에서 NPE가 발생한다. 처리하지 않는 Bean은 반드시 `return bean`으로 원본을 반환해야 한다. ② `@Autowired private UserService` — BPP는 일반 Bean보다 먼저 생성되므로, `UserService`가 아직 BPP 처리를 받기 전에 이 BPP가 생성될 수 있다. `UserService`가 `@Transactional` 등 프록시 처리가 필요하다면 프록시 없는 원본이 주입될 수 있다.
>
> **Q3.** `@Transactional` Bean은 `AbstractAutoProxyCreator`(AOP 프록시 생성기)가 처리한다. 이 BPP는 `postProcessAfterInitialization`에서 동작하고, 원본 `OrderService` 인스턴스 대신 CGLIB 서브클래스(`OrderService$$EnhancerBySpringCGLIB`)를 생성해 반환한다. 컨테이너는 이 반환값을 원본 대신 등록하므로, `ctx.getBean()`은 프록시를 반환한다. 이것이 같은 클래스 내부에서 `this.method()`로 호출하면 `@Transactional`이 동작하지 않는 이유이기도 하다 — `this`는 프록시가 아닌 원본을 가리키기 때문이다.

---

<div align="center">

**[⬅️ 이전: BeanDefinition과 Bean 메타데이터](./02-bean-definition-metadata.md)** | **[다음: ApplicationContext 계층 구조 ➡️](./04-applicationcontext-hierarchy.md)**

</div>

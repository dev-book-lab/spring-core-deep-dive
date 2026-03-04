# Bean 생성 전체 과정 — AbstractAutowireCapableBeanFactory 완전 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스프링이 Bean을 만들 때 거치는 단계는 정확히 몇 단계이고, 각 단계의 책임은?
- `AbstractAutowireCapableBeanFactory`의 `doCreateBean()`에서 어떤 일이 순서대로 벌어지는가?
- `createBeanInstance()`, `populateBean()`, `initializeBean()`은 각각 무엇을 하는가?
- BeanPostProcessor가 끼어드는 정확한 위치는 어디인가?
- 생성 중 예외 발생 시 이미 등록된 소멸 콜백은 어떻게 처리되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: "스프링이 알아서 Bean을 만든다"는 블랙박스

```java
@Service
public class OrderService {
    @Autowired private PaymentService paymentService;

    @PostConstruct
    public void init() { System.out.println("초기화"); }
}
```

```
"스프링이 알아서 만든다"는 설명으로는:
  - @Autowired 주입이 언제 일어나는지 모름
  - @PostConstruct가 왜 필드 주입 후에 실행되는지 모름
  - BeanPostProcessor가 언제 끼어드는지 모름
  - 초기화 콜백 순서를 예측할 수 없음

블랙박스를 열어야 하는 이유:
  - 초기화 순서 버그 디버깅
  - BeanPostProcessor 작성 시 올바른 단계 선택
  - 순환 참조 해결 원리 이해
  - Aware 인터페이스 호출 시점 예측
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 생성자에서 @Autowired 필드를 사용한다

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;

    public OrderService() {
        // ❌ 생성자에서 @Autowired 필드 사용
        paymentService.validate();  // NullPointerException!
    }
}
```

```
오해: "생성자가 실행되면 필드 주입도 완료됐을 것이다"

실제 순서:
  1. 생성자 호출 (paymentService = null)
  2. populateBean() → @Autowired 필드 주입
  3. 초기화 콜백

생성자 실행 시점에는 @Autowired 필드가 아직 null
→ 초기화 로직은 @PostConstruct에서 수행
```

---

## ✨ 올바른 이해와 사용

### After: 8단계 생성 흐름을 머릿속에 그린다

```
Bean 생성 8단계:

1. createBeanInstance()       인스턴스 생성 (생성자 호출)
2. applyMergedBeanDefinition  @Autowired 메타데이터 수집
3. addSingletonFactory()      순환 참조 대비 early ref 등록
4. populateBean()             프로퍼티/의존성 주입
   └── postProcessProperties()  @Autowired, @Value 실제 주입
5. invokeAwareMethods()       BeanNameAware, BeanFactoryAware
6. BPP Before                 @PostConstruct 실행
7. invokeInitMethods()        afterPropertiesSet(), init-method
8. BPP After                  AOP 프록시 생성 (@Transactional 등)
```

---

## 🔬 내부 동작 원리

### 1. createBean() → doCreateBean() 진입

```java
// AbstractAutowireCapableBeanFactory.createBean()
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) {

    // BPP에게 "인스턴스 생성 전 가로채기" 기회 제공
    // (InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation)
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;  // BPP가 직접 객체 반환하면 doCreateBean() 건너뜀 (드문 케이스)
    }

    return doCreateBean(beanName, mbdToUse, args);
}
```

### 2. doCreateBean() — 핵심 생성 로직 전체

```java
// AbstractAutowireCapableBeanFactory.doCreateBean()
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {

    // ── 1단계: 인스턴스 생성 ───────────────────────────────────────
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    // 결과: new OrderService() — 모든 필드 null

    // ── 2단계: @Autowired 메타데이터 수집 (캐싱) ───────────────────
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            // AutowiredAnnotationBPP: 어떤 필드/메서드에 @Autowired 붙었는지 스캔
            // 결과를 InjectionMetadata로 캐싱 → 실제 주입은 4단계에서
            mbd.postProcessed = true;
        }
    }

    // ── 3단계: 순환 참조 대비 Early Reference 등록 ────────────────
    boolean earlySingletonExposure = (mbd.isSingleton()
            && this.allowCircularReferences
            && isSingletonCurrentlyInCreation(beanName));

    if (earlySingletonExposure) {
        addSingletonFactory(beanName,
            () -> getEarlyBeanReference(beanName, mbd, bean));
        // ObjectFactory 람다 등록
        // AOP 프록시 있으면: 람다 실행 시 프록시 early ref 반환
    }

    Object exposedObject = bean;
    try {
        // ── 4단계: 프로퍼티 주입 ──────────────────────────────────
        populateBean(beanName, mbd, instanceWrapper);
        // @Autowired, @Value, @Resource 실제 주입 발생

        // ── 5~8단계: 초기화 ───────────────────────────────────────
        exposedObject = initializeBean(beanName, exposedObject, mbd);

    } catch (Throwable ex) {
        // 생성 중 예외 → 등록된 소멸 콜백 정리
        if (bean != exposedObject) {
            destroyBean(beanName, bean);
        }
        throw new BeanCreationException(beanName, "...", ex);
    }

    // ── 순환 참조 후처리 ──────────────────────────────────────────
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
                // AOP 프록시가 early ref로 노출됐으면 최종 객체도 프록시로 통일
            }
        }
    }

    return exposedObject;
}
```

### 3. createBeanInstance() — 생성자 선택과 인스턴스 생성

```java
// AbstractAutowireCapableBeanFactory.createBeanInstance()
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {

    // 1. 팩토리 메서드 방식 (@Bean 메서드)
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 2. 생성자 주입 — @Autowired 생성자 또는 단일 생성자
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR) {
        return autowireConstructor(beanName, mbd, ctors, args);
        // 생성자 파라미터 Bean 탐색 + 생성자 호출
    }

    // 3. 기본 생성자
    return instantiateBean(beanName, mbd);
    // SimpleInstantiationStrategy.instantiate() → Constructor.newInstance()
}
```

```
생성자 선택 우선순위:
  @Bean 팩토리 메서드 → 우선
  @Autowired 붙은 생성자 → 다음
  단일 생성자 (Spring 4.3+) → 다음
  기본 생성자 → 마지막
```

### 4. populateBean() — 의존성 주입 단계

```java
// AbstractAutowireCapableBeanFactory.populateBean()
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {

    // InstantiationAwareBPP.postProcessAfterInstantiation()
    // → 기본 구현: true 반환 (계속 진행)
    // → false 반환하면 이후 주입 전부 건너뜀 (드문 케이스)
    if (!continueWithPropertyPopulation(beanName, mbd, bw)) {
        return;
    }

    // XML <property> 등 전통 방식 처리
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // InstantiationAwareBPP.postProcessProperties() 호출
    // → AutowiredAnnotationBPP: @Autowired, @Value 주입
    // → CommonAnnotationBPP: @Resource 주입
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        pvs = pvsToUse;
    }

    // XML 명시 프로퍼티 적용
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

### 5. initializeBean() — 초기화 단계 (5~8단계)

```java
// AbstractAutowireCapableBeanFactory.initializeBean()
protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {

    // ── 5단계: Aware 인터페이스 콜백 ──────────────────────────────
    invokeAwareMethods(beanName, bean);
    // BeanNameAware.setBeanName()
    // BeanClassLoaderAware.setBeanClassLoader()
    // BeanFactoryAware.setBeanFactory()

    // ── 6단계: BPP Before ─────────────────────────────────────────
    Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    // ApplicationContextAwareProcessor → ApplicationContextAware 등 처리
    // CommonAnnotationBPP → @PostConstruct 실행 ← 여기서!

    // ── 7단계: 초기화 콜백 ────────────────────────────────────────
    invokeInitMethods(beanName, wrappedBean, mbd);
    // InitializingBean.afterPropertiesSet()
    // init-method (XML 또는 @Bean(initMethod="..."))

    // ── 8단계: BPP After ──────────────────────────────────────────
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    // AbstractAutoProxyCreator → @Transactional, @Async 프록시 생성 ← 여기서!

    return wrappedBean;
}
```

```
invokeAwareMethods() — BPP Before 이전에 실행되는 Aware:
  BeanNameAware       → setBeanName(beanName)
  BeanClassLoaderAware→ setBeanClassLoader(classLoader)
  BeanFactoryAware    → setBeanFactory(beanFactory)

  ApplicationContextAware 등 나머지 Aware는
  ApplicationContextAwareProcessor(BPP Before)에서 처리됨
```

### 6. 예외 발생 시 소멸 콜백 처리

```java
// doCreateBean() 예외 처리 블록
try {
    populateBean(beanName, mbd, instanceWrapper);
    exposedObject = initializeBean(beanName, exposedObject, mbd);
} catch (Throwable ex) {
    if (bean != exposedObject) {
        // BPP After에서 프록시로 교체된 경우
        // 원본 Bean의 소멸 콜백(@PreDestroy 등) 등록 해제
        destroyBean(beanName, bean);
    }
    throw new BeanCreationException(beanName, ex.getMessage(), ex);
}
```

```
예외 발생 시:
  이미 등록된 DisposableBean 어댑터 제거
  소멸 콜백이 절반만 실행되는 상황 방지
  BeanCreationException으로 래핑하여 상위 전파
  → 컨텍스트 refresh() 실패 → 애플리케이션 시작 실패
```

---

## 💻 실험으로 확인하기

### 실험 1: 전체 생성 순서 출력

```java
@Service
public class LifecycleBean implements BeanNameAware, BeanFactoryAware,
                                      ApplicationContextAware, InitializingBean {

    @Autowired private SomeService someService;

    public LifecycleBean() {
        System.out.println("1. 생성자 호출 (someService=" + someService + ")");
    }

    @Override public void setBeanName(String name) {
        System.out.println("5a. BeanNameAware.setBeanName: " + name);
    }

    @Override public void setBeanFactory(BeanFactory bf) {
        System.out.println("5b. BeanFactoryAware.setBeanFactory");
    }

    @Override public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("6a. ApplicationContextAware (BPP Before 내)");
    }

    @PostConstruct public void postConstruct() {
        System.out.println("6b. @PostConstruct (someService=" + someService + ")");
    }

    @Override public void afterPropertiesSet() {
        System.out.println("7. InitializingBean.afterPropertiesSet");
    }
}
```

```
출력:
1. 생성자 호출 (someService=null)     ← 주입 전
4. populateBean → someService 주입    ← 로그 없지만 여기서 발생
5a. BeanNameAware.setBeanName: lifecycleBean
5b. BeanFactoryAware.setBeanFactory
6a. ApplicationContextAware (BPP Before 내)
6b. @PostConstruct (someService=com.example.SomeService@...)  ← 주입 완료
7. InitializingBean.afterPropertiesSet
```

### 실험 2: BPP After에서 프록시 교체 확인

```java
@Service @Transactional
class TransactionalService {
    public void doWork() {}
}

// ctx.getBean(TransactionalService.class).getClass()
// → TransactionalService$$EnhancerBySpringCGLIB$$...
// BPP After(8단계)에서 CGLIB 프록시로 교체됨
```

---

## 🤔 트레이드오프

```
초기화 콜백 위치 선택:

생성자
  적합: 불변 필드 초기화, 파라미터 검증
  부적합: @Autowired 필드 사용 (아직 null)

@PostConstruct (6단계 BPP Before)
  적합: @Autowired 필드 사용한 초기화
  부적합: 트랜잭션 컨텍스트 필요 (프록시 미완성)

afterPropertiesSet() (7단계)
  @PostConstruct와 거의 동일, 스프링 인터페이스 의존

init-method (7단계, afterPropertiesSet 후)
  XML/외부 설정으로 초기화 메서드 지정 시

BPP After 이후 (8단계)
  트랜잭션 등 AOP 기능이 필요한 초기화는 이후 불가
  → SmartInitializingSingleton.afterSingletonsInstantiated() 활용
     (모든 Singleton 생성 완료 후 콜백)
```

---

## 📌 핵심 정리

```
doCreateBean() 8단계

1. createBeanInstance()       생성자 호출, 팩토리 메서드
2. applyMergedBeanDefinition  @Autowired 메타데이터 캐싱
3. addSingletonFactory()      순환 참조용 ObjectFactory 등록
4. populateBean()             @Autowired, @Value, @Resource 주입
5. invokeAwareMethods()       BeanNameAware, BeanFactoryAware
6. BPP Before                 ApplicationContextAware, @PostConstruct
7. invokeInitMethods()        afterPropertiesSet(), init-method
8. BPP After                  AOP 프록시 생성

핵심 포인트
  생성자 실행 시점: @Autowired 필드 null
  @PostConstruct: @Autowired 주입 완료 후 (4단계 이후)
  AOP 프록시: 가장 마지막 (8단계)
  예외 발생: BeanCreationException 래핑 후 컨텍스트 시작 실패
```

---

## 🤔 생각해볼 문제

**Q1.** `@PostConstruct` 메서드에서 `@Transactional`이 동작하지 않는 이유를 Bean 생성 단계로 설명하라.

**Q2.** `SmartInitializingSingleton.afterSingletonsInstantiated()`는 언제 호출되는가? `@PostConstruct`와 어떻게 다른가?

**Q3.** `resolveBeforeInstantiation()`에서 BPP가 null이 아닌 객체를 반환하면 어떤 일이 벌어지는가? 어떤 상황에서 이 경로가 사용되는가?

> 💡 **해설**
>
> **Q1.** `@Transactional` AOP 프록시는 8단계 `BPP After`(`applyBeanPostProcessorsAfterInitialization`)에서 `AbstractAutoProxyCreator`가 생성한다. `@PostConstruct`는 6단계 `BPP Before`에서 실행된다. 즉 `@PostConstruct` 실행 시점에는 프록시가 아직 존재하지 않는다. `this.doSomething()`을 `@PostConstruct`에서 호출하면 프록시를 거치지 않고 원본 메서드를 직접 호출하게 되어 트랜잭션이 시작되지 않는다.
>
> **Q2.** `SmartInitializingSingleton.afterSingletonsInstantiated()`는 `preInstantiateSingletons()` 루프가 완전히 끝난 뒤, 즉 **모든 Singleton Bean이 생성 완료된 후** 한 번 호출된다. `@PostConstruct`는 **각 Bean 생성 시**(`initializeBean()` 내) 호출된다. 따라서 `afterSingletonsInstantiated()`에서는 컨테이너의 다른 모든 Bean이 완전히 초기화된 상태를 보장받을 수 있다. 특정 Bean이 모든 다른 Bean의 초기화 완료를 전제로 하는 작업(예: 캐시 워밍업, 플러그인 등록 확인)은 여기서 수행해야 한다.
>
> **Q3.** `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()`이 null이 아닌 객체를 반환하면, 스프링은 `doCreateBean()`을 건너뛰고 해당 객체를 Bean으로 사용한다. 이어서 `postProcessAfterInitialization()`만 추가로 호출된다. 이 경로는 주로 AOP 프레임워크에서 타겟 클래스를 완전히 대체하는 프록시를 생성할 때나, 커스텀 인스턴스화 전략이 필요할 때 사용된다. 일반적인 애플리케이션 코드에서는 거의 사용되지 않는 특수 경로다.

---

<div align="center">

**[다음: @PostConstruct vs InitializingBean 실행 순서 ➡️](./02-postconstruct-vs-initializingbean.md)**

</div>

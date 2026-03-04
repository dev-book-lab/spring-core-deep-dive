# @PostConstruct vs InitializingBean — 초기화 콜백 세 가지의 실행 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@PostConstruct`, `InitializingBean.afterPropertiesSet()`, `init-method` 세 가지의 실행 순서는?
- 같은 Bean에 세 가지 모두 있으면 어떤 순서로 실행되는가?
- 각 방법을 처리하는 내부 클래스와 메서드는 무엇인가?
- `@PostConstruct`가 JSR-250 표준임에도 스프링에서 권장되는 이유는?
- 초기화 콜백에서 예외가 발생하면 Bean 생성은 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean 생성 후 추가 초기화가 필요한 경우

```java
@Service
public class CacheService {
    @Autowired private DataSource dataSource;

    // dataSource가 주입된 이후에 캐시를 초기화해야 함
    // 생성자에서는 불가 (주입 전)
    // 어디서 초기화해야 하는가?
}
```

```
초기화 시점 요구:
  @Autowired 주입 완료 이후
  → 세 가지 방법 모두 populateBean() 이후에 실행

세 가지 방법이 생긴 역사적 배경:
  init-method    스프링 초창기부터 (XML 시대)
  InitializingBean  스프링 고유 인터페이스 (코드 결합)
  @PostConstruct JSR-250 표준 (Java EE 5, 2006)
                 → 스프링 의존 없는 표준 방식
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 세 가지가 같은 시점에 실행된다고 생각한다

```java
@Service
public class MultiInitBean implements InitializingBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("언제 실행?");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("언제 실행?");
    }

    @Bean(initMethod = "customInit")
    // or XML: init-method="customInit"
    public void customInit() {
        System.out.println("언제 실행?");
    }
}
```

```
❌ "셋 다 동시에 실행된다" 또는 "순서는 랜덤이다"

✅ 실제 순서 (항상 고정):
  1. @PostConstruct
  2. InitializingBean.afterPropertiesSet()
  3. init-method (customInit)
```

### Before: 셋 다 써도 문제없다

```java
// ❌ 같은 초기화 작업을 세 곳에 중복 작성
@PostConstruct
public void init1() { loadCache(); }

@Override
public void afterPropertiesSet() { loadCache(); }  // 중복 실행!

public void customInit() { loadCache(); }          // 또 중복!
```

---

## ✨ 올바른 이해와 사용

### After: 세 가지의 처리 경로와 순서를 명확히 구분한다

```
처리 경로:

@PostConstruct
  CommonAnnotationBeanPostProcessor (BPP Before)
  → applyBeanPostProcessorsBeforeInitialization()에서 실행

afterPropertiesSet()
  AbstractAutowireCapableBeanFactory.invokeInitMethods()
  → InitializingBean 구현 여부 체크 후 호출

init-method
  invokeInitMethods() 내부
  → afterPropertiesSet() 호출 직후
  → BeanDefinition에 저장된 메서드명으로 리플렉션 호출

실행 단계:
  6단계 BPP Before → @PostConstruct
  7단계 invokeInitMethods → afterPropertiesSet → init-method
```

---

## 🔬 내부 동작 원리

### 1. @PostConstruct 처리 — CommonAnnotationBeanPostProcessor

```java
// InitDestroyAnnotationBeanPostProcessor (CommonAnnotationBPP의 부모)
public Object postProcessBeforeInitialization(Object bean, String beanName) {

    // 클래스에서 @PostConstruct / @PreDestroy 메서드 메타데이터 수집 (캐시)
    LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());

    // @PostConstruct 메서드 실행
    metadata.invokeInitMethods(bean, beanName);

    return bean;
}

// findLifecycleMetadata() — 리플렉션으로 메서드 탐색
private LifecycleMetadata buildLifecycleMetadata(Class<?> clazz) {
    List<LifecycleElement> initMethods = new ArrayList<>();
    List<LifecycleElement> destroyMethods = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        // 각 메서드에서 @PostConstruct / @PreDestroy 탐색
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            if (this.initAnnotationType != null
                    && method.isAnnotationPresent(this.initAnnotationType)) {
                initMethods.add(0, new LifecycleElement(method));
                // 부모 클래스 메서드를 앞에 추가 → 부모 @PostConstruct 먼저
            }
        });
        targetClass = targetClass.getSuperclass();
    } while (targetClass != null && targetClass != Object.class);

    return new LifecycleMetadata(clazz, initMethods, destroyMethods);
}
```

```
@PostConstruct 특징:
  JSR-250 어노테이션 → 스프링 미의존
  반환 타입: void 강제
  파라미터: 없어야 함
  접근 제한자: public / protected / package-private 모두 가능
  부모 클래스 @PostConstruct → 자식보다 먼저 실행
  같은 클래스에 여러 개 → 순서 불보장 (하나만 쓸 것)
```

### 2. afterPropertiesSet() 처리 — invokeInitMethods()

```java
// AbstractAutowireCapableBeanFactory.invokeInitMethods()
protected void invokeInitMethods(String beanName, Object bean, RootBeanDefinition mbd) {

    // InitializingBean 구현 여부 확인
    boolean isInitializingBean = (bean instanceof InitializingBean);

    if (isInitializingBean
            && (mbd == null || !mbd.hasAnyExternallyManagedInitMethod("afterPropertiesSet"))) {
        // afterPropertiesSet()이 init-method로도 등록된 경우 중복 실행 방지

        ((InitializingBean) bean).afterPropertiesSet();  // 직접 메서드 호출
    }

    // init-method 처리
    if (mbd != null && bean.getClass() != NullBean.class) {
        String[] initMethodNames = mbd.getInitMethodNames();

        if (initMethodNames != null) {
            for (String initMethodName : initMethodNames) {
                if (StringUtils.hasLength(initMethodName)
                        && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName))) {
                    invokeCustomInitMethod(beanName, bean, mbd, initMethodName);
                    // 리플렉션으로 메서드 호출
                }
            }
        }
    }
}
```

```
afterPropertiesSet() 특징:
  InitializingBean 인터페이스 구현 필요 → 스프링 코드 결합
  직접 메서드 호출 (리플렉션 아님) → 약간 더 빠름
  init-method와 이름이 같으면 중복 실행 방지 로직 존재
```

### 3. init-method 처리

```java
// invokeCustomInitMethod()
protected void invokeCustomInitMethod(String beanName, Object bean,
                                       RootBeanDefinition mbd, String initMethodName) {
    Method initMethod = (mbd.isNonPublicAccessAllowed()
            ? BeanUtils.findMethod(bean.getClass(), initMethodName)
            : ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

    if (initMethod == null) {
        if (mbd.isEnforceInitMethod()) {
            throw new BeanDefinitionValidationException("...");
        }
        return;  // enforceInitMethod=false면 없어도 무시
    }

    // 리플렉션으로 호출
    ReflectionUtils.makeAccessible(initMethod);
    initMethod.invoke(bean);
}
```

```
init-method 등록 방법:
  @Bean(initMethod = "customInit")  → Java Config
  XML: <bean init-method="customInit">
  @PostConstruct와 달리 외부에서 지정 → 스프링 코드 비의존 가능
  (라이브러리 클래스에 초기화 메서드 지정 시 유용)
```

### 4. 전체 실행 순서 — 코드 흐름으로 정리

```
initializeBean() 호출:

  invokeAwareMethods()
    BeanNameAware, BeanClassLoaderAware, BeanFactoryAware

  applyBeanPostProcessorsBeforeInitialization()
    ApplicationContextAwareProcessor
      → EnvironmentAware, ApplicationContextAware 등
    CommonAnnotationBeanPostProcessor
      → @PostConstruct 실행  ← 1순위

  invokeInitMethods()
    InitializingBean.afterPropertiesSet()  ← 2순위
    init-method                            ← 3순위

  applyBeanPostProcessorsAfterInitialization()
    AOP 프록시 생성
```

### 5. 부모-자식 클래스에서의 실행 순서

```java
public class BaseService {
    @PostConstruct
    public void baseInit() {
        System.out.println("부모 @PostConstruct");
    }
}

@Service
public class OrderService extends BaseService implements InitializingBean {

    @PostConstruct
    public void childInit() {
        System.out.println("자식 @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("afterPropertiesSet");
    }
}
```

```
출력:
부모 @PostConstruct    ← 부모 클래스 먼저
자식 @PostConstruct    ← 자식 클래스
afterPropertiesSet     ← InitializingBean

이유:
  buildLifecycleMetadata()에서 부모 → 자식 순으로 메서드 수집
  initMethods.add(0, ...) → 부모 메서드를 리스트 앞에 삽입
```

### 6. 초기화 콜백에서 예외 발생

```java
@PostConstruct
public void init() {
    throw new RuntimeException("초기화 실패");
}
```

```
예외 전파 경로:
  @PostConstruct 예외
  → CommonAnnotationBPP.postProcessBeforeInitialization() 전파
  → initializeBean() 전파
  → doCreateBean() catch 블록
  → BeanCreationException("Error creating bean with name '...'")
  → 컨텍스트 refresh() 실패
  → 애플리케이션 시작 실패

→ 초기화 콜백의 예외는 반드시 처리하거나
  의도적 실패라면 명확한 메시지 포함 권장
```

---

## 💻 실험으로 확인하기

### 실험 1: 세 가지 초기화 순서 직접 확인

```java
@Component
public class OrderedInitBean implements InitializingBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. afterPropertiesSet");
    }

    public void customInit() {
        System.out.println("3. init-method");
    }
}

// Java Config
@Bean(initMethod = "customInit")
public OrderedInitBean orderedInitBean() {
    return new OrderedInitBean();
}
```

```
출력:
1. @PostConstruct
2. afterPropertiesSet
3. init-method
```

### 실험 2: 라이브러리 클래스에 init-method 적용

```java
// 외부 라이브러리 클래스 (수정 불가)
public class ThirdPartyConnectionPool {
    public void initialize() {
        // 연결 풀 초기화
    }
}

// @Bean의 initMethod로 외부 클래스 초기화
@Bean(initMethod = "initialize")
public ThirdPartyConnectionPool connectionPool() {
    return new ThirdPartyConnectionPool();
}
// → @PostConstruct, InitializingBean 없이 초기화 가능
```

---

## 🤔 트레이드오프

```
@PostConstruct (권장):
  장점  JSR-250 표준, 스프링 미의존, 간결
  단점  어노테이션 처리 오버헤드 (미미)

InitializingBean:
  장점  직접 메서드 호출 (리플렉션 없음), IDE 리팩토링 안전
  단점  스프링 인터페이스 구현 → 코드 결합

init-method:
  장점  외부 클래스 초기화, 설정과 코드 분리
  단점  문자열로 메서드명 지정 → 오타 위험, IDE 리팩토링 비대응

실무 권장:
  기본: @PostConstruct
  외부 라이브러리 클래스: @Bean(initMethod = "...")
  InitializingBean: 성능이 매우 중요한 내부 인프라 코드
```

---

## 📌 핵심 정리

```
실행 순서 (항상 고정)
  1. @PostConstruct      (BPP Before — CommonAnnotationBPP)
  2. afterPropertiesSet  (invokeInitMethods — 직접 호출)
  3. init-method         (invokeInitMethods — 리플렉션)

처리 클래스
  @PostConstruct   → CommonAnnotationBeanPostProcessor
  afterPropertiesSet, init-method → AbstractAutowireCapableBeanFactory

부모-자식 순서
  부모 @PostConstruct → 자식 @PostConstruct → afterPropertiesSet → init-method

예외 처리
  모든 초기화 콜백 예외 → BeanCreationException → 컨텍스트 시작 실패

선택 기준
  기본: @PostConstruct
  외부 클래스: init-method
  성능 민감 인프라: InitializingBean
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 Bean에서 세 초기화 메서드의 실행 순서와 출력 결과를 예측하라.

```java
public class Parent {
    @PostConstruct
    void parentInit() { System.out.println("A"); }
}

@Component
public class Child extends Parent implements InitializingBean {
    @PostConstruct
    void childInit() { System.out.println("B"); }

    @Override
    public void afterPropertiesSet() { System.out.println("C"); }
}
// @Bean(initMethod = "extraInit") 적용
    public void extraInit() { System.out.println("D"); }
```

**Q2.** `@PostConstruct` 메서드가 `private`이면 실행되는가?

**Q3.** `InitializingBean.afterPropertiesSet()`과 `@Bean(initMethod = "afterPropertiesSet")`을 동시에 사용하면 두 번 실행되는가?

> 💡 **해설**
>
> **Q1.** 출력 순서: A → B → C → D. `buildLifecycleMetadata()`에서 부모 클래스의 `@PostConstruct`를 리스트 앞에 삽입하므로 부모 A가 먼저, 자식 B가 다음. 이후 `invokeInitMethods()`에서 `afterPropertiesSet()`(C), `init-method`인 `extraInit()`(D) 순으로 실행된다.
>
> **Q2.** 실행된다. `ReflectionUtils.makeAccessible(method)`로 `private` 접근 제한을 해제한 후 호출한다. 단, Java 모듈 시스템이 적용된 환경(Java 9+)에서는 모듈 경계를 넘는 `private` 접근이 제한될 수 있어 `InaccessibleObjectException`이 발생할 수 있다. 접근 제한자를 `protected`나 `public`으로 하는 것이 안전하다.
>
> **Q3.** 두 번 실행되지 않는다. `invokeInitMethods()` 내부에서 Bean이 `InitializingBean`을 구현하고 init-method 이름이 `"afterPropertiesSet"`인 경우를 명시적으로 감지해 중복 호출을 방지한다. `isInitializingBean && "afterPropertiesSet".equals(initMethodName)` 조건이 true면 `invokeCustomInitMethod()`를 건너뛴다.

---

<div align="center">

**[⬅️ 이전: Bean 생성 전체 과정](./01-bean-creation-process.md)** | **[다음: @PreDestroy vs DisposableBean 정리 과정 ➡️](./03-predestroy-vs-disposablebean.md)**

</div>

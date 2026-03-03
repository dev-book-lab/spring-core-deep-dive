# BeanFactory vs ApplicationContext — 컨테이너 계층 구조의 비밀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- BeanFactory와 ApplicationContext는 어떤 관계인가?
- 인터페이스 계층 구조는 어떻게 설계되어 있고, 각 계층의 책임은 무엇인가?
- `getBean()`을 호출하면 내부에서 정확히 무슨 일이 벌어지는가?
- Lazy Initialization과 Eager Initialization은 어느 계층에서 결정되는가?
- 실무에서 BeanFactory를 직접 써야 하는 경우가 있는가?
- `AnnotationConfigApplicationContext` vs `ClassPathXmlApplicationContext`는 무엇이 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: 객체의 생성과 의존 관계를 누가 관리할 것인가

```java
// 스프링 없이 직접 관리
DataSource dataSource = new HikariDataSource(config);
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
UserRepository userRepository = new UserRepository(jdbcTemplate);
UserService userService = new UserService(userRepository);
OrderService orderService = new OrderService(userService, paymentService, ...);
// 수백 개의 객체, 수백 개의 의존 관계
```

```
문제 1: 의존 관계가 바뀌면?
  UserService에 새 의존성 추가
  → 모든 생성 코드 수정 필요

문제 2: 테스트할 때?
  UserService 단위 테스트
  → 실제 DB 연결 없이 Mock 주입 어려움

문제 3: 환경별 구현체 교체?
  개발 환경: H2, 운영 환경: MySQL
  → 분기 코드 난무
```

스프링은 이 문제를 **IoC Container**로 해결한다.  
객체 생성과 의존 관계 연결의 제어권을 컨테이너에게 역전(Inversion of Control)시킨다.

그 컨테이너의 최소 계약이 `BeanFactory`이고,  
실무에서 사용하는 풀스택 인터페이스가 `ApplicationContext`다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: "ApplicationContext가 BeanFactory를 감싸고 있다(has-a)"

```java
// ❌ 잘못된 이해 — Composition이라고 착각
public class MyApplicationContext {
    private BeanFactory beanFactory;  // 내부에 BeanFactory를 포함?

    public Object getBean(String name) {
        return beanFactory.getBean(name);  // 위임?
    }
}
```

```
잘못된 이해:
  ApplicationContext
      └── (has-a) BeanFactory  ← Wrapper / Composition

실제 관계:
  ApplicationContext
      └── (is-a) BeanFactory   ← Inheritance (인터페이스 상속)
```

많은 개발자가 ApplicationContext가 BeanFactory를 **포함(Composition)**한다고 생각하지만,  
실제로는 `ApplicationContext`가 `BeanFactory` 인터페이스를 **상속**하는 계층 구조다.

또 다른 오해: "BeanFactory를 쓰면 더 가볍다, 실무에서도 쓸 수 있다"

```java
// ❌ 직접 BeanFactory 사용 — 대부분의 실무에서 부적합
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("applicationContext.xml");

// 문제:
// - refresh() 없음 → BeanPostProcessor 자동 등록 안 됨
// - @Autowired 작동 안 함 (AutowiredAnnotationBeanPostProcessor 미등록)
// - 이벤트 발행 불가
// - 프로퍼티 주입 안 됨
```

---

## ✨ 올바른 이해와 사용

### After: 인터페이스 계층 구조 전체를 파악한다

```
BeanFactory                        (최상위: Bean 조회 계약)
    ↑                  ↑
HierarchicalBeanFactory    ListableBeanFactory
(부모 컨텍스트 탐색)         (타입으로 목록 조회)
    ↑                  ↑
    └──────────┬────────┘
               ↑
        ApplicationContext          (+ 이벤트, i18n, 리소스, 환경)
               ↑
  ConfigurableApplicationContext    (+ refresh, close, addBeanFactoryPostProcessor)
               ↑
  AbstractApplicationContext        (추상 구현체 — refresh() 핵심 로직)
               ↑
    ┌──────────┴──────────────────────────┐
    ↑                                     ↑
AnnotationConfigApplicationContext    ClassPathXmlApplicationContext
(Java Config / @Component 스캔)        (XML 설정)
    ↑
GenericWebApplicationContext          (웹 환경 — Spring Boot)
```

---

## 🔬 내부 동작 원리

### 1. BeanFactory — 컨테이너의 최소 계약

```java
// spring-beans/src/main/java/org/springframework/beans/factory/BeanFactory.java
public interface BeanFactory {

    // FactoryBean 자체를 가져올 때 쓰는 접두사 ("&myFactory")
    String FACTORY_BEAN_PREFIX = "&";

    // Bean 조회 — 이름 / 타입 / 이름+타입 / 이름+생성인자
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;

    // 지연/선택적 조회 (Optional 대안)
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    // 메타 정보 조회
    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    String[] getAliases(String name);
}
```

```
BeanFactory가 정의하는 것:
  - Bean 조회       (getBean 오버로드 4종)
  - Bean 존재 확인   (containsBean)
  - Scope 확인      (isSingleton, isPrototype)
  - 타입 확인       (getType)
  - 별칭 조회       (getAliases)

BeanFactory가 정의하지 않는 것:
  - Bean 등록/스캔
  - 이벤트 발행/구독
  - 국제화 (MessageSource)
  - 리소스 로딩 (classpath*, file:)
  - 환경 프로파일 (@Profile)
  → 모두 ApplicationContext의 영역
```

### 2. ApplicationContext — 실무 컨테이너의 전체 계약

```java
// spring-context/.../ApplicationContext.java
public interface ApplicationContext
        extends EnvironmentCapable,         // getEnvironment() — 프로파일, 프로퍼티
                ListableBeanFactory,        // getBeanNamesForType(), getBeansOfType()
                HierarchicalBeanFactory,    // getParentBeanFactory()
                MessageSource,              // getMessage() — 다국어
                ApplicationEventPublisher,  // publishEvent()
                ResourcePatternResolver {   // getResources("classpath*:...")

    String getId();
    String getApplicationName();
    String getDisplayName();
    long getStartupDate();
    ApplicationContext getParent();
    AutowireCapableBeanFactory getAutowireCapableBeanFactory();
}
```

```
추가된 6가지 능력:

EnvironmentCapable
  → @Profile, @Value("${...}"), application.yml 접근

ListableBeanFactory
  → ctx.getBeansOfType(Repository.class)
  → 타입으로 Bean 목록 전체 조회

HierarchicalBeanFactory
  → getParentBeanFactory()
  → Spring MVC Root/Servlet 컨텍스트 계층

MessageSource
  → ctx.getMessage("greeting", null, Locale.KOREAN)
  → messages.properties 다국어 처리

ApplicationEventPublisher
  → ctx.publishEvent(new UserCreatedEvent(user))
  → @EventListener 자동 호출

ResourcePatternResolver
  → ctx.getResources("classpath*:config/*.yml")
  → 여러 JAR에 걸친 리소스 탐색
```

### 3. getBean() 내부 흐름 — AbstractBeanFactory 소스 추적

`getBean()`이 호출될 때 `AbstractBeanFactory.doGetBean()`이 실행된다.

```java
// AbstractBeanFactory.java (핵심 로직 단순화)
protected <T> T doGetBean(String name, Class<T> requiredType, Object[] args, boolean typeCheckOnly) {

    // ── 1단계: 이름 정규화 ──────────────────────────────────────────
    String beanName = transformedBeanName(name);
    // "userService"  → alias 해석 → 실제 beanName
    // "&myFactory"   → FactoryBean 자체 조회 플래그

    // ── 2단계: Singleton 캐시 조회 ──────────────────────────────────
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 이미 생성된 Singleton → 즉시 반환 (대부분 여기서 끝)
        return (T) getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    // ── 3단계: 부모 컨텍스트 위임 ───────────────────────────────────
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        // 현재 컨텍스트에 없으면 부모에게 위임
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }

    // ── 4단계: BeanDefinition 가져오기 ──────────────────────────────
    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    // 부모 BeanDefinition merge (상속 처리)

    // ── 5단계: depends-on 선행 Bean 생성 ────────────────────────────
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
        for (String dep : dependsOn) {
            registerDependentBean(dep, beanName);
            getBean(dep);  // 재귀 — 선행 Bean 먼저 생성
        }
    }

    // ── 6단계: Scope에 따른 생성 ────────────────────────────────────
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName,
            () -> createBean(beanName, mbd, args));  // 동기화 보장
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);

    } else if (mbd.isPrototype()) {
        bean = createBean(beanName, mbd, args);      // 매번 새 인스턴스

    } else {
        // Request, Session, 커스텀 Scope
        Scope scope = this.scopes.get(mbd.getScope());
        bean = scope.get(beanName, () -> createBean(beanName, mbd, args));
    }

    // ── 7단계: 타입 불일치 시 변환 ──────────────────────────────────
    if (requiredType != null && !requiredType.isInstance(bean)) {
        bean = getTypeConverter().convertIfNecessary(bean, requiredType);
    }

    return (T) bean;
}
```

```
doGetBean() 7단계 요약:

1. 이름 정규화     alias → 실제 beanName, "&" 접두사 처리
2. 캐시 조회       Singleton이면 3단계 캐시에서 즉시 반환
3. 부모 위임       현재 컨텍스트에 없으면 부모 BeanFactory로
4. 메타데이터      BeanDefinition 조회 (merge 포함)
5. 선행 의존       depends-on Bean 재귀 생성
6. Scope 생성      Singleton(동기화) / Prototype(새 인스턴스) / 커스텀
7. 타입 변환       ConversionService로 타입 불일치 해결
```

### 4. Lazy vs Eager — 어느 계층에서 결정되는가

```java
// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {
    // ... 여러 단계 ...
    // 마지막 단계: Non-Lazy Singleton 모두 미리 생성
    finishBeanFactoryInitialization(beanFactory);
    // ...
}

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // ConversionService, StringValueResolver 등 초기화 ...
    beanFactory.preInstantiateSingletons();  // ← Eager Init 핵심
}

// DefaultListableBeanFactory.java
public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

        if (!bd.isAbstract()
                && bd.isSingleton()
                && !bd.isLazyInit()) {          // ← @Lazy 있으면 건너뜀
            if (isFactoryBean(beanName)) {
                getBean(FACTORY_BEAN_PREFIX + beanName);  // FactoryBean 자체 생성
            } else {
                getBean(beanName);              // ← Eager Initialization 발생
            }
        }
    }
    // SmartInitializingSingleton 콜백 호출 (afterSingletonsInstantiated)
}
```

```
Lazy vs Eager 결정 구조:

순수 BeanFactory
  → preInstantiateSingletons() 없음
  → 항상 Lazy (getBean 호출 시 생성)
  → 컨테이너 시작 시 오류 감지 불가

ApplicationContext (refresh() 포함)
  → refresh() → finishBeanFactoryInitialization()
  → Singleton + Non-Lazy → 컨테이너 시작 시 즉시 생성
  → 시작 단계에서 설정 오류 즉시 발견

@Lazy 어노테이션
  → BeanDefinition.isLazyInit() = true
  → preInstantiateSingletons() 루프에서 제외
  → 첫 getBean() 시점까지 생성 지연

spring.main.lazy-initialization=true (Spring Boot)
  → 모든 Bean에 LazyInit 적용
  → 시작 속도 향상, 첫 요청 지연 트레이드오프
```

### 5. 주요 구현체 — 무엇이 다른가

```
AnnotationConfigApplicationContext
  생성:  new AnnotationConfigApplicationContext(AppConfig.class)
  내부:  AnnotatedBeanDefinitionReader
         → @Bean, @Import, @ComponentScan 처리
         ClassPathBeanDefinitionScanner
         → @Component, @Service, @Repository 스캔
  용도:  Spring Boot 기반 / Java Config

ClassPathXmlApplicationContext
  생성:  new ClassPathXmlApplicationContext("applicationContext.xml")
  내부:  XmlBeanDefinitionReader
         → <bean>, <context:component-scan> 파싱
  용도:  레거시 XML 기반 프로젝트

GenericWebApplicationContext (Spring Boot 웹)
  생성:  EmbeddedWebApplicationContext (내부적으로)
  내부:  Servlet Container와 통합
         → WebApplicationContext.getServletContext()
  용도:  Spring Boot 웹 애플리케이션

공통점:
  모두 AbstractApplicationContext.refresh() 실행
  → 동일한 Bean 생명주기 (BeanPostProcessor 체인 등)
  → 동일한 getBean() 구현 (AbstractBeanFactory)
```

---

## 💻 실험으로 확인하기

### 실험 1: Eager vs Lazy 초기화 시점 비교

```java
public class InitTimingTest {

    static class HeavyBean {
        public HeavyBean() {
            System.out.println("[생성] HeavyBean @ " + System.currentTimeMillis());
        }
    }

    public static void main(String[] args) {
        // ── ApplicationContext (Eager) ──────────────────────────────
        System.out.println("=== ApplicationContext ===");
        System.out.println("refresh() 전");

        GenericApplicationContext ctx = new GenericApplicationContext();
        ctx.registerBeanDefinition("heavy",
            BeanDefinitionBuilder.genericBeanDefinition(HeavyBean.class)
                .getBeanDefinition());

        ctx.refresh();  // ← 여기서 HeavyBean 생성
        System.out.println("refresh() 후");
        System.out.println("getBean() 호출");
        ctx.getBean("heavy");  // 캐시에서 반환, 재생성 없음
        System.out.println();

        // ── 순수 BeanFactory (Lazy) ──────────────────────────────────
        System.out.println("=== BeanFactory (순수) ===");
        DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
        factory.registerBeanDefinition("heavy",
            BeanDefinitionBuilder.genericBeanDefinition(HeavyBean.class)
                .getBeanDefinition());

        System.out.println("등록 후 (아직 생성 안 됨)");
        factory.getBean("heavy");  // ← 여기서 HeavyBean 생성
        System.out.println("getBean() 후");
    }
}
```

```
출력:
=== ApplicationContext ===
refresh() 전
[생성] HeavyBean @ 1700000001000  ← refresh() 안에서 생성
refresh() 후
getBean() 호출                    ← 재생성 없음 (캐시 반환)

=== BeanFactory (순수) ===
등록 후 (아직 생성 안 됨)
[생성] HeavyBean @ 1700000002000  ← getBean() 호출 시 생성
getBean() 후
```

### 실험 2: 부모-자식 컨텍스트 계층 확인

```java
@Configuration
class ParentConfig {
    @Bean
    public String sharedService() { return "부모 Bean"; }
}

@Configuration
class ChildConfig {
    @Bean
    public String childOnlyService() { return "자식 Bean"; }
}

public class HierarchyTest {
    public static void main(String[] args) {
        // 부모 컨텍스트 생성
        AnnotationConfigApplicationContext parent =
            new AnnotationConfigApplicationContext(ParentConfig.class);

        // 자식 컨텍스트 — refresh() 전에 setParent 필수
        AnnotationConfigApplicationContext child =
            new AnnotationConfigApplicationContext();
        child.setParent(parent);           // ← refresh() 이전에!
        child.register(ChildConfig.class);
        child.refresh();

        // 자식에서 부모 Bean 조회 가능
        System.out.println(child.getBean("sharedService"));    // "부모 Bean"
        System.out.println(child.getBean("childOnlyService")); // "자식 Bean"

        // 부모에서 자식 Bean 조회 불가
        try {
            parent.getBean("childOnlyService");
        } catch (NoSuchBeanDefinitionException e) {
            System.out.println("부모는 자식 Bean 모름");
        }
    }
}
```

```
출력:
부모 Bean
자식 Bean
부모는 자식 Bean 모름
```

### 실험 3: Singleton 캐시 확인

```java
@Configuration
class Config {
    @Bean
    public List<String> myList() {
        return new ArrayList<>();
    }
}

public class SingletonCacheTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext(Config.class);

        List<String> list1 = ctx.getBean("myList", List.class);
        List<String> list2 = ctx.getBean("myList", List.class);

        System.out.println("같은 인스턴스? " + (list1 == list2));  // true

        list1.add("hello");
        System.out.println("list2 확인: " + list2);  // [hello] — 동일 참조
    }
}
```

```
출력:
같은 인스턴스? true
list2 확인: [hello]
```

---

## 🤔 트레이드오프

```
BeanFactory를 직접 사용해야 하는 상황:
  - 메모리 극도로 제한된 환경 (임베디드, Android)
    ApplicationContext의 이벤트/i18n/리소스 로딩 비용 제거
  - 커스텀 컨테이너 구현
    최소 인터페이스만 구현해 경량 IoC 제작
  - 단위 테스트에서 최소 컨텍스트

ApplicationContext Eager Init 트레이드오프:
  장점
    시작 시 설정 오류 즉시 발견 (@Autowired 실패 등)
    첫 요청 지연(Cold Start) 없음
  단점
    시작 시간 증가 (Bean 수에 비례)
    실제로 호출 안 하는 Bean도 생성

@Lazy 전략적 활용:
  무거운 초기화 Bean에만 선택적 적용
  spring.main.lazy-initialization=true (전체 Lazy)
    → 개발 환경 시작 속도 향상용
    → 운영 환경에는 주의 (첫 요청 응답 지연)
```

---

## 📌 핵심 정리

```
is-a 관계
  ApplicationContext는 BeanFactory를 "포함"하지 않음
  ApplicationContext는 BeanFactory 인터페이스를 "상속"

BeanFactory 책임
  Bean 조회 (getBean)
  Bean 존재/타입/Scope 확인

ApplicationContext 추가 능력
  EnvironmentCapable   → 프로파일, 프로퍼티
  ListableBeanFactory  → 타입으로 목록 조회
  MessageSource        → 다국어
  EventPublisher       → publishEvent
  ResourcePattern      → classpath*: 탐색

getBean() 7단계
  이름 정규화 → 캐시 조회 → 부모 위임
  → BeanDefinition → depends-on → Scope 생성 → 타입 변환

Eager vs Lazy
  BeanFactory: 항상 Lazy
  ApplicationContext: refresh() 시 Non-Lazy Singleton Eager 생성
  @Lazy: preInstantiateSingletons() 제외

주요 구현체
  AnnotationConfigApplicationContext  → Java Config
  ClassPathXmlApplicationContext      → XML 레거시
  GenericWebApplicationContext        → Spring Boot 웹
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 코드의 `UserService` 생성 시점 차이를 설명하라.

```java
// Code A
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
factory.registerBeanDefinition("userService", ...);
UserService svc = factory.getBean(UserService.class);

// Code B
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);
UserService svc = ctx.getBean(UserService.class);
```

**Q2.** 다음 코드에서 `NoSuchBeanDefinitionException`이 발생하는 이유는?

```java
AnnotationConfigApplicationContext child =
    new AnnotationConfigApplicationContext(ChildConfig.class);  // refresh() 포함

AnnotationConfigApplicationContext parent =
    new AnnotationConfigApplicationContext(ParentConfig.class);

child.setParent(parent);  // refresh() 이후에 호출

child.getBean("parentBean");  // ??
```

**Q3.** `BeanFactory`가 `MessageSource`를 제공하지 않는 이유를 스프링 설계 원칙 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** Code A는 `DefaultListableBeanFactory`를 직접 사용하므로 `preInstantiateSingletons()`가 호출되지 않는다. `UserService`는 `factory.getBean()` 호출 시점에 처음 생성된다. Code B는 생성자 내부에서 `refresh()`가 자동 호출되고, `finishBeanFactoryInitialization()` → `preInstantiateSingletons()`가 실행돼 `ctx.getBean()` 호출 **전에** `UserService`가 이미 생성되어 캐시에 있다. `getBean()`은 단순히 캐시에서 꺼내는 것이다.
>
> **Q2.** `child`의 생성자에서 이미 `refresh()`가 완료됐다. `refresh()` 안의 `finishBeanFactoryInitialization()`은 그 시점의 부모 컨텍스트 참조(null)를 기준으로 Bean을 초기화한다. 이후에 `setParent(parent)`를 호출해봤자 이미 완료된 초기화 단계에는 영향을 주지 못한다. `setParent()`는 반드시 `refresh()` **이전**에 호출해야 한다.
>
> **Q3.** 스프링은 **인터페이스 분리 원칙(ISP)**에 따라 설계했다. `BeanFactory`는 "Bean 저장소와 조회"라는 단일 책임만 가진다. 국제화(`MessageSource`)는 Bean 관리와 무관한 기능이므로 별도 인터페이스로 분리했다. 덕분에 메모리 제한 환경에서 `BeanFactory`만 구현하면 불필요한 기능을 강제로 포함하지 않아도 된다. `ApplicationContext`는 다양한 인터페이스를 합성해 "일반 애플리케이션에 필요한 모든 것"을 제공하는 편의 Facade 역할이다.

---

## 📚 참고 자료

- [Spring Docs — BeanFactory](https://docs.spring.io/spring-framework/reference/core/beans/beanfactory.html)
- [Spring Source — AbstractBeanFactory.doGetBean()](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractBeanFactory.java)
- [Spring Source — AbstractApplicationContext.refresh()](https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java)
- [Spring Source — DefaultListableBeanFactory.preInstantiateSingletons()](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java)

---

<div align="center">

**[⬅️ 목차로](../README.md)** | **[다음: BeanDefinition과 Bean 메타데이터 ➡️](./02-bean-definition-metadata.md)**

</div>

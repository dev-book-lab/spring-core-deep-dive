# BeanDefinition과 Bean 메타데이터 — 스프링이 Bean을 기억하는 방식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스프링은 Bean의 어떤 정보를 어떻게 저장하는가?
- `BeanDefinition`은 어떤 필드로 구성되어 있는가?
- XML / 어노테이션 / Java Config는 각각 어떻게 `BeanDefinition`으로 변환되는가?
- `BeanDefinitionRegistry`에 직접 Bean을 등록할 수 있는가?
- `BeanFactoryPostProcessor`와 `BeanDefinition`의 관계는 무엇인가?
- 런타임에 BeanDefinition을 수정하면 어떤 일이 벌어지는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 컨테이너는 Bean을 어떻게 "기억"하는가

```java
@Component
public class UserService {
    @Autowired
    private UserRepository userRepository;

    @Scope("prototype")
    // ... 기타 설정
}
```

```
컨테이너가 알아야 할 것:
  - 어떤 클래스인가?          (UserService.class)
  - Scope는?                  (singleton / prototype / ...)
  - 어떤 의존성이 있는가?      (UserRepository)
  - 초기화 메서드는?           (@PostConstruct, init-method)
  - Lazy 초기화인가?           (@Lazy)
  - Primary Bean인가?          (@Primary)
  - 어떤 어노테이션이 붙었나?   (메타데이터)

이 모든 정보를 담는 객체가 BeanDefinition이다.
```

스프링은 **실제 Bean 인스턴스를 만들기 전에** 먼저 `BeanDefinition`이라는 설계도를 만든다.  
인스턴스 생성은 이 설계도를 기반으로 이루어진다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: "스프링은 어노테이션을 보고 바로 인스턴스를 만든다"

```java
// ❌ 잘못된 이해
@Component
public class MyService { ... }

// 스프링이 하는 일 (잘못된 이해):
// @Component 발견 → new MyService() 즉시 실행
// → 이미 완성된 인스턴스 저장

// 실제로 하는 일:
// @Component 발견 → BeanDefinition 생성 → 저장
//                → 나중에 getBean() / refresh() 시 인스턴스 생성
```

```
잘못된 흐름:
  @Component 스캔 → new MyService() → 저장

실제 흐름:
  @Component 스캔 → BeanDefinition 생성 → BeanDefinitionRegistry 등록
                 → (나중에) BeanDefinition 기반 → createBean() → 인스턴스
```

설계도(`BeanDefinition`)와 실제 객체(`Bean 인스턴스`)는 **별개로 존재**한다.  
이 분리 덕분에 `BeanFactoryPostProcessor`로 인스턴스 생성 전에 설계도를 수정할 수 있다.

---

## ✨ 올바른 이해와 사용

### After: BeanDefinition은 Bean의 설계도다

```
스캔/파싱 단계 (설계도 생성):
  XML           → XmlBeanDefinitionReader    → BeanDefinition
  @Component    → ClassPathBeanDefinitionScanner → BeanDefinition
  @Bean         → ConfigurationClassParser   → BeanDefinition
                         ↓
                  BeanDefinitionRegistry
                  (설계도 저장소)
                         ↓
              BeanFactoryPostProcessor 개입 가능 (설계도 수정)
                         ↓
인스턴스 생성 단계:
              BeanDefinition → createBean() → 실제 인스턴스
```

---

## 🔬 내부 동작 원리

### 1. BeanDefinition 인터페이스 전체 필드

```java
// spring-beans/.../BeanDefinition.java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    // ── Scope ────────────────────────────────────────────────────────
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";

    void setScope(String scope);
    String getScope();
    boolean isSingleton();
    boolean isPrototype();

    // ── 클래스 정보 ──────────────────────────────────────────────────
    void setBeanClassName(String beanClassName);
    String getBeanClassName();

    // ── 팩토리 정보 (@Bean 메서드) ───────────────────────────────────
    void setFactoryBeanName(String factoryBeanName);
    String getFactoryBeanName();  // @Configuration 클래스 Bean 이름
    void setFactoryMethodName(String factoryMethodName);
    String getFactoryMethodName();  // @Bean 메서드 이름

    // ── 생성자 인자 ──────────────────────────────────────────────────
    ConstructorArgumentValues getConstructorArgumentValues();

    // ── 프로퍼티 값 ──────────────────────────────────────────────────
    MutablePropertyValues getPropertyValues();

    // ── 생명주기 콜백 ────────────────────────────────────────────────
    void setInitMethodName(String initMethodName);
    String getInitMethodName();
    void setDestroyMethodName(String destroyMethodName);
    String getDestroyMethodName();

    // ── 초기화 제어 ──────────────────────────────────────────────────
    void setLazyInit(boolean lazyInit);
    boolean isLazyInit();

    // ── 의존 관계 ────────────────────────────────────────────────────
    void setDependsOn(String... dependsOn);
    String[] getDependsOn();

    // ── 자동 주입 ────────────────────────────────────────────────────
    void setAutowireCandidate(boolean autowireCandidate);
    boolean isAutowireCandidate();
    void setPrimary(boolean primary);
    boolean isPrimary();

    // ── 역할 구분 ────────────────────────────────────────────────────
    int ROLE_APPLICATION = 0;   // 개발자가 정의한 Bean
    int ROLE_SUPPORT = 1;       // 인프라 지원 Bean
    int ROLE_INFRASTRUCTURE = 2; // 스프링 내부 Bean
    void setRole(int role);
    int getRole();

    // ── 설명 ─────────────────────────────────────────────────────────
    void setDescription(String description);
    String getDescription();

    // ── 출처 ─────────────────────────────────────────────────────────
    ResolvableType getResolvableType();
    boolean isAbstract();
    String getResourceDescription(); // "file [UserService.class]" 등
    BeanDefinition getOriginatingBeanDefinition();
}
```

### 2. 실제 구현체 — AbstractBeanDefinition

```java
// 실무에서 쓰이는 주요 구현체들
AbstractBeanDefinition               // 공통 필드 구현
    ↑
    ├── RootBeanDefinition           // 최종 Merged 결과 (getBean 시 사용)
    ├── ChildBeanDefinition          // 부모 BeanDefinition 상속 (XML parent=)
    └── GenericBeanDefinition        // 어노테이션 스캔 시 생성
```

```java
// AbstractBeanDefinition 주요 필드 (실제 저장되는 값)
public abstract class AbstractBeanDefinition implements BeanDefinition {

    private volatile Object beanClass;     // Class<?> 또는 String (클래스명)
    private String scope = SCOPE_DEFAULT;  // "" → Singleton으로 처리
    private boolean abstractFlag = false;
    private Boolean lazyInit;
    private int autowireMode = AUTOWIRE_NO;
    private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    private String[] dependsOn;
    private boolean autowireCandidate = true;
    private boolean primary = false;
    private String initMethodName;
    private String destroyMethodName;
    private boolean enforceInitMethod = true;
    private boolean enforceDestroyMethod = true;
    private boolean synthetic = false;    // true = 스프링 내부 생성 Bean
    private int role = ROLE_APPLICATION;
    private String description;
    private Resource resource;            // 이 BeanDef가 어느 파일에서 왔는가

    private ConstructorArgumentValues constructorArgumentValues;
    private MutablePropertyValues propertyValues;
    private MethodOverrides methodOverrides;  // lookup-method, replaced-method
    private List<AutowireCandidateQualifier> qualifiers;
    // ...
}
```

### 3. BeanDefinition 생성 경로 — 3가지 방식

#### 경로 1: @Component 스캔

```java
// ClassPathScanningCandidateComponentProvider → ScannedGenericBeanDefinition 생성
// ASM 기반으로 클래스 로딩 없이 바이트코드에서 메타데이터 읽음

@Component
@Scope("prototype")
@Lazy
public class UserService { ... }
```

```
스캔 결과로 생성되는 BeanDefinition:
  ScannedGenericBeanDefinition {
      beanClassName = "com.example.UserService"
      scope = "prototype"
      lazyInit = true
      role = ROLE_APPLICATION
      resource = "file [UserService.class]"
  }
```

#### 경로 2: Java Config @Bean

```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "init", destroyMethod = "cleanup")
    @Scope("singleton")
    @Primary
    public DataSource dataSource() {
        return new HikariDataSource(...);
    }
}
```

```
ConfigurationClassParser가 생성하는 BeanDefinition:
  ConfigurationClassBeanDefinition {
      beanClassName = "com.zaxxer.hikari.HikariDataSource"
      factoryBeanName = "appConfig"       ← @Configuration 클래스
      factoryMethodName = "dataSource"    ← @Bean 메서드 이름
      scope = "singleton"
      primary = true
      initMethodName = "init"
      destroyMethodName = "cleanup"
  }
```

#### 경로 3: 직접 등록 (프로그래밍 방식)

```java
// BeanDefinitionBuilder 사용
GenericBeanDefinition bd = BeanDefinitionBuilder
    .genericBeanDefinition(UserService.class)
    .addConstructorArgReference("userRepository")
    .setScope(BeanDefinition.SCOPE_SINGLETON)
    .setLazyInit(true)
    .setInitMethodName("init")
    .setPrimary(true)
    .getBeanDefinition();

BeanDefinitionRegistry registry = (BeanDefinitionRegistry) ctx.getBeanFactory();
registry.registerBeanDefinition("userService", bd);
```

### 4. BeanDefinitionRegistry — 설계도 저장소

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException;

    void removeBeanDefinition(String beanName)
            throws NoSuchBeanDefinitionException;

    BeanDefinition getBeanDefinition(String beanName)
            throws NoSuchBeanDefinitionException;

    boolean containsBeanDefinition(String beanName);

    String[] getBeanDefinitionNames();

    int getBeanDefinitionCount();

    boolean isBeanNameInUse(String beanName);
}
```

```
DefaultListableBeanFactory (BeanDefinitionRegistry 구현체):

내부 저장 구조:
  Map<String, BeanDefinition> beanDefinitionMap
    = new ConcurrentHashMap<>(256);
  
  List<String> beanDefinitionNames
    = new ArrayList<>(256);  // 등록 순서 보존

등록 순서가 중요한 이유:
  동일 Bean 이름 중복 등록 시 나중에 등록된 것이 덮어씀
  @Primary 없으면 등록 순서가 선택에 영향
```

### 5. BeanFactoryPostProcessor — 설계도 수정 가능 지점

```java
// BeanDefinition이 모두 등록된 후, 인스턴스 생성 전에 개입 가능
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException;
}
```

```java
// 실제 사용 예: 특정 Bean의 Scope를 런타임에 변경
@Component
public class ScopeModifier implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        BeanDefinition bd = beanFactory.getBeanDefinition("userService");
        
        System.out.println("변경 전 scope: " + bd.getScope());
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
        System.out.println("변경 후 scope: " + bd.getScope());
        
        // 인스턴스 생성 전이므로 변경이 실제 반영됨
    }
}
```

```
BeanFactoryPostProcessor 실행 시점:

refresh() 내부:
  1. BeanDefinition 등록 (스캔 / 파싱)
  2. BeanFactoryPostProcessor 실행   ← 설계도 수정 가능
  3. BeanPostProcessor 등록
  4. 인스턴스 생성 (preInstantiateSingletons)

대표적인 BeanFactoryPostProcessor:
  PropertySourcesPlaceholderConfigurer
    → @Value("${...}") 플레이스홀더를 실제 값으로 교체
  ConfigurationClassPostProcessor
    → @Configuration 클래스 처리, CGLIB 프록시 적용 표시
```

---

## 💻 실험으로 확인하기

### 실험 1: BeanDefinition 내용 직접 출력

```java
@Configuration
class AppConfig {
    @Bean
    @Scope("prototype")
    @Primary
    public UserService userService() {
        return new UserService();
    }
}

public class BeanDefinitionInspector {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext(AppConfig.class);

        ConfigurableListableBeanFactory factory = ctx.getBeanFactory();

        BeanDefinition bd = factory.getBeanDefinition("userService");

        System.out.println("클래스명:       " + bd.getBeanClassName());
        System.out.println("Scope:          " + bd.getScope());
        System.out.println("Primary:        " + bd.isPrimary());
        System.out.println("LazyInit:       " + bd.isLazyInit());
        System.out.println("팩토리 Bean:    " + bd.getFactoryBeanName());
        System.out.println("팩토리 메서드:  " + bd.getFactoryMethodName());
        System.out.println("출처:           " + bd.getResourceDescription());
        System.out.println("역할:           " + bd.getRole()); // 0 = APPLICATION
    }
}
```

```
출력:
클래스명:       com.example.UserService
Scope:          prototype
Primary:        true
LazyInit:       false
팩토리 Bean:    appConfig
팩토리 메서드:  userService
출처:           com.example.AppConfig
역할:           0
```

### 실험 2: BeanFactoryPostProcessor로 런타임 수정

```java
static class MyBean {
    public MyBean() {
        System.out.println("MyBean 생성 — scope: singleton 이었나? prototype 이었나?");
    }
}

@Component
static class BdModifier implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        BeanDefinition bd = bf.getBeanDefinition("myBean");
        System.out.println("수정 전: " + bd.getScope());
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);
        System.out.println("수정 후: " + bd.getScope());
    }
}

public class ModifyBdTest {
    public static void main(String[] args) {
        GenericApplicationContext ctx = new GenericApplicationContext();
        ctx.registerBeanDefinition("myBean",
            BeanDefinitionBuilder.genericBeanDefinition(MyBean.class).getBeanDefinition());
        ctx.registerBeanDefinition("bdModifier",
            BeanDefinitionBuilder.genericBeanDefinition(BdModifier.class).getBeanDefinition());
        ctx.refresh();

        Object a = ctx.getBean("myBean");
        Object b = ctx.getBean("myBean");
        System.out.println("같은 인스턴스? " + (a == b));  // false (prototype)
    }
}
```

```
출력:
수정 전: singleton
수정 후: prototype
MyBean 생성 — ...   ← getBean() 할 때마다 새로 생성
MyBean 생성 — ...
같은 인스턴스? false
```

### 실험 3: 등록된 모든 BeanDefinition 목록 출력

```java
public class BdListInspector {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext(AppConfig.class);

        ConfigurableListableBeanFactory factory = ctx.getBeanFactory();
        String[] names = factory.getBeanDefinitionNames();

        System.out.println("총 BeanDefinition 수: " + names.length);

        for (String name : names) {
            BeanDefinition bd = factory.getBeanDefinition(name);
            // ROLE_INFRASTRUCTURE(2)는 스프링 내부 Bean — 필터링
            if (bd.getRole() == BeanDefinition.ROLE_APPLICATION) {
                System.out.printf("%-40s scope=%-12s primary=%b%n",
                    name, bd.getScope(), bd.isPrimary());
            }
        }
    }
}
```

---

## 🤔 트레이드오프

```
BeanDefinition 직접 조작의 트레이드오프:

장점
  인스턴스 생성 전 설정 변경 가능
  동적 Bean 등록 (플러그인, 멀티테넌트 등)
  테스트 환경에서 Bean 교체

단점
  refresh() 이후 조작은 효과 없거나 위험
    → 이미 생성된 Singleton에는 반영 안 됨
  @Autowired 등 후처리 타이밍과 충돌 가능
  코드 가독성 저하 (암묵적 동작)

BeanFactoryPostProcessor vs BeanPostProcessor:
  BeanFactoryPostProcessor
    → 인스턴스 생성 전, BeanDefinition(설계도) 수정
    → PropertySourcesPlaceholderConfigurer가 대표 예
  BeanPostProcessor
    → 인스턴스 생성 후, 실제 객체에 개입
    → @Autowired 처리, @Transactional 프록시 생성
```

---

## 📌 핵심 정리

```
BeanDefinition
  Bean의 설계도 (인스턴스가 아님)
  클래스명, Scope, Primary, LazyInit, 생명주기 메서드 등 저장

생성 경로 3가지
  @Component 스캔 → ScannedGenericBeanDefinition
  @Bean 메서드   → ConfigurationClassBeanDefinition
  직접 등록       → BeanDefinitionBuilder

BeanDefinitionRegistry
  설계도 저장소
  DefaultListableBeanFactory가 구현체
  내부: ConcurrentHashMap<String, BeanDefinition>

설계도 → 인스턴스 흐름
  스캔/파싱 → BeanDefinitionRegistry 등록
  → BeanFactoryPostProcessor 개입 (설계도 수정 가능)
  → createBean() → 실제 인스턴스

BeanFactoryPostProcessor
  인스턴스 생성 전 개입 지점
  BeanDefinition을 읽거나 수정 가능
  대표: PropertySourcesPlaceholderConfigurer, ConfigurationClassPostProcessor

ROLE 구분
  ROLE_APPLICATION(0)   : 개발자가 정의한 Bean
  ROLE_SUPPORT(1)       : 인프라 지원
  ROLE_INFRASTRUCTURE(2): 스프링 내부 Bean
```

---

## 🤔 생각해볼 문제

**Q1.** `@Bean` 메서드로 등록된 Bean의 `BeanDefinition`에서 `getBeanClassName()`과 `getFactoryMethodName()`은 각각 무엇을 반환하는가? `@Component`로 등록된 Bean과 비교하라.

**Q2.** 다음 코드가 의도대로 동작하지 않는 이유는? 어느 시점에 실행해야 하는가?

```java
// 어딘가에서 refresh() 이후 호출
BeanDefinition bd = ctx.getBeanFactory().getBeanDefinition("userService");
bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);

Object a = ctx.getBean("userService");
Object b = ctx.getBean("userService");
System.out.println(a == b);  // true? false?
```

**Q3.** `BeanFactoryPostProcessor`와 `BeanPostProcessor`의 실행 시점 차이를 `refresh()` 흐름 기준으로 설명하고, 각각 어떤 작업에 적합한지 예시를 들어라.

> 💡 **해설**
>
> **Q1.** `@Bean` 메서드로 등록된 경우: `getBeanClassName()`은 반환 타입인 실제 구현 클래스(예: `"com.zaxxer.hikari.HikariDataSource"`), `getFactoryBeanName()`은 `@Configuration` 클래스의 Bean 이름(예: `"appConfig"`), `getFactoryMethodName()`은 `@Bean` 메서드 이름(예: `"dataSource"`)을 반환한다. `@Component`로 등록된 경우: `getBeanClassName()`은 해당 클래스(예: `"com.example.UserService"`), `getFactoryBeanName()`과 `getFactoryMethodName()`은 `null`이다. 스프링은 팩토리 메서드 정보가 있으면 `factoryBean.factoryMethod()` 방식으로 인스턴스를 생성하고, 없으면 직접 생성자를 호출한다.
>
> **Q2.** `refresh()` 이후 수정은 이미 생성된 Singleton 인스턴스에 반영되지 않는다. `userService`가 Singleton이었다면 `refresh()` 시점에 이미 인스턴스가 캐시(`singletonObjects`)에 저장되어 있다. 이후에 `BeanDefinition`의 Scope를 변경해도 `getBean()`은 캐시에서 기존 Singleton을 반환하므로 `a == b`는 `true`다. `BeanDefinition`을 수정하려면 반드시 `refresh()` 이전, 즉 `BeanFactoryPostProcessor` 내부에서 해야 한다.
>
> **Q3.** `refresh()` 흐름에서 `BeanFactoryPostProcessor`는 모든 `BeanDefinition`이 등록된 직후, 인스턴스 생성 전에 실행된다. 설계도(메타데이터)를 읽거나 수정하는 작업에 적합하다. 예: `@Value("${db.url}")` 플레이스홀더를 실제 값으로 교체하는 `PropertySourcesPlaceholderConfigurer`. `BeanPostProcessor`는 각 Bean 인스턴스가 생성된 직후, 초기화 전후에 실행된다. 실제 객체에 개입하는 작업에 적합하다. 예: `@Autowired` 필드를 리플렉션으로 채우는 `AutowiredAnnotationBeanPostProcessor`, `@Transactional`이 붙은 Bean에 CGLIB 프록시를 씌우는 `AnnotationAwareAspectJAutoProxyCreator`.

---

<div align="center">

**[⬅️ 이전: BeanFactory vs ApplicationContext](./01-beanfactory-vs-applicationcontext.md)** | **[다음: BeanPostProcessor 체인 ➡️](./03-beanpostprocessor-chain.md)**

</div>

# ApplicationContext 계층 구조 — Parent-Child 컨텍스트의 비밀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ApplicationContext 계층 구조란 무엇이고 왜 필요한가?
- 자식 컨텍스트는 부모 Bean을 쓸 수 있지만, 왜 반대는 안 되는가?
- Spring MVC에서 Root Context와 Servlet Context가 분리된 이유는?
- Spring Boot는 왜 대부분의 경우 단일 컨텍스트를 사용하는가?
- 부모 컨텍스트에 같은 이름의 Bean이 있을 때 자식에서 재정의하면 어떻게 되는가?
- `HierarchicalBeanFactory.getParentBeanFactory()`는 어떻게 동작하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 하나의 컨테이너로 모든 것을 관리하면 무슨 문제가 생기는가

```
단일 ApplicationContext로 모든 Bean 관리:

  [Root Context]
    DataSource, UserService, OrderService,
    DispatcherServlet, Controller, ViewResolver, ...
    (수백 개 Bean이 한 공간에 혼재)
```

```
문제 1: 관심사 분리 실패
  비즈니스 레이어 Bean과 웹 레이어 Bean이 섞임
  → Service가 Controller를 @Autowired 할 수 있는 상황 (설계 오염)

문제 2: 독립 배포 불가
  여러 DispatcherServlet을 사용하는 경우
  (예: API용 Servlet, Admin용 Servlet)
  → 공통 Service Bean을 별도 컨텍스트에 두고 싶음

문제 3: 테스트 격리
  웹 레이어 없이 서비스 레이어만 테스트하고 싶음
```

이 문제를 해결하기 위해 ApplicationContext는 **Parent-Child 계층 구조**를 지원한다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 부모-자식 방향을 반대로 이해한다

```java
// ❌ 잘못된 이해
// "부모 컨텍스트가 자식 컨텍스트를 포함하고 있다"
// "부모에서 자식 Bean을 조회할 수 있다"

AnnotationConfigApplicationContext parent =
    new AnnotationConfigApplicationContext(ParentConfig.class);

AnnotationConfigApplicationContext child =
    new AnnotationConfigApplicationContext();
child.setParent(parent);
child.register(ChildConfig.class);
child.refresh();

// ❌ 이렇게 동작한다고 착각
parent.getBean("childOnlyBean");  // NoSuchBeanDefinitionException!
```

```
실제 탐색 방향:

  자식 → 부모 (O)
    자식에 없으면 부모에서 찾음

  부모 → 자식 (X)
    부모는 자식의 존재를 모름

구조:
  [Parent Context]
    sharedService ← 자식도 접근 가능
    dataSource    ← 자식도 접근 가능

  [Child Context]
    controller    ← 자식만 접근 가능
    viewResolver  ← 자식만 접근 가능
    (parent → child 탐색 불가)
```

### Before: Spring MVC에서 Bean 위치 혼동

```java
// ❌ 자주 발생하는 실수 — Spring MVC 레거시 환경
// Root Context에 Controller를 등록하고
// Servlet Context에 Service를 등록하는 경우

// web.xml (Root Context 설정)
// <context-param>
//   <param-name>contextConfigLocation</param-name>
//   <param-value>classpath:spring/service-context.xml</param-value>
// </context-param>

// service-context.xml
@Controller  // ← Root Context에 Controller 등록 (잘못된 위치)
public class UserController { }

// servlet-context.xml
@Service     // ← Servlet Context에 Service 등록 (잘못된 위치)
public class UserService { }

// 결과: UserController가 UserService를 @Autowired로 주입 불가
//       자식(Servlet)의 Bean을 부모(Root)에서 찾을 수 없음
```

---

## ✨ 올바른 이해와 사용

### After: 계층별 책임을 명확히 분리한다

```
올바른 Spring MVC 계층 구조:

  [Root ApplicationContext]         ← ContextLoaderListener 생성
    @Service (UserService)
    @Repository (UserRepository)
    DataSource, TransactionManager
    → 공통 비즈니스/인프라 Bean
    → 여러 Servlet이 공유

        ↑ (부모)
        |
  [Servlet ApplicationContext]      ← DispatcherServlet 생성
    @Controller (UserController)
    ViewResolver, HandlerMapping
    → 웹 레이어 Bean만
    → 특정 DispatcherServlet 전용

탐색 규칙:
  Controller에서 Service @Autowired
  → Servlet Context에서 먼저 탐색
  → 없으면 Root Context로 위임 → Service 발견 (O)

  Service에서 Controller @Autowired (잘못된 설계)
  → Root Context에서 탐색
  → Root는 Servlet Context 모름 → 발견 불가 (X) ← 설계 보호
```

---

## 🔬 내부 동작 원리

### 1. HierarchicalBeanFactory — 계층 탐색의 구현

```java
// HierarchicalBeanFactory.java
public interface HierarchicalBeanFactory extends BeanFactory {

    // 부모 BeanFactory 반환
    @Nullable
    BeanFactory getParentBeanFactory();

    // 현재 컨텍스트에서만 찾음 (부모 탐색 안 함)
    boolean containsLocalBean(String name);
}
```

```java
// AbstractBeanFactory.doGetBean() — 부모 위임 로직
BeanFactory parentBeanFactory = getParentBeanFactory();

if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // 현재 컨텍스트에 BeanDefinition 없음 → 부모로 위임
    String nameToLookup = originalBeanName(name);

    if (parentBeanFactory instanceof AbstractBeanFactory abf) {
        return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
    } else if (args != null) {
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    } else if (requiredType != null) {
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    } else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
// 부모에도 없으면 NoSuchBeanDefinitionException
```

```
탐색 알고리즘:

getBean("userService") on child:
  1. child.containsBeanDefinition("userService") → false
  2. parent != null → parent.getBean("userService")
  3. parent.containsBeanDefinition("userService") → true
  4. parent에서 생성/반환

getBean("controller") on parent:
  1. parent.containsBeanDefinition("controller") → false
  2. parent.getParentBeanFactory() → null (부모 없음)
  3. NoSuchBeanDefinitionException 발생
```

### 2. Spring MVC 전통 계층 구조 (레거시)

```
웹 애플리케이션 시작 순서:

1. Servlet Container (Tomcat) 시작
   ↓
2. ContextLoaderListener.contextInitialized()
   → Root ApplicationContext 생성 (XML 또는 Java Config)
   → Service, Repository, DataSource 등 등록
   → ServletContext에 저장:
     servletContext.setAttribute(
       WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE,
       rootContext);
   ↓
3. DispatcherServlet.init()
   → Servlet ApplicationContext 생성
   → 부모로 Root Context 설정:
     wac.setParent(rootContext);  // ← 여기서 계층 연결
   → Controller, ViewResolver 등 등록
   → refresh()

결과:
  Servlet Context 부모 = Root Context
  Controller → Service 탐색: Servlet → Root 위임 → 성공
  Service → Controller 탐색: Root에서 탐색 → 실패 (설계 의도)
```

### 3. Bean 재정의 — 자식이 부모 Bean을 오버라이딩

```java
// 부모 Config
@Configuration
class ParentConfig {
    @Bean
    public DataSource dataSource() {
        // 운영 DB
        return new HikariDataSource(prodConfig);
    }
}

// 자식 Config
@Configuration
class ChildConfig {
    @Bean
    public DataSource dataSource() {
        // 테스트 DB (같은 이름으로 재정의)
        return new EmbeddedDatabaseBuilder().setType(H2).build();
    }
}
```

```
Bean 재정의 탐색 순서:

getBean("dataSource") on child:
  1. child.containsBeanDefinition("dataSource") → true  ← 자식에 있음
  2. 부모로 위임하지 않고 자식 것 반환

→ 자식 Bean이 부모 Bean을 가림 (Shadow)
→ 테스트 환경에서 DataSource 교체에 활용
→ 단, 이름 충돌은 명시적으로 인지하고 사용해야 함
```

### 4. Spring Boot의 단일 컨텍스트 전략

```java
// SpringApplication.java (Boot)
ConfigurableApplicationContext run(String... args) {
    // 1. ApplicationContext 생성 (단일)
    context = createApplicationContext();
    // → AnnotationConfigServletWebServerApplicationContext (웹)
    // → AnnotationConfigApplicationContext (비웹)

    // 2. 준비
    prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

    // 3. refresh
    refreshContext(context);

    // 별도의 Root/Servlet 컨텍스트 분리 없음
}
```

```
Spring Boot가 단일 컨텍스트를 쓰는 이유:

전통적 분리의 문제:
  - XML 기반 설정의 복잡성
  - 계층 구조 이해 어려움
  - 실수로 Bean을 잘못된 컨텍스트에 등록

Boot의 해결책:
  - 모든 Bean을 단일 컨텍스트에서 관리
  - @SpringBootApplication → 단일 ComponentScan
  - 레이어 분리는 패키지 구조와 @Component 계층으로

단일 컨텍스트의 트레이드오프:
  장점: 간단, 실수 적음, 모든 Bean 상호 참조 가능
  단점: 웹 Bean이 서비스 Bean을 직접 참조 가능 (설계 규율 필요)

계층 구조가 여전히 필요한 경우:
  - 여러 DispatcherServlet 사용
  - 플러그인 아키텍처 (각 플러그인이 독립 컨텍스트)
  - @SpringBootTest의 슬라이스 테스트
    (@WebMvcTest → 웹 레이어만 컨텍스트 로드)
```

### 5. 컨텍스트 계층과 ApplicationContextAware

```java
// 자식 컨텍스트에서 부모 컨텍스트 접근
@Component
public class ContextInspector implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }

    public void printHierarchy() {
        ApplicationContext current = context;
        int level = 0;

        while (current != null) {
            System.out.println("Level " + level + ": "
                + current.getDisplayName()
                + " [" + current.getBeanDefinitionCount() + " beans]");
            current = current.getParent();
            level++;
        }
    }
}
```

```
출력 예시 (Spring MVC 레거시):
Level 0: WebApplicationContext for namespace 'dispatcher-servlet'  [45 beans]
Level 1: Root WebApplicationContext                                 [120 beans]
```

---

## 💻 실험으로 확인하기

### 실험 1: 부모-자식 탐색 방향 확인

```java
@Configuration
class ParentConfig {
    @Bean
    public String parentBean() { return "parent"; }
}

@Configuration
class ChildConfig {
    @Bean
    public String childBean() { return "child"; }
}

public class HierarchyDirectionTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext parent =
            new AnnotationConfigApplicationContext(ParentConfig.class);

        AnnotationConfigApplicationContext child =
            new AnnotationConfigApplicationContext();
        child.setParent(parent);
        child.register(ChildConfig.class);
        child.refresh();

        // 자식 → 부모 탐색 (O)
        System.out.println(child.getBean("parentBean"));  // "parent"
        System.out.println(child.getBean("childBean"));   // "child"

        // 현재 컨텍스트에만 있는지 확인
        System.out.println(child.containsLocalBean("parentBean"));  // false
        System.out.println(child.containsBean("parentBean"));       // true (부모 포함)

        // 부모 → 자식 탐색 (X)
        System.out.println(parent.containsBean("childBean"));       // false
    }
}
```

```
출력:
parent
child
false   ← 자식 컨텍스트 로컬에는 없음
true    ← 부모까지 포함하면 있음
false   ← 부모는 자식 Bean 모름
```

### 실험 2: 자식에서 부모 Bean 재정의

```java
@Configuration
class ParentConf {
    @Bean
    public String greeting() { return "Hello from Parent"; }
}

@Configuration
class ChildConf {
    @Bean
    public String greeting() { return "Hello from Child"; }  // 같은 이름
}

public class OverrideTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext parent =
            new AnnotationConfigApplicationContext(ParentConf.class);

        AnnotationConfigApplicationContext child =
            new AnnotationConfigApplicationContext();
        child.setParent(parent);
        child.register(ChildConf.class);
        child.refresh();

        System.out.println(child.getBean("greeting"));   // "Hello from Child" (자식 우선)
        System.out.println(parent.getBean("greeting"));  // "Hello from Parent" (부모 불변)
    }
}
```

```
출력:
Hello from Child    ← 자식이 부모를 가림 (Shadow)
Hello from Parent   ← 부모 컨텍스트는 영향 없음
```

---

## 🤔 트레이드오프

```
계층 구조 사용 시:
  장점
    레이어 간 의존 방향 강제 (Service → Controller 불가)
    공통 Bean 공유 (여러 Servlet이 같은 DataSource 사용)
    독립 테스트 용이 (Root Context만 로드)
  단점
    설정 복잡도 증가
    어느 컨텍스트에 Bean을 넣어야 하는지 혼동 위험
    같은 이름 Bean 재정의로 디버깅 어려움

Spring Boot 단일 컨텍스트:
  장점
    간단, 실수 적음
    @SpringBootTest 테스트 편리
  단점
    레이어 간 의존 방향 컴파일러/컨테이너가 강제 안 함
    → 코드 리뷰와 아키텍처 규칙으로 보완 필요

containsBean vs containsLocalBean:
  containsBean("name")      → 부모 포함 탐색
  containsLocalBean("name") → 현재 컨텍스트만
  → 계층 구조에서 Bean 존재 여부 판단 시 의도 명확히
```

---

## 📌 핵심 정리

```
계층 구조 기본 규칙
  자식 → 부모 탐색 가능
  부모 → 자식 탐색 불가
  자식 Bean이 부모 동명 Bean을 가림 (Shadow)

HierarchicalBeanFactory
  getParentBeanFactory() → 부모 BeanFactory 반환
  containsLocalBean()    → 현재 컨텍스트만 탐색
  containsBean()         → 부모 포함 탐색

Spring MVC 레거시 구조
  Root Context (ContextLoaderListener)
    → Service, Repository, DataSource
  Servlet Context (DispatcherServlet)
    → Controller, ViewResolver
    → 부모 = Root Context

탐색 위임
  doGetBean() → containsBeanDefinition() → false
  → parentBeanFactory.getBean() 재귀 위임
  → 부모에도 없으면 NoSuchBeanDefinitionException

Spring Boot
  단일 AnnotationConfigApplicationContext
  계층 구조 대신 패키지/어노테이션으로 레이어 분리
  @WebMvcTest 등 슬라이스 테스트로 부분 로드 지원
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `child.getBean("dataSource")`는 어떤 `DataSource`를 반환하는가? `containsLocalBean`과 `containsBean`의 결과도 예측하라.

```java
@Configuration
class ParentConf {
    @Bean DataSource dataSource() { return new HikariDataSource(); }
}

@Configuration
class ChildConf {
    // dataSource Bean 없음
    @Bean UserService userService() { return new UserService(); }
}

// parent, child 계층 구조 설정 후
child.getBean("dataSource");
child.containsLocalBean("dataSource");
child.containsBean("dataSource");
```

**Q2.** Spring MVC에서 `@Controller` Bean을 Root Context에 등록하면 `@Autowired`로 `@Service` Bean을 주입받을 수 있는가? 그 이유를 계층 구조 탐색 방향으로 설명하라.

**Q3.** Spring Boot에서 계층 구조를 사용하지 않을 때 발생할 수 있는 아키텍처 문제와, 이를 코드 수준에서 보완하는 방법을 제시하라.

> 💡 **해설**
>
> **Q1.** `child.getBean("dataSource")`는 Parent의 `HikariDataSource`를 반환한다. Child 컨텍스트에는 `dataSource` BeanDefinition이 없으므로 `doGetBean()`이 부모에게 위임하고, 부모에서 찾아 반환한다. `child.containsLocalBean("dataSource")` → `false` (Child 로컬에 없음). `child.containsBean("dataSource")` → `true` (부모 포함 탐색하면 있음).
>
> **Q2.** `@Controller`를 Root Context에 등록하면, Root Context의 `getBean("userService")` 탐색은 Root 자체에서만 이루어진다. Root Context는 Servlet Context(자식)를 모르므로, Servlet Context에 있는 `@Service`를 찾을 수 없어 `NoSuchBeanDefinitionException`이 발생한다. 반대로 `@Service`를 Servlet Context에 등록하면, Controller가 Service를 조회할 때 Servlet Context → 없음 → Root Context 위임 → 없음 → 예외가 발생한다. 올바른 방법은 `@Service`는 Root Context, `@Controller`는 Servlet Context에 각각 등록하는 것이다.
>
> **Q3.** 단일 컨텍스트에서 모든 Bean이 서로를 `@Autowired`할 수 있으므로, `@Controller`가 다른 `@Controller`를, `@Service`가 `@Controller`를 주입받는 등 레이어 역전 의존이 컴파일/런타임에서 막히지 않는다. 보완 방법으로는 ArchUnit 라이브러리를 사용해 레이어 의존 방향을 테스트 코드로 강제하거나, Checkstyle / custom annotation processor로 정적 분석을 추가하거나, 명시적인 패키지 구조 (`controller`, `service`, `repository`)와 코드 리뷰 규칙으로 팀 내 규율을 유지하는 방법이 있다.

---

<div align="center">

**[⬅️ 이전: BeanPostProcessor 체인](./03-beanpostprocessor-chain.md)** | **[다음: Environment & PropertySource 우선순위 ➡️](./05-environment-propertysource.md)**

</div>

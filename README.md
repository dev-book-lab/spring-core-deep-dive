<div align="center">

# 🌱 Spring Core Deep Dive

**"스프링 컨테이너가 Bean을 관리하는 모든 비밀"**

<br/>

> *"스프링을 사용하는 것과, 스프링이 어떻게 작동하는지 아는 것은 다르다"*

IoC 컨테이너 구조부터 AOP 바이트코드, 트랜잭션 프록시, SpEL 파싱까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring](https://img.shields.io/badge/Spring-6.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io)
[![Docs](https://img.shields.io/badge/Docs-50개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@Transactional`을 붙이면 트랜잭션이 시작됩니다" | CGLIB 프록시가 생성되고 `TransactionInterceptor`가 Advice 체인에 끼어드는 과정 |
| "`@Autowired`로 의존성을 주입받습니다" | `AutowiredAnnotationBeanPostProcessor`가 리플렉션으로 필드를 채우는 내부 코드 |
| "순환 참조는 에러가 납니다" | 3단계 캐시(`singletonObjects` / `earlySingletonObjects` / `singletonFactories`)로 해결되는 메커니즘 |
| "`private` 메서드에는 AOP가 안 됩니다" | JDK Proxy와 CGLIB의 바이트코드 차이, 왜 오버라이딩 기반인가 |
| "`@Configuration`은 `@Component`와 비슷합니다" | CGLIB 프록시가 씌워져 `@Bean` 메서드 호출을 가로채는 Full Mode vs Lite Mode |
| 이론 나열 | 실행 가능한 코드 + Spring 소스코드 직접 추적 + `javap` 바이트코드 분석 |

---

## 🔗 연관 레포지토리

이 레포는 **Dev Book Lab** 시리즈의 일부입니다.

| 레포 | 연결고리 |
|------|---------|
| [JVM Deep Dive](https://github.com/dev-book-lab/jvm-deep-dive) | Spring이 JVM 위에서 동작하는 원리 — CGLIB가 바이트코드를 조작하는 방식, 프록시 객체의 메모리 구조 |
| [Unit Testing Patterns](https://github.com/dev-book-lab/unit-testing-patterns) | Spring Bean 테스트 전략 — `@SpringBootTest` vs `@WebMvcTest`, Mock Bean 주입 원리 |
| [Design Patterns](https://github.com/dev-book-lab/design-patterns) | Spring이 사용하는 패턴들 — Template Method(JdbcTemplate), Proxy(AOP), Factory(BeanFactory) |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![IoC Container](https://img.shields.io/badge/🔹_IoC_Container-BeanFactory_vs_ApplicationContext-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./01-ioc-container/01-beanfactory-vs-applicationcontext.md)
[![DI Internals](https://img.shields.io/badge/🔹_DI_Internals-생성자_vs_필드_주입_바이트코드-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./02-dependency-injection/01-constructor-vs-field-injection-bytecode.md)
[![Bean Lifecycle](https://img.shields.io/badge/🔹_Bean_Lifecycle-Bean_생성_전체_과정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./03-bean-lifecycle/01-bean-creation-process.md)
[![AOP](https://img.shields.io/badge/🔹_AOP-JDK_Proxy_vs_CGLIB-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./04-aop/01-jdk-proxy-vs-cglib.md)
[![Component Scan](https://img.shields.io/badge/🔹_Component_Scan-@ComponentScan_동작_원리-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./05-component-scanning/01-componentscan-internals.md)
[![Configuration](https://img.shields.io/badge/🔹_Configuration-@Configuration_vs_@Component-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./06-configuration/01-configuration-vs-component.md)
[![Events](https://img.shields.io/badge/🔹_Events-ApplicationEvent_발행_구독-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./07-events/01-application-event-mechanism.md)
[![SpEL](https://img.shields.io/badge/🔹_SpEL-Expression_파싱_과정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./08-spel/01-spel-parsing-process.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: IoC Container Architecture

> **핵심 질문:** 스프링 컨테이너는 어떻게 Bean을 찾고, 만들고, 관리하는가?

<details>
<summary><b>BeanFactory부터 ApplicationContext까지, 컨테이너 내부 해부 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. BeanFactory vs ApplicationContext](./01-ioc-container/01-beanfactory-vs-applicationcontext.md) | 인터페이스 계층 구조, `getBean()` 내부 동작, Lazy vs Eager 초기화 결정 지점 |
| [02. BeanDefinition과 Bean 메타데이터](./01-ioc-container/02-bean-definition-metadata.md) | `BeanDefinition` 인터페이스 필드 전체, XML / 어노테이션 / Java Config → BeanDefinition 변환 과정 |
| [03. BeanPostProcessor 체인](./01-ioc-container/03-beanpostprocessor-chain.md) | `postProcessBeforeInitialization` / `After` 호출 순서, `@Autowired`와 `@Transactional`이 이 체인에서 처리되는 방식 |
| [04. ApplicationContext 계층 구조](./01-ioc-container/04-applicationcontext-hierarchy.md) | Parent-Child 컨텍스트 분리 이유, Spring MVC의 Root / Servlet Context 구조 |
| [05. Environment & PropertySource 우선순위](./01-ioc-container/05-environment-propertysource.md) | `PropertySource` 체인, OS 환경변수 / JVM 프로퍼티 / `application.yml` 우선순위 결정 메커니즘 |
| [06. Resource Abstraction 패턴](./01-ioc-container/06-resource-abstraction.md) | `Resource` 인터페이스 계층, `classpath:` / `file:` / `http:` prefix 처리, `ResourceLoader` 전략 패턴 |

</details>

<br/>

### 🔹 Chapter 2: Dependency Injection Internals

> **핵심 질문:** `@Autowired`는 어떻게 타입을 찾고, 어떻게 주입하는가?

<details>
<summary><b>주입 방식의 차이와 의존성 해결 메커니즘 완전 분석 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 생성자 vs 필드 vs 세터 주입 바이트코드 비교](./02-dependency-injection/01-constructor-vs-field-injection-bytecode.md) | `javap`로 생성자 주입 vs 리플렉션 필드 주입 바이트코드 차이 실측, 불변성과 테스트 용이성의 기술적 근거 |
| [02. 순환 참조 해결 — 3단계 캐시](./02-dependency-injection/02-circular-dependency-3cache.md) | `singletonObjects` / `earlySingletonObjects` / `singletonFactories` 각 캐시의 역할, 생성자 주입에서 왜 해결이 안 되는가 |
| [03. @Autowired vs @Inject vs @Resource](./02-dependency-injection/03-autowired-inject-resource.md) | JSR-330 vs Spring 어노테이션, 탐색 순서(타입 → 이름) 차이, `AutowiredAnnotationBeanPostProcessor` 처리 경로 |
| [04. @Qualifier & @Primary 우선순위](./02-dependency-injection/04-qualifier-primary-priority.md) | 동일 타입 Bean 다수 존재 시 해결 알고리즘, `@Qualifier` → `@Primary` → Bean 이름 폴백 순서 |
| [05. Optional Dependency 처리](./02-dependency-injection/05-optional-dependency.md) | `required=false` / `Optional<T>` / `@Nullable` / `ObjectProvider<T>` 각각의 내부 처리 차이 |
| [06. Constructor Binding의 장점](./02-dependency-injection/06-constructor-binding.md) | 불변 객체 보장 원리, `@ConfigurationProperties`에서 생성자 바인딩이 권장되는 이유 |
| [07. Lazy Initialization 동작 원리](./02-dependency-injection/07-lazy-initialization.md) | `@Lazy` 어노테이션이 만드는 Proxy, `DefaultListableBeanFactory`의 Lazy 처리 경로 |

</details>

<br/>

### 🔹 Chapter 3: Bean Lifecycle Deep Dive

> **핵심 질문:** `new MyBean()`이 아닌 스프링이 Bean을 만들 때, 정확히 어떤 순서로 무슨 일이 벌어지는가?

<details>
<summary><b>Instantiation부터 Destruction까지, Bean의 일생 전체 추적 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Bean 생성 전체 과정](./03-bean-lifecycle/01-bean-creation-process.md) | Instantiation → Populate Properties → Aware 체인 → BeanPostProcessor → Init Method 전체 흐름, `AbstractAutowireCapableBeanFactory` 소스 추적 |
| [02. @PostConstruct vs InitializingBean 실행 순서](./03-bean-lifecycle/02-postconstruct-vs-initializingbean.md) | `@PostConstruct`(JSR-250) vs `InitializingBean.afterPropertiesSet()` vs `init-method` 실행 순서와 이유 |
| [03. @PreDestroy vs DisposableBean 정리 과정](./03-bean-lifecycle/03-predestroy-vs-disposablebean.md) | 컨테이너 종료 시 소멸 콜백 순서, `ShutdownHook` 등록 원리, 소멸이 보장되지 않는 경우 |
| [04. Aware 인터페이스 체인](./03-bean-lifecycle/04-aware-interfaces.md) | `BeanNameAware` / `BeanFactoryAware` / `ApplicationContextAware` 호출 시점, 컨테이너 인프라 접근의 트레이드오프 |
| [05. Bean Scope와 프록시](./03-bean-lifecycle/05-bean-scope-and-proxy.md) | Singleton / Prototype / Request / Session Scope 생명주기 차이, Prototype을 Singleton에 주입할 때의 함정과 Scope Proxy 해결 원리 |
| [06. FactoryBean vs ObjectProvider](./03-bean-lifecycle/06-factorybean-vs-objectprovider.md) | `FactoryBean<T>` 패턴의 존재 이유, `&`로 FactoryBean 자체를 가져오는 방법, `ObjectProvider`로 지연/선택적 조회 |
| [07. 순환 의존 내부 해결 과정](./03-bean-lifecycle/07-circular-dependency-resolution.md) | 3단계 캐시 + `ObjectFactory` 조합으로 순환 참조가 해결되는 단계별 시나리오 코드 추적 |

</details>

<br/>

### 🔹 Chapter 4: AOP Implementation

> **핵심 질문:** `@Transactional`은 어떻게 트랜잭션을 시작하고 커밋하는가? 왜 `private`에서는 안 되는가?

<details>
<summary><b>JDK Proxy부터 CGLIB까지, 프록시 기반 AOP의 비밀 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. JDK Dynamic Proxy vs CGLIB 바이트코드 비교](./04-aop/01-jdk-proxy-vs-cglib.md) | `Proxy.newProxyInstance()` vs ASM 바이트코드 생성, `javap`로 두 프록시의 클래스 구조 비교 |
| [02. ProxyFactoryBean 내부 구조](./04-aop/02-proxyfactorybean-internals.md) | `ProxyFactoryBean` → `ProxyFactory` → `AopProxy` 생성 체인, `AdvisedSupport` 메타데이터 |
| [03. @AspectJ 어노테이션 처리 과정](./04-aop/03-aspectj-annotation-processing.md) | `AnnotationAwareAspectJAutoProxyCreator`가 `@Aspect`를 `Advisor`로 변환하는 과정 |
| [04. Pointcut Expression 파싱과 매칭](./04-aop/04-pointcut-expression.md) | `execution()` / `within()` / `@annotation()` 파싱 원리, `AspectJExpressionPointcut` 내부 매처 |
| [05. Advice 체인 실행 순서](./04-aop/05-advice-chain-execution.md) | `Around` → `Before` → `AfterReturning` / `AfterThrowing` 실행 순서, `ReflectiveMethodInvocation.proceed()` 재귀 호출 구조 |
| [06. @Transactional이 프록시인 이유](./04-aop/06-transactional-proxy-mechanism.md) | `TransactionInterceptor`가 Advice 체인에 들어가는 과정, `PlatformTransactionManager` 위임 구조 |
| [07. private 메서드에 AOP가 안 되는 이유](./04-aop/07-why-private-aop-fails.md) | 오버라이딩 불가 → 프록시 적용 불가의 바이트코드 수준 설명, 같은 클래스 내부 메서드 호출(Self-Invocation) 함정 |
| [08. Proxy 성능 비교](./04-aop/08-proxy-performance.md) | JDK Proxy vs CGLIB vs AspectJ Weaving 성능 실측 (`JMH`), Spring Boot 기본값이 CGLIB인 이유 |

</details>

<br/>

### 🔹 Chapter 5: Component Scanning

> **핵심 질문:** `@ComponentScan`은 어떻게 수백 개의 클래스 중 Bean 후보를 찾아내는가?

<details>
<summary><b>클래스패스 스캔부터 BeanDefinition 등록까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. @ComponentScan 동작 원리](./05-component-scanning/01-componentscan-internals.md) | `ClassPathBeanDefinitionScanner` 스캔 흐름, ASM 기반 메타데이터 리더로 클래스 로딩 없이 어노테이션 탐색하는 방법 |
| [02. ClassPathScanningCandidateComponentProvider](./05-component-scanning/02-candidate-component-provider.md) | 후보 컴포넌트 필터링 알고리즘, `@Component` 계층 어노테이션(`@Service`, `@Repository`) 인식 방식 |
| [03. TypeFilter Include & Exclude](./05-component-scanning/03-typefilter-include-exclude.md) | `includeFilters` / `excludeFilters` 적용 순서, `AssignableTypeFilter` / `AnnotationTypeFilter` / Custom Filter 구현 |
| [04. @Conditional 평가 과정](./05-component-scanning/04-conditional-evaluation.md) | `ConditionEvaluator`가 `@Conditional`을 평가하는 시점과 순서, `@ConditionalOnClass` 내부 구현 |
| [05. BeanDefinition 등록 과정](./05-component-scanning/05-beandefinition-registration.md) | 스캔 결과가 `BeanDefinitionRegistry`에 등록되는 과정, 중복 등록 충돌 해결 규칙 |
| [06. 컴포넌트 인덱스를 통한 스캔 최적화](./05-component-scanning/06-component-index-optimization.md) | `spring.components` 인덱스 파일, `@Indexed` 어노테이션, 대규모 프로젝트에서 스캔 시간을 줄이는 원리 |

</details>

<br/>

### 🔹 Chapter 6: Configuration Classes

> **핵심 질문:** `@Configuration`이 `@Component`와 다른 이유는 무엇인가? CGLIB는 왜 필요한가?

<details>
<summary><b>Java Config의 내부 구현과 CGLIB 프록시의 역할 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. @Configuration vs @Component](./06-configuration/01-configuration-vs-component.md) | Full Mode vs Lite Mode 차이, `@Bean` 메서드를 직접 호출하면 생기는 일 |
| [02. CGLIB 프록시가 적용되는 이유](./06-configuration/02-cglib-proxy-on-configuration.md) | `ConfigurationClassEnhancer`가 CGLIB 서브클래스를 만드는 과정, `$$EnhancerBySpringCGLIB` 클래스 분석 |
| [03. @Bean 메서드 호출 가로채기](./06-configuration/03-bean-method-interception.md) | `BeanMethodInterceptor`가 이미 등록된 Bean을 반환하는 메커니즘, Singleton 보장이 이루어지는 지점 |
| [04. Lite Mode vs Full Mode 트레이드오프](./06-configuration/04-lite-mode-vs-full-mode.md) | CGLIB 없는 Lite Mode의 성능상 이점과 Singleton 보장 불가 트레이드오프, 선택 기준 |
| [05. @Import의 3가지 방식](./06-configuration/05-import-three-ways.md) | 일반 클래스 Import / `ImportSelector` / `ImportBeanDefinitionRegistrar` 각각의 처리 시점과 활용 사례 |
| [06. ImportSelector & ImportBeanDefinitionRegistrar](./06-configuration/06-importselector-registrar.md) | `AutoConfigurationImportSelector`(Spring Boot 자동 설정)가 동작하는 원리, 조건부 Bean 등록 패턴 |

</details>

<br/>

### 🔹 Chapter 7: Event & Listener

> **핵심 질문:** `ApplicationEvent`는 어떻게 발행되고, 리스너는 어떻게 호출되는가?

<details>
<summary><b>스프링 이벤트 시스템의 발행-구독 메커니즘 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. ApplicationEvent 발행-구독 메커니즘](./07-events/01-application-event-mechanism.md) | `ApplicationEventMulticaster` 내부 구조, `publishEvent()` 호출 시 리스너 탐색 및 호출 흐름 |
| [02. @EventListener vs ApplicationListener](./07-events/02-eventlistener-vs-applicationlistener.md) | `EventListenerMethodProcessor`가 `@EventListener` 메서드를 `ApplicationListener`로 변환하는 과정 |
| [03. 비동기 이벤트 처리 @Async](./07-events/03-async-event-processing.md) | `@Async`와 결합된 비동기 이벤트의 스레드 분리 원리, `SimpleApplicationEventMulticaster` 스레드 풀 설정 |
| [04. 이벤트 실행 순서 보장](./07-events/04-event-ordering.md) | `@Order` / `Ordered` 인터페이스로 리스너 순서 제어, 순서가 보장되지 않는 경우 |
| [05. 트랜잭션 바운드 이벤트](./07-events/05-transaction-bound-events.md) | `@TransactionalEventListener`의 Phase(`AFTER_COMMIT` 등) 동작 원리, 트랜잭션 컨텍스트에서 이벤트 발행의 함정 |

</details>

<br/>

### 🔹 Chapter 8: SpEL & Type Conversion

> **핵심 질문:** `@Value("#{...}")`는 어떻게 표현식을 평가하는가?

<details>
<summary><b>Spring Expression Language와 타입 변환 시스템 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. SpEL 파싱 과정](./08-spel/01-spel-parsing-process.md) | `ExpressionParser` → `Expression` → `EvaluationContext` 구조, Tokenizer / AST 파싱 단계 |
| [02. @Value("${...}") vs @Value("#{...}")](./08-spel/02-value-property-vs-spel.md) | Property Placeholder와 SpEL의 처리 시점 차이, `PropertySourcesPlaceholderConfigurer` vs `ExpressionEvaluator` |
| [03. PropertyEditor vs Converter 변환](./08-spel/03-propertyeditor-vs-converter.md) | JavaBeans `PropertyEditor`(구)와 Spring `Converter<S,T>`(신) 설계 차이, 타입 변환 순서 |
| [04. ConversionService 등록과 우선순위](./08-spel/04-conversionservice-registration.md) | `DefaultConversionService` 내장 Converter 목록, 커스텀 Converter 등록 방법과 우선순위 결정 |
| [05. Custom Type Converter 작성](./08-spel/05-custom-type-converter.md) | `Converter<S,T>` / `GenericConverter` / `ConverterFactory` 세 가지 구현 방식과 사용 시점 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "@Autowired는 어떻게 동작하나?" — 스프링 입문자 / 면접 준비 (2주)</b></summary>

<br/>

**Week 1 — 컨테이너와 의존성 주입 원리**
```
01 BeanFactory vs ApplicationContext
01 Bean 생성 전체 과정
01 생성자 vs 필드 vs 세터 주입 바이트코드 비교
02 순환 참조 해결 — 3단계 캐시
```

**Week 2 — AOP와 트랜잭션의 비밀**
```
01 JDK Proxy vs CGLIB 바이트코드 비교
06 @Transactional이 프록시인 이유
07 private 메서드에 AOP가 안 되는 이유
01 @Configuration vs @Component
```

</details>

<details>
<summary><b>🔵 Spring 내부를 원리로 이해하고 싶은 개발자 (6주)</b></summary>

<br/>

```
Chapter 1 전체 (IoC 컨테이너 구조)
→ Chapter 2 전체 (DI 메커니즘)
→ Chapter 3 전체 (Bean 생명주기)
→ Chapter 4 전체 (AOP + 직접 바이트코드 분석)
→ Chapter 5, 6 (컴포넌트 스캔 + Java Config)
```

</details>

<details>
<summary><b>🔴 Spring 소스코드 레벨 이해 목표 (3개월)</b></summary>

<br/>

```
전체 순서대로 + 각 문서의 실험 코드 직접 실행
→ Spring Framework GitHub에서 연관 소스 직접 추적
→ Chapter 7, 8로 이벤트 / SpEL까지 완성
→ 직접 BeanPostProcessor / ImportBeanDefinitionRegistrar 구현해보기
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이게 존재하는가** | 문제 상황과 설계 배경 |
| 😱 **흔한 오해 또는 잘못된 사용** | Before — 많은 개발자가 틀리는 방식 |
| ✨ **올바른 이해와 사용** | After — 원리를 알고 난 후의 올바른 접근 |
| 🔬 **내부 동작 원리** | Spring 소스코드 직접 추적 + 다이어그램 |
| 💻 **실험으로 확인하기** | 직접 실행 가능한 코드 + 예상 결과 |
| 🤔 **트레이드오프** | 이 설계의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🙏 Reference

- [Spring Framework Reference Documentation](https://docs.spring.io/spring-framework/reference/)
- [Spring Framework Source Code (GitHub)](https://github.com/spring-projects/spring-framework)
- [Spring in Action — Craig Walls](https://www.manning.com/books/spring-in-action-sixth-edition)
- [Expert Spring MVC and Web Flow — Seth Ladd](https://link.springer.com/book/10.1007/978-1-4302-0024-7)
- [JEP Index (연관 JDK 기능)](https://openjdk.org/jeps/0)
- [Baeldung Spring Guides](https://www.baeldung.com/spring-tutorial)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"스프링을 사용하는 것과, 스프링이 어떻게 작동하는지 아는 것은 다르다"*

</div>

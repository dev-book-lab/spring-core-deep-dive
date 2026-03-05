# @Import의 3가지 방식 — 일반 클래스 / ImportSelector / ImportBeanDefinitionRegistrar

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Import`의 3가지 방식이 내부에서 처리되는 시점은 각각 언제인가?
- `ImportSelector`와 `DeferredImportSelector`의 평가 순서 차이는?
- `ImportBeanDefinitionRegistrar`가 `ImportSelector`보다 더 강력한 이유는?
- `@EnableXxx` 패턴이 `@Import`를 어떻게 활용하는가?
- `ConfigurationClassParser`가 세 방식을 처리하는 코드 경로는?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean 등록을 모듈화하고 조건부로 활성화해야 한다

```java
// 사용자 코드 — "트랜잭션 관리 활성화"
@Configuration
@EnableTransactionManagement  // ← @Import 내부 사용
public class AppConfig {}

// @EnableTransactionManagement 정의
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)  // ← ImportSelector
public @interface EnableTransactionManagement { ... }
```

```
@Import가 해결하는 것:
  @ComponentScan 없이 특정 클래스/설정을 명시적으로 가져오기
  → 모듈화: 기능 단위로 @Configuration 분리
  → 조건부 활성화: @EnableXxx 패턴
  → 동적 등록: 런타임 조건에 따라 다른 Bean 등록

3가지 방식:
  방식 1: 일반 클래스 (@Configuration, @Component 등)
  방식 2: ImportSelector — 조건에 따라 다른 클래스 선택
  방식 3: ImportBeanDefinitionRegistrar — BeanDefinition 직접 생성/등록
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Import는 항상 즉시 처리된다

```
❌ 잘못된 이해:
  "@Import → 그 자리에서 즉시 처리"

✅ 실제:
  일반 클래스 @Import → 즉시 처리 (현재 파싱 라운드)
  ImportSelector    → 즉시 처리 (현재 파싱 라운드)
  DeferredImportSelector → 지연 처리 (모든 @Configuration 파싱 완료 후)
  ImportBeanDefinitionRegistrar → 가장 나중 처리 (BeanDefinition 로딩 단계)

순서가 중요한 이유:
  DeferredImportSelector: 다른 @Configuration의 Bean 존재 여부 파악 후 결정 가능
  → Spring Boot 자동 설정(AutoConfigurationImportSelector)이 여기에 해당
```

### Before: ImportSelector는 Bean을 직접 등록한다

```
❌ 잘못된 이해:
  "ImportSelector → 반환된 클래스들이 바로 Bean으로 등록"

✅ 실제:
  ImportSelector → 클래스 이름 문자열 배열 반환
  → ConfigurationClassParser가 반환된 클래스들을 다시 파싱 대상으로 추가
  → 파싱 후 BeanDefinition으로 등록
  → 반환 클래스가 @Configuration이면 그 안의 @Bean도 처리됨

  ImportBeanDefinitionRegistrar → BeanDefinition을 직접 레지스트리에 추가
  → 파싱 단계 없이 바로 등록
```

---

## ✨ 올바른 이해와 사용

### After: 3가지 방식의 처리 시점과 역할 비교

```
방식 1: 일반 클래스 Import
  @Import(SomeConfig.class)
  처리 시점: configureClass() 파싱 중 즉시
  역할: 해당 클래스를 @Configuration처럼 파싱
  사용: 명시적 설정 클래스 추가

방식 2: ImportSelector
  @Import(MySelector.class)  // MySelector implements ImportSelector
  처리 시점: 파싱 중 즉시 (DeferredImportSelector는 마지막)
  역할: selectImports() → 클래스명 배열 반환 → 해당 클래스 파싱
  사용: 조건에 따라 다른 설정 선택 (@EnableXxx)

방식 3: ImportBeanDefinitionRegistrar
  @Import(MyRegistrar.class)  // MyRegistrar implements ImportBeanDefinitionRegistrar
  처리 시점: BeanDefinition 로딩 단계 (파싱 완료 후)
  역할: registerBeanDefinitions() → 직접 BeanDefinition 생성/등록
  사용: 커스텀 BeanDefinition 생성, 동적 Bean 등록
```

---

## 🔬 내부 동작 원리

### 1. ConfigurationClassParser.processImports() — 분기 처리

```java
// ConfigurationClassParser.processImports()
private void processImports(ConfigurationClass configClass,
                              SourceClass currentSourceClass,
                              Collection<SourceClass> importCandidates, ...) {

    for (SourceClass candidate : importCandidates) {

        // ① ImportSelector 구현체
        if (candidate.isAssignableTo(ImportSelector.class)) {
            Class<?> candidateClass = candidate.loadClass();
            ImportSelector selector = ParserStrategyUtils.instantiateClass(
                candidateClass, ImportSelector.class, ...);

            // DeferredImportSelector → 지연 처리 목록에 추가
            if (selector instanceof DeferredImportSelector dis) {
                this.deferredImportSelectorHandler.handle(configClass, dis);
            } else {
                // 즉시 처리: selectImports() 호출
                String[] importClassNames = selector.selectImports(
                    currentSourceClass.getMetadata());
                // 반환된 클래스명 배열을 재귀적으로 processImports()
                Collection<SourceClass> importSourceClasses =
                    asSourceClasses(importClassNames, ...);
                processImports(configClass, currentSourceClass, importSourceClasses, ...);
            }
        }

        // ② ImportBeanDefinitionRegistrar 구현체
        else if (candidate.isAssignableTo(ImportBeanDefinitionRegistrar.class)) {
            Class<?> candidateClass = candidate.loadClass();
            ImportBeanDefinitionRegistrar registrar =
                ParserStrategyUtils.instantiateClass(candidateClass,
                    ImportBeanDefinitionRegistrar.class, ...);
            // 즉시 처리하지 않고 configClass에 저장 → 나중에 loadBeanDefinitions()에서 처리
            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
        }

        // ③ 일반 클래스 (@Configuration, @Component 등)
        else {
            this.importStack.registerImport(currentSourceClass.getMetadata(),
                candidate.getMetadata().getClassName());
            // 일반 @Configuration처럼 파싱
            processConfigurationClass(candidate.asConfigClass(configClass), ...);
        }
    }
}
```

### 2. 방식 1: 일반 클래스 Import

```java
// 일반 클래스 Import — 가장 단순
@Configuration
@Import(SecurityConfig.class)  // SecurityConfig를 가져옴
public class AppConfig {}

@Configuration
public class SecurityConfig {
    @Bean
    public SecurityManager securityManager() { ... }
}
```

```
처리 흐름:
  AppConfig 파싱 → @Import(SecurityConfig.class) 발견
  → processImports() → 일반 클래스 분기
  → processConfigurationClass(SecurityConfig) 재귀 호출
  → SecurityConfig 내부 @Bean 메서드 처리
  → securityManager BeanDefinition 등록

특징:
  @ComponentScan 없어도 특정 설정을 명시적으로 포함
  SecurityConfig에 @Configuration 없어도 됨 (Import되면 파싱 대상)
  단, @Configuration이 없으면 @Bean 메서드 처리는 Lite Mode로
```

### 3. 방식 2: ImportSelector

```java
// ImportSelector 구현
public class TransactionManagementConfigurationSelector
        implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // @EnableTransactionManagement의 속성 읽기
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(
            importingClassMetadata.getAnnotationAttributes(
                EnableTransactionManagement.class.getName()));
        AdviceMode mode = attrs.getEnum("mode");

        return switch (mode) {
            case PROXY    -> new String[]{
                AutoProxyRegistrar.class.getName(),
                ProxyTransactionManagementConfiguration.class.getName()
            };
            case ASPECTJ  -> new String[]{
                TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME
            };
        };
    }
}
```

```java
// 사용 측
@Configuration
@EnableTransactionManagement(mode = AdviceMode.PROXY)
public class AppConfig {}

// 처리 흐름:
// @EnableTransactionManagement → @Import(TransactionManagementConfigurationSelector)
// → processImports() → ImportSelector 분기
// → selectImports(AppConfig 메타데이터) 호출
// → ["AutoProxyRegistrar", "ProxyTransactionManagementConfiguration"] 반환
// → 두 클래스 재귀 파싱
// → AutoProxyRegistrar → @ComponentScan 없이 AOP 인프라 등록
// → ProxyTransactionManagementConfiguration → @Bean TransactionInterceptor 등록
```

### 4. DeferredImportSelector — 지연 평가

```java
// DeferredImportSelector extends ImportSelector
// → 모든 @Configuration 파싱 완료 후 처리

public interface DeferredImportSelector extends ImportSelector {

    // 그룹화 지원 (Spring Boot 자동 설정에서 활용)
    @Nullable
    default Class<? extends Group> getImportGroup() { return null; }

    interface Group {
        void process(AnnotationMetadata metadata, DeferredImportSelector selector);
        Iterable<Entry> selectImports();
    }
}
```

```java
// DeferredImportSelectorHandler.process() 처리 시점
// → ConfigurationClassParser.parse() 완료 후 호출

void process() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    this.deferredImportSelectors = null;

    // Group별로 정렬 후 처리
    DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
    deferredImports.stream()
        .sorted(DEFERRED_IMPORT_COMPARATOR)
        .forEach(handler::register);
    handler.processGroupImports();
}

// 중요: 이 시점에 다른 @Configuration의 BeanDefinition이 모두 등록된 상태
// → @ConditionalOnMissingBean 등 조건 판단 가능
```

### 5. 방식 3: ImportBeanDefinitionRegistrar

```java
// ImportBeanDefinitionRegistrar — 가장 강력한 방식
public interface ImportBeanDefinitionRegistrar {

    default void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry,
            BeanNameGenerator importBeanNameGenerator) {
        registerBeanDefinitions(importingClassMetadata, registry);
    }
}

// 구현 예: @EnableAspectJAutoProxy 내부
public class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                         BeanDefinitionRegistry registry) {
        // AopConfigUtils: AnnotationAwareAspectJAutoProxyCreator BeanDefinition 직접 등록
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata,
                EnableAspectJAutoProxy.class);

        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}
```

```java
// 처리 시점: ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass()
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, ...) {

    // ... @Bean 메서드 처리 ...

    // ImportBeanDefinitionRegistrar 처리 (가장 나중)
    for (Map.Entry<ImportBeanDefinitionRegistrar, AnnotationMetadata> entry :
            configClass.getImportBeanDefinitionRegistrars().entrySet()) {
        entry.getKey().registerBeanDefinitions(entry.getValue(), this.registry);
    }
}
```

### 6. @Enable 패턴 전체 구조

```
@EnableTransactionManagement
  └── @Import(TransactionManagementConfigurationSelector)  ← ImportSelector
        → selectImports() → [AutoProxyRegistrar, ProxyTransactionManagementConfiguration]
          → AutoProxyRegistrar implements ImportBeanDefinitionRegistrar
            → InfrastructureAdvisorAutoProxyCreator BeanDefinition 직접 등록
          → ProxyTransactionManagementConfiguration (@Configuration)
            → @Bean TransactionInterceptor, BeanFactoryTransactionAttributeSourceAdvisor

@EnableAspectJAutoProxy
  └── @Import(AspectJAutoProxyRegistrar)  ← ImportBeanDefinitionRegistrar
        → AnnotationAwareAspectJAutoProxyCreator BeanDefinition 직접 등록

@EnableScheduling
  └── @Import(SchedulingConfiguration)  ← 일반 @Configuration 클래스
        → @Bean ScheduledAnnotationBeanPostProcessor

@SpringBootApplication
  └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector)  ← DeferredImportSelector
              → 조건부로 수백 개 자동 설정 클래스 선택
```

---

## 💻 실험으로 확인하기

### 실험 1: @Import 일반 클래스

```java
public class DatabaseConfig {
    // @Configuration 없음
    @Bean
    public DataSource dataSource() { return new EmbeddedDatabase(); }
}

@Configuration
@Import(DatabaseConfig.class)
public class AppConfig {}

// DatabaseConfig가 @Configuration 없이 Import됨
// → Lite Mode로 처리 (@Bean 메서드 파싱됨)
// → dataSource Bean 등록됨
System.out.println(ctx.containsBean("dataSource"));  // true
```

### 실험 2: ImportSelector 직접 구현

```java
public class MySelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 특정 조건에 따라 다른 설정 반환
        boolean useCache = metadata.getAnnotationAttributes(
            EnableMyFeature.class.getName()).getBoolean("cache");
        return useCache
            ? new String[]{"com.example.CachedServiceConfig"}
            : new String[]{"com.example.SimpleServiceConfig"};
    }
}

@EnableMyFeature(cache = true)  // → CachedServiceConfig 선택
@Configuration
public class AppConfig {}
```

### 실험 3: ImportBeanDefinitionRegistrar로 동적 BeanDefinition

```java
public class MyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                         BeanDefinitionRegistry registry) {
        // BeanDefinition 직접 생성
        RootBeanDefinition bd = new RootBeanDefinition(MyService.class);
        bd.getPropertyValues().add("name", "dynamic");
        registry.registerBeanDefinition("myDynamicService", bd);
    }
}

@Configuration
@Import(MyRegistrar.class)
public class AppConfig {}

// myDynamicService Bean 등록 확인
System.out.println(ctx.getBean("myDynamicService"));
```

---

## 🤔 트레이드오프

```
방식 1 (일반 클래스):
  장점  단순, 명시적
  단점  조건부 선택 불가, 유연성 낮음
  사용  단순 설정 모듈화, @Configuration 추가

방식 2 (ImportSelector):
  장점  조건에 따라 다른 설정 선택 가능
       AnnotationMetadata로 어노테이션 속성 접근 가능
  단점  직접 BeanDefinition 생성 불가 (클래스명만 반환)
  사용  @EnableXxx 패턴, 조건부 설정 선택

DeferredImportSelector:
  장점  모든 @Configuration 파싱 후 실행 → 전체 컨텍스트 파악 가능
  단점  처리 순서가 나중 → 의존 관계 주의
  사용  Spring Boot 자동 설정 (사용자 설정 반영 후 자동 설정 결정)

방식 3 (ImportBeanDefinitionRegistrar):
  장점  BeanDefinition 완전 제어 (scope, 생성자 인수 등)
       파싱 단계 없이 직접 등록
  단점  복잡성 높음, 스프링 내부 API 의존
  사용  @EnableAspectJAutoProxy처럼 인프라 Bean 직접 등록
       MyBatis @MapperScan — Mapper 인터페이스별 FactoryBean 동적 등록
```

---

## 📌 핵심 정리

```
3가지 방식 비교
  일반 클래스       → processConfigurationClass() 재귀 파싱
  ImportSelector    → selectImports() → 클래스명 배열 → 재귀 파싱
  DeferredImportSelector → 모든 파싱 완료 후 처리 (Spring Boot 자동 설정)
  Registrar         → loadBeanDefinitions() 단계에서 직접 등록

처리 시점 순서
  일반 클래스 / ImportSelector → 현재 파싱 라운드
  DeferredImportSelector    → 전체 파싱 완료 후
  ImportBeanDefinitionRegistrar → BeanDefinition 로딩 단계 (가장 나중)

@Enable 패턴 핵심
  @EnableXxx = @Import + (ImportSelector 또는 Registrar)
  → 어노테이션 하나로 기능 모듈 전체 활성화

selectImports() vs registerBeanDefinitions()
  selectImports(): 클래스명 반환 → 파싱 후 등록 → @Conditional 적용됨
  registerBeanDefinitions(): BeanDefinition 직접 등록 → @Conditional 우회 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `ImportSelector.selectImports()`가 반환한 클래스에 `@Conditional`이 붙어 있으면 조건 평가가 이루어지는가?

**Q2.** `DeferredImportSelector`가 Spring Boot 자동 설정에 반드시 필요한 이유는?

**Q3.** `ImportBeanDefinitionRegistrar`에서 등록한 BeanDefinition은 `@Conditional`의 영향을 받는가?

> 💡 **해설**
>
> **Q1.** 이루어진다. `selectImports()`가 반환한 클래스명들은 `ConfigurationClassParser`가 일반 `@Configuration` 클래스처럼 재파싱한다. 이 파싱 과정에서 `ConditionEvaluator.shouldSkip()`이 `@Conditional`을 평가한다. 따라서 `ImportSelector`가 반환한 클래스에 `@ConditionalOnClass`, `@ConditionalOnMissingBean` 등이 붙어 있으면 조건에 따라 건너뛰어질 수 있다. Spring Boot 자동 설정 클래스들이 이 방식으로 `@Conditional`을 활용한다.
>
> **Q2.** 일반 `ImportSelector`는 현재 파싱 라운드에서 즉시 실행된다. 이 시점에는 사용자가 작성한 `@Configuration` 클래스의 `@Bean` 메서드들이 아직 BeanDefinition으로 등록되지 않았을 수 있다. `@ConditionalOnMissingBean(DataSource.class)`가 즉시 평가되면 사용자의 `DataSource`가 아직 없는 상태이므로 자동 설정 `DataSource`가 등록된다. `DeferredImportSelector`는 모든 `@Configuration` 파싱이 완료된 후에 실행되므로 사용자 Bean이 먼저 등록된 상태에서 조건을 판단할 수 있다.
>
> **Q3.** `registerBeanDefinitions()`에서 직접 `BeanDefinitionRegistry.registerBeanDefinition()`을 호출해 등록한 BeanDefinition은 `ConditionEvaluator`를 거치지 않는다. 즉 `@Conditional`이 붙어 있어도 조건 평가 없이 강제로 등록된다. 조건부 등록이 필요하다면 `registerBeanDefinitions()` 내부에서 직접 조건을 확인하고 등록 여부를 결정해야 한다. `ConditionContext`를 생성해 `Condition.matches()`를 직접 호출하거나, BeanDefinition에 `@Conditional`을 붙여도 이 경로에서는 무시된다.

---

<div align="center">

**[⬅️ 이전: Lite Mode vs Full Mode](./04-lite-mode-vs-full-mode.md)** | **[다음: ImportSelector & ImportBeanDefinitionRegistrar ➡️](./06-importselector-registrar.md)**

</div>

# ImportSelector & ImportBeanDefinitionRegistrar — Spring Boot 자동 설정의 핵심 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AutoConfigurationImportSelector`가 자동 설정 클래스를 로드하는 정확한 경로는?
- `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일이 어떻게 처리되는가?
- `AutoConfigurationImportSelector.AutoConfigurationGroup`이 `DeferredImportSelector.Group`을 구현하는 이유는?
- `@MapperScan`이 `ImportBeanDefinitionRegistrar`로 Mapper 인터페이스별 FactoryBean을 등록하는 방식은?
- 커스텀 `ImportBeanDefinitionRegistrar`에서 제네릭 타입 정보를 어떻게 활용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 수백 개의 자동 설정을 조건부로, 사용자 설정 이후에 처리해야 한다

```
Spring Boot 자동 설정 요구사항:
  1. 수백 개의 자동 설정 클래스가 클래스패스에 존재
  2. 각각 @ConditionalOnClass, @ConditionalOnMissingBean 등 조건 보유
  3. 사용자 @Configuration이 먼저 처리된 후에 조건 평가해야 함
  4. 자동 설정끼리도 @AutoConfigureBefore/@AutoConfigureAfter 순서 존재

일반 @ComponentScan으로는 불가:
  → 스캔 범위 제어 어려움
  → 처리 순서 보장 어려움
  → 조건 평가 시점 제어 불가

AutoConfigurationImportSelector (DeferredImportSelector):
  → 사용자 설정 파싱 완료 후 실행
  → imports 파일에서 설정 클래스 목록 로드
  → 필터링 + 정렬 후 조건부 등록
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Spring Boot 자동 설정은 @ComponentScan으로 동작한다

```
❌ 잘못된 이해:
  "Spring Boot가 spring-boot-autoconfigure JAR을 스캔해서 자동 설정 클래스를 찾는다"

✅ 실제:
  AutoConfigurationImportSelector가 특정 파일에서 클래스 목록을 읽음

  Spring Boot 2.x: META-INF/spring.factories
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    com.example.DataSourceAutoConfiguration,\
    com.example.JacksonAutoConfiguration,...

  Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    com.example.DataSourceAutoConfiguration
    com.example.JacksonAutoConfiguration
    ...

  → @ComponentScan 없음, 파일 기반 명시적 목록
  → 각 JAR이 자신의 자동 설정 클래스를 이 파일에 선언
```

### Before: @MapperScan은 @ComponentScan과 동일하게 동작한다

```
❌ 잘못된 이해:
  "@MapperScan = Mapper 인터페이스를 스캔해서 @Component처럼 등록"

✅ 실제:
  @MapperScan → @Import(MapperScannerRegistrar.class)
  → ImportBeanDefinitionRegistrar 구현체
  → registerBeanDefinitions()에서 각 Mapper 인터페이스마다
     MapperFactoryBean BeanDefinition 직접 생성 및 등록
  → 인터페이스는 인스턴스화 불가 → FactoryBean으로 프록시 생성
  → @ComponentScan과 다른 완전히 별개의 메커니즘
```

---

## ✨ 올바른 이해와 사용

### After: 두 구현체의 역할 분담

```
AutoConfigurationImportSelector (DeferredImportSelector):
  역할: Spring Boot 자동 설정 클래스 목록 제공
  시점: 사용자 @Configuration 파싱 완료 후
  핵심: imports 파일 로드 → 필터링 → @AutoConfigureOrder 정렬

MapperScannerRegistrar (ImportBeanDefinitionRegistrar):
  역할: Mapper 인터페이스마다 MapperFactoryBean 등록
  시점: BeanDefinition 로딩 단계 (가장 나중)
  핵심: ClassPathMapperScanner로 인터페이스 탐색 → MapperFactoryBean BeanDefinition 생성
```

---

## 🔬 내부 동작 원리

### 1. AutoConfigurationImportSelector — DeferredImportSelector 구현

```java
// AutoConfigurationImportSelector
public class AutoConfigurationImportSelector
        implements DeferredImportSelector, BeanClassLoaderAware,
                   ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // DeferredImportSelector → 이 메서드는 Group 기반 처리로 대체됨
        // getImportGroup()이 AutoConfigurationGroup을 반환하면
        // Group.process() + Group.selectImports()가 대신 호출됨
        if (!isEnabled(annotationMetadata)) return NO_IMPORTS;
        AutoConfigurationEntry entry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(entry.getConfigurations());
    }

    @Override
    public Class<? extends Group> getImportGroup() {
        return AutoConfigurationGroup.class;
        // → Group 기반 처리 사용
        // → 여러 DeferredImportSelector를 그룹화해서 한 번에 처리
    }
}
```

### 2. AutoConfigurationGroup — 그룹 기반 처리

```java
private static class AutoConfigurationGroup
        implements DeferredImportSelector.Group, BeanClassLoaderAware, ... {

    // 여러 @EnableAutoConfiguration이 있을 때 결과를 합산
    private final Map<String, AnnotationMetadata> entries = new LinkedHashMap<>();
    private final List<AutoConfigurationEntry> autoConfigurationEntries = new ArrayList<>();

    @Override
    public void process(AnnotationMetadata annotationMetadata,
                        DeferredImportSelector deferredImportSelector) {
        // 자동 설정 엔트리 수집
        AutoConfigurationEntry entry =
            ((AutoConfigurationImportSelector) deferredImportSelector)
                .getAutoConfigurationEntry(annotationMetadata);
        this.autoConfigurationEntries.add(entry);
        for (String importClassName : entry.getConfigurations()) {
            this.entries.putIfAbsent(importClassName, annotationMetadata);
        }
    }

    @Override
    public Iterable<Entry> selectImports() {
        // 모든 엔트리 수집 후 정렬하여 반환
        if (this.autoConfigurationEntries.isEmpty()) return Collections.emptyList();

        Set<String> allExclusions = collectExclusions();
        Set<String> processedConfigurations = collectConfigurations();
        processedConfigurations.removeAll(allExclusions);

        // @AutoConfigureOrder / @AutoConfigureBefore / @AutoConfigureAfter 기반 정렬
        return sortAutoConfigurations(processedConfigurations, ...)
            .stream()
            .map(importClassName -> new Entry(entries.get(importClassName), importClassName))
            .toList();
    }
}
```

### 3. getAutoConfigurationEntry() — imports 파일 로드

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata metadata) {
    if (!isEnabled(metadata)) return EMPTY_ENTRY;

    // @EnableAutoConfiguration의 exclude/excludeName 속성 수집
    AnnotationAttributes attributes = getAttributes(metadata);
    List<String> configurations = getCandidateConfigurations(metadata, attributes);

    // 중복 제거
    configurations = removeDuplicates(configurations);

    // exclusion 적용
    Set<String> exclusions = getExclusions(metadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);

    // AutoConfigurationImportFilter 적용
    // → @ConditionalOnClass 등 빠른 필터링 (클래스 로드 없이)
    configurations = getConfigurationClassFilter().filter(configurations);

    // AutoConfigurationImportEvent 발행
    fireAutoConfigurationImportEvents(configurations, exclusions);

    return new AutoConfigurationEntry(configurations, exclusions);
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, ...) {
    // Spring Boot 3.x: ImportCandidates.load() 사용
    List<String> configurations = ImportCandidates.load(
        AutoConfiguration.class, getBeanClassLoader()).getCandidates();
    // → META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 읽기

    // Spring Boot 2.x 호환: spring.factories도 확인
    configurations.addAll(SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));

    return configurations;
}
```

```
imports 파일 형식 (Spring Boot 3.x):
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

com.example.autoconfigure.DataSourceAutoConfiguration
com.example.autoconfigure.JacksonAutoConfiguration
com.example.autoconfigure.TransactionAutoConfiguration
...

→ 클래스패스의 모든 JAR에서 이 파일을 수집
→ AutoConfigurationGroup.process()에서 합산
→ selectImports()에서 필터링 + 정렬 후 반환
```

### 4. AutoConfigurationImportFilter — 빠른 필터링

```java
// AutoConfigurationImportFilter
// → @ConditionalOnClass처럼 클래스 로드 없이 빠르게 필터링

public interface AutoConfigurationImportFilter {
    boolean[] match(String[] autoConfigurationClasses,
                    AutoConfigurationMetadata autoConfigurationMetadata);
}

// OnClassCondition implements AutoConfigurationImportFilter
// → autoConfigurationMetadata (spring-autoconfigure-metadata.properties) 활용
// → 클래스 로드 없이 클래스패스 존재 여부만 확인
// → 로드 불가능한 자동 설정을 조기에 제거 → 처리 속도 향상
```

### 5. MapperScannerRegistrar — ImportBeanDefinitionRegistrar 활용

```java
// MyBatis @MapperScan 처리
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ... {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                         BeanDefinitionRegistry registry) {
        // @MapperScan 속성 읽기
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(
            importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
        String[] basePackages = attrs.getStringArray("basePackages");

        // ClassPathMapperScanner로 Mapper 인터페이스 탐색
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setAnnotationClass(attrs.getClass("annotationClass"));
        scanner.setMarkerInterface(attrs.getClass("markerInterface"));

        scanner.scan(basePackages);
        // → 각 Mapper 인터페이스에 대해:
        //   GenericBeanDefinition bd = new GenericBeanDefinition()
        //   bd.setBeanClass(MapperFactoryBean.class)
        //   bd.getConstructorArgumentValues().addGenericArgumentValue(mapperInterface)
        //   registry.registerBeanDefinition(beanName, bd)
    }
}
```

```
MapperFactoryBean BeanDefinition 구조:
  beanClass: MapperFactoryBean<OrderMapper>
  constructorArgs[0]: OrderMapper.class
  propertyValues["sqlSessionFactory"]: ref → sqlSessionFactory Bean

Bean 생성 시:
  MapperFactoryBean<OrderMapper>.getObject()
    → sqlSession.getMapper(OrderMapper.class)
    → MyBatis 동적 프록시 생성 → Mapper 인터페이스 구현체 반환

결과:
  @Autowired OrderMapper mapper
  → MapperFactoryBean이 생성한 프록시 주입
```

### 6. 커스텀 ImportBeanDefinitionRegistrar 패턴

```java
// 커스텀 어노테이션 기반 동적 Bean 등록
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(ClientRegistrar.class)  // ImportBeanDefinitionRegistrar
public @interface EnableClients {
    String[] basePackages() default {};
}

public class ClientRegistrar implements ImportBeanDefinitionRegistrar,
                                         EnvironmentAware {
    private Environment environment;

    @Override
    public void setEnvironment(Environment env) { this.environment = env; }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                         BeanDefinitionRegistry registry) {
        // @EnableClients 속성 읽기
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(EnableClients.class.getName()));
        String[] basePackages = attrs.getStringArray("basePackages");

        // ClassPathScanningCandidateComponentProvider로 @FeignClient 인터페이스 탐색
        ClassPathScanningCandidateComponentProvider scanner =
            new ClassPathScanningCandidateComponentProvider(false);
        scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));

        for (String basePackage : basePackages) {
            for (BeanDefinition candidate : scanner.findCandidateComponents(basePackage)) {
                // FeignClient 인터페이스마다 FeignClientFactoryBean 등록
                String className = candidate.getBeanClassName();
                BeanDefinitionBuilder builder =
                    BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
                builder.addPropertyValue("type", className);
                builder.addPropertyValue("url",
                    environment.getProperty("feign.url"));
                registry.registerBeanDefinition(
                    ClassUtils.getShortName(className), builder.getBeanDefinition());
            }
        }
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 자동 설정 클래스 목록 출력

```java
// Spring Boot 자동 설정 클래스 목록 확인
List<String> autoConfigs = ImportCandidates
    .load(AutoConfiguration.class, getClass().getClassLoader())
    .getCandidates();
autoConfigs.forEach(System.out::println);
// → 수백 개 자동 설정 클래스 출력

// 실제 적용된 자동 설정 확인
ctx.getBean(AutoConfigurationReport.class)
   .getConditionAndOutcomesBySource()
   .forEach((key, val) -> System.out.println(key + " → " + val));
```

### 실험 2: 자동 설정 처리 순서 확인

```yaml
# 자동 설정 디버그 로그
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
```

```
DEBUG --- [main] o.s.b.a.AutoConfigurationSorter:
  Sorting auto-configuration classes using: DataSourceAutoConfiguration, JpaAutoConfiguration
  DataSourceAutoConfiguration before JpaAutoConfiguration (via @AutoConfigureBefore)
```

### 실험 3: 커스텀 DeferredImportSelector

```java
public class MyDeferredSelector implements DeferredImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 이 시점: 모든 일반 @Configuration 파싱 완료
        System.out.println("DeferredImportSelector 실행");
        return new String[]{"com.example.MyAutoConfig"};
    }
}

// 일반 ImportSelector와 처리 순서 비교
public class MyImmediateSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        System.out.println("즉시 ImportSelector 실행");
        return new String[]{};
    }
}
// 출력: "즉시 ImportSelector 실행" → (다른 파싱) → "DeferredImportSelector 실행"
```

---

## 🤔 트레이드오프

```
DeferredImportSelector 처리 순서 보장:
  장점  사용자 Bean 등록 후 조건 평가 → @ConditionalOnMissingBean 정확히 동작
  단점  다른 @Configuration보다 나중 처리 → 의존 관계 방향성 주의

AutoConfigurationImportFilter (빠른 필터):
  장점  클래스 로드 없이 조기 필터링 → 시작 시간 단축
  단점  spring-autoconfigure-metadata.properties 생성 필요 (별도 프로세서)

ImportBeanDefinitionRegistrar vs @Bean:
  @Bean:
    스프링 생명주기 내에서 처리
    @Conditional 적용됨
    단순 Bean 등록에 적합

  ImportBeanDefinitionRegistrar:
    BeanDefinition 완전 제어 가능
    @Conditional 우회 가능
    인터페이스 → FactoryBean 패턴에 필수
    복잡하지만 강력

imports 파일 방식 (Spring Boot 3.x):
  장점  spring.factories보다 단순하고 명확한 구조
        클래스별 한 줄
  단점  spring.factories와 병용 필요 (하위 호환)
        파일 위치 정확히 지정 필요
```

---

## 📌 핵심 정리

```
AutoConfigurationImportSelector
  DeferredImportSelector + Group(AutoConfigurationGroup) 구현
  사용자 @Configuration 파싱 완료 후 실행
  imports 파일(Spring Boot 3.x) 또는 spring.factories(2.x) 로드
  AutoConfigurationImportFilter로 클래스 로드 없이 조기 필터링
  @AutoConfigureOrder/Before/After 기반 정렬 후 반환

imports 파일 경로 (Spring Boot 3.x)
  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  각 JAR이 자신의 자동 설정 클래스를 선언
  ImportCandidates.load()로 클래스패스 전체 수집

MapperScannerRegistrar (ImportBeanDefinitionRegistrar)
  @MapperScan → @Import(MapperScannerRegistrar)
  registerBeanDefinitions(): 각 Mapper 인터페이스마다
    MapperFactoryBean BeanDefinition 직접 생성 + 등록
  MapperFactoryBean.getObject() → MyBatis 프록시 생성

커스텀 활용 패턴
  ImportSelector: 어노테이션 속성 기반 설정 클래스 선택
  DeferredImportSelector: 전체 컨텍스트 파악 후 선택
  ImportBeanDefinitionRegistrar: 인터페이스 → FactoryBean 패턴
                                  스캔 기반 동적 Bean 등록
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 3.x에서 커스텀 자동 설정 클래스를 만들 때 `META-INF/spring/...AutoConfiguration.imports` 파일에 등록하는 것 외에 필요한 것은?

**Q2.** `ImportBeanDefinitionRegistrar.registerBeanDefinitions()`에서 등록한 Bean이 `BeanPostProcessor`의 처리를 받는가?

**Q3.** `AutoConfigurationImportFilter`가 클래스 로드 없이 필터링할 수 있는 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** 클래스 자체를 `@AutoConfiguration` (또는 `@Configuration(proxyBeanMethods=false)`)으로 선언하고, 필요한 `@Conditional` 조건을 붙여야 한다. `@AutoConfigureBefore`/`@AutoConfigureAfter`로 처리 순서를 명시하는 것도 중요하다. 또한 `spring-autoconfigure-processor` 의존성을 추가하면 컴파일 시 `spring-autoconfigure-metadata.properties` 파일이 생성되어 `AutoConfigurationImportFilter`가 클래스 로드 없이 빠른 필터링을 수행할 수 있다. 테스트에서 `@SpringBootTest`가 이 자동 설정을 포함하도록 하려면 테스트 클래스패스에도 imports 파일이 있어야 한다.
>
> **Q2.** 받는다. `ImportBeanDefinitionRegistrar`로 등록된 BeanDefinition도 일반 BeanDefinition과 동일하게 `BeanDefinitionRegistry`에 저장된다. `refresh()` → `finishBeanFactoryInitialization()` 단계에서 Singleton Bean이 초기화될 때 `BeanPostProcessor` 체인을 정상적으로 통과한다. `@Autowired` 주입(`AutowiredAnnotationBeanPostProcessor`)이나 AOP 프록시 생성(`AbstractAutoProxyCreator`) 모두 적용된다. 단, `BeanDefinition.setRole(ROLE_INFRASTRUCTURE)`로 설정하면 일부 모니터링 도구에서 인프라 Bean으로 분류된다.
>
> **Q3.** `spring-autoconfigure-processor`가 컴파일 시 `spring-autoconfigure-metadata.properties` 파일을 생성하는데, 이 파일에 각 자동 설정 클래스의 `@ConditionalOnClass` 요구 클래스 목록이 사전 저장된다. `AutoConfigurationImportFilter.match()`는 이 파일을 참조해 클래스패스 존재 여부를 `ClassLoader.getResource()` 방식으로 확인한다. 실제 자동 설정 클래스를 로드하거나 `@Conditional`을 평가하는 복잡한 과정 없이 파일 조회만으로 빠르게 필터링할 수 있다. 이는 자동 설정 클래스를 실제로 로드하면 그 클래스가 참조하는 타입들까지 로드될 수 있어 오히려 `ClassNotFoundException`이 발생할 수 있기 때문이기도 하다.

---

<div align="center">

**[⬅️ 이전: @Import의 3가지 방식](./05-import-three-ways.md)** | **[홈으로 🏠](../README.md)**

</div>

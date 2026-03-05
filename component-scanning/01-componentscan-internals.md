# @ComponentScan 동작 원리 — 클래스패스 스캔 전체 파이프라인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@ComponentScan`이 실행되는 정확한 시점은 `refresh()` 어디인가?
- `ClassPathBeanDefinitionScanner`가 `.class` 파일을 탐색하는 경로는?
- ASM 기반 메타데이터 리더가 클래스 로드 없이 어노테이션을 읽는 원리는?
- `basePackages`를 지정하지 않으면 어떤 패키지가 기준이 되는가?
- 스캔 → 필터링 → BeanDefinition 등록까지의 단계별 흐름은?

---

## 🔍 왜 이게 존재하는가

### 문제: XML 없이 Bean을 자동으로 등록해야 한다

```xml
<!-- XML 시대: Bean마다 수동 등록 -->
<bean id="orderService" class="com.example.service.OrderService"/>
<bean id="paymentService" class="com.example.service.PaymentService"/>
<!-- ... 수백 줄 -->
```

```java
// @ComponentScan: 패키지만 지정하면 자동 등록
@Configuration
@ComponentScan("com.example")
public class AppConfig {}
```

```
자동화가 가능한 이유:
  .class 파일에는 어노테이션 메타데이터가 포함돼 있음
  ASM 라이브러리로 .class 바이트코드를 읽으면
  → 실제 Class 객체 로드 없이 어노테이션 정보 파싱 가능
  → @Component 붙은 것만 골라 BeanDefinition으로 등록

핵심 설계:
  수천 개 .class 파일 중 소수의 Bean 후보만 선별
  → 선별된 것만 실제로 클래스 로드
  → 메모리/시간 효율 압도적으로 우수
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: basePackages를 항상 명시해야 한다

```java
// ❌ 불필요한 명시
@SpringBootApplication(scanBasePackages = "com.example")
public class Application {}
// Application 클래스가 이미 com.example 패키지에 있음
```

```
✅ 기본 동작:
  basePackages 미지정 → @ComponentScan 선언 클래스의 패키지 기준
  → com.example.Application → "com.example" 자동 스캔

흔한 실수:
  Application 클래스를 default 패키지(패키지 선언 없음)에 두면
  → 모든 JAR의 모든 클래스 스캔 → 시작 시간 폭발
  → 항상 패키지 안에 Application 클래스 배치할 것
```

### Before: 스캔 시 모든 클래스가 실제로 로드된다

```
❌ 잘못된 이해:
  "스캔 = Class.forName()으로 클래스를 전부 로드한다"

✅ 실제:
  ASM ClassReader가 .class 파일 바이트코드만 읽음
  → ClassLoader.loadClass() 호출 없음
  → 어노테이션·클래스 선언만 파싱
  → 필터 통과한 Bean 후보만 실제로 클래스 로드됨
```

---

## ✨ 올바른 이해와 사용

### After: 스캔 파이프라인 전체 흐름

```
refresh()
  → invokeBeanFactoryPostProcessors()
    → ConfigurationClassPostProcessor (BeanDefinitionRegistryPostProcessor)
      → ConfigurationClassParser.parse()
        → @ComponentScan 메타데이터 수집
        → ComponentScanAnnotationParser.parse()
          → ClassPathBeanDefinitionScanner.doScan(basePackages)
            → findCandidateComponents()
              PathMatchingResourcePatternResolver: classpath*:com/example/**/*.class
              → SimpleMetadataReaderFactory: ASM으로 .class 읽기
              → includeFilter / excludeFilter 평가
              → @Conditional 평가
              → ScannedGenericBeanDefinition 생성
            → @Scope / @Lazy / @Primary 처리
            → BeanDefinitionRegistry 등록
            → 새로 발견된 @Configuration 클래스 재귀 처리
```

---

## 🔬 내부 동작 원리

### 1. 처리 시작 — ConfigurationClassPostProcessor

```java
// ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor
// → refresh()의 invokeBeanFactoryPostProcessors() 단계에서 실행

public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    processConfigBeanDefinitions(registry);
}

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    // 등록된 BeanDefinition 중 @Configuration 후보 탐색
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    for (String beanName : registry.getBeanDefinitionNames()) {
        BeanDefinition bd = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isConfigurationCandidate(bd)) {
            configCandidates.add(new BeanDefinitionHolder(bd, beanName));
        }
    }

    // @ComponentScan, @Import, @Bean 등 전체 처리
    ConfigurationClassParser parser = new ConfigurationClassParser(...);
    parser.parse(configCandidates);

    // 파싱 결과로 BeanDefinition 로딩
    this.reader.loadBeanDefinitions(configurationClasses);
}
```

### 2. ComponentScanAnnotationParser — basePackages 결정

```java
Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, String declaringClass) {

    ClassPathBeanDefinitionScanner scanner =
        new ClassPathBeanDefinitionScanner(registry,
            componentScan.getBoolean("useDefaultFilters"), ...);
    // useDefaultFilters=true: AnnotationTypeFilter(@Component) 기본 포함

    // includeFilters / excludeFilters 등록
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
        scanner.addIncludeFilter(...);
    }
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
        scanner.addExcludeFilter(...);
    }

    // basePackages 결정 (우선순위)
    Set<String> basePackages = new LinkedHashSet<>();

    // 1. 명시적 basePackages
    String[] specified = componentScan.getStringArray("basePackages");
    if (specified.length > 0) basePackages.addAll(Arrays.asList(specified));

    // 2. basePackageClasses → 해당 클래스의 패키지
    for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
        basePackages.add(ClassUtils.getPackageName(clazz));
    }

    // 3. 아무것도 없으면 → 선언 클래스의 패키지
    if (basePackages.isEmpty()) {
        basePackages.add(ClassUtils.getPackageName(declaringClass));
    }

    return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

### 3. ClassPathBeanDefinitionScanner.doScan()

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();

    for (String basePackage : basePackages) {

        // ASM 기반 후보 탐색
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);

        for (BeanDefinition candidate : candidates) {

            // @Scope 처리
            ScopeMetadata scopeMetadata =
                this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());

            // Bean 이름 결정 (기본: 클래스명 camelCase)
            String beanName = this.beanNameGenerator.generateBeanName(candidate, registry);

            // @Lazy, @Primary, @DependsOn 등 공통 처리
            if (candidate instanceof AbstractBeanDefinition abd) {
                postProcessBeanDefinition(abd, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition abd) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
            }

            // 충돌 확인 후 등록
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder holder = new BeanDefinitionHolder(candidate, beanName);
                holder = AnnotationConfigUtils.applyScopedProxyMode(
                    scopeMetadata, holder, registry);
                beanDefinitions.add(holder);
                registerBeanDefinition(holder, registry);
            }
        }
    }
    return beanDefinitions;
}
```

### 4. findCandidateComponents() — ASM 스캔 핵심

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();

    // "com.example" → "classpath*:com/example/**/*.class"
    String packageSearchPath = CLASSPATH_ALL_URL_PREFIX
        + resolveBasePackage(basePackage) + '/' + this.resourcePattern;

    // 패턴에 맞는 모든 .class 파일 Resource 탐색
    Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);

    for (Resource resource : resources) {
        if (!resource.isReadable()) continue;

        // ASM으로 .class 파일 메타데이터만 읽음 (클래스 로드 없음)
        MetadataReader metadataReader =
            getMetadataReaderFactory().getMetadataReader(resource);

        // 1차 필터: excludeFilter → includeFilter → @Conditional
        if (isCandidateComponent(metadataReader)) {
            ScannedGenericBeanDefinition sbd =
                new ScannedGenericBeanDefinition(metadataReader);
            sbd.setSource(resource);

            // 2차 필터: 독립 클래스이고 구체 클래스인지 확인
            if (isCandidateComponent(sbd)) {
                candidates.add(sbd);
            }
        }
    }
    return candidates;
}
```

### 5. ScannedGenericBeanDefinition — 클래스 로드 없는 BeanDefinition

```java
public class ScannedGenericBeanDefinition extends GenericBeanDefinition
        implements AnnotatedBeanDefinition {

    private final AnnotationMetadata metadata;  // ASM이 읽은 메타데이터

    public ScannedGenericBeanDefinition(MetadataReader metadataReader) {
        this.metadata = metadataReader.getAnnotationMetadata();
        setBeanClassName(this.metadata.getClassName());
        // → "com.example.service.OrderService" 문자열만 저장
        //   Class 객체 로드 없음!
    }
}

// 실제 Class 로드 시점:
// AbstractBeanFactory.resolveBeanClass()
//   → Class.forName(beanClassName, classLoader)
//   → 최초 Bean 인스턴스 생성 직전
```

### 6. @SpringBootApplication의 내장 @ComponentScan

```java
@ComponentScan(
    excludeFilters = {
        // 테스트 전용 TypeExcludeFilter 구현체 지원
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        // 자동 설정 클래스를 일반 스캔에서 제외
        // (@EnableAutoConfiguration 경로로만 처리되도록)
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
    }
)
public @interface SpringBootApplication {}
```

---

## 💻 실험으로 확인하기

### 실험 1: 스캔된 Bean 목록 확인

```java
@Configuration
@ComponentScan("com.example.service")
public class AppConfig {}

// 등록된 Bean 이름 출력
Arrays.stream(ctx.getBeanDefinitionNames())
      .filter(n -> !n.contains("org.springframework"))
      .sorted()
      .forEach(System.out::println);
// → com.example.service 패키지의 @Component 클래스만 출력
```

### 실험 2: basePackages 기본값 확인

```java
// com.example.config 패키지에 위치
@Configuration
@ComponentScan  // basePackages 미지정
public class AppConfig {}

// → "com.example.config" 패키지부터 스캔
// ClassUtils.getPackageName("com.example.config.AppConfig") = "com.example.config"
```

### 실험 3: ASM 스캔 — 클래스 로드 없음 확인

```java
// BeanDefinition의 beanClass 필드가 null임을 확인
ConfigurableListableBeanFactory bf = ctx.getBeanFactory();
BeanDefinition bd = bf.getBeanDefinition("orderService");

System.out.println(bd.getBeanClassName());  // "com.example.service.OrderService" (문자열)
System.out.println(bd instanceof ScannedGenericBeanDefinition);  // true

// ScannedGenericBeanDefinition은 클래스 로드 전까지 Class 객체 없음
// → 실제 Class 객체: ctx.getBean("orderService").getClass()로 획득
```

---

## 🤔 트레이드오프

```
넓은 스캔 범위 ("com.example"):
  장점  설정 간단, 새 클래스 자동 등록
  단점  의도치 않은 Bean 등록, 시작 시간 증가

좁은 범위 + 명시적 등록:
  장점  Bean 목록 명확, 시작 빠름
  단점  새 클래스마다 설정 추가 필요

basePackageClasses 관례:
  @ComponentScan(basePackageClasses = Markers.class)
  장점  문자열 오타 없음, IDE 리팩토링 지원
  관례  각 모듈의 루트 패키지에 빈 마커 인터페이스/클래스 배치

Spring Index (대규모 최적화):
  spring-context-indexer 의존성 추가
  → 컴파일 시 META-INF/spring.components 생성
  → 런타임 클래스패스 스캔 없이 인덱스 파일만 읽음
  → 수천 개 클래스 프로젝트에서 시작 시간 크게 단축
```

---

## 📌 핵심 정리

```
처리 시점
  refresh() → invokeBeanFactoryPostProcessors()
  → ConfigurationClassPostProcessor
  → ConfigurationClassParser
  → ComponentScanAnnotationParser
  → ClassPathBeanDefinitionScanner.doScan()

basePackages 결정 (우선순위)
  명시적 basePackages > basePackageClasses > 선언 클래스의 패키지

스캔 핵심 동작
  classpath*:패키지/**/*.class 패턴으로 Resource 탐색
  ASM MetadataReader: .class 파일 바이트코드만 읽음 (클래스 로드 없음)
  excludeFilter → includeFilter → @Conditional 순서로 평가
  통과한 것만 ScannedGenericBeanDefinition으로 생성 → 레지스트리 등록

ScannedGenericBeanDefinition 특징
  beanClassName: 문자열만 (Class 객체 없음)
  실제 클래스 로드: Bean 인스턴스 생성 직전 (AbstractBeanFactory.resolveBeanClass)
```

---

## 🤔 생각해볼 문제

**Q1.** `@ComponentScan`이 선언된 `@Configuration` 클래스 자신도 스캔 대상 패키지에 있다면 무한 재귀가 발생하는가?

**Q2.** 두 개의 `@ComponentScan`이 같은 패키지를 중복 스캔하면 어떻게 되는가?

**Q3.** `@SpringBootApplication`의 `AutoConfigurationExcludeFilter`가 없다면 어떤 문제가 생기는가?

> 💡 **해설**
>
> **Q1.** 발생하지 않는다. `ConfigurationClassParser`가 이미 파싱한 클래스를 `alreadyParsed` Set에 기록하고 동일 클래스는 건너뛴다. `ClassPathBeanDefinitionScanner.checkCandidate()`도 이미 등록된 BeanDefinition을 감지해 중복 등록을 방지한다.
>
> **Q2.** `checkCandidate()` 내부 `isCompatible()`이 같은 클래스(동일 `beanClassName`)면 호환 가능으로 판단해 조용히 넘어간다. 다른 클래스인데 Bean 이름이 충돌하면 `ConflictingBeanDefinitionException`이 발생한다. 중복 스캔은 오류 없이 넘어가지만 불필요한 파일 I/O가 발생한다.
>
> **Q3.** `spring.factories`에 등록된 자동 설정 클래스들이 `@ComponentScan` 일반 스캔 대상에도 포함된다. `@EnableAutoConfiguration`이 별도로 이들을 처리할 때 이미 등록된 BeanDefinition과 충돌하거나 조건부 평가(`@ConditionalOnMissingBean` 등)가 올바르게 작동하지 않을 수 있다. `AutoConfigurationExcludeFilter`는 자동 설정 클래스가 반드시 `@EnableAutoConfiguration` 경로를 통해서만 등록되도록 보장한다.

---

<div align="center">

**[다음: ClassPathScanningCandidateComponentProvider ➡️](./02-candidate-component-provider.md)**

</div>

# @Conditional 평가 과정 — ConditionEvaluator가 조건을 판단하는 시점과 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Conditional`이 평가되는 정확한 시점은 스캔 파이프라인 어디인가?
- `ConditionEvaluator`가 `Condition.matches()`를 호출하는 흐름은?
- `@ConditionalOnClass`는 클래스 로드 없이 어떻게 클래스 존재 여부를 확인하는가?
- `@ConditionalOnMissingBean`이 다른 Condition보다 늦게 평가되어야 하는 이유는?
- `ConfigurationPhase.PARSE_CONFIGURATION` vs `REGISTER_BEAN`의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 환경에 따라 Bean 등록 여부를 동적으로 결정해야 한다

```java
// 하드코딩 방식 — 환경마다 코드 변경 필요
@Bean
public DataSource dataSource() {
    if (isTestEnvironment()) return new H2DataSource();
    else return new MySQLDataSource();
}

// @Conditional 방식 — 조건만 선언, 코드 변경 없음
@Bean
@ConditionalOnProperty("spring.datasource.url")
public DataSource dataSource() { return new MySQLDataSource(); }

@Bean
@ConditionalOnMissingBean(DataSource.class)
public DataSource defaultDataSource() { return new H2DataSource(); }
```

```
@Conditional이 해결하는 것:
  Bean 등록 여부를 런타임 환경 조건으로 결정
  → 프로파일, 클래스패스, 프로퍼티, 기존 Bean 존재 여부 등

Spring Boot Auto-configuration 핵심:
  수백 개의 @ConditionalOnClass / @ConditionalOnMissingBean 조건으로
  필요한 Bean만 선택적으로 등록
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @ConditionalOnClass는 클래스를 로드해서 확인한다

```java
@ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver")
```

```
❌ 잘못된 이해:
  "Class.forName()으로 로드해서 존재 여부를 확인한다"
  → 클래스가 없으면 ClassNotFoundException → 처리 복잡

✅ 실제:
  클래스패스에서 리소스 존재 여부만 확인
  ClassLoader.getResource("com/mysql/cj/jdbc/Driver.class") != null
  → 실제 클래스 로드 없음
  → 없어도 예외 발생 안 함
```

### Before: @ConditionalOnMissingBean은 언제 평가해도 같은 결과다

```java
// ❌ 잘못된 이해:
// "@ConditionalOnMissingBean은 평가 시점과 무관하게 동작한다"

// ✅ 실제:
// Auto-configuration 클래스들은 일반 @Configuration보다 나중에 처리됨
// → 사용자 Bean이 먼저 등록되고 나서 @ConditionalOnMissingBean 평가
// → 사용자가 DataSource를 정의했으면 → 자동 설정 DataSource 건너뜀
// 평가 순서가 잘못되면 사용자 Bean이 덮어씌워질 수 있음
```

---

## ✨ 올바른 이해와 사용

### After: Condition 평가 시점과 단계를 구분

```
@Conditional 평가 시점:

시점 1 — 컴포넌트 스캔 중 (PARSE_CONFIGURATION 페이즈)
  ClassPathBeanDefinitionScanner.isCandidateComponent()
  → isConditionMatch() 호출
  → 스캔 대상 클래스 자체가 Bean 후보에서 탈락 가능

시점 2 — @Configuration 파싱 중 (PARSE_CONFIGURATION 페이즈)
  ConfigurationClassParser.processConfigurationClass()
  → shouldSkip() 호출
  → @Configuration 클래스 자체 또는 @Bean 메서드 건너뜀

시점 3 — BeanDefinition 등록 시 (REGISTER_BEAN 페이즈)
  ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()
  → shouldSkip() 호출
  → @Bean 메서드의 Bean 등록 여부 결정

ConfigurationPhase:
  PARSE_CONFIGURATION → @Configuration 파싱 단계에서 평가
  REGISTER_BEAN       → Bean 등록 단계에서 평가 (기본값)
```

---

## 🔬 내부 동작 원리

### 1. ConditionEvaluator.shouldSkip() — 핵심 평가 진입점

```java
// ConditionEvaluator.java
public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {

    // @Conditional 어노테이션이 없으면 건너뜀 없음
    if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
        return false;
    }

    // phase가 null이면 페이즈 판단
    if (phase == null) {
        if (metadata instanceof AnnotationMetadata am
                && ConfigurationClassUtils.isConfigurationCandidate(am)) {
            return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
        }
        return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
    }

    // @Conditional에서 Condition 클래스 목록 수집
    List<Condition> conditions = new ArrayList<>();
    for (String[] conditionClasses : getConditionClasses(metadata)) {
        for (String conditionClass : conditionClasses) {
            Condition condition = getCondition(conditionClass, context.getClassLoader());
            conditions.add(condition);
        }
    }

    // Condition 정렬 (OrderComparator 기반)
    AnnotationAwareOrderComparator.sort(conditions);

    for (Condition condition : conditions) {
        ConfigurationPhase requiredPhase = null;
        if (condition instanceof ConfigurationCondition cc) {
            requiredPhase = cc.getConfigurationPhase();
        }

        // 현재 페이즈와 Condition의 요구 페이즈가 맞지 않으면 건너뜀
        if (requiredPhase != null && requiredPhase != phase) {
            continue;
        }

        // Condition 평가
        if (!condition.matches(context, metadata)) {
            return true;  // 조건 불충족 → 이 Bean/클래스 건너뜀
        }
    }
    return false;  // 모든 Condition 통과 → 등록
}
```

### 2. ConditionContext — Condition에 제공되는 컨텍스트

```java
// Condition.matches()에 전달되는 ConditionContext
public interface ConditionContext {

    BeanDefinitionRegistry getRegistry();
    // → 이미 등록된 BeanDefinition 조회 가능
    // → @ConditionalOnMissingBean이 여기서 기존 Bean 확인

    ConfigurableListableBeanFactory getBeanFactory();
    // → Bean 타입 조회, 존재 여부 확인

    Environment getEnvironment();
    // → 프로퍼티, 프로파일 조회
    // → @ConditionalOnProperty, @Profile이 여기서 확인

    ResourceLoader getResourceLoader();
    // → 클래스패스 리소스 존재 여부 확인

    ClassLoader getClassLoader();
    // → 클래스패스 탐색 (로드 없이)
}
```

### 3. @ConditionalOnClass — 클래스패스 존재 여부 확인

```java
// OnClassCondition (Spring Boot)
// ConfigurationCondition 구현 → 페이즈 제어 가능
public class OnClassCondition extends FilteringSpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
                                             AnnotatedTypeMetadata metadata) {
        ClassLoader classLoader = context.getClassLoader();

        // @ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver") 속성 읽기
        List<String> classNames = getCandidates(metadata, ConditionalOnClass.class);

        List<String> missing = filter(classNames, ClassNameFilter.MISSING, classLoader);
        // ClassNameFilter.MISSING:
        //   classLoader.getResource(className.replace('.', '/') + ".class") == null
        //   → 실제 클래스 로드 없이 리소스 존재 여부만 확인!

        if (!missing.isEmpty()) {
            return ConditionOutcome.noMatch(
                ConditionMessage.forCondition(ConditionalOnClass.class)
                    .didNotFind("required class", "required classes")
                    .items(Style.QUOTE, missing));
        }
        return ConditionOutcome.match();
    }
}
```

```
ClassNameFilter.MISSING 구현:
  classLoader.getResource("com/mysql/cj/jdbc/Driver.class")
  → null이면 클래스패스에 없음 → MISSING = true
  → Class.forName() 없음 → ClassNotFoundException 없음
  → 클래스가 없어도 안전하게 false 반환
```

### 4. @ConditionalOnMissingBean — REGISTER_BEAN 페이즈

```java
// OnMissingBeanCondition
// ConfigurationCondition.getConfigurationPhase() = REGISTER_BEAN
// → BeanDefinition 등록 시점에 평가 (PARSE_CONFIGURATION에서는 건너뜀)

public class OnMissingBeanCondition extends FilteringSpringBootCondition
        implements ConfigurationCondition {

    @Override
    public ConfigurationPhase getConfigurationPhase() {
        return ConfigurationPhase.REGISTER_BEAN;
        // ← 핵심: 반드시 등록 시점에 평가해야 함
        // 이미 등록된 BeanDefinition을 기준으로 판단하므로
        // 파싱 시점에는 아직 모든 Bean이 등록 안 됐을 수 있음
    }

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, ...) {
        BeanDefinitionRegistry registry = context.getRegistry();

        // @ConditionalOnMissingBean(DataSource.class) 속성
        Set<String> beanNames = collectBeanNamesForType(
            context.getBeanFactory(), DataSource.class);
        // → registry에서 DataSource 타입의 BeanDefinition 탐색

        if (!beanNames.isEmpty()) {
            return ConditionOutcome.noMatch(...);
            // DataSource Bean이 이미 등록됨 → 이 Bean 건너뜀
        }
        return ConditionOutcome.match();
        // DataSource Bean 없음 → 이 Bean 등록
    }
}
```

```
REGISTER_BEAN 페이즈 필요 이유:

Auto-configuration 처리 순서:
  1. 사용자 @Configuration 클래스 파싱 (PARSE_CONFIGURATION)
  2. 사용자 @Bean 메서드 → BeanDefinition 등록
  3. Auto-configuration 클래스 파싱 + 등록 (나중에)
     → @ConditionalOnMissingBean 평가 시 1~2의 결과 반영됨

만약 PARSE_CONFIGURATION에서 평가하면:
  아직 사용자 Bean이 등록 안 된 시점
  → @ConditionalOnMissingBean이 항상 true → 자동 설정 Bean 중복 등록
```

### 5. ConfigurationPhase — 두 단계 평가 시점

```java
// ConfigurationCondition 인터페이스
public interface ConfigurationCondition extends Condition {

    enum ConfigurationPhase {
        // @Configuration 파싱 단계
        // → 이 단계에서 false면 @Configuration 클래스 자체를 건너뜀
        // → @Bean 메서드도 모두 건너뜀
        PARSE_CONFIGURATION,

        // BeanDefinition 등록 단계
        // → 이 단계에서 false면 해당 @Bean만 건너뜀
        // → @Configuration 클래스 자체는 살아있음
        REGISTER_BEAN
    }

    ConfigurationPhase getConfigurationPhase();
}
```

```
페이즈 선택 가이드:

PARSE_CONFIGURATION 사용:
  @ConditionalOnClass → 라이브러리 자체가 없으면 @Configuration 전체 무의미
  @Profile            → 프로파일 자체가 맞지 않으면 @Configuration 전체 건너뜀

REGISTER_BEAN 사용 (기본값):
  @ConditionalOnMissingBean → 다른 Bean 등록 결과 봐야 함
  @ConditionalOnBean        → 특정 Bean이 있어야 함
  @ConditionalOnProperty    → 프로퍼티 값 기준
```

### 6. @Conditional 커스텀 구현

```java
// 커스텀 Condition 구현
public class OnLinuxCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // Environment에서 OS 정보 확인
        String os = context.getEnvironment().getProperty("os.name", "");
        return os.toLowerCase().contains("linux");
    }
}

// 커스텀 어노테이션으로 래핑
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(OnLinuxCondition.class)
public @interface ConditionalOnLinux {}

// 사용
@Bean
@ConditionalOnLinux
public FileWatcher linuxFileWatcher() { ... }
```

---

## 💻 실험으로 확인하기

### 실험 1: @ConditionalOnClass 동작 확인

```java
// classpath에 없는 클래스 조건
@Configuration
@ConditionalOnClass(name = "com.nonexistent.Library")
public class LibraryAutoConfig {
    @Bean
    public LibraryBean libraryBean() { return new LibraryBean(); }
}

// → LibraryAutoConfig 자체가 건너뜀 (PARSE_CONFIGURATION 페이즈)
// → libraryBean도 등록 안 됨
// → ClassNotFoundException 없음
```

### 실험 2: @ConditionalOnMissingBean 순서 확인

```java
@Configuration
class UserConfig {
    @Bean
    DataSource userDataSource() { return new HikariDataSource(); }
}

@Configuration
@AutoConfigureAfter(UserConfig.class)  // 순서 보장
class DataSourceAutoConfig {
    @Bean
    @ConditionalOnMissingBean(DataSource.class)
    DataSource defaultDataSource() { return new EmbeddedDatabase(); }
    // → UserConfig의 userDataSource가 먼저 등록됨
    // → @ConditionalOnMissingBean 평가 시 DataSource 존재 → 건너뜀
}
```

### 실험 3: Condition 결과 로그로 확인

```yaml
# application.yml
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
```

```
출력 예시:
DataSourceAutoConfiguration:
  Did not match:
    - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)
  Matched:
    - @ConditionalOnMissingBean (types: javax.sql.DataSource) did not find any beans (OnMissingBeanCondition)
```

---

## 🤔 트레이드오프

```
@Conditional vs @Profile:
  @Profile     → Environment의 activeProfiles 기반, 간단
  @Conditional → 임의 Condition 로직, 더 유연
  @Profile은 내부적으로 @Conditional(ProfileCondition.class)로 구현됨

PARSE_CONFIGURATION vs REGISTER_BEAN:
  PARSE_CONFIGURATION: @Configuration 전체 포함 여부 결정
    → Bean 등록 전에 평가 → 다른 Bean 상태 볼 수 없음
    → 클래스패스, 프로퍼티, 환경 기반 조건에 적합

  REGISTER_BEAN: 개별 @Bean 포함 여부 결정
    → 다른 Bean 등록 이후 평가 → 기존 Bean 상태 볼 수 있음
    → @ConditionalOnMissingBean, @ConditionalOnBean에 필수

@ConditionalOnMissingBean 주의:
  BeanDefinition 기준으로 확인 (인스턴스 아님)
  → BeanDefinition만 등록돼도 "있음"으로 판단
  → 이름 기반이 아닌 타입 기반 확인 권장
  → 부모 컨텍스트 Bean은 기본적으로 고려 안 됨
```

---

## 📌 핵심 정리

```
평가 진입점
  ConditionEvaluator.shouldSkip()
  → 컴포넌트 스캔 중: isConditionMatch()
  → @Configuration 파싱 중: processConfigurationClass()
  → @Bean 등록 시: loadBeanDefinitionsForBeanMethod()

ConfigurationPhase
  PARSE_CONFIGURATION: @Configuration 클래스 수준 판단
  REGISTER_BEAN:       @Bean 메서드 수준 판단 (기본값)

핵심 Condition 구현
  OnClassCondition     classLoader.getResource() → 클래스 로드 없이 존재 확인
  OnMissingBeanCondition registry에서 타입 검색 → REGISTER_BEAN 페이즈 필수

ConditionContext가 제공하는 것
  BeanDefinitionRegistry  이미 등록된 Bean 조회
  Environment             프로퍼티, 프로파일 조회
  ClassLoader             클래스패스 리소스 확인

Condition 정렬
  AnnotationAwareOrderComparator → @Order 또는 Ordered 기준
  낮은 order = 먼저 평가
```

---

## 🤔 생각해볼 문제

**Q1.** `@ConditionalOnClass`를 `@Configuration` 클래스 레벨에 붙이는 것과 `@Bean` 메서드 레벨에 붙이는 것의 차이는?

**Q2.** 커스텀 `Condition` 구현에서 `ConditionContext.getRegistry()`로 아직 등록되지 않은 Bean을 조회하면 어떻게 되는가?

**Q3.** 두 개의 `@Conditional`이 같은 Bean에 붙어 있을 때 하나가 `false`를 반환하면 다른 하나도 평가되는가?

> 💡 **해설**
>
> **Q1.** 클래스 레벨에 붙이면 `PARSE_CONFIGURATION` 페이즈에서 평가되어 해당 `@Configuration` 클래스 전체가 건너뛰어진다. 즉 클래스 안의 모든 `@Bean` 메서드도 함께 건너뛴다. 메서드 레벨에 붙이면 `REGISTER_BEAN` 페이즈에서 해당 `@Bean` 메서드만 건너뛰어지고 나머지 `@Bean` 메서드는 정상 처리된다. `@ConditionalOnClass`를 클래스 레벨에 붙이는 것이 일반적으로 권장되는 이유는, 해당 라이브러리 없이 `@Configuration` 클래스 자체를 파싱하려 하면 반환 타입 등에서 `ClassNotFoundException`이 발생할 수 있기 때문이다.
>
> **Q2.** `getRegistry()`는 현재까지 등록된 `BeanDefinition`만 반환한다. 아직 처리되지 않은 `@Configuration` 클래스의 `@Bean`이나 스캔되지 않은 `@Component`는 아직 레지스트리에 없다. 이 경우 해당 Bean이 없는 것으로 판단된다. 이 때문에 `@ConditionalOnMissingBean`은 `REGISTER_BEAN` 페이즈에서 평가되어야 하고, 자동 설정 클래스들은 사용자 `@Configuration`보다 나중에 처리되도록 `@AutoConfigureAfter`로 순서를 보장한다.
>
> **Q3.** `ConditionEvaluator.shouldSkip()`은 Condition 목록을 순서대로 평가하다가 하나라도 `false`(matches 반환 false)를 반환하면 즉시 `shouldSkip = true`를 반환하고 나머지 Condition 평가를 중단한다. 단락 평가(short-circuit)처럼 동작한다. 따라서 첫 번째 Condition이 `false`이면 두 번째 Condition은 평가되지 않는다. 이는 AND 조건으로 처리된다는 의미이며, 여러 `@Conditional`을 붙이면 모두 통과해야 Bean이 등록된다.

---

<div align="center">

**[⬅️ 이전: TypeFilter Include & Exclude](./03-typefilter-include-exclude.md)** | **[다음: BeanDefinition 등록 과정 ➡️](./05-beandefinition-registration.md)**

</div>

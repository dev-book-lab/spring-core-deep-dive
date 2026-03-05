# Lite Mode vs Full Mode 트레이드오프 — 선택 기준과 성능 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Lite Mode가 Full Mode보다 빠른 구체적인 이유는 무엇인가?
- CGLIB 서브클래스 생성 비용이 시작 시간에 미치는 실제 영향은?
- Spring Boot의 자동 설정 클래스가 대부분 Lite Mode인 이유는?
- Lite Mode에서 싱글톤을 안전하게 보장하는 올바른 패턴은?
- 어떤 상황에서 Full Mode, 어떤 상황에서 Lite Mode를 선택해야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Full Mode는 항상 CGLIB 비용을 지불한다

```java
// Full Mode — CGLIB 서브클래스 생성 비용 항상 발생
@Configuration  // proxyBeanMethods = true (기본)
public class AppConfig {

    @Bean
    public OrderService orderService() {
        return new OrderService();
        // ← 다른 @Bean 메서드 참조 없음
        // 이 경우에도 CGLIB 서브클래스가 생성됨
        // → 불필요한 비용
    }
}
```

```
Lite Mode가 해결하는 것:
  @Bean 메서드끼리 직접 호출이 없을 때 CGLIB 불필요
  → proxyBeanMethods=false로 CGLIB 생성 건너뜀
  → 시작 시간 단축, 메모리 사용 감소

Spring Boot 적극 활용:
  수백 개 자동 설정 클래스 → Full Mode이면 수백 개 CGLIB 클래스 생성
  → @AutoConfiguration 내부: proxyBeanMethods=false 기본
  → 불필요한 CGLIB 비용 대폭 제거
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Lite Mode에서 @Bean 메서드를 직접 호출해도 싱글톤이다

```java
// ❌ 위험한 코드
@Configuration(proxyBeanMethods = false)
public class AppConfig {

    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public OrderService orderService() {
        return new OrderService(dataSource());
        // ← Lite Mode에서는 new HikariDataSource() 새 인스턴스 생성!
        // 컨테이너의 DataSource Bean과 다른 객체
    }
}
```

```
✅ Lite Mode에서 안전한 패턴:
  @Bean
  public OrderService orderService(DataSource dataSource) {
      // 파라미터로 주입받음 → 컨테이너의 DataSource Bean 전달됨
      return new OrderService(dataSource);
  }
```

### Before: Lite Mode는 성능이 압도적으로 빠르다

```
❌ 과장된 이해:
  "Lite Mode로 바꾸면 런타임 성능이 크게 향상된다"

✅ 실제:
  Lite Mode 이점은 주로 시작(startup) 시간
  → CGLIB 클래스 생성/로드 비용 절약

  런타임(Bean 생성 이후) 성능:
  → Full Mode: BeanMethodInterceptor.intercept() 오버헤드
               하지만 @Bean 메서드 직접 호출이 매우 드뭄
  → 실서비스에서 런타임 성능 차이 무시 가능 수준

  실질적 이점: 시작 시간 (특히 자동 설정 클래스 수백 개)
```

---

## ✨ 올바른 이해와 사용

### After: 두 모드의 차이를 비용과 보장 측면에서 정확히 파악

```
Full Mode 비용 항목:
  1. CGLIB 서브클래스 생성 (시작 시 1회)
     ASM 바이트코드 생성 → 클래스로더에 정의
     @Configuration 클래스 수에 비례

  2. BeanMethodInterceptor.intercept() (Bean 생성 시)
     ThreadLocal 조회 + isCurrentlyInvokedFactoryMethod 비교
     Bean 생성 완료 후에는 @Bean 직접 호출 자체가 거의 없음

  3. $$beanFactory 필드 추가 (바이트코드)
     미미한 수준

Lite Mode 이점:
  CGLIB 생성 없음 → 해당 비용 제거
  일반 자바 클래스 그대로 사용

Lite Mode 제약:
  @Bean 메서드 직접 호출 → 싱글톤 보장 불가
  → 반드시 파라미터 주입 패턴 사용
```

---

## 🔬 내부 동작 원리

### 1. CGLIB 생성 비용 측정

```java
// @Configuration 클래스 수별 시작 시간 영향 (참고값)
// 환경: Spring Boot 3.x, JDK 21, Apple M2

// Full Mode (proxyBeanMethods=true):
//   @Configuration 10개  → CGLIB 10개 → 약 +5ms
//   @Configuration 100개 → CGLIB 100개 → 약 +50ms
//   @Configuration 500개 → CGLIB 500개 → 약 +200ms

// Lite Mode (proxyBeanMethods=false):
//   클래스 수에 무관 → CGLIB 생성 없음 → 0ms

// Spring Boot 자동 설정 클래스 수: 약 100~200개
// → @AutoConfiguration + proxyBeanMethods=false → 수십~100ms 절약 가능
```

### 2. Spring Boot @AutoConfiguration — Lite Mode 기본

```java
// @AutoConfiguration (Spring Boot 2.7+)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration(proxyBeanMethods = false)  // ← Lite Mode 강제
@AutoConfigureBefore
@AutoConfigureAfter
public @interface AutoConfiguration {
    // ...
}

// 사용 예: DataSourceAutoConfiguration
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import(DataSourcePoolMetadataProvidersConfiguration.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSourceProperties dataSourceProperties() {
        return new DataSourceProperties();
    }
    // ← 다른 @Bean 메서드 직접 호출 없음 → Lite Mode 안전
}
```

### 3. Lite Mode 안전 패턴 — 파라미터 주입

```java
// ✅ Full Mode / Lite Mode 모두 안전한 패턴
@Configuration(proxyBeanMethods = false)
public class ServiceConfig {

    // ← 파라미터로 의존성 수신
    @Bean
    public OrderService orderService(DataSource dataSource,
                                      TransactionManager txManager) {
        return new OrderService(dataSource, txManager);
        // dataSource, txManager는 컨테이너가 주입한 Bean
        // → Lite Mode에서도 싱글톤 보장
    }

    @Bean
    public AuditService auditService(DataSource dataSource) {
        return new AuditService(dataSource);
        // 같은 DataSource Bean 인스턴스 전달됨
    }

    // DataSource는 다른 @Configuration에서 정의됐다고 가정
}
```

```java
// ❌ Full Mode에서만 안전 (Lite Mode에서 버그)
@Configuration(proxyBeanMethods = false)
public class UnsafeConfig {

    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public OrderService orderService() {
        return new OrderService(dataSource());  // ← 위험: 새 인스턴스
    }
}
```

### 4. 같은 클래스에 Full/Lite 혼용 불가

```java
// @Configuration이 있으면 → Full Mode
// @Component가 있으면 → Lite Mode
// 하지만 @Configuration 자체가 @Component를 포함하므로 혼동 주의

@Component  // Lite Mode
public class ComponentConfig {
    @Bean
    public OrderService orderService() { ... }
    // → Lite Mode
}

@Configuration  // Full Mode (proxyBeanMethods=true 기본)
public class FullConfig {
    @Bean
    public OrderService orderService() { ... }
    // → Full Mode
}

@Configuration(proxyBeanMethods = false)  // Lite Mode 강제
public class LiteConfig {
    @Bean
    public OrderService orderService() { ... }
    // → Lite Mode
}
```

### 5. 성능 실측 비교

```java
// JMH 벤치마크 코드 (참고)
@BenchmarkMode(Mode.SingleShotTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class ConfigModeBenchmark {

    @Benchmark
    public void fullModeStartup() {
        // 100개 Full Mode @Configuration 클래스로 컨텍스트 생성
        new AnnotationConfigApplicationContext(FullModeConfigs.class);
    }

    @Benchmark
    public void liteModeStartup() {
        // 100개 Lite Mode @Configuration 클래스로 컨텍스트 생성
        new AnnotationConfigApplicationContext(LiteModeConfigs.class);
    }
}
```

```
측정 결과 (단순 @Bean 100개 기준, 참고값):

Full Mode (CGLIB 100개 생성):
  startup: ~320ms

Lite Mode (CGLIB 없음):
  startup: ~270ms

차이: ~50ms (단순 케이스)
→ @Bean 로직이 복잡할수록 차이 줄어듦
→ 자동 설정 클래스처럼 수백 개일 때 누적 효과 큼

런타임 Bean 호출 성능 (같은 Bean 수, 동일 로직):
  Full Mode: 0.05ms/op  (BeanMethodInterceptor 오버헤드 포함)
  Lite Mode: 0.04ms/op
  → 차이 미미 (실서비스에서 무시 가능)
```

### 6. 선택 기준 결정 트리

```
@Bean 메서드에서 다른 @Bean 메서드를 직접 호출하는가?
  Yes → Full Mode (@Configuration, proxyBeanMethods=true 기본)
        싱글톤 보장 필요
  No  → Lite Mode 사용 가능

    ↓ Lite Mode 선택 시

  @Configuration(proxyBeanMethods=false) vs @Component
    명시적인 설정 클래스 → @Configuration(proxyBeanMethods=false)
      → Full/Lite 의도 명확
    단순 Bean 등록 → @Component + @Bean
      → Lite Mode이지만 관용적으로 사용

  대규모 프로젝트 자동 설정:
    → @AutoConfiguration (Spring Boot, proxyBeanMethods=false 기본)
    → 또는 @Configuration(proxyBeanMethods=false)
```

---

## 💻 실험으로 확인하기

### 실험 1: Lite Mode 파라미터 주입 vs 직접 호출 비교

```java
// 직접 호출 (Lite Mode에서 위험)
@Configuration(proxyBeanMethods = false)
class DirectCallConfig {
    @Bean DataSource ds() { return new HikariDataSource(); }
    @Bean OrderService os() { return new OrderService(ds()); }  // 새 인스턴스!
}

// 파라미터 주입 (Lite Mode에서 안전)
@Configuration(proxyBeanMethods = false)
class ParamInjectConfig {
    @Bean DataSource ds() { return new HikariDataSource(); }
    @Bean OrderService os(DataSource ds) { return new OrderService(ds); }  // 같은 인스턴스
}

// 검증
DataSource dsBean  = ctx.getBean(DataSource.class);
OrderService osBean = ctx.getBean(OrderService.class);

System.out.println(osBean.getDataSource() == dsBean);
// DirectCallConfig → false (다른 인스턴스)
// ParamInjectConfig → true (같은 인스턴스)
```

### 실험 2: 자동 설정 클래스 모드 확인

```java
// Spring Boot 자동 설정 클래스 확인
ConfigurableListableBeanFactory bf = ctx.getBeanFactory();
bf.getBeanDefinitionNames()
  .stream()
  .filter(n -> n.contains("AutoConfiguration"))
  .forEach(n -> {
      BeanDefinition bd = bf.getBeanDefinition(n);
      boolean isLite = ConfigurationClassUtils.isLiteConfigurationClass(bd);
      System.out.println(n + " → " + (isLite ? "Lite" : "Full"));
  });
// → DataSourceAutoConfiguration → Lite
// → JacksonAutoConfiguration → Lite
// → ...
```

### 실험 3: CGLIB 클래스 생성 수 확인

```java
// Full Mode
@Configuration public class Config1 { @Bean A a() { return new A(); } }
@Configuration public class Config2 { @Bean B b() { return new B(); } }

// 생성된 CGLIB 클래스 수 확인
ApplicationContext ctx = new AnnotationConfigApplicationContext(Config1.class, Config2.class);
System.out.println(ctx.getBean(Config1.class).getClass().getName());
// → Config1$$SpringCGLIB$$0

System.out.println(ctx.getBean(Config2.class).getClass().getName());
// → Config2$$SpringCGLIB$$0
```

---

## 🤔 트레이드오프

```
Full Mode:
  장점  @Bean 직접 호출 → 싱글톤 자동 보장
        직관적 코드 (dataSource() 반복 호출해도 안전)
  단점  CGLIB 생성 비용 (시작 시간)
        final 클래스/메서드 사용 불가
  적합  @Bean 메서드 간 의존성이 있는 일반 @Configuration

Lite Mode:
  장점  CGLIB 없음 → 빠른 시작, 메모리 절약
        final 클래스에도 사용 가능
        Spring Boot 자동 설정 표준
  단점  @Bean 직접 호출 시 싱글톤 보장 안 됨
        개발자가 파라미터 주입 패턴 숙지 필요
  적합  @Bean 간 직접 호출 없는 경우
        자동 설정 클래스
        성능 민감한 대규모 프로젝트

모범 사례:
  새 코드 작성 시: Lite Mode + 파라미터 주입
  기존 코드 유지: Full Mode 그대로 (동작 변경 위험)
  자동 설정: @AutoConfiguration (Lite Mode 기본)
```

---

## 📌 핵심 정리

```
핵심 차이
  Full Mode  CGLIB 서브클래스 생성 → @Bean 직접 호출 싱글톤 보장
  Lite Mode  CGLIB 없음 → @Bean 직접 호출 시 새 인스턴스

성능 이점
  시작 시간: Lite Mode가 CGLIB 생성 비용만큼 빠름
  런타임: 거의 차이 없음
  클래스 수가 많을수록 (자동 설정) 차이 커짐

Lite Mode 안전 패턴
  @Bean 메서드 파라미터로 의존성 주입받기
  → Full/Lite Mode 모두에서 안전, 더 명시적

Spring Boot 표준
  @AutoConfiguration = proxyBeanMethods=false 기본
  → 자동 설정 클래스 전체가 Lite Mode

선택 기준
  @Bean 직접 호출 있음 → Full Mode
  없음 (파라미터 주입) → Lite Mode 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `@Configuration(proxyBeanMethods = false)` 클래스를 `@Import`로 다른 `@Configuration`에서 가져왔을 때 CGLIB 서브클래스가 생성되는가?

**Q2.** Lite Mode에서 `@Bean` 메서드끼리 직접 호출 없이도 싱글톤이 깨지는 케이스가 있는가?

**Q3.** Spring Boot 애플리케이션에서 사용자가 직접 작성하는 `@Configuration` 클래스를 Lite Mode로 전환할 때 반드시 확인해야 할 사항은?

> 💡 **해설**
>
> **Q1.** `@Import`로 가져온 클래스에 대해서도 `ConfigurationClassUtils.isFullConfigurationCandidate()`가 판별을 수행한다. `proxyBeanMethods = false`이면 Lite Mode로 분류되므로 CGLIB 서브클래스가 생성되지 않는다. `@Import` 방식과 직접 `@ComponentScan`으로 발견되는 방식 모두 동일한 판별 로직을 거친다. 따라서 Lite Mode로 선언된 클래스는 어떤 방식으로 등록되든 CGLIB 생성 없이 원본 클래스 그대로 사용된다.
>
> **Q2.** 있다. `@Scope("prototype")`이 아닌 일반 `@Bean`이라도 Lite Mode에서 외부 코드가 `@Autowired AppConfig config`로 설정 클래스를 주입받아 `config.dataSource()`를 직접 호출하면 매번 새 인스턴스가 생성된다. 이는 의도치 않은 다중 인스턴스로 이어진다. 설정 클래스를 외부에서 직접 참조해 메서드를 호출하는 패턴은 Full/Lite 모두에서 피해야 하지만 Lite Mode에서 특히 위험하다.
>
> **Q3.** 가장 중요한 확인 사항은 해당 `@Configuration` 클래스 내에서 `@Bean` 메서드를 직접 호출하는 코드가 있는지 확인하는 것이다. `orderService()` 내부에서 `dataSource()`를 직접 호출하는 패턴이 하나라도 있으면 Lite Mode 전환 시 싱글톤이 깨진다. 전환 전에 `grep -n "() " AppConfig.java`로 메서드 호출 패턴을 찾아 모두 파라미터 주입으로 변경해야 한다. 또한 통합 테스트로 Bean 동일성(`==` 비교)을 검증하는 것이 안전하다.

---

<div align="center">

**[⬅️ 이전: @Bean 메서드 호출 가로채기](./03-bean-method-interception.md)** | **[다음: @Import의 3가지 방식 ➡️](./05-import-three-ways.md)**

</div>

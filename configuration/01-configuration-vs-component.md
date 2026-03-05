# @Configuration vs @Component — Full Mode vs Lite Mode의 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Configuration`과 `@Component`의 근본적인 차이는 무엇인가?
- Full Mode와 Lite Mode는 내부에서 어떻게 구분되는가?
- `@Bean` 메서드를 같은 클래스 안에서 직접 호출하면 무슨 일이 생기는가?
- `ConfigurationClassUtils.isFullConfigurationClass()`가 판별하는 기준은?
- CGLIB 서브클래스가 생성되는 조건은 정확히 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Java Config에서 @Bean 메서드를 직접 호출하면 싱글톤이 깨진다

```java
@Configuration
public class AppConfig {

    @Bean
    public OrderService orderService() {
        return new OrderService(dataSource());  // dataSource() 직접 호출
    }

    @Bean
    public AuditService auditService() {
        return new AuditService(dataSource());  // dataSource() 또 직접 호출
    }

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();  // 매번 새 인스턴스?
    }
}
```

```
직관적 문제:
  dataSource()를 두 번 호출 → new HikariDataSource() 두 번 생성?
  → 싱글톤 Bean이 두 개?

@Configuration Full Mode의 답:
  dataSource() 직접 호출 → CGLIB 인터셉터가 가로챔
  → 이미 컨테이너에 등록된 DataSource Bean 반환
  → 항상 같은 인스턴스 → 싱글톤 보장
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Bean 메서드는 어디서 호출해도 같은 객체를 반환한다

```java
// ❌ Lite Mode에서는 싱글톤 보장 안 됨
@Component  // @Configuration 아님
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    @Bean
    public OrderService orderService() {
        return new OrderService(dataSource());
        // → 새 HikariDataSource() 생성
        // → 컨테이너의 dataSource Bean과 다른 인스턴스!
    }
}
```

```
✅ Full Mode (@Configuration):
  CGLIB 서브클래스가 dataSource() 호출을 가로채
  → 컨테이너에 이미 등록된 Bean 반환 → 싱글톤

✅ Lite Mode (@Component, @Bean):
  CGLIB 없음 → dataSource() 그냥 실행 → 새 객체 생성
  → orderService의 DataSource ≠ 컨테이너의 DataSource Bean
  → 의도치 않은 다중 인스턴스
```

### Before: @Configuration을 쓰면 항상 CGLIB 프록시가 생성된다

```
❌ 잘못된 이해:
  "@Configuration 붙이면 무조건 CGLIB 서브클래스"

✅ 실제:
  @Configuration(proxyBeanMethods = false) → Lite Mode로 강제
  → CGLIB 생성 안 함
  → @Bean 메서드 직접 호출 시 싱글톤 보장 안 됨
  → 단, 다른 @Bean 메서드를 호출하지 않는 독립적 @Bean이라면 안전
```

---

## ✨ 올바른 이해와 사용

### After: Full Mode / Lite Mode 판별 기준 명확히 파악

```
Full Mode (@Configuration):
  METADATA 속성 "configurationClass" = "full"
  → ConfigurationClassEnhancer가 CGLIB 서브클래스 생성
  → @Bean 메서드 호출 → BeanMethodInterceptor 가로챔 → 싱글톤 보장

Lite Mode:
  다음 조건 중 하나:
  a. @Component, @ComponentScan, @Import, @ImportResource 선언
  b. @Bean 메서드만 있는 클래스 (위 어노테이션 없이)
  c. @Configuration(proxyBeanMethods = false)

  METADATA 속성 "configurationClass" = "lite"
  → CGLIB 생성 안 함 → @Bean 메서드 = 일반 메서드
  → @Bean 직접 호출 시 새 인스턴스 생성
```

---

## 🔬 내부 동작 원리

### 1. ConfigurationClassUtils — Full / Lite 판별

```java
// ConfigurationClassUtils.java
public abstract class ConfigurationClassUtils {

    private static final String CONFIGURATION_CLASS_FULL  = "full";
    private static final String CONFIGURATION_CLASS_LITE  = "lite";
    private static final String CONFIGURATION_CLASS_ATTRIBUTE =
        Conventions.getQualifiedAttributeName(
            ConfigurationClassPostProcessor.class, "configurationClass");

    // BeanDefinition이 @Configuration 후보인지, Full/Lite 판별 후 속성 저장
    public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {

        // 인터페이스는 후보 아님
        if (metadata.isInterface()) return false;

        // Full Mode 조건: @Configuration 직접 선언
        // (proxyBeanMethods = true, 기본값)
        if (isFullConfigurationCandidate(metadata)) {
            metadata.getAnnotationAttributes(Configuration.class.getName())
                    .put(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
            return true;
        }

        // Lite Mode 조건
        if (isLiteConfigurationCandidate(metadata)) {
            metadata.getAnnotationAttributes(...)
                    .put(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
            return true;
        }

        return false;
    }

    // Full Mode: @Configuration(proxyBeanMethods=true) 또는 기본
    static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
        return metadata.isAnnotated(Configuration.class.getName());
        // @Configuration이 붙어있고 proxyBeanMethods = true (기본)
    }

    // Lite Mode: @Component 계열이거나 @Bean 메서드가 있는 클래스
    static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
        // @Component, @ComponentScan, @Import, @ImportResource 중 하나
        for (String indicator : LITE_CONFIGURATION_CLASS_INDICATORS) {
            if (metadata.isAnnotated(indicator)) return true;
        }
        // 또는 @Bean 메서드가 하나라도 있는 경우
        return metadata.hasAnnotatedMethods(Bean.class.getName());
    }

    private static final Set<String> LITE_CONFIGURATION_CLASS_INDICATORS = Set.of(
        Component.class.getName(),
        ComponentScan.class.getName(),
        Import.class.getName(),
        ImportResource.class.getName()
    );
}
```

### 2. proxyBeanMethods = false — Lite Mode 강제

```java
// @Configuration 어노테이션 정의
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

    String value() default "";

    // proxyBeanMethods = false → Lite Mode 강제
    // @Bean 메서드 간 직접 호출 없을 때 사용 → CGLIB 오버헤드 제거
    boolean proxyBeanMethods() default true;  // 기본 true = Full Mode
}
```

```java
// proxyBeanMethods 체크
static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
    Map<String, Object> attributes =
        metadata.getAnnotationAttributes(Configuration.class.getName());

    // @Configuration이 있고 proxyBeanMethods = true면 Full Mode
    return (attributes != null
        && !Boolean.FALSE.equals(attributes.get("proxyBeanMethods")));
}
```

### 3. @Bean 메서드 직접 호출 — Full vs Lite 비교

```java
// Full Mode (@Configuration, proxyBeanMethods = true)
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        System.out.println("DataSource 생성");
        return new HikariDataSource();
    }

    @Bean
    public OrderService orderService() {
        // CGLIB 인터셉터가 가로챔
        // → 컨테이너의 dataSource Bean 반환 (싱글톤)
        // → "DataSource 생성" 출력 안 됨 (이미 있으므로)
        return new OrderService(dataSource());
    }

    @Bean
    public AuditService auditService() {
        // 마찬가지로 인터셉터 → 같은 DataSource 인스턴스
        return new AuditService(dataSource());
    }
}
// 결과: "DataSource 생성" 1번 출력, 모두 같은 DataSource 인스턴스
```

```java
// Lite Mode (@Configuration(proxyBeanMethods = false))
@Configuration(proxyBeanMethods = false)
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        System.out.println("DataSource 생성");
        return new HikariDataSource();
    }

    @Bean
    public OrderService orderService() {
        // CGLIB 없음 → dataSource() 그냥 호출 → 새 인스턴스
        return new OrderService(dataSource());
        // "DataSource 생성" 출력 됨!
    }

    @Bean
    public AuditService auditService() {
        return new AuditService(dataSource());
        // "DataSource 생성" 또 출력!
    }
}
// 결과: "DataSource 생성" 3번 출력, 각각 다른 DataSource 인스턴스
```

### 4. CGLIB 서브클래스 생성 — 어느 단계에서?

```java
// ConfigurationClassPostProcessor.enhanceConfigurationClasses()
// → refresh()의 postProcessBeanFactory() 단계에서 실행
// → invokeBeanFactoryPostProcessors() 이후, finishBeanFactoryInitialization() 이전

public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
    Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();

    for (String beanName : beanFactory.getBeanDefinitionNames()) {
        AbstractBeanDefinition beanDef =
            (AbstractBeanDefinition) beanFactory.getBeanDefinition(beanName);

        // Full Mode BeanDefinition만 선별
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
            configBeanDefs.put(beanName, beanDef);
        }
    }

    if (configBeanDefs.isEmpty()) return;

    // ConfigurationClassEnhancer로 CGLIB 서브클래스 생성
    ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
    for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
        AbstractBeanDefinition beanDef = entry.getValue();
        Class<?> configClass = beanDef.getBeanClass();

        // CGLIB 서브클래스 생성
        Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);

        if (configClass != enhancedClass) {
            // BeanDefinition의 클래스를 CGLIB 서브클래스로 교체
            beanDef.setBeanClass(enhancedClass);
            // → Bean 인스턴스 생성 시 CGLIB 서브클래스가 인스턴스화됨
        }
    }
}
```

```
처리 순서:
  1. invokeBeanFactoryPostProcessors()
     → ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
     → @ComponentScan, @Import, @Bean 등 파싱 → BeanDefinition 등록

  2. ConfigurationClassPostProcessor.postProcessBeanFactory()
     → enhanceConfigurationClasses()
     → Full Mode BeanDefinition의 beanClass를 CGLIB 서브클래스로 교체

  3. finishBeanFactoryInitialization()
     → Singleton Bean 인스턴스 생성
     → AppConfig 생성 시 AppConfig$$SpringCGLIB$$0 인스턴스 생성됨
```

### 5. CGLIB 서브클래스 구조

```java
// 생성된 클래스 (개념적 표현)
public class AppConfig$$SpringCGLIB$$0 extends AppConfig {

    // 스프링 컨테이너 참조 (BeanFactory)
    private BeanFactory $$beanFactory;

    @Override
    public DataSource dataSource() {
        // BeanMethodInterceptor가 이 메서드를 가로챔
        // → 상세 동작은 03 문서에서 다룸

        // 이미 컨테이너에 있으면 → 기존 Bean 반환
        // 없으면 → super.dataSource() 호출 → 실제 생성
        return (DataSource) $$beanFactory.getBean("dataSource");
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Full Mode vs Lite Mode 인스턴스 비교

```java
// Full Mode
@Configuration
public class FullConfig {
    @Bean DataSource ds() { return new HikariDataSource(); }
    @Bean OrderService os() { return new OrderService(ds()); }
    @Bean AuditService as() { return new AuditService(ds()); }
}

OrderService os = ctx.getBean(OrderService.class);
AuditService as = ctx.getBean(AuditService.class);
DataSource ds  = ctx.getBean(DataSource.class);

System.out.println(os.getDataSource() == ds);  // true (같은 인스턴스)
System.out.println(as.getDataSource() == ds);  // true (같은 인스턴스)
```

```java
// Lite Mode
@Configuration(proxyBeanMethods = false)
public class LiteConfig {
    @Bean DataSource ds() { return new HikariDataSource(); }
    @Bean OrderService os() { return new OrderService(ds()); }
    @Bean AuditService as() { return new AuditService(ds()); }
}

OrderService os = ctx.getBean(OrderService.class);
AuditService as = ctx.getBean(AuditService.class);
DataSource ds  = ctx.getBean(DataSource.class);

System.out.println(os.getDataSource() == ds);  // false! (다른 인스턴스)
System.out.println(as.getDataSource() == ds);  // false! (다른 인스턴스)
```

### 실험 2: CGLIB 서브클래스 클래스명 확인

```java
@Autowired AppConfig appConfig;
System.out.println(appConfig.getClass().getName());
// Full Mode: com.example.AppConfig$$SpringCGLIB$$0
// Lite Mode: com.example.AppConfig
```

### 실험 3: BeanDefinition의 beanClass 변경 확인

```java
ConfigurableListableBeanFactory bf = ctx.getBeanFactory();
BeanDefinition bd = bf.getBeanDefinition("appConfig");

System.out.println(bd.getBeanClassName());
// Full Mode: com.example.config.AppConfig$$SpringCGLIB$$0
// Lite Mode: com.example.config.AppConfig
```

---

## 🤔 트레이드오프

```
Full Mode (@Configuration, proxyBeanMethods=true):
  장점  @Bean 메서드 간 직접 호출 → 싱글톤 보장
        직관적인 Java Config 작성 가능
  단점  CGLIB 서브클래스 생성 비용 (시작 시간 미미하게 증가)
        final 클래스 / final 메서드에 사용 불가
  사용  @Bean 메서드끼리 서로 참조하는 경우 (일반적인 @Configuration)

Lite Mode (@Configuration(proxyBeanMethods=false) 또는 @Component):
  장점  CGLIB 없음 → 시작 시간 약간 단축
        final 클래스에도 사용 가능
        Spring Boot 내부 자동 설정 클래스에서 주로 사용
  단점  @Bean 메서드 직접 호출 → 새 인스턴스 생성 → 싱글톤 깨짐
  사용  @Bean 메서드 간 직접 호출이 없는 경우
        성능 최적화가 필요한 자동 설정 클래스

Spring Boot 자동 설정 클래스:
  @AutoConfiguration (내부적으로 proxyBeanMethods=false)
  → 수백 개 자동 설정 클래스에 CGLIB 없음 → 시작 시간 최적화
```

---

## 📌 핵심 정리

```
Full Mode vs Lite Mode 판별
  Full: @Configuration(proxyBeanMethods=true, 기본)
        → "configurationClass" = "full"
        → CGLIB 서브클래스 생성

  Lite: @Component / @ComponentScan / @Import / @ImportResource
        또는 @Bean 메서드만 있는 클래스
        또는 @Configuration(proxyBeanMethods=false)
        → "configurationClass" = "lite"
        → CGLIB 없음

CGLIB 생성 시점
  ConfigurationClassPostProcessor.postProcessBeanFactory()
  → enhanceConfigurationClasses()
  → Full Mode BeanDefinition의 beanClass를 CGLIB 서브클래스로 교체

@Bean 메서드 직접 호출
  Full Mode: BeanMethodInterceptor 가로챔 → 컨테이너 Bean 반환 → 싱글톤
  Lite Mode: 그냥 실행 → 새 인스턴스 → 싱글톤 보장 안 됨

실무 선택
  @Bean끼리 서로 호출 → Full Mode (@Configuration)
  독립 @Bean, 자동 설정 → Lite Mode (proxyBeanMethods=false)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Configuration(proxyBeanMethods = false)` 상태에서 `@Bean` 메서드 간 싱글톤을 보장하려면 어떻게 해야 하는가?

**Q2.** `@Configuration` 클래스가 `final`이면 어떤 일이 발생하는가?

**Q3.** `@SpringBootApplication`이 붙은 메인 클래스에 `@Bean` 메서드를 선언하면 Full Mode인가 Lite Mode인가?

> 💡 **해설**
>
> **Q1.** `@Bean` 메서드를 직접 호출하는 대신 파라미터로 의존성을 주입받으면 된다. 스프링은 `@Bean` 메서드의 파라미터를 자동으로 Bean으로 주입해준다. 예를 들어 `@Bean public OrderService orderService(DataSource ds)` 처럼 선언하면 컨테이너의 `DataSource` Bean이 주입된다. 이 방식은 Lite Mode에서도 싱글톤을 보장하며, Full Mode에서도 더 명시적이고 테스트하기 쉬운 코드가 된다.
>
> **Q2.** `ConfigurationClassEnhancer`가 CGLIB 서브클래스를 만들려면 원본 클래스를 상속해야 한다. `final` 클래스는 상속 불가이므로 CGLIB 서브클래스 생성 시 `IllegalArgumentException` 또는 `CannotLoadBeanClassException`이 발생한다. Spring Boot는 `@SpringBootApplication`의 자동 설정 클래스에 `final`을 허용하기 위해 `proxyBeanMethods=false`(Lite Mode)를 기본으로 사용하는 경우가 있다. `@Configuration` 클래스를 `final`로 선언하지 않는 것이 기본 규칙이다.
>
> **Q3.** `@SpringBootApplication`은 내부에 `@SpringBootConfiguration`을 포함하고, `@SpringBootConfiguration`은 `@Configuration`을 포함한다. `proxyBeanMethods`는 기본값 `true`이므로 `@SpringBootApplication`이 붙은 메인 클래스에 `@Bean` 메서드를 선언하면 Full Mode로 처리된다. 즉 CGLIB 서브클래스가 생성되고 `@Bean` 메서드 직접 호출 시 싱글톤이 보장된다. 단 메인 클래스를 `@Configuration`으로 활용하는 것은 관심사 분리 측면에서 별도 `@Configuration` 클래스를 사용하는 편이 낫다.

---

<div align="center">

**[다음: CGLIB 프록시가 적용되는 이유 ➡️](./02-cglib-proxy-on-configuration.md)**

</div>

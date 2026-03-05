# @Value("${...}") vs @Value("#{...}") — 처리 시점과 동작 경로의 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `${...}`와 `#{...}`는 각각 어느 단계에서, 어떤 컴포넌트가 처리하는가?
- `PropertySourcesPlaceholderConfigurer`가 `BeanFactoryPostProcessor`인 이유는?
- `${...}` 안에 `#{...}`를 중첩하면 어떻게 처리되는가?
- `@Value` 어노테이션이 없는 생성자 파라미터에서 값 주입은 어떻게 동작하는가?
- `${...}` 기본값 구문 `${key:default}`와 SpEL Elvis 연산자 `#{expr ?: default}`의 차이는?

---

## 🔍 왜 이게 존재하는가

### 두 가지 주입 요구사항은 서로 다른 수준이다

```java
// ${...} — 외부 설정 값 주입
@Value("${server.port}")          // application.properties의 값
private int serverPort;

@Value("${db.url:jdbc:h2:mem:}")  // 없으면 기본값
private String dbUrl;

// #{...} — 런타임 표현식 계산
@Value("#{systemProperties['os.name']}")   // 시스템 프로퍼티
@Value("#{@dataSource.maxPoolSize * 2}")   // Bean 프로퍼티 참조
@Value("#{T(Math).PI}")                    // 클래스 정적 값
private double piValue;
```

```
처리 수준 차이:
  ${...}: 텍스트 치환 — PropertySource에서 키로 값 조회
          Bean 정의 단계에서 처리
          결과: 항상 String → 이후 타입 변환

  #{...}: 표현식 평가 — SpEL AST 파싱 후 컨텍스트에서 평가
          Bean 초기화 단계에서 처리
          결과: 표현식 타입에 따라 다양
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: ${...}와 #{...}는 처리 시점이 같다

```
❌ 잘못된 이해:
  "두 구문 모두 @Autowired 처리 시점에 함께 평가된다"

✅ 실제:
  ${...}: BeanFactoryPostProcessor 단계
          → PropertySourcesPlaceholderConfigurer.postProcessBeanFactory()
          → BeanDefinition의 propertyValues 치환
          → Bean 인스턴스 생성 전

  #{...}: Bean 초기화 단계 (인스턴스 생성 후)
          → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
          → BeanExpressionResolver.evaluate()
          → 이미 Bean들이 존재하는 시점
```

### Before: ${...}에서 Bean 프로퍼티를 참조할 수 있다

```java
// ❌ 잘못된 시도
@Value("${@someBean.value}")   // 동작 안 함
@Value("${myBean.property}")   // 동작 안 함

// ${...}는 PropertySource(properties 파일, 환경 변수 등)에서만 값 조회
// Bean 참조는 #{...}로만 가능

// ✅ 올바른 Bean 참조
@Value("#{@someBean.value}")
private String beanValue;
```

---

## ✨ 올바른 이해와 사용

### After: 두 구문의 처리 경로 전체 비교

```
${server.port} 처리 경로:

  refresh()
    → invokeBeanFactoryPostProcessors()
      → PropertySourcesPlaceholderConfigurer.postProcessBeanFactory()
        → BeanDefinition 순회
        → @Value("${server.port}") 발견
        → PropertySourcesPropertyResolver.resolveRequiredPlaceholders()
          → environment.getProperty("server.port")
          → "8080" 반환
        → BeanDefinition.propertyValues에 "8080" 저장
    (Bean 생성 전 BeanDefinition 수정 완료)

  → finishBeanFactoryInitialization()
    → MyBean 인스턴스 생성
    → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
      → @Value("8080") → int 타입 변환 → 필드에 주입


#{@dataSource.maxPoolSize * 2} 처리 경로:

  refresh()
    → invokeBeanFactoryPostProcessors()
      → PropertySourcesPlaceholderConfigurer: #{...}는 건드리지 않음
         (기본적으로 #{로 시작하는 값은 PlaceholderConfigurer가 처리 안 함)

  → finishBeanFactoryInitialization()
    → dataSource Bean 먼저 생성 (의존 관계 때문)
    → MyBean 인스턴스 생성
    → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
      → @Value 값 = "#{@dataSource.maxPoolSize * 2}"
      → AbstractBeanFactory.evaluateBeanDefinitionString()
        → BeanExpressionResolver.evaluate()
          → SpelExpressionParser.parseExpression(...)
          → StandardBeanExpressionContext로 평가
          → @dataSource Bean 조회 → .maxPoolSize → * 2
          → 결과값 반환
      → 결과를 int 타입으로 변환 → 필드에 주입
```

---

## 🔬 내부 동작 원리

### 1. PropertySourcesPlaceholderConfigurer — ${...} 처리

```java
// PropertySourcesPlaceholderConfigurer implements BeanFactoryPostProcessor
// → refresh() 중 invokeBeanFactoryPostProcessors()에서 실행
// → Bean 인스턴스 생성 전에 BeanDefinition 문자열 치환

public class PropertySourcesPlaceholderConfigurer
        extends PlaceholderConfigurerSupport implements EnvironmentAware {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Environment의 PropertySource 목록 기반으로 PropertyResolver 구성
        MutablePropertySources appliedSources = new MutablePropertySources();
        // application.properties, 환경 변수, JVM 시스템 프로퍼티 순서
        appliedSources.addLast(new PropertySourcesPropertyResolver(this.environment));

        // BeanDefinition 내 플레이스홀더 치환
        processProperties(beanFactory,
            new PropertySourcesPropertyResolver(appliedSources));
    }
}

// PlaceholderConfigurerSupport.processProperties()
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
                                   StringValueResolver valueResolver) {
    BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
    // 모든 BeanDefinition 순회
    for (String beanName : beanFactoryToProcess.getBeanDefinitionNames()) {
        BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(beanName);
        visitor.visitBeanDefinition(bd);
        // → propertyValues, constructorArgValues의 모든 문자열에서
        //   ${...} 패턴 탐색 → PropertySource에서 값 조회 → 치환
    }
}
```

```
PropertySource 조회 우선순위 (기본):
  1. System Properties (JVM -D 옵션)
  2. System Environment (OS 환경 변수)
  3. application.properties / application.yml
  4. @PropertySource 지정 파일
  → 우선순위 높은 것이 낮은 것 덮어씀
```

### 2. BeanExpressionResolver — #{...} 처리

```java
// AbstractBeanFactory.evaluateBeanDefinitionString()
protected Object evaluateBeanDefinitionString(String value, BeanDefinition bd) {
    if (this.beanExpressionResolver == null) return value;

    Scope scope = (bd != null ? getRegisteredScope(bd.getScope()) : null);

    return this.beanExpressionResolver.evaluate(value,
        new BeanExpressionContext(this, scope));
}

// StandardBeanExpressionResolver (기본 BeanExpressionResolver)
public class StandardBeanExpressionResolver implements BeanExpressionResolver {

    private final SpelExpressionParser expressionParser = new SpelExpressionParser();
    // 파싱 결과 캐시 (표현식 문자열 → Expression)
    private final Map<String, Expression> expressionCache = new ConcurrentHashMap<>();

    @Override
    public Object evaluate(String value, BeanExpressionContext evalContext) {
        if (!StringUtils.hasLength(value)) return value;

        // #{...} 패턴 확인
        String expression = extractExpression(value);
        if (expression == null) return value;  // #{...} 아니면 그대로 반환

        // 캐시에서 Expression 조회 (없으면 파싱)
        Expression expr = this.expressionCache.computeIfAbsent(expression,
            k -> this.expressionParser.parseExpression(k, TEMPLATE_PARSER_CONTEXT));

        // BeanExpressionContext 기반 EvaluationContext 구성
        StandardEvaluationContext sec = new StandardEvaluationContext(evalContext);
        sec.addPropertyAccessor(new BeanExpressionContextAccessor());
        // → @beanName → BeanFactory에서 조회 가능
        sec.setBeanResolver(new BeanFactoryResolver(evalContext.getBeanFactory()));
        // → T(Class) 사용 가능
        sec.setTypeLocator(new StandardTypeLocator(evalContext.getBeanFactory()
            .getBeanClassLoader()));

        return expr.getValue(sec);
    }
}
```

### 3. AutowiredAnnotationBeanPostProcessor — @Value 처리 흐름

```java
// AutowiredAnnotationBeanPostProcessor.postProcessProperties()
// → finishBeanFactoryInitialization() 중 Bean 초기화 단계

public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, ...) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    metadata.inject(bean, beanName, pvs);
    return pvs;
}

// @Value 필드 주입 과정
// AutowiredFieldElement.resolveFieldValue()
protected Object resolveFieldValue(Field field, Object bean, String beanName) {
    // @Value 값 ("${server.port}" 또는 "#{...}")
    DependencyDescriptor desc = new DependencyDescriptor(field, true);
    Object value = beanFactory.resolveDependency(desc, beanName, ...);
    return value;
}

// DefaultListableBeanFactory.resolveDependency()
// → @Value 어노테이션 값 추출
// → AbstractBeanFactory.resolveEmbeddedValue() 호출
//   → StringValueResolver(PropertySourcesPlaceholderConfigurer) → ${...} 치환
//   → evaluateBeanDefinitionString() → #{...} SpEL 평가
// → TypeConverter로 대상 타입 변환
```

### 4. ${...}와 #{...} 중첩 사용

```java
// ${...} 안에 #{...}: 동작 안 함
@Value("${#{expr}}")     // PropertySource에서 "#{expr}" 키를 찾음 → 실패

// #{...} 안에 ${...}: 동작함
@Value("#{${server.port} * 2}")
// 처리 순서:
// 1. PropertySourcesPlaceholderConfigurer: ${server.port} → "8080" 치환
//    결과: @Value("#{8080 * 2}")
// 2. SpEL 평가: 8080 * 2 = 16160

// 실용적 패턴: 환경 변수로 SpEL 표현식 조절
@Value("#{${myApp.multiplier:1} > 1 ? 'SCALED' : 'NORMAL'}")
private String mode;
```

### 5. 기본값 구문 비교

```java
// ${...} 기본값 — ':' 구분자
@Value("${server.port:8080}")
// → "server.port" 없으면 "8080" 사용
// → 항상 문자열 기반

// #{...} Elvis 연산자 — '?:' 구분자
@Value("#{environment['server.port'] ?: '8080'}")
// → environment['server.port']가 null이면 '8080' 사용
// → SpEL 평가 결과 기반 (타입 유연)

// ${...} 기본값의 실제 동작
// PropertyPlaceholderHelper.parseStringValue()
String propVal = this.propertyResolver.getProperty(placeholder);
if (propVal == null) {
    // ':' 뒤가 기본값
    String defaultValue = extractDefaultValue(placeholder);
    propVal = (defaultValue != null ? defaultValue : null);
}
```

### 6. Spring Boot의 @ConfigurationProperties와 @Value 비교

```java
// @Value — 개별 값 주입
@Value("${app.timeout}")
private int timeout;
@Value("${app.maxRetry}")
private int maxRetry;

// @ConfigurationProperties — 그룹 바인딩 (내부적으로 Binder 활용)
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private int timeout;
    private int maxRetry;
    // → app.timeout, app.maxRetry 자동 바인딩
    // SpEL 미사용, Relaxed Binding 지원 (camelCase ↔ kebab-case)
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 처리 시점 차이 확인

```java
@Component
public class ValueDemo {

    // ${...}: BeanDefinition 단계에서 치환 → 실제 값이 BeanDefinition에 저장됨
    @Value("${server.port:8080}")
    private int port;

    // #{...}: Bean 초기화 단계에서 평가
    @Value("#{T(System).currentTimeMillis()}")
    private long initTime;

    // 확인
    System.out.println(port);     // 8080 (또는 설정된 포트)
    System.out.println(initTime); // 초기화 시점의 타임스탬프
}
```

### 실험 2: 중첩 사용

```properties
# application.properties
multiplier=3
```

```java
@Value("#{${multiplier} * 100}")
private int scaledValue;
// → multiplier=3 → @Value("#{3 * 100}") → 300
```

### 실험 3: 기본값 비교

```java
// property 없을 때
@Value("${missing.key:DEFAULT}")
private String v1;  // "DEFAULT"

@Value("#{environment['missing.key'] ?: 'DEFAULT'}")
private String v2;  // "DEFAULT"

// null vs 빈 문자열 차이
@Value("${empty.key:}")   // empty.key="" → "" (빈 문자열)
@Value("${missing.key:}") // missing.key 없음 → "" (빈 문자열)
```

---

## 🤔 트레이드오프

```
${...} 사용 적합:
  외부 설정 파일/환경 변수 값 주입
  단순 문자열, 숫자 값
  환경별 다른 값 (dev/prod 분리)
  타입 변환은 ConversionService가 담당

#{...} 사용 적합:
  Bean 프로퍼티 참조 (@bean.property)
  런타임 계산 값 (Math.PI, currentTimeMillis)
  조건부 값 (삼항 연산자)
  컬렉션 필터링 (.?[condition])

둘 다 피해야 할 상황:
  복잡한 비즈니스 로직을 @Value SpEL로 표현
  → 가독성, 테스트, 유지보수 어려움
  → @Bean 메서드나 @PostConstruct에서 자바 코드로 처리

@ConfigurationProperties vs @Value:
  @ConfigurationProperties: 관련 설정 그룹화, 유효성 검증(JSR-303), IDE 지원
  @Value: 개별 값, SpEL 표현식 필요 시
```

---

## 📌 핵심 정리

```
${...} 처리 경로
  BeanFactoryPostProcessor 단계
  → PropertySourcesPlaceholderConfigurer.postProcessBeanFactory()
  → BeanDefinition 문자열에서 ${key} 탐색
  → PropertySource 계층에서 값 조회 → 문자열 치환
  → Bean 인스턴스 생성 전 처리 완료

#{...} 처리 경로
  Bean 초기화 단계
  → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
  → AbstractBeanFactory.evaluateBeanDefinitionString()
  → StandardBeanExpressionResolver.evaluate()
  → SpelExpressionParser → AST → BeanExpressionContext로 평가

중첩 순서
  #{${key}} → ${key}가 먼저 PlaceholderConfigurer에서 치환
  → 이후 #{치환된값} SpEL 평가 가능
  ${#{expr}} → 동작 안 함 (PlaceholderConfigurer는 #{...} 무시)

기본값
  ${key:default} — PropertySource 미존재 시 'default' 문자열
  #{expr ?: default} — SpEL null-safe Elvis 연산자
```

---

## 🤔 생각해볼 문제

**Q1.** `@Value("${db.password}")`에서 `db.password`가 존재하지 않으면 어떻게 되는가? 예외를 방지하려면?

**Q2.** `PropertySourcesPlaceholderConfigurer`가 `BeanFactoryPostProcessor`가 아닌 `BeanPostProcessor`로 구현됐다면 어떤 문제가 생겼을까?

**Q3.** `@Value("#{@orderService}")`처럼 Bean 자체를 주입하는 것과 `@Autowired OrderService orderService`의 차이는 무엇인가?

> 💡 **해설**
>
> **Q1.** 기본적으로 `PropertySourcesPlaceholderConfigurer`는 플레이스홀더를 해석할 수 없으면 `IllegalArgumentException`을 던진다. 방지 방법은 두 가지다. 첫째, 기본값 구문 사용: `@Value("${db.password:}")` — 없으면 빈 문자열. 둘째, `ignoreUnresolvablePlaceholders=true` 설정 — 단 이 경우 `${db.password}` 문자열이 그대로 값이 되어 타입 불일치 오류가 날 수 있다. Spring Boot의 `@ConfigurationProperties`는 `@Value`보다 유연한 바인딩과 유효성 검증을 제공하므로 선택적 설정 처리에 더 적합하다.
>
> **Q2.** `BeanPostProcessor`는 Bean 인스턴스 초기화 단계(생성 후)에 실행된다. 이 시점에는 이미 일부 Bean들이 `${...}` 플레이스홀더가 치환되지 않은 채 생성됐을 수 있다. `BeanFactoryPostProcessor`는 BeanDefinition 등록 완료 후, 인스턴스 생성 전에 실행되기 때문에 모든 Bean의 설정을 일괄적으로 치환할 수 있다. 특히 `@Configuration` 클래스 내 `@Bean` 메서드에 `@Value`가 있을 때 `BeanPostProcessor` 시점에서는 너무 늦어서 일부 Bean이 잘못된 값으로 생성될 수 있다.
>
> **Q3.** `@Value("#{@orderService}")`는 SpEL `BeanReference`를 통해 `BeanFactory.getBean("orderService")`를 호출해 Bean을 반환한다. `@Autowired`는 타입 기반으로 `DefaultListableBeanFactory.doResolveDependency()`를 통해 의존성을 해결한다. 실질적 차이는 거의 없지만 `@Value` SpEL 방식은 빈 이름이 하드코딩되고 타입 안전성이 없으며(Object 반환 후 캐스팅 필요), IDE의 자동완성과 리팩토링 지원이 약하다. Bean 주입에는 `@Autowired`나 생성자 주입이 권장된다.

---

<div align="center">

**[⬅️ 이전: SpEL 파싱 과정](./01-spel-parsing-process.md)** | **[다음: PropertyEditor vs Converter 변환 ➡️](./03-propertyeditor-vs-converter.md)**

</div>

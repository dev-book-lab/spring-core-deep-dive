# Constructor Binding의 장점 — 불변 객체 보장과 @ConfigurationProperties

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 생성자 바인딩이 필드 바인딩과 다른 기술적 이유는 무엇인가?
- `@ConfigurationProperties`에서 생성자 바인딩이 권장되는 이유는?
- `@ConstructorBinding`은 언제 필요하고 언제 생략 가능한가?
- 생성자 바인딩에서 기본값과 중첩 객체는 어떻게 처리되는가?
- 불변 설정 객체가 가져다주는 실질적 이점은 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 설정값이 중간에 바뀔 수 있다

```java
// 기존 필드 바인딩 방식
@ConfigurationProperties(prefix = "app.database")
@Component
public class DatabaseProperties {
    private String url;
    private int maxPoolSize;
    private Duration timeout;

    // 모든 필드에 getter/setter 필요
    public void setUrl(String url) { this.url = url; }
    public void setMaxPoolSize(int maxPoolSize) { this.maxPoolSize = maxPoolSize; }
    // ...
}
```

```
문제:
  setter가 있으면 애플리케이션 실행 중 언제든 값 변경 가능
  → 설정값은 "한 번 정해지면 불변"이어야 하는데
  → DatabaseProperties.setUrl("hack") 누군가 호출할 수 있음
  → 멀티스레드 환경에서 가시성 문제 가능

이상적인 설정 객체:
  생성 시점에 모든 값 확정
  이후 변경 불가 (final 필드)
  → 불변(Immutable) 설정 객체
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @ConfigurationProperties에서 setter가 반드시 필요하다

```java
// ❌ 잘못된 이해
// "스프링이 리플렉션으로 setter를 호출해야 하니까 setter 없으면 동작 안 한다"

@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private final String name;  // final → setter 불가?

    public AppProperties(String name) {
        this.name = name;
    }
    // setter 없음 → 바인딩 실패?

// ✅ 실제:
// 생성자 바인딩을 사용하면 setter 없이도 정상 바인딩됨
// Spring Boot 2.2+부터 지원
```

### Before: @ConstructorBinding을 항상 명시해야 한다

```java
// ❌ Spring Boot 2.x 시절의 습관
@ConfigurationProperties(prefix = "app")
@ConstructorBinding  // Spring Boot 3.x에서는 단일 생성자면 생략 가능
public class AppProperties {
    private final String name;
    // ...
}
```

```
Spring Boot 3.0 변경사항:
  단일 생성자 → @ConstructorBinding 생략 가능 (자동 감지)
  생성자 여러 개 → 명시 필요
```

---

## ✨ 올바른 이해와 사용

### After: 생성자 바인딩으로 완전한 불변 설정 객체

```java
@ConfigurationProperties(prefix = "app.database")
public class DatabaseProperties {
    private final String url;
    private final int maxPoolSize;
    private final Duration timeout;
    private final PoolProperties pool;  // 중첩 객체도 불변

    // Spring Boot 3.x: 단일 생성자면 @ConstructorBinding 생략 가능
    public DatabaseProperties(String url,
                               int maxPoolSize,
                               Duration timeout,
                               PoolProperties pool) {
        this.url = url;
        this.maxPoolSize = maxPoolSize;
        this.timeout = timeout;
        this.pool = pool;
    }

    // getter만 제공, setter 없음
    public String getUrl() { return url; }
    // ...
}

public record PoolProperties(int minIdle, int maxSize) {}
// record: 자동으로 생성자 바인딩 + 불변
```

---

## 🔬 내부 동작 원리

### 1. 바인딩 처리 클래스 — Binder와 BindConstructorProvider

```java
// Spring Boot ConfigurationPropertiesBindingPostProcessor
// → ConfigurationPropertiesBinder.bind()
// → Binder.bind()

// Binder.java — 생성자 vs setter 바인딩 분기
private <T> Object bindDataObject(ConfigurationPropertyName name,
                                   Bindable<T> target, ...) {

    // 생성자 바인딩 대상인지 확인
    Constructor<?> bindConstructor = BindConstructorProvider.DEFAULT
            .getBindConstructor(target, false);

    if (bindConstructor != null) {
        // 생성자 바인딩 경로
        return bindViaConstructor(name, target, bindConstructor, ...);
    }

    // setter 바인딩 경로 (기본)
    return bindViaSetters(name, target, ...);
}
```

```java
// BindConstructorProvider — 생성자 결정 로직
public Constructor<?> getBindConstructor(Bindable<?> bindable, boolean isNestedConstructorBinding) {

    Class<?> type = bindable.getType().resolve();

    // 1. Record 타입 → 자동으로 canonical 생성자 선택
    if (type.isRecord()) {
        return type.getDeclaredConstructors()[0];
    }

    // 2. @ConstructorBinding 명시된 생성자 탐색
    Constructor<?>[] constructors = type.getDeclaredConstructors();
    Constructor<?> annotated = findAnnotatedConstructor(constructors);
    if (annotated != null) return annotated;

    // 3. 생성자가 1개이고 @DefaultConstructor 없으면 → 생성자 바인딩
    if (constructors.length == 1 && hasNoDefaultConstructor(type)) {
        return constructors[0];
    }

    return null;  // null → setter 바인딩
}
```

### 2. 생성자 바인딩 실행 과정

```java
// bindViaConstructor() — 단순화
private <T> Object bindViaConstructor(ConfigurationPropertyName name,
                                       Bindable<T> target,
                                       Constructor<?> constructor, ...) {

    // 생성자 파라미터 각각 바인딩
    Parameter[] parameters = constructor.getParameters();
    Object[] args = new Object[parameters.length];

    for (int i = 0; i < parameters.length; i++) {
        Parameter parameter = parameters[i];
        String propertyName = getParameterName(parameter);  // 파라미터명 → 프로퍼티 키

        // 각 파라미터에 해당하는 프로퍼티 값 조회
        Object value = bind(name.append(propertyName),
                            Bindable.of(parameter.getParameterizedType()),
                            ...);

        // @DefaultValue 처리
        if (value == null) {
            value = getDefaultValue(parameter);
        }

        args[i] = value;
    }

    // 모든 파라미터 준비 완료 → 생성자 호출
    return BeanUtils.instantiateClass(constructor, args);
}
```

```
프로퍼티 키 → 파라미터명 변환:

application.yml:
  app:
    database:
      max-pool-size: 10   ← kebab-case

파라미터명: maxPoolSize    ← camelCase

스프링이 자동 변환:
  "max-pool-size" ↔ "maxPoolSize" ↔ "MAX_POOL_SIZE"
  RelaxedPropertyResolver가 처리
```

### 3. @DefaultValue — 기본값 처리

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private final String name;
    private final int timeout;
    private final List<String> allowedOrigins;

    public AppProperties(
            String name,
            @DefaultValue("30") int timeout,              // 기본값 30
            @DefaultValue({"*"}) List<String> allowedOrigins) {  // 기본값 ["*"]
        this.name = name;
        this.timeout = timeout;
        this.allowedOrigins = allowedOrigins;
    }
}
```

```yaml
# application.yml — name만 설정, 나머지는 기본값
app:
  name: "MyApp"
  # timeout → 기본값 30
  # allowedOrigins → 기본값 ["*"]
```

```
@DefaultValue 처리:
  바인딩 시 해당 파라미터 프로퍼티 값 없음
  → @DefaultValue 어노테이션 확인
  → 값 변환 (String "30" → int 30)
  → 생성자 인자로 전달

setter 바인딩과 차이:
  setter: 값 없으면 setter 미호출 → 필드 초기화값 유지
  생성자: 값 없으면 @DefaultValue 또는 null → 생성자에서 처리
```

### 4. 중첩 객체 바인딩

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private final DatabaseProperties database;
    private final CacheProperties cache;

    public AppProperties(DatabaseProperties database, CacheProperties cache) {
        this.database = database;
        this.cache = cache;
    }
}

// 중첩 객체도 생성자 바인딩
public class DatabaseProperties {
    private final String url;
    private final int maxPoolSize;

    public DatabaseProperties(String url, @DefaultValue("10") int maxPoolSize) {
        this.url = url;
        this.maxPoolSize = maxPoolSize;
    }
}
```

```yaml
app:
  database:
    url: jdbc:mysql://localhost:3306/mydb
    max-pool-size: 20
  cache:
    ttl: 300s
```

```
중첩 바인딩 흐름:
  AppProperties 생성자 파라미터 분석
  → DatabaseProperties 파라미터 발견
  → DatabaseProperties도 생성자 바인딩 대상 확인
  → 재귀적으로 DatabaseProperties 생성자 바인딩
  → 완성된 DatabaseProperties를 AppProperties 생성자에 전달
```

### 5. record를 활용한 최단 경로

```java
// Java record: 생성자 바인딩 + 불변 + getter 자동 생성
@ConfigurationProperties(prefix = "app.database")
public record DatabaseProperties(
    String url,
    @DefaultValue("10") int maxPoolSize,
    Duration timeout,
    PoolConfig pool
) {}

public record PoolConfig(
    @DefaultValue("2") int minIdle,
    @DefaultValue("10") int maxSize
) {}
```

```
record의 장점:
  canonical 생성자 자동 생성 → 생성자 바인딩 자동 적용
  final 필드 자동 → 불변 보장
  getter 자동 생성 (url(), maxPoolSize())
  equals, hashCode, toString 자동

Spring Boot 3.x + record 조합이 현재 권장 방식
```

---

## 💻 실험으로 확인하기

### 실험 1: setter 바인딩 vs 생성자 바인딩 불변성 차이

```java
// setter 바인딩 — 변경 가능
@ConfigurationProperties(prefix = "app")
@Component
class MutableProps {
    private String name;
    public void setName(String name) { this.name = name; }
    public String getName() { return name; }
}

// 생성자 바인딩 — 변경 불가
@ConfigurationProperties(prefix = "app")
class ImmutableProps {
    private final String name;
    public ImmutableProps(String name) { this.name = name; }
    public String getName() { return name; }
    // setter 없음 → 외부에서 변경 불가
}

// 테스트
mutableProps.setName("hacked");    // 가능 — 런타임 변경 허용
immutableProps.setName("hacked");  // 컴파일 오류 — setter 없음
```

### 실험 2: @DefaultValue 동작 확인

```java
// application.yml에 timeout 미설정
@ConfigurationProperties(prefix = "service")
public class ServiceProperties {
    private final int timeout;
    private final int retryCount;

    public ServiceProperties(
            @DefaultValue("30") int timeout,
            @DefaultValue("3") int retryCount) {
        this.timeout = timeout;
        this.retryCount = retryCount;
    }
}

// 결과: timeout=30, retryCount=3 (기본값 적용)
```

### 실험 3: record 바인딩

```java
@ConfigurationProperties(prefix = "mail")
public record MailProperties(
    String host,
    @DefaultValue("587") int port,
    boolean ssl
) {}

// application.yml
// mail:
//   host: smtp.gmail.com
//   ssl: true
// → MailProperties("smtp.gmail.com", 587, true)
```

---

## 🤔 트레이드오프

```
생성자 바인딩 장점:
  final 필드 → 불변 보장
  생성 시점에 필수값 검증 (@Validated와 조합)
  setter 불필요 → 코드 간결
  스레드 안전 (불변 객체)

생성자 바인딩 단점:
  파라미터 많으면 생성자 길어짐
  @DefaultValue가 어노테이션으로 표현돼 가독성 저하 가능
  record 활용하면 완화됨

setter 바인딩이 여전히 필요한 경우:
  레거시 설정 클래스 유지
  선택적 설정이 매우 많아 생성자가 지나치게 길어지는 경우
  외부 라이브러리 설정 클래스 확장

@Validated와 조합:
  @ConfigurationProperties
  @Validated
  public record DbProps(@NotBlank String url, @Min(1) @Max(100) int maxPool) {}
  → 컨텍스트 시작 시 설정값 유효성 검사 자동 수행
```

---

## 📌 핵심 정리

```
생성자 바인딩 결정 로직
  record → 자동으로 canonical 생성자
  @ConstructorBinding 명시 → 해당 생성자
  생성자 1개 + 기본 생성자 없음 → 자동 감지 (Boot 3.x)

바인딩 실행
  파라미터명(camelCase) ↔ 프로퍼티 키(kebab-case) 자동 변환
  값 없으면 @DefaultValue → null → NPE 주의

중첩 객체
  파라미터 타입이 설정 클래스면 재귀 바인딩

불변성 장점
  final 필드 → 런타임 변경 불가
  스레드 안전 → 설정 객체 공유 안전

권장 패턴 (Boot 3.x)
  record + @ConfigurationProperties + @Validated
  → 불변 + 간결 + 시작 시 검증
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `@ConstructorBinding`을 생략해도 생성자 바인딩이 적용되는가? 이유는?

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private final String name;
    private final int timeout;

    public AppProperties(String name, int timeout) {
        this.name = name;
        this.timeout = timeout;
    }
}
```

**Q2.** 다음 record 설정에서 `port`가 application.yml에 없을 때 어떤 값이 바인딩되는가?

```java
@ConfigurationProperties(prefix = "server")
public record ServerProperties(
    String host,
    @DefaultValue("8080") int port
) {}
```

**Q3.** `@ConfigurationProperties` + `@Validated` + 생성자 바인딩 조합에서 유효성 검사 실패 시 어떤 시점에 어떤 예외가 발생하는가?

> 💡 **해설**
>
> **Q1.** Spring Boot 3.x 기준으로 생략 가능하다. `BindConstructorProvider`가 생성자를 분석할 때, 클래스에 생성자가 하나뿐이고 기본 생성자(no-arg)가 없으면 자동으로 생성자 바인딩 대상으로 간주한다. Spring Boot 2.x에서는 `@ConstructorBinding`을 명시해야 했으나 3.0부터 이 규칙이 추가됐다. 단, 생성자가 두 개 이상이면 어느 것을 쓸지 알 수 없으므로 `@ConstructorBinding`을 명시해야 한다.
>
> **Q2.** `8080`이 바인딩된다. `bindViaConstructor()` 내부에서 `port` 파라미터에 해당하는 프로퍼티 값이 없으면 `@DefaultValue("8080")`을 읽어 `String "8080"`을 `int 8080`으로 변환해 생성자 인자로 전달한다. `@DefaultValue`가 없고 값도 없으면 기본 타입(`int`)은 `0`이 들어가고, 참조 타입은 `null`이 들어가 이후 `@NotNull` 검증 실패나 NPE로 이어질 수 있다.
>
> **Q3.** `ConfigurationPropertiesBindingPostProcessor`가 Bean 초기화 단계에서 바인딩을 수행하고, `@Validated`가 붙어 있으면 바인딩 직후 `Validator`로 검증을 수행한다. 검증 실패 시 `BindValidationException`이 발생하고, 이것이 `BeanCreationException`으로 감싸져 컨텍스트 `refresh()` 단계에서 전파된다. 결과적으로 애플리케이션이 시작되지 않으며, 어떤 프로퍼티가 잘못됐는지 명확한 메시지가 출력된다.

---

<div align="center">

**[⬅️ 이전: Optional Dependency 처리](./05-optional-dependency.md)** | **[다음: Lazy Initialization 동작 원리 ➡️](./07-lazy-initialization.md)**

</div>

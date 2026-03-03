# Environment & PropertySource 우선순위 — 프로퍼티가 결정되는 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Environment`는 무엇이고, `PropertySource`와 어떤 관계인가?
- OS 환경변수, JVM 시스템 프로퍼티, `application.yml`은 우선순위가 어떻게 결정되는가?
- `@Value("${...}")`는 어떤 순서로 값을 찾는가?
- `@Profile`은 내부에서 어떻게 평가되는가?
- 같은 키가 여러 PropertySource에 있을 때 어떤 값이 이기는가?
- Spring Boot에서 프로퍼티 우선순위는 어떻게 달라지는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 환경마다 다른 설정값을 어떻게 관리할 것인가

```java
// 환경마다 달라야 하는 값들
String dbUrl = ???;      // 개발: H2, 운영: MySQL
String apiKey = ???;     // 로컬: 테스트키, CI: 환경변수로 주입
int timeout = ???;       // 기본값: 3000, 특정 환경: 5000
```

```
단순한 접근 — Properties 파일 하드코딩:
  application.properties에 모든 값 고정
  → 환경마다 파일을 바꿔야 함
  → 운영 비밀키가 소스코드에 노출될 위험

이상적인 구조:
  "우선순위가 높은 곳의 값이 낮은 곳을 덮는다"

  커맨드라인 인수 (최우선)
    ↓ 없으면
  OS 환경변수 / JVM -D 옵션
    ↓ 없으면
  application.yml
    ↓ 없으면
  @Value의 기본값 (코드에 명시)
```

이 "우선순위 체인"을 구현하는 것이 `Environment`와 `PropertySource`다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 프로퍼티 오버라이딩 방향을 반대로 이해한다

```yaml
# application.yml
server:
  port: 8080
```

```bash
java -Dserver.port=9090 -jar app.jar
```

```
❌ 잘못된 이해:
  application.yml이 "설정 파일"이니까 더 구체적이고 우선순위가 높다.
  → 8080이 적용된다?

✅ 실제:
  JVM 시스템 프로퍼티(-D) > application.yml
  → 9090이 적용됨

이유:
  외부에서 주입하는 값이 파일보다 항상 높은 우선순위를 가짐
  → 배포 시 파일 수정 없이 동작 변경 가능
  → 12-Factor App 원칙
```

### Before: `@Profile`이 런타임 내내 동적으로 평가된다고 생각한다

```java
// ❌ 잘못된 이해
// "실행 중에 프로파일을 바꾸면 Bean이 교체된다?"

ctx.getEnvironment().setActiveProfiles("prod");  // refresh() 이후 변경 시도

// ✅ 실제:
// @Profile은 refresh() 시점에 한 번만 평가됨
// 이후 프로파일 변경은 이미 등록된 Bean에 영향 없음
```

---

## ✨ 올바른 이해와 사용

### After: PropertySource 체인과 우선순위를 명확히 파악한다

```
Spring Boot 기본 PropertySource 우선순위 (높음 → 낮음):

1.  @TestPropertySource                  테스트 전용
2.  커맨드라인 인수                       --server.port=9090
3.  SPRING_APPLICATION_JSON              환경변수로 JSON 주입
4.  ServletConfig / ServletContext       웹 컨테이너 파라미터
5.  JNDI (java:comp/env)
6.  JVM 시스템 프로퍼티                  -Dserver.port=9090
7.  OS 환경변수                          SERVER_PORT=9090
8.  RandomValuePropertySource            ${random.int}
9.  application-{profile}.yml           프로파일별 파일
10. application.yml                      기본 설정 파일
11. @PropertySource                      @PropertySource("classpath:...")
12. SpringApplication 기본값
```

---

## 🔬 내부 동작 원리

### 1. Environment 인터페이스 계층

```java
// PropertyResolver: 키로 값을 조회하는 최소 계약
public interface PropertyResolver {
    boolean containsProperty(String key);
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);
    String getRequiredProperty(String key) throws IllegalStateException;
}

// Environment: PropertyResolver + 프로파일 관리
public interface Environment extends PropertyResolver {
    String[] getActiveProfiles();
    String[] getDefaultProfiles();
    boolean matchesProfiles(String... profileExpressions);
}

// ConfigurableEnvironment: PropertySource 체인 직접 조작
public interface ConfigurableEnvironment extends Environment {
    void setActiveProfiles(String... profiles);
    MutablePropertySources getPropertySources();  // ← 체인 직접 접근
    Map<String, Object> getSystemProperties();    // JVM -D
    Map<String, Object> getSystemEnvironment();   // OS 환경변수
}
```

```
계층 요약:
  PropertyResolver          키 → 값 조회 (최소 계약)
      ↑
  Environment               + 프로파일 관리
      ↑
  ConfigurableEnvironment   + PropertySource 체인 수정
      ↑
  StandardEnvironment       기본 구현 (JVM + OS)
      ↑
  StandardServletEnvironment 웹 환경 (Servlet 파라미터 추가)
```

### 2. PropertySource 체인 — MutablePropertySources

```java
// MutablePropertySources: PropertySource 목록을 순서 있게 관리
public class MutablePropertySources implements PropertySources {

    private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();

    public void addFirst(PropertySource<?> propertySource) { ... }  // 최우선
    public void addLast(PropertySource<?> propertySource) { ... }   // 최저순위
    public void addBefore(String relativePropertySourceName, PropertySource<?> propertySource) { ... }
    public void addAfter(String relativePropertySourceName, PropertySource<?> propertySource) { ... }
}

// 프로퍼티 조회 — AbstractEnvironment
public String getProperty(String key) {
    // PropertySource 목록을 순서대로 순회 (앞이 우선순위 높음)
    for (PropertySource<?> propertySource : this.propertySources) {
        Object value = propertySource.getProperty(key);
        if (value != null) {
            return convertValueIfNecessary(value, String.class);
        }
    }
    return null;  // 전체 체인에 없으면 null
}
```

```
체인 탐색 흐름:

getProperty("server.port"):
  1. CommandLinePropertySource       → 없음
  2. SystemProperties (JVM -D)       → "9090" 발견 → 즉시 반환
     (이하 탐색 중단)

  만약 -D도 없었다면:
  3. SystemEnvironment (OS env)      → 없음
  4. application-prod.yml            → 없음
  5. application.yml                 → "8080" 발견 → 반환
```

### 3. @Value("${...}") 처리 과정

```java
@Service
public class MyService {
    @Value("${server.timeout:3000}")  // 기본값: 3000
    private int timeout;
}
```

```java
// PropertySourcesPropertyResolver.getProperty() — 단순화
protected <T> T getProperty(String key, Class<T> targetValueType) {

    for (PropertySource<?> propertySource : this.propertySources) {
        Object value = propertySource.getProperty(key);

        if (value != null) {
            // 중첩 플레이스홀더 재귀 해석 ("${inner.key}" 포함된 경우)
            if (value instanceof String strValue) {
                value = resolveNestedPlaceholders(strValue);
            }
            // String → int, boolean 등 타입 변환
            return convertValueIfNecessary(value, targetValueType);
        }
    }
    return null;
}
```

```
"${server.timeout:3000}" 파싱:

PropertyPlaceholderHelper 처리:
  키:     "server.timeout"
  기본값: "3000"  (콜론 뒤)

탐색:
  체인 순회 → 없으면 → 기본값 "3000" 사용
  있으면 → 발견된 값 사용

타입 변환:
  String "3000" → int 3000
  ConversionService 또는 TypeConverter 사용

기본값 없이 키가 없으면:
  IllegalArgumentException: Could not resolve placeholder 'server.timeout'
  → BeanCreationException으로 감싸져 컨텍스트 시작 실패
```

### 4. @Profile 평가 메커니즘

```java
// @Profile은 @Conditional(ProfileCondition.class) 메타 어노테이션으로 구현됨
class ProfileCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());

        if (attrs != null) {
            for (Object value : attrs.get("value")) {
                if (context.getEnvironment().matchesProfiles((String[]) value)) {
                    return true;
                }
            }
            return false;  // 아무 프로파일도 일치 안 함 → Bean 등록 안 함
        }
        return true;
    }
}
```

```
@Profile 평가 시점:

refresh() → ComponentScan 단계:
  @Component 후보 클래스 발견
  → @Profile 있으면 ProfileCondition.matches() 호출
  → Environment.getActiveProfiles() 확인
  → 일치 → BeanDefinition 등록
  → 불일치 → 건너뜀 (Bean 등록 자체 안 됨)

활성 프로파일 설정 방법 (우선순위 그대로 적용):
  1. -Dspring.profiles.active=prod          (JVM 옵션)
  2. SPRING_PROFILES_ACTIVE=prod            (OS 환경변수)
  3. spring.profiles.active=prod            (application.yml)
  4. ctx.getEnvironment().setActiveProfiles ("코드, refresh() 전에만 유효")

프로파일 표현식 (Spring 5.1+):
  @Profile("prod")           → "prod" 활성 시
  @Profile("!prod")          → "prod" 비활성 시
  @Profile("prod | staging") → "prod" 또는 "staging"
  @Profile("prod & cloud")   → 둘 다 활성 시
```

### 5. 커스텀 PropertySource 추가

```java
// EnvironmentPostProcessor로 커스텀 PropertySource 삽입
public class VaultPropertySourcePostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
                                       SpringApplication application) {
        // 예: Vault나 외부 시크릿 저장소에서 프로퍼티 로드
        Map<String, Object> secrets = loadFromVault();

        MapPropertySource secretSource =
            new MapPropertySource("vaultSecrets", secrets);

        // 기존 체인의 맨 앞에 삽입 → 최우선
        environment.getPropertySources().addFirst(secretSource);
    }

    private Map<String, Object> loadFromVault() {
        // ...
        return Map.of("db.password", "secret-from-vault");
    }
}
```

```
등록 방법 (Spring Boot):

# src/main/resources/META-INF/spring/
# org.springframework.boot.env.EnvironmentPostProcessor.imports

com.example.VaultPropertySourcePostProcessor
```

---

## 💻 실험으로 확인하기

### 실험 1: PropertySource 체인 목록 확인

```java
@SpringBootApplication
public class PropertyInspectApp {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx =
            SpringApplication.run(PropertyInspectApp.class, args);

        ConfigurableEnvironment env = ctx.getEnvironment();

        System.out.println("=== PropertySource 체인 (우선순위 순) ===");
        for (PropertySource<?> ps : env.getPropertySources()) {
            System.out.printf("%-50s [%s]%n",
                ps.getName(), ps.getClass().getSimpleName());
        }

        System.out.println("\n활성 프로파일: "
            + Arrays.toString(env.getActiveProfiles()));
    }
}
```

```
출력 (java -Dserver.port=9090 -jar app.jar):
=== PropertySource 체인 (우선순위 순) ===
systemProperties                          [PropertiesPropertySource]
systemEnvironment                         [SystemEnvironmentPropertySource]
Config resource 'application.yml'         [OriginTrackedMapPropertySource]

활성 프로파일: []
```

### 실험 2: 우선순위 오버라이딩 확인

```yaml
# application.yml
app.timeout: 1000
```

```bash
# 결과 비교
java -jar app.jar
# → 1000  (application.yml)

java -Dapp.timeout=3000 -jar app.jar
# → 3000  (JVM -D가 yml 덮어씀)

java -Dapp.timeout=3000 -jar app.jar --app.timeout=5000
# → 5000  (커맨드라인이 -D도 덮어씀)
```

### 실험 3: @Profile로 Bean 등록 제어

```java
public interface NotificationService {
    void send(String msg);
}

@Service @Profile("dev")
class ConsoleNotificationService implements NotificationService {
    public void send(String msg) { System.out.println("[DEV] " + msg); }
}

@Service @Profile("prod")
class EmailNotificationService implements NotificationService {
    public void send(String msg) { /* 실제 이메일 발송 */ }
}

// java -Dspring.profiles.active=dev -jar app.jar
// → ConsoleNotificationService만 등록
// → EmailNotificationService 등록 안 됨
```

---

## 🤔 트레이드오프

```
외부 설정 우선순위가 높은 이유:
  장점
    배포 시 파일 수정 없이 동작 변경
    운영 비밀키를 소스코드/파일에 두지 않아도 됨
  단점
    어떤 값이 실제 적용됐는지 파악 어려움
    → /actuator/env 엔드포인트 활용 권장

@Profile vs @ConditionalOnProperty:
  @Profile                  → 환경 프로파일 기반 (dev, prod)
  @ConditionalOnProperty    → 특정 프로퍼티 값 기반
    (예: feature.flag.enabled=true 일 때만 Bean 등록)
  → 단순 환경 분기: @Profile
  → 피처 플래그 / 세밀한 조건: @ConditionalOnProperty

application-{profile}.yml 전략:
  application.yml           → 공통 설정
  application-dev.yml       → 개발 환경 차이만
  application-prod.yml      → 운영 환경 차이만
  우선순위: 프로파일 파일 > 기본 파일
  → 중복 최소화, 환경별 차이만 작성
```

---

## 📌 핵심 정리

```
Environment
  PropertyResolver(키→값 조회) + 프로파일 관리
  ConfigurableEnvironment → PropertySource 체인 직접 조작

PropertySource 체인
  MutablePropertySources가 순서 있는 목록 관리
  getProperty() → 목록 앞부터 탐색, 첫 발견값 즉시 반환

우선순위 (높음 → 낮음)
  커맨드라인 > JVM -D > OS 환경변수 > application-{profile}.yml > application.yml

@Value("${key:default}")
  PropertySource 체인 순회 → 없으면 기본값 사용
  기본값 없고 키 없으면 BeanCreationException

@Profile
  refresh() 시 한 번만 평가 (런타임 변경 불가)
  ProfileCondition → Environment.matchesProfiles()
  불일치 시 BeanDefinition 등록 자체 안 됨

프로파일 파일 우선순위
  application-{profile}.yml > application.yml
  공통은 기본 파일, 환경별 차이만 프로파일 파일에 작성
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 `app.timeout`의 최종 적용값은 무엇인가?

```yaml
# application.yml
app.timeout: 1000
```
```bash
export APP_TIMEOUT=2000
java -Dapp.timeout=3000 -jar app.jar --app.timeout=4000
```

**Q2.** 다음 두 Bean이 동시에 등록되는 조건은? 그리고 그 경우 어떤 문제가 발생할 수 있는가?

```java
@Service @Profile("dev")
class DevService implements GreetingService { }

@Service @Profile("prod")
class ProdService implements GreetingService { }
```

**Q3.** `@Value("${db.url}")`을 사용하는 Bean에서 어떤 환경에서도 `db.url`이 설정되지 않을 경우 발생하는 일과, 이를 방어하는 방법 두 가지를 설명하라.

> 💡 **해설**
>
> **Q1.** 최종값은 `4000`이다. 우선순위: 커맨드라인(`--app.timeout=4000`) > JVM 시스템 프로퍼티(`-Dapp.timeout=3000`) > OS 환경변수(`APP_TIMEOUT=2000`, 단 키 변환 규칙으로 `app.timeout`과 매핑) > `application.yml`(`1000`). 커맨드라인 인수가 가장 높은 우선순위를 가지므로 `4000`이 적용된다.
>
> **Q2.** `spring.profiles.active=dev,prod`처럼 두 프로파일을 동시에 활성화하면 두 Bean 모두 등록된다. 이 경우 `GreetingService` 타입 Bean이 두 개가 되므로, 단일 `@Autowired` 주입 시 `NoUniqueBeanDefinitionException`이 발생한다. 의도치 않은 상황을 막으려면 `@Profile("prod & !dev")` 같은 표현식을 쓰거나 `@Primary`로 우선순위를 명시하면 된다.
>
> **Q3.** 키가 없고 기본값도 없으면 `BeanCreationException`(내부적으로 `IllegalArgumentException: Could not resolve placeholder 'db.url'`)이 발생하며, `refresh()` 도중 즉시 실패한다. 방어 방법 두 가지: ① `@Value("${db.url:jdbc:h2:mem:default}")` — 콜론 뒤에 기본값을 명시해 키가 없어도 안전하게 초기화된다. ② `@ConfigurationProperties` + `@Validated` + `@NotNull` — 프로퍼티 전용 클래스로 바인딩하고 검증 어노테이션을 붙이면, 누락 시 어떤 프로퍼티가 빠졌는지 명확한 메시지와 함께 시작 단계에서 실패해 디버깅이 쉬워진다.

---

<div align="center">

**[⬅️ 이전: ApplicationContext 계층 구조](./04-applicationcontext-hierarchy.md)** | **[다음: Resource Abstraction 패턴 ➡️](./06-resource-abstraction.md)**

</div>

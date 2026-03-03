# Resource Abstraction 패턴 — 스프링이 파일을 다루는 통일된 방식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Resource` 인터페이스는 왜 존재하는가? `java.io.File`로 충분하지 않은가?
- `classpath:`, `file:`, `http:` prefix는 어디서 어떻게 처리되는가?
- `ResourceLoader`와 `ResourcePatternResolver`는 어떻게 다른가?
- `classpath:` vs `classpath*:`의 차이는 무엇인가?
- `@Value("classpath:...")`로 리소스를 주입하는 과정은 어떻게 동작하는가?
- 스프링이 내부적으로 Resource를 어떻게 활용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 리소스 위치가 다양하면 코드가 달라져야 하는가

```java
// 리소스가 classpath에 있을 때
InputStream is = getClass().getResourceAsStream("/config/settings.xml");

// 파일 시스템에 있을 때
InputStream is = new FileInputStream("/etc/app/settings.xml");

// URL(원격)에 있을 때
InputStream is = new URL("https://config.server.com/settings.xml").openStream();

// Servlet Context에 있을 때 (웹 환경)
InputStream is = servletContext.getResourceAsStream("/WEB-INF/settings.xml");
```

```
문제:
  리소스 위치에 따라 완전히 다른 API 사용
  → 환경이 바뀌면 코드를 수정해야 함
  → 단위 테스트에서 실제 파일 경로 하드코딩 문제
  → 추상화 없이 직접 사용하면 예외 처리, 인코딩, URL 인코딩 각각 처리

이상적인 해결책:
  위치에 상관없이 동일한 인터페이스로 접근
  prefix만 바꾸면 구현체가 자동으로 선택됨
```

스프링의 해답이 `Resource` 인터페이스다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: classpath: 와 classpath*: 를 같다고 생각한다

```java
// ❌ JAR가 여러 개인 환경에서 의도와 다르게 동작
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

// classpath: → 첫 번째로 발견된 단일 리소스만
Resource[] single = resolver.getResources("classpath:config/db.yml");

// classpath*: → 클래스패스 전체(모든 JAR)에서 패턴 매칭
Resource[] all = resolver.getResources("classpath*:config/db.yml");
```

```
차이:
  classpath:
    첫 번째 ClassLoader에서 발견된 하나만 반환
    멀티 모듈 / 멀티 JAR 환경에서 나머지 무시

  classpath*:
    모든 ClassLoader에서 동일 경로 전체 탐색
    여러 JAR에 같은 경로 파일이 있으면 모두 반환

Spring MVC에서 context:component-scan이 classpath*를 쓰는 이유:
  여러 모듈의 @Component를 모두 스캔해야 하기 때문
```

### Before: `Resource`를 `java.io.File`로 무조건 변환한다

```java
// ❌ 모든 Resource를 File로 변환하려는 시도
@Value("classpath:templates/email.html")
private Resource emailTemplate;

public String readTemplate() throws IOException {
    File file = emailTemplate.getFile();  // ← JAR 내부 리소스는 실패!
    return Files.readString(file.toPath());
}
```

```
문제:
  classpath 리소스가 JAR 내부에 있으면 File 시스템 경로가 없음
  → getFile() → FileNotFoundException

올바른 방법:
  public String readTemplate() throws IOException {
      return new String(emailTemplate.getInputStream().readAllBytes());
  }
  → InputStream은 항상 사용 가능 (어떤 Resource 구현체든)
```

---

## ✨ 올바른 이해와 사용

### After: prefix만 바꾸면 구현체가 자동으로 선택된다

```java
@Component
public class ResourceConsumer implements ResourceLoaderAware {

    private ResourceLoader resourceLoader;

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    public void loadResource(String location) throws IOException {
        Resource resource = resourceLoader.getResource(location);

        System.out.println("exists: " + resource.exists());
        System.out.println("readable: " + resource.isReadable());
        System.out.println("filename: " + resource.getFilename());

        try (InputStream is = resource.getInputStream()) {
            String content = new String(is.readAllBytes());
            System.out.println(content);
        }
    }
}

// 사용:
consumer.loadResource("classpath:config/app.yml");       // ClassPathResource
consumer.loadResource("file:/etc/app/config.yml");       // FileSystemResource
consumer.loadResource("https://config.server.com/app");  // UrlResource
// 코드는 완전히 동일, prefix만 다름
```

---

## 🔬 내부 동작 원리

### 1. Resource 인터페이스 전체 계약

```java
// spring-core/.../Resource.java
public interface Resource extends InputStreamSource {

    // 기본 존재/상태 확인
    boolean exists();
    boolean isReadable();
    boolean isOpen();       // InputStream이 이미 열려 있는가 (한 번만 읽기 가능한 경우)
    boolean isFile();       // 파일 시스템 파일인가

    // URL / URI / File 접근 (지원하는 구현체만)
    URL getURL() throws IOException;
    URI getURI() throws IOException;
    File getFile() throws IOException;  // JAR 내부는 실패 가능

    // 콘텐츠 관련
    long contentLength() throws IOException;
    long lastModified() throws IOException;

    // 상대 경로로 새 Resource 생성
    Resource createRelative(String relativePath) throws IOException;

    // 파일명, 설명
    @Nullable String getFilename();
    String getDescription();  // 디버깅용 설명 문자열
}

// InputStreamSource (부모)
public interface InputStreamSource {
    InputStream getInputStream() throws IOException;  // 핵심 메서드
}
```

### 2. Resource 구현체 계층

```
Resource (인터페이스)
    ↑
AbstractResource (공통 구현)
    ↑
    ├── ClassPathResource          "classpath:config/app.yml"
    │     ClassLoader.getResourceAsStream() 사용
    │     JAR 내부 리소스 지원
    │
    ├── FileSystemResource         "file:/etc/app/config.yml"
    │     java.nio.file.Path 기반
    │     getFile() 항상 성공
    │
    ├── UrlResource                "https://example.com/config"
    │                              "ftp://..."
    │     java.net.URL 기반
    │     HTTP, FTP, 파일 URL 모두 처리
    │
    ├── ServletContextResource     웹 환경 전용
    │     ServletContext.getResourceAsStream() 사용
    │
    ├── ByteArrayResource          byte[] 를 Resource로 래핑
    │     단위 테스트에서 유용
    │
    └── InputStreamResource        InputStream을 Resource로 래핑
          isOpen() = true (한 번만 읽기 가능)
```

### 3. ResourceLoader — prefix로 구현체 선택

```java
// DefaultResourceLoader.getResource() — prefix 기반 분기
public Resource getResource(String location) {

    // 1. ResourceLoader에 등록된 프로토콜 핸들러 확인
    for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) return resource;
    }

    // 2. "/" 로 시작하면 → getResourceByPath (기본: ClassPathResource)
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }

    // 3. "classpath:" prefix
    if (location.startsWith(ResourceUtils.CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(
            location.substring(ResourceUtils.CLASSPATH_URL_PREFIX.length()),
            getClassLoader());
    }

    // 4. URL 형식이면 → UrlResource
    try {
        URL url = ResourceUtils.toURL(location);
        return (ResourceUtils.isFileURL(url)
            ? new FileUrlResource(url)   // file: URL
            : new UrlResource(url));     // http:, ftp: 등
    } catch (MalformedURLException ex) {
        // 5. 아무것도 아니면 → classpath 상대 경로로 처리
        return getResourceByPath(location);
    }
}
```

```
prefix 매핑 요약:

"classpath:config/app.yml"    → ClassPathResource
"file:/etc/app/config.yml"    → FileSystemResource (FileUrlResource)
"https://example.com/config"  → UrlResource
"/WEB-INF/config.xml"         → (웹) ServletContextResource
                                 (비웹) ClassPathResource
상대경로 "config/app.yml"     → ClassPathResource (기본)
```

### 4. ResourcePatternResolver — 와일드카드 패턴 탐색

```java
// ResourcePatternResolver: 단일 → 복수 확장
public interface ResourcePatternResolver extends ResourceLoader {
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    // 패턴으로 여러 Resource 반환
    Resource[] getResources(String locationPattern) throws IOException;
}

// PathMatchingResourcePatternResolver 핵심 로직 (단순화)
public Resource[] getResources(String locationPattern) throws IOException {

    if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
        // classpath*: → 모든 ClassLoader + 패턴 매칭
        String locationPatternWithoutPrefix =
            locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length());

        if (getPathMatcher().isPattern(locationPatternWithoutPrefix)) {
            // 와일드카드 포함 ("classpath*:config/*.yml")
            return findPathMatchingResources(locationPattern);
        } else {
            // 정확한 경로 ("classpath*:config/app.yml")
            return findAllClassPathResources(locationPatternWithoutPrefix);
        }
    }
    // 단일 Resource로 처리
    return new Resource[] { getResourceLoader().getResource(locationPattern) };
}
```

```
classpath: vs classpath*: 차이:

"classpath:META-INF/spring.factories"
  → ClassLoader.getResource() 한 번 호출
  → 클래스패스 탐색 순서에서 첫 번째 발견 파일만 반환
  → 여러 JAR에 같은 경로 있어도 하나만

"classpath*:META-INF/spring.factories"
  → ClassLoader.getResources() (복수형) 호출
  → 클래스패스의 모든 JAR에서 동일 경로 전부 반환
  → Spring Boot 자동 설정이 이 방식으로 모든 JAR의
    spring.factories를 수집함

와일드카드:
"classpath*:config/**/*.yml"
  → 모든 JAR의 config/ 하위 임의 경로의 .yml 전부
  → AntPathMatcher로 ** / * 패턴 처리
```

### 5. ApplicationContext 자체가 ResourceLoader

```java
// ApplicationContext extends ResourcePatternResolver
AnnotationConfigApplicationContext ctx =
    new AnnotationConfigApplicationContext(AppConfig.class);

// ApplicationContext에서 직접 Resource 획득
Resource r1 = ctx.getResource("classpath:data/init.sql");
Resource[] r2 = ctx.getResources("classpath*:db/migration/*.sql");

// ResourceLoaderAware로 주입받기
@Component
public class SqlLoader implements ResourceLoaderAware {
    private ResourceLoader loader;

    @Override
    public void setResourceLoader(ResourceLoader loader) {
        this.loader = loader;  // ApplicationContext 자체가 주입됨
    }
}

// 또는 직접 @Autowired
@Component
public class SqlLoader {
    @Autowired
    private ResourceLoader resourceLoader;  // ApplicationContext 주입됨
}
```

---

## 💻 실험으로 확인하기

### 실험 1: prefix별 구현체 확인

```java
@Component
public class ResourceTypeChecker {

    @Autowired
    private ResourceLoader resourceLoader;

    public void check() {
        String[] locations = {
            "classpath:application.yml",
            "file:/tmp/test.txt",
            "https://example.com"
        };

        for (String loc : locations) {
            Resource r = resourceLoader.getResource(loc);
            System.out.printf("%-40s → %s%n", loc, r.getClass().getSimpleName());
        }
    }
}
```

```
출력:
classpath:application.yml              → ClassPathResource
file:/tmp/test.txt                     → FileUrlResource
https://example.com                    → UrlResource
```

### 실험 2: classpath: vs classpath*: 차이

```java
// 여러 JAR에 "META-INF/services/MyService" 파일이 있다고 가정
PathMatchingResourcePatternResolver resolver =
    new PathMatchingResourcePatternResolver();

Resource[] single = resolver.getResources("classpath:META-INF/services/MyService");
Resource[] all    = resolver.getResources("classpath*:META-INF/services/MyService");

System.out.println("classpath:  " + single.length + "개");  // 1개 (첫 JAR만)
System.out.println("classpath*: " + all.length    + "개");  // N개 (모든 JAR)
```

### 실험 3: @Value로 Resource 주입

```java
@Component
public class TemplateLoader {

    @Value("classpath:templates/welcome.html")
    private Resource welcomeTemplate;  // Resource 타입으로 직접 주입

    public String load() throws IOException {
        // getFile() 쓰지 말 것 — JAR 환경에서 실패
        try (InputStream is = welcomeTemplate.getInputStream()) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}
```

---

## 🤔 트레이드오프

```
getFile() 사용 주의:
  개발 환경 (IDE 실행): 파일 시스템 기반 → getFile() 성공
  운영 환경 (JAR 실행): JAR 내부 → getFile() 실패
  → 항상 getInputStream() 사용이 안전

classpath*: 성능:
  모든 JAR 탐색 → 클래스패스가 클수록 느림
  컴포넌트 스캔에서 classpath*: 사용이 시작 시간에 영향
  → @Indexed (spring-context-indexer) 로 최적화 가능

Resource vs ResourceLoader 주입:
  @Value("classpath:...")  → 컴파일 시점에 경로 고정
  ResourceLoader 주입      → 런타임에 동적으로 경로 결정
  → 설정 파일은 @Value, 동적 로딩은 ResourceLoader 권장
```

---

## 📌 핵심 정리

```
Resource 인터페이스
  다양한 리소스 위치(classpath, file, URL)를 단일 추상화
  getInputStream() → 항상 사용 가능
  getFile()        → 파일 시스템 기반만 (JAR 내부 불가)

구현체 분류
  ClassPathResource    → classpath:
  FileSystemResource   → file:
  UrlResource          → http:, https:, ftp:
  ByteArrayResource    → byte[] 래핑 (테스트 유용)

ResourceLoader
  getResource(location) → prefix로 구현체 자동 선택
  ApplicationContext 자체가 ResourceLoader 구현

ResourcePatternResolver
  getResources(pattern) → 와일드카드 패턴으로 복수 반환
  classpath:  → 첫 번째 발견 파일만
  classpath*: → 모든 JAR에서 전부 탐색

classpath*: 사용 사례
  Spring Boot 자동 설정 (spring.factories)
  멀티 모듈 컴포넌트 스캔
  플러그인 JAR 등록 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드가 IDE에서는 정상 동작하지만 JAR로 패키징 후 실행하면 실패하는 이유는?

```java
@Value("classpath:data/schema.sql")
private Resource schemaResource;

public void run() throws IOException {
    File file = schemaResource.getFile();
    List<String> lines = Files.readAllLines(file.toPath());
    // ...
}
```

**Q2.** 멀티 모듈 스프링 프로젝트에서 각 모듈의 `META-INF/db/migration/` 폴더에 SQL 마이그레이션 파일이 있다. 모든 모듈의 파일을 한 번에 로드하려면 어떤 패턴을 써야 하는가?

**Q3.** `ResourceLoader`를 `@Autowired`로 주입받을 때 실제로 주입되는 객체는 무엇인가? 그 이유를 인터페이스 계층 구조로 설명하라.

> 💡 **해설**
>
> **Q1.** IDE에서 실행할 때는 `classpath:data/schema.sql`이 파일 시스템의 실제 경로로 매핑되므로 `getFile()`이 성공한다. 하지만 JAR로 패키징되면 `schema.sql`은 JAR 아카이브 내부에 ZIP 엔트리로 존재한다. JAR 내부 파일은 독립적인 파일 시스템 경로가 없으므로 `getFile()`은 `FileNotFoundException`을 던진다. 해결법은 `getFile()` 대신 `getInputStream()`을 항상 사용하는 것이다.
>
> **Q2.** `classpath*:META-INF/db/migration/*.sql` 패턴을 사용해야 한다. `classpath:`는 클래스패스 탐색 순서에서 첫 번째 JAR의 파일만 반환한다. `classpath*:`는 `ClassLoader.getResources()`를 사용해 모든 JAR에서 동일 패턴을 탐색하므로 여러 모듈의 파일을 전부 수집할 수 있다. `PathMatchingResourcePatternResolver`가 `**`와 `*` 와일드카드를 `AntPathMatcher`로 처리한다.
>
> **Q3.** 주입되는 객체는 `ApplicationContext` 자체다. `ApplicationContext`가 `ResourcePatternResolver`를 상속하고, `ResourcePatternResolver`가 `ResourceLoader`를 상속하는 구조이므로, `@Autowired ResourceLoader`에 `ApplicationContext`가 타입 호환으로 주입된다. 이 덕분에 `ResourceLoader`를 주입받으면 실제로는 `classpath*:` 패턴 탐색이 가능한 `ResourcePatternResolver` 능력도 함께 사용할 수 있다(`(ResourcePatternResolver) resourceLoader`로 캐스팅 가능).

---

<div align="center">

**[⬅️ 이전: Environment & PropertySource 우선순위](./05-environment-propertysource.md)** | **[홈으로 🏠](../README.md)**

</div>

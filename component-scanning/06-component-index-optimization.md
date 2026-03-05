# 컴포넌트 인덱스를 통한 스캔 최적화 — spring.components와 @Indexed

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `META-INF/spring.components` 인덱스 파일이 생성되는 시점과 내용은?
- `@Indexed` 어노테이션의 역할은 무엇이며, 기존 `@Component`와 어떻게 연동되는가?
- 인덱스를 사용할 때와 클래스패스 스캔을 사용할 때의 내부 경로 차이는?
- 어떤 규모의 프로젝트에서 인덱스가 의미 있는 차이를 만드는가?
- Spring Native / GraalVM AOT와 인덱스의 관계는?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스패스 스캔은 .class 파일 수에 비례해 느려진다

```
일반 프로젝트 (200개 클래스):
  스캔 시간: 수십 ms → 큰 문제 없음

대규모 모노레포 (10,000개 클래스):
  JAR 100개 × 클래스 100개 = 10,000개 .class 파일
  → PathMatchingResourcePatternResolver가 JAR 내부를 모두 열어서 탐색
  → ASM MetadataReader로 각 파일 파싱
  → 시작 시간: 수 초 → 개발 생산성 저하

인덱스 해결:
  컴파일 시점에 "어떤 클래스가 @Component인가"를 미리 기록
  → 런타임에 파일 하나(spring.components)만 읽으면 됨
  → 수천 개 .class 파일 탐색 없음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Indexed를 직접 붙여야 동작한다

```java
// ❌ 잘못된 이해:
// "모든 @Component 클래스에 @Indexed도 함께 붙여야 한다"
@Component
@Indexed  // ← 매번 추가?
public class OrderService {}
```

```
✅ 실제:
  @Component 자체에 이미 @Indexed가 메타 어노테이션으로 포함됨

  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Indexed              // ← @Component에 내장됨
  public @interface Component {}

  → @Component, @Service, @Repository, @Controller 모두 자동으로 인덱싱 대상
  → @Indexed를 직접 붙일 필요 없음 (커스텀 스테레오타입 어노테이션에는 필요)
```

### Before: 인덱스 파일이 없으면 에러가 발생한다

```
❌ 잘못된 이해:
  "spring-context-indexer 의존성 추가했는데 인덱스 파일 없으면 에러"

✅ 실제:
  ClassPathScanningCandidateComponentProvider.findCandidateComponents()
  → componentsIndex가 null이거나 인덱스 미지원 필터가 있으면
  → 자동으로 기존 클래스패스 스캔으로 폴백
  → 인덱스는 선택적 최적화, 없어도 정상 동작
```

---

## ✨ 올바른 이해와 사용

### After: 인덱스 생성 → 저장 → 런타임 로드 전체 흐름

```
컴파일 타임 (spring-context-indexer):
  CandidateComponentsIndexer (어노테이션 프로세서)
  → 컴파일 중 @Indexed 붙은 클래스 탐색
  → META-INF/spring.components 파일 생성
     형식: 클래스명=어노테이션명(들)

런타임:
  CandidateComponentsIndexLoader.loadIndex()
  → 클래스패스의 모든 spring.components 파일 로드
  → CandidateComponentsIndex 객체 생성

  ClassPathScanningCandidateComponentProvider.findCandidateComponents()
  → componentsIndex != null → addCandidateComponentsFromIndex() 사용
  → classpath 파일 탐색 없음
  → 인덱스에서 basePackage에 해당하는 클래스만 필터링

인덱스 미사용 경로:
  componentsIndex == null (의존성 없음)
  또는 커스텀 includeFilter로 인덱스 지원 불가 판단
  → scanCandidateComponents() (기존 ASM 스캔) 로 폴백
```

---

## 🔬 내부 동작 원리

### 1. @Indexed — 인덱싱 마커

```java
// spring-context 모듈
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Indexed {
    // 속성 없음 — 단순 마커
}

// @Component에 내장
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed               // ← 내장
public @interface Component {
    String value() default "";
}
```

```
@Indexed가 있는 어노테이션:
  @Component (→ @Service, @Repository, @Controller, @RestController 등)
  커스텀 스테레오타입 어노테이션에 @Indexed 추가 → 자동 인덱싱

인덱싱 대상 결정:
  CandidateComponentsIndexer가 클래스의 어노테이션을 검사
  → @Indexed가 붙은 어노테이션을 가진 클래스 → 인덱스에 기록
  → 메타 어노테이션 체인도 탐색
```

### 2. CandidateComponentsIndexer — 컴파일 타임 어노테이션 프로세서

```java
// spring-context-indexer 모듈의 어노테이션 프로세서
// javac 컴파일 시 자동 실행

public class CandidateComponentsIndexer extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {

        // 이번 컴파일 라운드에서 처리된 모든 타입 순회
        for (Element element : env.getRootElements()) {
            if (element instanceof TypeElement typeElement) {
                // 해당 타입에 @Indexed (직접 또는 메타 어노테이션)가 있는지 확인
                String stereotype = getStereotype(typeElement);
                if (stereotype != null) {
                    // 클래스명 = 스테레오타입 어노테이션명 형식으로 기록
                    this.indexedClasses.add(
                        typeElement.getQualifiedName().toString(),
                        stereotype);
                }
            }
        }

        // 마지막 라운드에서 파일 쓰기
        if (env.processingOver()) {
            writeIndexFile();
        }
        return false;
    }

    private void writeIndexFile() {
        // META-INF/spring.components 생성
        try (PrintWriter writer = new PrintWriter(
                processingEnv.getFiler().createResource(
                    StandardLocation.CLASS_OUTPUT,
                    "", "META-INF/spring.components"))) {

            // 형식: 클래스명=어노테이션명
            for (Entry<String, Set<String>> entry : indexedClasses.entrySet()) {
                for (String stereotype : entry.getValue()) {
                    writer.println(entry.getKey() + "=" + stereotype);
                }
            }
        }
    }
}
```

```
생성된 META-INF/spring.components 예시:

com.example.service.OrderService=org.springframework.stereotype.Service
com.example.service.PaymentService=org.springframework.stereotype.Service
com.example.repository.OrderRepository=org.springframework.stereotype.Repository
com.example.web.OrderController=org.springframework.stereotype.Controller
com.example.config.AppConfig=org.springframework.context.annotation.Configuration

형식:
  FQCN=stereotype-annotation-FQCN
  한 클래스가 여러 어노테이션 → 여러 줄
```

### 3. CandidateComponentsIndexLoader — 런타임 인덱스 로드

```java
// CandidateComponentsIndexLoader.java
public final class CandidateComponentsIndexLoader {

    public static final String COMPONENTS_RESOURCE_LOCATION =
        "META-INF/spring.components";

    // 인덱스 캐시 (ClassLoader 기준)
    private static final ConcurrentReferenceHashMap<ClassLoader, CandidateComponentsIndex>
        cache = new ConcurrentReferenceHashMap<>();

    public static CandidateComponentsIndex loadIndex(ClassLoader classLoader) {
        return cache.computeIfAbsent(classLoader, cl -> {
            // 클래스패스의 모든 spring.components 파일 로드
            // (여러 JAR에 분산된 파일 모두 수집)
            Enumeration<URL> urls = cl.getResources(COMPONENTS_RESOURCE_LOCATION);

            Properties result = new Properties();
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                try (InputStream is = url.openStream()) {
                    result.load(is);  // "클래스명=어노테이션명" 파싱
                }
            }

            return new CandidateComponentsIndex(result);
        });
    }
}
```

### 4. addCandidateComponentsFromIndex() — 인덱스 기반 후보 수집

```java
// ClassPathScanningCandidateComponentProvider.addCandidateComponentsFromIndex()
private Set<BeanDefinition> addCandidateComponentsFromIndex(
        CandidateComponentsIndex index, String basePackage) {

    Set<BeanDefinition> candidates = new LinkedHashSet<>();

    // includeFilter별로 인덱스 조회
    for (TypeFilter includeFilter : this.includeFilters) {

        // 인덱스가 이 필터를 지원하는가?
        String stereotype = extractStereotype(includeFilter);
        if (stereotype == null) {
            // 지원 안 함 → 이 메서드 포기, 기존 스캔으로 폴백
            return null;
        }

        // 인덱스에서 stereotype에 해당하는 클래스 목록 조회
        // + basePackage 필터링
        for (EntryReader.Entry entry : index.getCandidateTypes(basePackage, stereotype)) {
            // .class 파일 탐색 없이 인덱스 Map에서 직접 조회!

            MetadataReader metadataReader =
                getMetadataReaderFactory().getMetadataReader(entry.getType());
            // → 이 시점에 실제 .class 파일을 읽음 (하지만 인덱스 대상만)

            if (isCandidateComponent(metadataReader)) {
                ScannedGenericBeanDefinition sbd =
                    new ScannedGenericBeanDefinition(metadataReader);
                sbd.setSource(metadataReader.getResource());
                if (isCandidateComponent(sbd)) {
                    candidates.add(sbd);
                }
            }
        }
    }
    return candidates;
}
```

```
인덱스 사용 시 차이:

기존 스캔:
  classpath*:com/example/**/*.class 패턴
  → JAR 파일들 순회 → 각 .class 파일 열기 → ASM 파싱
  → 대부분 필터 탈락 → 소수만 후보

인덱스 스캔:
  spring.components 파일 로드 (이미 캐시됨)
  → basePackage 기준 필터링 (Map 조회, O(1))
  → 후보로 확인된 클래스만 .class 파일 읽음
  → JAR 내 불필요한 클래스 탐색 없음

성능 차이:
  클래스 수 N이 클수록 효과 큼
  N=200   → 큰 차이 없음
  N=5,000 → 수백 ms 단축 가능
  N=20,000 → 초 단위 단축 가능
```

### 5. indexSupportsIncludeFilters() — 폴백 조건

```java
// 인덱스를 쓸 수 없는 경우 기존 스캔으로 폴백
private boolean indexSupportsIncludeFilters() {
    for (TypeFilter includeFilter : this.includeFilters) {
        if (!indexSupportsIncludeFilter(includeFilter)) {
            return false;
        }
    }
    return true;
}

private boolean indexSupportsIncludeFilter(TypeFilter filter) {
    // AnnotationTypeFilter만 인덱스 지원
    if (filter instanceof AnnotationTypeFilter atf) {
        Class<? extends Annotation> ann = atf.getAnnotationType();
        // @Indexed가 메타 어노테이션으로 있거나 직접 붙어있어야 함
        return (AnnotationUtils.getAnnotation(ann, Indexed.class) != null
            || ann.getName().startsWith("javax."));
    }
    // AssignableTypeFilter, RegexPatternTypeFilter, Custom → 인덱스 미지원
    if (filter instanceof AssignableTypeFilter atf) {
        Class<?> target = atf.getTargetType();
        return AnnotationUtils.getAnnotation(target, Indexed.class) != null;
    }
    return false;
}
```

```
인덱스 폴백 조건:
  excludeFilters에 비 AnnotationTypeFilter 포함 → 폴백
  커스텀 includeFilter 사용 → 폴백
  → 인덱스는 표준 @Component 기반 스캔에만 완전히 적용
```

### 6. Spring Native / GraalVM AOT와의 관계

```
GraalVM Native Image:
  런타임 클래스패스 스캔 불가 (정적 분석 기반)
  → spring.components 인덱스 활용 또는 AOT 처리로 대체

Spring Boot 3.x AOT(Ahead-of-Time):
  build-time에 BeanDefinition 분석 → 소스 코드로 미리 생성
  → 런타임에 클래스패스 스캔 없음
  → spring-context-indexer와 유사한 컨셉이지만 더 포괄적

  AOT가 활성화되면 spring.components 인덱스는 대부분 대체됨
  → 스탠다드 JVM 배포에서는 여전히 spring-context-indexer 유용
```

---

## 💻 실험으로 확인하기

### 실험 1: 의존성 추가 후 생성된 인덱스 파일 확인

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-indexer</artifactId>
    <optional>true</optional>
</dependency>
```

```bash
# 컴파일 후 확인
cat target/classes/META-INF/spring.components

# 출력 예시:
# com.example.service.OrderService=org.springframework.stereotype.Service
# com.example.repository.OrderRepository=org.springframework.stereotype.Repository
# com.example.web.OrderController=org.springframework.stereotype.Controller
```

### 실험 2: 인덱스 사용 여부 로그 확인

```yaml
# application.yml
logging:
  level:
    org.springframework.context.annotation: TRACE
```

```
인덱스 사용 시:
  "Using candidate component index" 로그 출력

인덱스 미사용 (폴백):
  "findCandidateComponents" 스캔 로그
```

### 실험 3: 커스텀 스테레오타입 어노테이션 인덱싱

```java
// 커스텀 어노테이션에 @Indexed 추가
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
@Indexed   // ← 직접 추가 (또는 @Component 통해 자동)
public @interface DomainService {}

@DomainService
public class OrderDomainService {}

// 컴파일 후 spring.components에 자동 기록:
// com.example.OrderDomainService=com.example.DomainService
```

---

## 🤔 트레이드오프

```
spring-context-indexer 도입 비용 vs 효과:

도입 비용:
  의존성 추가 1줄
  빌드 시간 약간 증가 (어노테이션 프로세서 실행)
  spring.components 파일 관리 (git 추적 여부 결정)

효과:
  클래스 수 < 500: 체감 효과 거의 없음
  클래스 수 500~2000: 수백 ms 단축
  클래스 수 2000+: 수 초 단축 (개발 루프 체감)

주의사항:
  IDE incremental compile: 어노테이션 프로세서 누락 가능
  → 전체 빌드 후 인덱스 파일 동기화 필요

  커스텀 includeFilter 사용 시 인덱스 효과 없음 → 폴백
  → 표준 @Component 기반 스캔 위주 설계 권장

Spring Boot 3.x 환경:
  AOT가 컴파일 타임 분석을 대부분 담당
  → Native 빌드에서는 spring-context-indexer 효과 적음
  → 표준 JVM 배포에서는 여전히 유효
```

---

## 📌 핵심 정리

```
인덱스 생성 (컴파일 타임)
  spring-context-indexer 의존성 추가
  → CandidateComponentsIndexer (어노테이션 프로세서)
  → @Indexed가 있는 어노테이션을 가진 클래스 탐색
  → META-INF/spring.components 파일 생성
     형식: FQCN=stereotype-annotation-FQCN

@Indexed 내장
  @Component에 이미 @Indexed 내장
  → @Service, @Repository, @Controller 자동 포함
  → 커스텀 스테레오타입에는 직접 @Indexed 추가 필요

런타임 로드
  CandidateComponentsIndexLoader.loadIndex()
  → classpath의 모든 spring.components 수집
  → CandidateComponentsIndex (ClassLoader 기준 캐시)

인덱스 사용 분기
  componentsIndex != null
  && indexSupportsIncludeFilters() (AnnotationTypeFilter만 지원)
  → addCandidateComponentsFromIndex() : Map 조회, O(1)
  그 외 → scanCandidateComponents() : ASM 파일 스캔 폴백

효과
  대규모 프로젝트(2000+ 클래스)에서 시작 시간 수 초 단축
  소규모 프로젝트는 효과 미미
```

---

## 🤔 생각해볼 문제

**Q1.** `spring-context-indexer`를 추가한 후 새 `@Service` 클래스를 작성했는데 IDE에서 incremental compile만 했다면 인덱스 파일이 올바르게 갱신되는가?

**Q2.** 멀티모듈 프로젝트에서 각 모듈이 독립적으로 `spring.components`를 생성한다면, 런타임에 어떻게 병합되는가?

**Q3.** `AssignableTypeFilter`를 포함한 커스텀 `includeFilter`가 있을 때 인덱스가 폴백되는 이유는?

> 💡 **해설**
>
> **Q1.** IDE의 incremental compile은 변경된 파일만 재컴파일하는데, 어노테이션 프로세서(`CandidateComponentsIndexer`)가 incremental 모드에서 올바르게 동작하려면 IDE가 어노테이션 프로세서를 지원해야 한다. IntelliJ IDEA는 `Enable annotation processing` 옵션이 켜져 있으면 incremental 재컴파일 시 어노테이션 프로세서도 실행하지만, 새로 추가된 클래스만 처리하고 기존 `spring.components`를 갱신한다. 그러나 일부 환경에서는 전체 빌드(`mvn compile` 또는 `./gradlew compileJava`) 후에 인덱스가 완전히 동기화된다. 의심스럽다면 인덱스 파일을 직접 열어 새 클래스가 포함됐는지 확인한다.
>
> **Q2.** 각 모듈의 JAR에 독립적인 `META-INF/spring.components` 파일이 포함된다. `CandidateComponentsIndexLoader.loadIndex()`에서 `classLoader.getResources("META-INF/spring.components")`를 호출하면 클래스패스의 모든 JAR에서 해당 경로의 파일을 수집한다. `Enumeration<URL>`로 모든 파일을 순회하며 `Properties`에 병합한다. 따라서 모듈 A의 `spring.components`와 모듈 B의 `spring.components`가 하나의 `CandidateComponentsIndex`로 합쳐진다. 런타임 클래스패스에 있는 모든 JAR의 인덱스가 자동으로 통합되므로 별도 설정이 필요 없다.
>
> **Q3.** `AssignableTypeFilter`는 타입 계층 관계를 확인하기 위해 런타임에 클래스를 실제로 로드해야 한다. `isAssignableFrom()` 검사가 필요하기 때문이다. 인덱스는 컴파일 타임에 어노테이션 정보만 기록하며 타입 계층 정보는 담지 않는다. 따라서 `AssignableTypeFilter`가 특정 인터페이스의 구현체를 찾으려 해도 인덱스에서는 그 정보를 알 수 없다. `indexSupportsIncludeFilter()`가 `AssignableTypeFilter`에 대해 `false`를 반환하므로 `indexSupportsIncludeFilters()` 전체가 `false`가 되어 인덱스 경로를 포기하고 기존 ASM 스캔으로 폴백한다.

---

<div align="center">

**[⬅️ 이전: BeanDefinition 등록 과정](./05-beandefinition-registration.md)** | **[홈으로 🏠](../README.md)**

</div>

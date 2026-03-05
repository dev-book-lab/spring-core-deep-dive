# ClassPathScanningCandidateComponentProvider — 후보 필터링 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ClassPathScanningCandidateComponentProvider`가 Bean 후보를 거르는 정확한 알고리즘은?
- `@Service`, `@Repository`, `@Controller`가 `@Component` 없이도 스캔되는 이유는?
- `MetadataReader`와 `AnnotationMetadata`가 협력하는 방식은?
- ASM `ClassReader`가 클래스 로드 없이 어노테이션을 읽는 원리는?
- `CachingMetadataReaderFactory`의 LRU 캐시가 왜 중요한가?

---

## 🔍 왜 이게 존재하는가

### 문제: 수천 개의 .class 파일 중 Bean 후보를 빠르게 골라야 한다

```
방법 A: Class.forName()으로 전부 로드 후 리플렉션으로 어노테이션 확인
  → 5,000개 클래스 모두 Metaspace에 적재
  → 시작 시간 수 배 증가, GC 불가 영역 차지

방법 B: ASM으로 .class 바이트코드만 파싱
  → 클래스 로드 없이 어노테이션 정보만 읽음
  → Bean 후보만 선별 후 그것만 실제 로드

스프링 선택: 방법 B → ClassPathScanningCandidateComponentProvider 핵심 설계
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Service / @Repository는 별도 includeFilter가 필요하다

```java
// ❌ 불필요한 설정
@ComponentScan(
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Service.class)
)
```

```
✅ 실제:
  useDefaultFilters=true(기본) → AnnotationTypeFilter(@Component) 하나만 등록
  @Service, @Repository, @Controller는 @Component를 메타 어노테이션으로 포함
  → hasMetaAnnotation("Component") = true → 자동 감지
  → 별도 필터 추가는 불필요한 중복
```

### Before: MetadataReader가 내부적으로 클래스를 로드한다

```
❌ 잘못된 이해: "MetadataReader = 리플렉션으로 어노테이션을 읽는다"

✅ 실제:
  MetadataReader = ASM ClassReader 래퍼
  .class 파일 바이트 배열을 직접 파싱
  ClassLoader.loadClass() 호출 없음
```

---

## ✨ 올바른 이해와 사용

### After: 필터링 알고리즘과 메타데이터 레이어 구조

```
Resource (.class 파일)
  ↓ CachingMetadataReaderFactory.getMetadataReader()
MetadataReader
  ├── ClassMetadata     이름, 수퍼클래스, 인터페이스, abstract/interface 여부
  └── AnnotationMetadata 클래스/메서드 어노테이션, 메타 어노테이션 포함 확인

필터링 단계:
  1단계 isCandidateComponent(MetadataReader)
    excludeFilter 평가 → includeFilter 평가 → @Conditional 평가

  2단계 isCandidateComponent(AnnotatedBeanDefinition)
    isIndependent() && (isConcrete() || @Lookup 보유 abstract)
```

---

## 🔬 내부 동작 원리

### 1. ASM ClassReader — 바이트코드에서 메타데이터 읽기

```java
final class SimpleMetadataReader implements MetadataReader {

    SimpleMetadataReader(Resource resource, ClassLoader classLoader) throws IOException {
        try (InputStream is = new BufferedInputStream(resource.getInputStream())) {

            ClassReader classReader = new ClassReader(is);
            SimpleAnnotationMetadataReadingVisitor visitor =
                new SimpleAnnotationMetadataReadingVisitor(classLoader);

            // SKIP_CODE:   메서드 바이트코드 건너뜀 (어노테이션만 필요)
            // SKIP_DEBUG:  디버그 정보 건너뜀
            // SKIP_FRAMES: 스택 프레임 건너뜀
            classReader.accept(visitor,
                ClassReader.SKIP_CODE | ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);

            this.annotationMetadata = visitor.getMetadata();
        }
    }
}
```

```
.class 파일에서 읽는 정보:

access_flags           ← public / abstract / interface / final
this_class             ← 이 클래스의 이름
super_class            ← 수퍼클래스
interfaces             ← 구현 인터페이스 목록
attributes
  → RuntimeVisibleAnnotations  ← 클래스 레벨 @Component, @Service 등
methods
  → RuntimeVisibleAnnotations  ← @Bean, @PostConstruct 등
```

### 2. AnnotationMetadata — 메타 어노테이션 탐색

```java
// @Service 감지 흐름

// AnnotationTypeFilter.match() — 기본 includeFilter
public boolean match(MetadataReader metadataReader, ...) {
    AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();

    // @Component 직접 선언 OR @Component를 메타 어노테이션으로 포함
    return metadata.hasAnnotation(this.annotationType.getName())
        || (this.considerMetaAnnotations
            && metadata.hasMetaAnnotation(this.annotationType.getName()));
    //                                    ↑
    //   @Service → @Component 체인 탐색 → true
    //   @Repository, @Controller, @RestController 모두 동일
}
```

```
메타 어노테이션 체인 예시:

@RestController
  @Controller
    @Component  ← AnnotationTypeFilter가 여기서 매칭
  @ResponseBody

→ @RestController만 있어도 자동으로 Bean 후보 선별
```

### 3. SimpleAnnotationMetadataReadingVisitor — 방문자 패턴

```java
class SimpleAnnotationMetadataReadingVisitor extends ClassVisitor {

    private String className;
    private int access;
    private String superClassName;
    private String[] interfaceNames;

    // 클래스 선언 방문 (가장 먼저 호출)
    @Override
    public void visit(int version, int access, String name,
                      String signature, String superName, String[] interfaces) {
        this.className = ClassUtils.convertResourcePathToClassName(name);
        this.access = access;
        this.superClassName = ClassUtils.convertResourcePathToClassName(superName);
        this.interfaceNames = /* interfaces 변환 */ ;
    }

    // 클래스 레벨 어노테이션 방문
    @Override
    public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
        if (!visible) return null;  // RetentionPolicy.RUNTIME만 처리
        String annotationType = Type.getType(descriptor).getClassName();
        return MergedAnnotationReadingVisitor.get(
            classLoader, this::addAnnotation, annotationType);
        // 어노테이션 속성값(value 등)도 재귀 수집
    }

    // 메서드 방문 — @Bean, @PostConstruct 등 감지
    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, ...) {
        return new SimpleMethodMetadataReadingVisitor(
            classLoader, className, access, name, descriptor, methods);
    }
}
```

### 4. isCandidateComponent() — 두 단계 필터

```java
// 1단계: MetadataReader 기반
protected boolean isCandidateComponent(MetadataReader metadataReader) {

    // excludeFilter 먼저 (제외 우선)
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return false;  // 즉시 탈락, includeFilter 평가 안 함
        }
    }

    // includeFilter 평가
    for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return isConditionMatch(metadataReader);  // @Conditional 평가
        }
    }
    return false;
}

// 2단계: AnnotatedBeanDefinition 기반 (구체 클래스 여부 확인)
protected boolean isCandidateComponent(AnnotatedBeanDefinition sbd) {
    AnnotationMetadata metadata = sbd.getMetadata();

    return (metadata.isIndependent()
        // 최상위 클래스 또는 static 내부 클래스
        // 비정적 내부 클래스는 외부 인스턴스 필요 → 제외

        && (metadata.isConcrete()
        // abstract / interface 아님

            || (metadata.isAbstract()
                && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
                // abstract이지만 @Lookup 메서드 있으면 허용
}
```

```
isIndependent() = false 예시:

public class Outer {
    @Component
    class Inner { ... }  // 비정적 내부 클래스 → 제외

    @Component
    static class StaticInner { ... }  // 정적 내부 클래스 → 허용
}
```

### 5. CachingMetadataReaderFactory — LRU 캐시

```java
public class CachingMetadataReaderFactory extends SimpleMetadataReaderFactory {

    public static final int DEFAULT_CACHE_LIMIT = 256;

    // LinkedHashMap을 LRU 방식으로 사용
    private Map<Resource, MetadataReader> metadataReaderCache =
        new LinkedHashMap<>(256, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Resource, MetadataReader> eldest) {
                return size() > cacheLimit;
            }
        };

    @Override
    public MetadataReader getMetadataReader(Resource resource) throws IOException {
        synchronized (this.metadataReaderCache) {
            MetadataReader reader = this.metadataReaderCache.get(resource);
            if (reader == null) {
                reader = super.getMetadataReader(resource);  // 파일 I/O
                this.metadataReaderCache.put(resource, reader);
            }
            return reader;
        }
    }
}
```

```
캐시가 중요한 이유:
  스캔 + @Conditional 평가 + @Import 처리 등에서
  동일 .class 파일을 여러 번 읽을 수 있음
  → ConfigurationClassPostProcessor가 동일 팩토리 인스턴스를 공유
  → 스캔 전체에서 중복 파일 I/O 방지
```

---

## 💻 실험으로 확인하기

### 실험 1: MetadataReader로 클래스 정보 직접 읽기

```java
SimpleMetadataReaderFactory factory = new SimpleMetadataReaderFactory();
Resource resource = new ClassPathResource("com/example/service/OrderService.class");
MetadataReader reader = factory.getMetadataReader(resource);

ClassMetadata cm = reader.getClassMetadata();
System.out.println(cm.getClassName());       // "com.example.service.OrderService"
System.out.println(cm.getSuperClassName());  // "java.lang.Object"
System.out.println(cm.isAbstract());         // false
System.out.println(cm.isIndependent());      // true

AnnotationMetadata am = reader.getAnnotationMetadata();
System.out.println(am.hasAnnotation(
    "org.springframework.stereotype.Service"));        // true
System.out.println(am.hasMetaAnnotation(
    "org.springframework.stereotype.Component"));      // true ← 핵심
```

### 실험 2: @Component 메타 어노테이션 체인 확인

```java
// @Service의 메타 어노테이션 직접 확인
for (Annotation a : Service.class.getAnnotations()) {
    System.out.println(a.annotationType().getSimpleName());
    // Component
    // Documented
}
// → @Service가 @Component를 직접 메타 어노테이션으로 포함
```

### 실험 3: CachingMetadataReaderFactory 캐시 효과

```java
CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
Resource resource = new ClassPathResource("com/example/service/OrderService.class");

long t1 = System.nanoTime();
MetadataReader r1 = factory.getMetadataReader(resource);  // 파일 I/O
System.out.printf("첫 번째: %d ns%n", System.nanoTime() - t1);

long t2 = System.nanoTime();
MetadataReader r2 = factory.getMetadataReader(resource);  // 캐시
System.out.printf("두 번째: %d ns%n", System.nanoTime() - t2);

System.out.println(r1 == r2);  // true — 같은 인스턴스
```

---

## 🤔 트레이드오프

```
ASM 기반 메타데이터 읽기:
  장점  클래스 로드 없이 빠른 필터링, 메모리 효율
  단점  RetentionPolicy.RUNTIME 어노테이션만 읽음
       런타임 동적 어노테이션 추가 감지 불가

CachingMetadataReaderFactory 캐시 크기:
  기본 256 — 소규모 프로젝트 충분
  대규모: spring.context.cache.limit 조정 고려
  너무 크면 메모리 낭비, 너무 작으면 캐시 효과 감소

SKIP_CODE 옵션:
  메서드 바디 건너뜀 → 파싱 속도 최적화
  메서드 시그니처(반환 타입, 파라미터 어노테이션)는 여전히 읽음
  → @Bean 메서드 반환 타입, @Qualifier 등 정상 처리

2단계 isCandidateComponent:
  1단계: 어노테이션 기반 필터 (빠름)
  2단계: 구조 기반 필터 (비정적 내부 클래스, 인터페이스, abstract 제거)
```

---

## 📌 핵심 정리

```
핵심 클래스 역할
  ClassPathScanningCandidateComponentProvider
    스캔 실행, includeFilter / excludeFilter 보유
  CachingMetadataReaderFactory
    Resource → MetadataReader, LRU 캐시(기본 256)
  SimpleMetadataReader
    ASM ClassReader 래퍼, ClassMetadata + AnnotationMetadata 제공

AnnotationMetadata 핵심 메서드
  hasAnnotation()     직접 선언된 어노테이션
  hasMetaAnnotation() 메타 어노테이션 체인 탐색
  → @Service / @Repository 파생 어노테이션 자동 감지의 핵심

두 단계 필터
  1단계: excludeFilter → includeFilter(메타 어노테이션 포함) → @Conditional
  2단계: isIndependent() && isConcrete()

ASM 파싱 플래그
  SKIP_CODE | SKIP_DEBUG | SKIP_FRAMES
  → 어노테이션 정보만 읽고 나머지 건너뜀
```

---

## 🤔 생각해볼 문제

**Q1.** `@Service` 어노테이션이 `hasAnnotation("Component")`가 아닌 `hasMetaAnnotation("Component")`으로 감지되는 이유는?

**Q2.** `abstract` 클래스에 `@Component`를 붙이면 Bean으로 등록되는가?

**Q3.** `CachingMetadataReaderFactory`의 캐시 크기(기본 256)를 초과하면 어떤 일이 벌어지는가?

> 💡 **해설**
>
> **Q1.** `OrderService` 클래스에 실제로 붙어 있는 것은 `@Service`이지 `@Component`가 아니다. `hasAnnotation("Component")`는 `false`다. ASM이 `@Service` 어노테이션 자체의 어노테이션(`@Component`)까지 재귀적으로 탐색하며 `hasMetaAnnotation("Component")`가 `true`가 된다. `AnnotationTypeFilter.match()`는 두 조건을 OR로 평가하기 때문에 `@Service`, `@Repository`, `@Controller` 등 `@Component` 파생 어노테이션이 모두 자동으로 감지된다.
>
> **Q2.** 기본적으로 등록되지 않는다. 2단계 `isCandidateComponent()`에서 `metadata.isConcrete()`가 `false`(abstract 클래스)이므로 탈락한다. 예외적으로 abstract 클래스에 `@Lookup` 어노테이션이 붙은 메서드가 있으면 `hasAnnotatedMethods(Lookup.class.getName())`가 `true`가 되어 Bean 후보로 허용된다. 스프링이 CGLIB으로 abstract 메서드를 구현해 Prototype Bean을 반환하는 기능이다.
>
> **Q3.** LRU 정책에 따라 가장 오래 접근되지 않은 엔트리가 제거된다. 제거된 `Resource`에 다음 번 접근 시 다시 파일 I/O가 발생한다. 대규모 프로젝트에서 스캔 대상이 256개를 초과하면 캐시 효율이 떨어져 시작 시간이 증가할 수 있다. `spring.context.cache.limit` 프로퍼티로 캐시 크기를 조정할 수 있다.

---

<div align="center">

**[⬅️ 이전: @ComponentScan 동작 원리](./01-componentscan-internals.md)** | **[다음: TypeFilter Include & Exclude ➡️](./03-typefilter-include-exclude.md)**

</div>

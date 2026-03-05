# TypeFilter Include & Exclude — 스캔 대상을 정밀하게 제어하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `includeFilters`와 `excludeFilters`의 평가 순서는? `excludeFilter`가 우선인가?
- `FilterType`의 5가지 종류(`ANNOTATION`, `ASSIGNABLE_TYPE`, `ASPECTJ`, `REGEX`, `CUSTOM`)는 내부에서 어떻게 처리되는가?
- `useDefaultFilters=false`로 설정하면 정확히 무슨 일이 생기는가?
- 커스텀 `TypeFilter`를 구현할 때 `MetadataReader`에서 어떤 정보를 꺼낼 수 있는가?
- Spring Boot 테스트 슬라이스(`@WebMvcTest`, `@DataJpaTest`)가 TypeFilter를 어떻게 활용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: `@Component`만으로는 스캔 대상을 세밀하게 제어할 수 없다

```java
// @Component 붙은 모든 클래스가 등록되는 기본 동작
@ComponentScan("com.example")

// 원하는 것:
// - 특정 패키지 클래스 제외
// - @Component 없어도 특정 타입은 포함
// - 클래스명 패턴으로 필터링
// - 커스텀 조건 적용 (@Conditional과 다른 레이어)
```

```
TypeFilter가 해결하는 것:
  includeFilters → @Component 없어도 Bean으로 등록
  excludeFilters → @Component 있어도 Bean 제외

  두 필터는 ASM MetadataReader 레벨에서 동작
  → 클래스 로드 없이 빠르게 판별
  → @Conditional보다 먼저, 더 낮은 비용으로 처리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: includeFilter가 excludeFilter보다 먼저 평가된다

```java
@ComponentScan(
    basePackages = "com.example",
    includeFilters  = @Filter(type = REGEX, pattern = ".*Service"),
    excludeFilters  = @Filter(type = REGEX, pattern = ".*Internal.*")
)
```

```
❌ 잘못된 이해:
  "includeFilter 통과 → excludeFilter 검사"

✅ 실제 (isCandidateComponent 코드):
  excludeFilter 먼저 평가 → 제외 확정 시 즉시 탈락
  excludeFilter 통과 후 → includeFilter 평가

  "OrderInternalService"
  → excludeFilter(".*Internal.*") = true → 즉시 탈락
  → includeFilter 평가 없음

이유: 제외가 포함보다 의미론적으로 강함
     excludeFilter 통과 못 하면 포함 여부를 따질 필요도 없음
```

### Before: `useDefaultFilters=false`면 아무것도 등록 안 된다

```java
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false
)
```

```
❌ 잘못된 이해: "아무 Bean도 등록 안 된다"

✅ 실제:
  useDefaultFilters=false → AnnotationTypeFilter(@Component) 기본 필터만 제거
  → @Component / @Service / @Repository / @Controller 자동 감지 OFF

  명시적 includeFilters가 있으면 그것만 기준으로 등록
  없으면 → 아무것도 등록 안 됨

활용:
  특정 어노테이션만 Bean으로 등록하고 싶을 때
  @ComponentScan(useDefaultFilters=false,
                 includeFilters = @Filter(MyAnnotation.class))
```

---

## ✨ 올바른 이해와 사용

### After: FilterType별 동작과 평가 순서를 명확히 구분

```
FilterType 5종류:

ANNOTATION    (기본) 어노테이션 타입으로 필터
              → AnnotationTypeFilter 사용
              → 메타 어노테이션 체인 탐색 가능

ASSIGNABLE_TYPE  특정 타입의 서브클래스/구현체 필터
              → AssignableTypeFilter 사용
              → 실제 클래스 로드 필요 (isAssignableFrom 검사)

ASPECTJ       AspectJ 타입 패턴
              → AspectJTypeFilter 사용
              → aspectjweaver 의존성 필요

REGEX         정규식으로 클래스명 필터
              → RegexPatternTypeFilter 사용
              → 클래스의 이진 이름(패키지 포함)에 매칭

CUSTOM        TypeFilter 직접 구현
              → MetadataReader / ClassMetadata / AnnotationMetadata 활용
              → 가장 유연
```

---

## 🔬 내부 동작 원리

### 1. isCandidateComponent() — 필터 평가 순서

```java
// ClassPathScanningCandidateComponentProvider.isCandidateComponent()
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {

    // ① excludeFilter 먼저 — 하나라도 매칭되면 즉시 탈락
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return false;
        }
    }

    // ② includeFilter — 하나라도 매칭되면 @Conditional 평가로 진행
    for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return isConditionMatch(metadataReader);
        }
    }

    // 아무 includeFilter도 매칭 안 됨 → 탈락
    return false;
}
```

### 2. AnnotationTypeFilter — 어노테이션 기반 (가장 일반적)

```java
// FilterType.ANNOTATION → AnnotationTypeFilter 생성
public class AnnotationTypeFilter extends AbstractTypeHierarchyTraversingFilter {

    private final Class<? extends Annotation> annotationType;
    private final boolean considerMetaAnnotations;  // 기본 true

    @Override
    protected boolean matchSelf(MetadataReader metadataReader) {
        AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();

        // 직접 선언 OR 메타 어노테이션으로 포함
        return metadata.hasAnnotation(this.annotationType.getName())
            || (this.considerMetaAnnotations
                && metadata.hasMetaAnnotation(this.annotationType.getName()));
    }

    // considerInheritedAnnotations=true 시 수퍼클래스도 탐색
    @Override
    protected Boolean matchSuperClass(String superClassName) {
        return hasAnnotation(superClassName);
    }
}
```

```java
// 사용 예
@ComponentScan(
    includeFilters = @Filter(
        type = FilterType.ANNOTATION,
        classes = MyCustomAnnotation.class
    )
)
// → @MyCustomAnnotation 붙은 클래스 + 해당 어노테이션을 메타 어노테이션으로 포함한 클래스 모두 포함
```

### 3. AssignableTypeFilter — 타입 계층 기반

```java
// FilterType.ASSIGNABLE_TYPE → AssignableTypeFilter 생성
public class AssignableTypeFilter extends AbstractTypeHierarchyTraversingFilter {

    private final Class<?> targetType;

    @Override
    protected boolean matchClassName(String className) {
        return this.targetType.getName().equals(className);
    }

    @Override
    protected Boolean matchSuperClass(String superClassName) {
        return isAssignable(superClassName);
    }

    @Override
    protected Boolean matchInterface(String interfaceName) {
        return isAssignable(interfaceName);
    }

    private Boolean isAssignable(String typeName) {
        try {
            Class<?> clazz = ClassUtils.forName(typeName, ...);
            // ← 실제 클래스 로드 발생!
            return this.targetType.isAssignableFrom(clazz);
        } catch (ClassNotFoundException ex) {
            return null;
        }
    }
}
```

```
AssignableTypeFilter 주의:
  수퍼클래스 / 인터페이스 체인 탐색 시 실제 클래스 로드 발생
  → AnnotationTypeFilter보다 비용 높음
  → 대규모 스캔에서 성능에 영향 가능

사용 예:
  @ComponentScan(
      includeFilters = @Filter(
          type = FilterType.ASSIGNABLE_TYPE,
          classes = Repository.class  // Repository 인터페이스 구현체 모두 포함
      )
  )
```

### 4. RegexPatternTypeFilter — 정규식 기반

```java
// FilterType.REGEX → RegexPatternTypeFilter 생성
public class RegexPatternTypeFilter implements TypeFilter {

    private final Pattern pattern;

    @Override
    public boolean match(MetadataReader metadataReader, ...) {
        // 클래스의 이진 이름(패키지.클래스명)에 정규식 매칭
        String className = metadataReader.getClassMetadata().getClassName();
        return this.pattern.matcher(className).matches();
    }
}
```

```java
@ComponentScan(
    excludeFilters = @Filter(
        type = FilterType.REGEX,
        pattern = "com\\.example\\..*\\.internal\\..*"
    )
)
// → "com.example.*.internal.*" 패키지의 클래스 모두 제외
```

### 5. Custom TypeFilter — 가장 유연한 방식

```java
// TypeFilter 인터페이스
public interface TypeFilter {
    boolean match(MetadataReader metadataReader,
                  MetadataReaderFactory metadataReaderFactory) throws IOException;
}

// 구현 예: @Component이면서 특정 메서드가 있는 클래스만 포함
public class HasHandleMethodFilter implements TypeFilter {

    @Override
    public boolean match(MetadataReader metadataReader, ...) {
        AnnotationMetadata am = metadataReader.getAnnotationMetadata();
        ClassMetadata cm = metadataReader.getClassMetadata();

        // @Component 계열 어노테이션이 있고
        if (!am.hasMetaAnnotation("org.springframework.stereotype.Component")) {
            return false;
        }

        // "handle"로 시작하는 메서드가 하나 이상 있으면 포함
        return am.getAnnotatedMethods("").stream()
            .anyMatch(m -> m.getMethodName().startsWith("handle"));
        // 또는 ClassMetadata, 수퍼클래스 탐색 등 조합 가능
    }
}

// 등록
@ComponentScan(
    includeFilters = @Filter(type = FilterType.CUSTOM,
                             classes = HasHandleMethodFilter.class)
)
```

### 6. Spring Boot 테스트 슬라이스 — TypeExcludeFilter

```java
// @WebMvcTest, @DataJpaTest 등 테스트 슬라이스의 핵심 메커니즘
// @SpringBootApplication이 기본 등록하는 excludeFilter

@ComponentScan(
    excludeFilters = @Filter(
        type = FilterType.CUSTOM,
        classes = TypeExcludeFilter.class  // ← 확장 가능한 exclude 메커니즘
    )
)

// TypeExcludeFilter는 BeanFactory에서 TypeExcludeFilter 구현 Bean을 탐색해 위임
public class TypeExcludeFilter implements TypeFilter, BeanFactoryAware {

    @Override
    public boolean match(MetadataReader metadataReader, ...) {
        if (this.beanFactory instanceof ListableBeanFactory lbf) {
            // 컨텍스트에 등록된 모든 TypeExcludeFilter 구현체에 위임
            for (TypeExcludeFilter delegate : getDelegates(lbf)) {
                if (delegate.match(metadataReader, metadataReaderFactory)) {
                    return true;
                }
            }
        }
        return false;
    }
}

// @WebMvcTest 내부:
// WebMvcTypeExcludeFilter implements TypeExcludeFilter
// → @Controller, @ControllerAdvice, @JsonComponent 등만 허용
// → @Service, @Repository 등 제외
// → 웹 레이어만 로드되는 슬라이스 구성
```

---

## 💻 실험으로 확인하기

### 실험 1: excludeFilter 우선 확인

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters  = @Filter(type = REGEX, pattern = ".*Service"),
    excludeFilters  = @Filter(type = REGEX, pattern = ".*Internal.*"),
    useDefaultFilters = false
)
class AppConfig {}

// com.example.OrderService        → include 매칭, exclude 미매칭 → 등록 O
// com.example.OrderInternalService → include 매칭이지만 exclude 매칭 → 등록 X
// com.example.UserRepository      → include 미매칭 → 등록 X
```

### 실험 2: useDefaultFilters=false + includeFilter 조합

```java
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false,  // @Component 자동 감지 OFF
    includeFilters = @Filter(
        type = FilterType.ANNOTATION,
        classes = MyDomainService.class  // 커스텀 어노테이션만 등록
    )
)
```

### 실험 3: Custom TypeFilter 구현

```java
// 클래스명이 "Impl"로 끝나는 클래스만 제외
public class ImplClassExcludeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader reader, ...) {
        return reader.getClassMetadata().getClassName().endsWith("Impl");
    }
}

@ComponentScan(
    excludeFilters = @Filter(
        type = FilterType.CUSTOM,
        classes = ImplClassExcludeFilter.class
    )
)
// → OrderServiceImpl, PaymentServiceImpl 등 제외
// → 인터페이스 구현체를 직접 Bean으로 노출하지 않을 때 유용
```

---

## 🤔 트레이드오프

```
FilterType 성능 비교:
  ANNOTATION    → ASM 메타데이터만 → 클래스 로드 없음 → 빠름
  REGEX         → 클래스명 문자열 비교 → 빠름
  ASSIGNABLE_TYPE → 수퍼클래스/인터페이스 탐색 시 클래스 로드 → 느림
  CUSTOM        → 구현에 따라 다름 (MetadataReader만 쓰면 빠름)

useDefaultFilters=false 활용:
  테스트 슬라이스: 특정 레이어 Bean만 로드
  멀티모듈: 모듈별 전용 어노테이션으로 스캔 범위 명확화
  레거시 통합: 기존 어노테이션 체계 유지하면서 스프링 통합

CUSTOM Filter 주의:
  TypeFilter는 MetadataReader에서 읽을 수 있는 정보로만 판별
  런타임 상태, 다른 Bean 존재 여부 등은 @Conditional로 처리
  무거운 로직(I/O, 네트워크) 금지 → 스캔 성능 저하

excludeFilter + includeFilter 혼용:
  excludeFilter가 항상 우선
  같은 클래스가 양쪽에 모두 매칭되면 → 항상 제외
```

---

## 📌 핵심 정리

```
평가 순서
  excludeFilter 먼저 → 통과 시 includeFilter → 통과 시 @Conditional
  excludeFilter 매칭 → 즉시 탈락 (includeFilter 평가 없음)

FilterType 5종류
  ANNOTATION      AnnotationTypeFilter — 메타 어노테이션 체인 포함, 빠름
  ASSIGNABLE_TYPE AssignableTypeFilter — 타입 계층 탐색, 클래스 로드 가능
  ASPECTJ         AspectJTypeFilter    — AspectJ 패턴 표현식
  REGEX           RegexPatternTypeFilter — 클래스 이진 이름에 정규식
  CUSTOM          TypeFilter 직접 구현 — 가장 유연

useDefaultFilters
  true (기본): AnnotationTypeFilter(@Component) 자동 포함
  false: @Component 자동 감지 OFF → includeFilters 명시 필요

TypeExcludeFilter (Spring Boot)
  @SpringBootApplication 기본 excludeFilter
  컨텍스트의 TypeExcludeFilter 구현 Bean에 위임
  → @WebMvcTest 등 테스트 슬라이스의 핵심 메커니즘
```

---

## 🤔 생각해볼 문제

**Q1.** `ASSIGNABLE_TYPE` 필터가 `ANNOTATION` 필터보다 느릴 수 있는 이유를 클래스 로드 관점에서 설명하라.

**Q2.** `useDefaultFilters=false` 설정 시 `@Configuration` 클래스도 스캔에서 제외되는가?

**Q3.** Custom TypeFilter 내에서 다른 Bean의 존재 여부를 확인해야 한다면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** `ANNOTATION` 필터는 ASM이 파싱한 `AnnotationMetadata`를 그대로 사용한다. 클래스 로드 없이 바이트코드에서 읽은 어노테이션 정보로만 판별한다. 반면 `ASSIGNABLE_TYPE` 필터는 수퍼클래스나 구현 인터페이스 체인을 따라가면서 타겟 타입의 하위 타입인지 확인해야 한다. 체인을 탐색할 때 중간 클래스들이 클래스패스에 있어야 하고 `isAssignableFrom()` 검사를 위해 `Class.forName()`이 호출될 수 있다. 이 클래스 로드 비용이 누적되면 대규모 스캔에서 성능 차이로 나타난다.
>
> **Q2.** `useDefaultFilters=false`는 `AnnotationTypeFilter(@Component)` 기본 필터만 제거한다. `@Configuration`은 `@Component`를 메타 어노테이션으로 포함하므로 기본 필터로 감지되는데, 기본 필터가 제거되면 `@Configuration` 클래스도 자동 감지 대상에서 빠진다. 단, `@SpringBootApplication`이나 명시적으로 지정된 `@Configuration` 클래스는 스캔을 통해 등록되는 것이 아니라 `ConfigurationClassPostProcessor`의 초기 후보 목록에서 처리되므로 영향을 받지 않는다. 스캔으로 추가 발견되는 `@Configuration` 클래스가 제외된다.
>
> **Q3.** TypeFilter는 스캔 단계에서 실행되는데, 이 시점은 대부분의 Bean이 아직 생성되지 않은 상태다. TypeFilter 내에서 `BeanFactory`를 참조하려면 `BeanFactoryAware` 인터페이스를 구현해 `BeanFactory`를 주입받으면 된다(Spring Boot의 `TypeExcludeFilter`가 이 방식). 그러나 스캔 시점에 `BeanFactory`에는 이미 등록된 `BeanDefinition`만 있고 완성된 Bean 인스턴스가 없다. Bean 존재 여부를 확인하려면 `beanFactory.containsBeanDefinition("beanName")`으로 정의 등록 여부만 확인할 수 있다. 실행 중인 Bean 상태에 의존하는 조건이라면 TypeFilter가 아닌 `@Conditional`로 처리하는 것이 올바르다.

---

<div align="center">

**[⬅️ 이전: ClassPathScanningCandidateComponentProvider](./02-candidate-component-provider.md)** | **[다음: @Conditional 평가 과정 ➡️](./04-conditional-evaluation.md)**

</div>

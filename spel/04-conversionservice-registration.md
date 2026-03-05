# ConversionService 등록과 우선순위 — DefaultConversionService와 커스텀 Converter

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DefaultConversionService`에 기본 등록된 Converter 목록과 카테고리는?
- `ConversionService`가 `ApplicationContext`에 어떻게 등록되는가?
- 커스텀 `Converter`를 등록하는 방법과 기존 Converter를 오버라이딩하는 방식은?
- `GenericConversionService.convert()`가 최적 Converter를 찾는 알고리즘은?
- Spring MVC의 `FormattingConversionService`는 `DefaultConversionService`와 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: 타입 변환이 여러 곳에 분산되면 일관성이 없어진다

```java
// 분산된 변환 — 일관성 없음
String value = "2024-01-15";

// 컨트롤러 A: 자체 변환
LocalDate date1 = LocalDate.parse(value, DateTimeFormatter.ISO_LOCAL_DATE);

// 서비스 B: 다른 방식
LocalDate date2 = LocalDate.parse(value);

// Bean 주입: PropertyEditor
@Value("${birth.date}")
private Date date3;  // 어떤 형식인가?
```

```
ConversionService가 해결하는 것:
  타입 변환 로직의 중앙화
  → 한 곳에 등록 → 어디서든 일관된 변환
  → 프레임워크(@Value, @RequestParam, SpEL)가 모두 같은 ConversionService 사용
  → 커스텀 변환 규칙을 한 번만 정의
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: ConversionService는 자동으로 사용된다

```
❌ 잘못된 이해:
  "Converter를 Bean으로 등록하면 자동으로 ConversionService가 감지한다"

✅ 실제:
  Converter를 단순히 @Component로 등록하면 동작 안 함
  → ConversionService에 명시적으로 추가해야 함

  방법 1: ConversionServiceFactoryBean 사용
    @Bean ConversionServiceFactoryBean: 커스텀 Converter Set 주입

  방법 2: WebMvcConfigurer.addFormatters()
    → Spring MVC FormattingConversionService에 등록

  방법 3: ApplicationConversionService (Spring Boot)
    → SpringApplication이 자동으로 ConversionService 구성
    → Converter Bean 자동 등록 지원
```

### Before: 같은 타입 쌍의 Converter를 두 개 등록하면 예외가 발생한다

```
❌ 잘못된 이해:
  "String → LocalDate Converter가 두 개면 충돌 오류"

✅ 실제:
  나중에 등록된 Converter가 이전 것을 덮어씀
  → converters Map의 키 = (sourceType, targetType) 쌍
  → 같은 키로 addConverter() 호출 시 덮어쓰기 (오류 없음)
  → 이 특성을 이용해 기본 Converter 오버라이딩 가능
```

---

## ✨ 올바른 이해와 사용

### After: ConversionService 계층 구조

```
ConversionService (인터페이스)
  canConvert(sourceType, targetType)
  convert(source, targetType)
  ↓
GenericConversionService (핵심 구현)
  converters: ConvertersForPair 맵 관리
  Converter/ConverterFactory/GenericConverter 지원
  ↓
DefaultConversionService extends GenericConversionService
  기본 Converter 200개+ 자동 등록
  ↓
ApplicationConversionService (Spring Boot)
  Spring Boot 전용 Converter 추가 등록
  @ConfigurationProperties 바인딩 지원
  ↓
FormattingConversionService extends GenericConversionService
  Formatter 지원 (Locale 기반 포맷)
  DefaultFormattingConversionService: 기본 포매터 등록
  ↓
(Spring MVC)
  WebConversionService: MVC 전용 추가 변환
```

---

## 🔬 내부 동작 원리

### 1. DefaultConversionService 기본 등록 Converter

```java
// DefaultConversionService.addDefaultConverters()
public static void addDefaultConverters(ConverterRegistry converterRegistry) {

    // 스칼라 변환 (문자열/숫자 기본형)
    addScalarConverters(converterRegistry);

    // 컬렉션 변환
    converterRegistry.addConverter(new ArrayToCollectionConverter(converterRegistry));
    converterRegistry.addConverter(new CollectionToArrayConverter(converterRegistry));
    converterRegistry.addConverter(new ArrayToArrayConverter(converterRegistry));
    converterRegistry.addConverter(new CollectionToCollectionConverter(converterRegistry));
    converterRegistry.addConverter(new MapToMapConverter(converterRegistry));
    converterRegistry.addConverter(new ArrayToStringConverter(converterRegistry));
    converterRegistry.addConverter(new StringToArrayConverter(converterRegistry));
    converterRegistry.addConverter(new ArrayToObjectConverter(converterRegistry));
    converterRegistry.addConverter(new ObjectToArrayConverter(converterRegistry));
    converterRegistry.addConverter(new CollectionToStringConverter(converterRegistry));
    converterRegistry.addConverter(new StringToCollectionConverter(converterRegistry));
    converterRegistry.addConverter(new CollectionToObjectConverter(converterRegistry));
    converterRegistry.addConverter(new ObjectToCollectionConverter(converterRegistry));
    converterRegistry.addConverter(new StreamConverter(converterRegistry));

    // Object 범용 변환
    converterRegistry.addConverter(new ObjectToStringConverter());
    converterRegistry.addConverter(new ObjectToObjectConverter());
    converterRegistry.addConverter(new IdToEntityConverter(converterRegistry));
    converterRegistry.addConverter(new FallbackObjectToStringConverter());
    converterRegistry.addConverter(new ObjectToOptionalConverter(converterRegistry));
}

private static void addScalarConverters(ConverterRegistry converterRegistry) {
    converterRegistry.addConverterFactory(new NumberToNumberConverterFactory());
    converterRegistry.addConverterFactory(new StringToNumberConverterFactory());
    // "8080" → Integer, Long, Double 등 모든 Number 타입
    converterRegistry.addConverter(new NumberToCharacterConverter());
    converterRegistry.addConverterFactory(new CharacterToNumberFactory());
    converterRegistry.addConverter(new StringToCharacterConverter());  // "A" → 'A'
    converterRegistry.addConverter(new NumberToStringConverter());
    converterRegistry.addConverterFactory(new StringToEnumConverterFactory()); // "ROLE_USER" → Enum
    converterRegistry.addConverter(new EnumToStringConverter(converterRegistry));
    converterRegistry.addConverterFactory(new IntegerToEnumConverterFactory()); // 0 → Enum
    converterRegistry.addConverter(new EnumToIntegerConverter(converterRegistry));
    converterRegistry.addConverter(new StringToBooleanConverter()); // "true"/"false"/"1"/"0"
    converterRegistry.addConverter(new StringToLocaleConverter()); // "en_US" → Locale
    converterRegistry.addConverter(new LocaleToStringConverter());
    converterRegistry.addConverter(new StringToCharsetConverter()); // "UTF-8" → Charset
    converterRegistry.addConverter(new StringToCurrencyConverter()); // "USD" → Currency
    converterRegistry.addConverter(new StringToPropertiesConverter());
    converterRegistry.addConverter(new StringToUUIDConverter()); // UUID 문자열 → UUID
    converterRegistry.addConverter(new StringToDurationConverter()); // "PT1H" → Duration
    converterRegistry.addConverter(new StringToPeriodConverter());  // "P1D" → Period
    converterRegistry.addConverter(new StringToDataSizeConverter()); // "10MB" → DataSize
}
```

### 2. GenericConversionService — Converter 조회 알고리즘

```java
// GenericConversionService.convert()
public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {

    GenericConverter converter = getConverter(sourceType, targetType);
    if (converter == null) {
        // 변환 불가 → ConverterNotFoundException
        throw new ConverterNotFoundException(sourceType, targetType,
            "No converter found for [" + sourceType + "] to [" + targetType + "]");
    }
    return converter.convert(source, sourceType, targetType);
}

// getConverter() 조회 순서
private GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // 1. 캐시 조회 (ConverterCacheKey → GenericConverter)
    ConverterCacheKey key = new ConverterCacheKey(sourceType, targetType);
    GenericConverter converter = this.converterCache.get(key);
    if (converter != null) {
        return (converter != NO_MATCH ? converter : null);
    }

    // 2. 정확한 타입 쌍 매칭
    converter = this.converters.find(sourceType, targetType);
    if (converter == null) {
        // 3. 타입 계층 탐색 (슈퍼클래스/인터페이스)
        converter = getDefaultConverter(sourceType, targetType);
    }

    // 4. 결과 캐시 저장
    this.converterCache.put(key, (converter != null ? converter : NO_MATCH));
    return converter;
}
```

```java
// Converters.find() — 타입 계층 탐색
GenericConverter find(TypeDescriptor sourceType, TypeDescriptor targetType) {
    // 소스 타입의 클래스 계층 (자신 → 슈퍼클래스 → 인터페이스)
    List<Class<?>> sourceCandidates = getClassHierarchy(sourceType.getType());
    // 타겟 타입의 클래스 계층
    List<Class<?>> targetCandidates = getClassHierarchy(targetType.getType());

    // 소스 계층 × 타겟 계층 조합으로 매칭 탐색
    for (Class<?> sourceCandidate : sourceCandidates) {
        for (Class<?> targetCandidate : targetCandidates) {
            ConvertiblePair convertiblePair =
                new ConvertiblePair(sourceCandidate, targetCandidate);
            GenericConverter converter = getRegisteredConverter(convertiblePair);
            if (converter != null) return converter;
        }
    }
    return null;
}
```

### 3. ConversionService 등록 방법

```java
// 방법 1: ConversionServiceFactoryBean (순수 Spring)
@Bean
public ConversionServiceFactoryBean conversionService() {
    ConversionServiceFactoryBean factory = new ConversionServiceFactoryBean();
    Set<Object> converters = new HashSet<>();
    converters.add(new StringToLocalDateConverter());
    converters.add(new OrderStatusConverter());
    factory.setConverters(converters);
    return factory;
}
// → "conversionService" 이름으로 등록 → AbstractBeanFactory가 자동 감지

// 방법 2: Spring MVC WebMvcConfigurer.addFormatters()
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToLocalDateConverter());
        registry.addConverter(new OrderStatusConverter());
        // FormatterRegistry는 ConverterRegistry를 확장
    }
}

// 방법 3: Spring Boot ApplicationConversionService
// → Converter 타입 Bean을 자동으로 등록
@Component
public class StringToLocalDateConverter implements Converter<String, LocalDate> {
    @Override
    public LocalDate convert(String source) {
        return LocalDate.parse(source);
    }
}
// Spring Boot가 자동으로 ApplicationConversionService에 등록
```

### 4. AbstractBeanFactory의 ConversionService 자동 감지

```java
// AbstractBeanFactory.getConversionService()
// → "conversionService" 이름의 Bean을 자동 감지
protected ConversionService getConversionService() {
    if (this.conversionService != null) return this.conversionService;

    // BeanFactory 자체에서 ConversionService Bean 조회
    if (containsLocalBean(CONVERSION_SERVICE_BEAN_NAME)) {
        this.conversionService = getBean(CONVERSION_SERVICE_BEAN_NAME,
            ConversionService.class);
    }
    return this.conversionService;
}

public static final String CONVERSION_SERVICE_BEAN_NAME = "conversionService";
// → 정확히 "conversionService" 이름이어야 자동 감지됨
```

### 5. 커스텀 Converter로 기본 Converter 오버라이딩

```java
// 기본 StringToEnumConverterFactory는 Enum.valueOf(name) 사용
// → "ACTIVE" → Status.ACTIVE (대소문자 민감)

// 커스텀: 대소문자 무시 변환
public class StringToStatusConverter implements Converter<String, Status> {
    @Override
    public Status convert(String source) {
        return Status.valueOf(source.toUpperCase().trim());
    }
}

// 등록 (나중에 등록 → 덮어쓰기)
@Override
public void addFormatters(FormatterRegistry registry) {
    // String → Status 타입 쌍에 대해 기존 ConverterFactory 대신 이 Converter 사용
    registry.addConverter(new StringToStatusConverter());
}

// 주의: ConverterFactory 전체가 아닌 특정 타입 쌍만 오버라이딩
// String → Status: 커스텀
// String → OtherEnum: 기본 StringToEnumConverterFactory 사용
```

### 6. FormattingConversionService — Locale 기반 포맷

```java
// DefaultFormattingConversionService
// = DefaultConversionService + Formatter 지원

// Formatter<T>: Printer<T> + Parser<T>
// → Locale을 고려한 포맷/파싱

public class LocalDateTimeFormatter implements Formatter<LocalDateTime> {

    @Override
    public String print(LocalDateTime object, Locale locale) {
        // Locale 기반 포맷 (US → "Jan 15, 2024", KR → "2024년 1월 15일")
        return DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(locale)
            .format(object);
    }

    @Override
    public LocalDateTime parse(String text, Locale locale) throws ParseException {
        return LocalDateTime.parse(text,
            DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
                .withLocale(locale));
    }
}

// 등록
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addFormatter(new LocalDateTimeFormatter());
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 기본 등록 Converter 확인

```java
DefaultConversionService cs = new DefaultConversionService();

// String → 기본 타입
System.out.println(cs.convert("8080", Integer.class));   // 8080
System.out.println(cs.convert("true", Boolean.class));   // true
System.out.println(cs.convert("USER", UserRole.class));  // UserRole.USER

// 컬렉션 변환
System.out.println(cs.convert("a,b,c", String[].class)); // ["a", "b", "c"]

// UUID
System.out.println(cs.convert(
    "550e8400-e29b-41d4-a716-446655440000", UUID.class));
```

### 실험 2: 커스텀 Converter 등록 및 오버라이딩

```java
DefaultConversionService cs = new DefaultConversionService();

// 기본: "true" → true
System.out.println(cs.convert("yes", Boolean.class));  // → IllegalArgumentException

// 커스텀 등록: "yes"/"no" → Boolean
cs.addConverter(String.class, Boolean.class, s ->
    "yes".equalsIgnoreCase(s) || "true".equalsIgnoreCase(s));

System.out.println(cs.convert("yes", Boolean.class));  // → true
System.out.println(cs.convert("no", Boolean.class));   // → false
```

### 실험 3: canConvert() 확인

```java
DefaultConversionService cs = new DefaultConversionService();

System.out.println(cs.canConvert(String.class, LocalDate.class));    // true
System.out.println(cs.canConvert(String.class, Integer.class));      // true
System.out.println(cs.canConvert(Integer.class, LocalDate.class));   // false (기본 등록 없음)
System.out.println(cs.canConvert(String.class, MyCustomClass.class));// false
```

---

## 🤔 트레이드오프

```
DefaultConversionService vs FormattingConversionService:
  Default:     순수 타입 변환 (Locale 무관)
  Formatting:  Locale 기반 포맷 + 타입 변환
  → 웹 애플리케이션: FormattingConversionService 권장
  → 배치/백엔드 서비스: DefaultConversionService로 충분

ConversionService 등록 방식:
  ConversionServiceFactoryBean: Spring 표준, 순수 환경
  WebMvcConfigurer.addFormatters(): Spring MVC 전용, 가장 일반적
  Spring Boot: Converter Bean 자동 등록, 설정 최소화

Converter 캐시:
  GenericConversionService 내부 ConcurrentHashMap 캐시
  → 동일 타입 쌍 반복 조회 시 O(1)
  → converterCache.invalidateCache()로 수동 무효화 가능 (Converter 추가 시 자동)
  → 대규모 타입 쌍이 있으면 캐시 크기 증가 주의

오버라이딩 주의:
  기본 Converter를 오버라이딩하면 예상치 못한 부작용 가능
  → String → Boolean 기본 변환을 바꾸면 전체 애플리케이션에 영향
  → 특정 컨텍스트만 다른 변환이 필요하면 @InitBinder로 컨트롤러 범위 제한
```

---

## 📌 핵심 정리

```
ConversionService 계층
  GenericConversionService → DefaultConversionService
  → ApplicationConversionService (Spring Boot)
  → FormattingConversionService (MVC, Locale 지원)

기본 등록 Converter 카테고리
  스칼라: String ↔ Number/Boolean/Enum/UUID/Duration 등
  컬렉션: Array ↔ Collection ↔ String, Stream 변환
  범용: Object → String, Optional 래핑

자동 감지
  "conversionService" 이름으로 Bean 등록 → AbstractBeanFactory 자동 인식

Converter 조회 알고리즘
  캐시 조회 → 정확한 타입 쌍 → 타입 계층 탐색 (소스×타겟 계층 조합)
  캐시 키: (sourceType, targetType) → ConcurrentHashMap

오버라이딩
  동일 타입 쌍으로 addConverter() → 이전 것 덮어씀 (오류 없음)
  Spring Boot: Converter Bean 자동 등록 → ApplicationConversionService에 추가
```

---

## 🤔 생각해볼 문제

**Q1.** `ConversionService`에 `String → MyEnum` Converter를 등록했는데 `@RequestParam`에서 `String → MyEnum` 변환이 동작하지 않는다. 이유는 무엇인가?

**Q2.** `GenericConversionService`의 타입 계층 탐색은 `Animal` 인터페이스를 구현한 `Dog` 클래스에서 `Animal` 로 등록된 Converter를 찾을 수 있는가?

**Q3.** Spring Boot에서 `Converter<String, LocalDate>` Bean을 등록했는데 `@Value("${birth.date}")` 주입에 적용되지 않는다. 이유와 해결책은?

> 💡 **해설**
>
> **Q1.** `@RequestParam`은 Spring MVC의 `FormattingConversionService`를 사용한다. `WebMvcConfigurer.addFormatters()`가 아닌 다른 방법(예: `ConversionServiceFactoryBean`)으로 등록한 `ConversionService`는 Spring MVC의 `FormattingConversionService`에 연결되지 않는다. 해결책은 `WebMvcConfigurer.addFormatters()`에서 `registry.addConverter()`로 등록하거나, Spring Boot라면 `Converter` Bean을 `@Component`로 등록하면 자동으로 `ApplicationConversionService`에 추가된다.
>
> **Q2.** 찾을 수 있다. `GenericConversionService.getClassHierarchy()`는 소스 타입 `Dog`의 클래스 계층을 탐색할 때 `Dog` → `Object` (슈퍼클래스 체인)와 `Dog`가 구현하는 인터페이스 `Animal`을 모두 포함한다. 따라서 `Animal → String` Converter가 등록돼 있으면 `Dog → String` 변환 요청 시 `Animal` 기반 Converter가 발견되어 사용된다. 단, 정확히 `Dog` 타입으로 등록된 Converter가 있으면 그것이 우선한다.
>
> **Q3.** `@Value("${...}")`의 타입 변환은 `AbstractBeanFactory`의 `TypeConverter`를 사용하며, 이는 `"conversionService"` 이름의 `ConversionService` Bean을 감지한다. Spring Boot의 `ApplicationConversionService`는 별도 이름으로 등록될 수 있어 `"conversionService"` Bean이 따로 없으면 감지되지 않는다. 해결책은 `@Bean(name = "conversionService")` 로 명시적 이름을 지정하거나 `ConversionServiceFactoryBean`으로 `"conversionService"` 이름의 Bean을 등록하는 것이다. Spring Boot에서는 `spring.mvc.converters.preferred-json-mapper` 등의 자동 설정 경로를 확인해야 한다.

---

<div align="center">

**[⬅️ 이전: PropertyEditor vs Converter](./03-propertyeditor-vs-converter.md)** | **[다음: Custom Type Converter 작성 ➡️](./05-custom-type-converter.md)**

</div>

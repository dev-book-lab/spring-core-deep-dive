# Custom Type Converter 작성 — Converter / GenericConverter / ConverterFactory 선택 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Converter<S, T>`, `GenericConverter`, `ConverterFactory<S, R>` 세 가지의 정확한 사용 시점은?
- `GenericConverter`가 `TypeDescriptor`를 받는 이유는 무엇인가?
- `ConditionalConverter`를 구현하면 어떤 유연성이 생기는가?
- `ConverterFactory`가 `Converter`보다 `Enum` 변환에 적합한 이유는?
- `@NumberFormat`, `@DateTimeFormat` 어노테이션은 어떻게 타입 변환에 연결되는가?

---

## 🔍 왜 이게 존재하는가

### 세 가지 인터페이스는 서로 다른 수준의 유연성을 제공한다

```java
// 요구사항별 선택:

// 1. String → LocalDate (1:1 단순 변환)
//    → Converter<String, LocalDate>

// 2. String → 모든 Enum 타입 (1:N 변환)
//    → ConverterFactory<String, Enum>
//    → 타겟 Enum 타입에 맞는 Converter를 동적으로 생성

// 3. List<String> → List<Integer> (제네릭 타입 정보 필요)
//    → GenericConverter
//    → TypeDescriptor로 List의 요소 타입까지 접근

// 4. @CsvFormat 어노테이션이 있을 때만 활성화
//    → GenericConverter + ConditionalConverter
//    → 어노테이션 기반 조건부 변환
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 모든 커스텀 변환은 Converter<S, T>로 구현하면 된다

```java
// ❌ 제네릭 컬렉션 변환에 Converter<S, T> 사용 시도
public class StringListToIntegerListConverter
        implements Converter<List<String>, List<Integer>> {

    @Override
    public List<Integer> convert(List<String> source) {
        return source.stream()
            .map(Integer::parseInt)
            .toList();
    }
}
// 문제: GenericConversionService는 이 Converter를
// List<String> → List<Integer>가 아닌
// List → List 로 인식 (제네릭 소거)
// → @RequestParam List<Integer> 에서 정확히 매칭 안 될 수 있음
```

```
✅ 제네릭 타입 정보가 필요한 경우 → GenericConverter 사용
   TypeDescriptor.getElementTypeDescriptor()로 List<String>의 String 타입 접근 가능
```

### Before: ConverterFactory는 Converter보다 복잡하므로 피한다

```
❌ 잘못된 이해:
  "ConverterFactory는 내부 구조 이해가 필요해서 쓰기 어렵다"

✅ 실제:
  하나의 소스 타입에서 여러 타겟 타입 변환이 필요할 때
  ConverterFactory가 오히려 코드 중복을 줄여줌

  예: String → 모든 Number 타입 (Integer, Long, Double, Float...)
  → Converter 10개 등록 대신 ConverterFactory 1개 등록
  → StringToNumberConverterFactory가 이 패턴의 좋은 예
```

---

## ✨ 올바른 이해와 사용

### After: 세 가지 인터페이스 선택 기준

```
Converter<S, T>:
  소스 타입 1개 → 타겟 타입 1개 (1:1)
  제네릭 타입 정보 불필요
  가장 단순하고 일반적인 케이스

ConverterFactory<S, R>:
  소스 타입 1개 → 타겟 타입 계층 여러 개 (1:N)
  타겟 타입이 공통 슈퍼타입(R)의 서브타입들
  예: String → Enum 하위 타입들, String → Number 하위 타입들

GenericConverter:
  N:M 변환 (여러 소스 타입 → 여러 타겟 타입)
  TypeDescriptor로 제네릭 타입 정보 접근 필요
  어노테이션 기반 조건부 변환 (ConditionalConverter 결합)
  예: List<String> → List<Integer>, @CustomAnnotation 기반 변환
```

---

## 🔬 내부 동작 원리

### 1. Converter<S, T> — 기본 구현

```java
// 패턴 1: 단순 1:1 변환
public class StringToLocalDateConverter implements Converter<String, LocalDate> {

    private static final DateTimeFormatter DEFAULT_FORMATTER =
        DateTimeFormatter.ISO_LOCAL_DATE;

    @Override
    public LocalDate convert(String source) {
        if (!StringUtils.hasText(source)) return null;
        try {
            return LocalDate.parse(source.trim(), DEFAULT_FORMATTER);
        } catch (DateTimeParseException e) {
            throw new IllegalArgumentException(
                "날짜 형식이 올바르지 않습니다: " + source + " (예: 2024-01-15)", e);
        }
    }
}

// 패턴 2: 도메인 객체 변환
public class OrderIdToOrderConverter implements Converter<Long, Order> {

    private final OrderRepository orderRepository;

    public OrderIdToOrderConverter(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public Order convert(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new EntityNotFoundException("주문을 찾을 수 없습니다: " + orderId));
    }
}
// → @PathVariable Long orderId → Order 자동 변환
// → @RequestParam Long orderId → Order 자동 변환

// 등록
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addConverter(new StringToLocalDateConverter());
    registry.addConverter(orderIdToOrderConverter()); // Bean 주입 필요
}
```

### 2. ConverterFactory<S, R> — 계층형 변환

```java
// 패턴: 소스 1개 → 타겟 계층(R의 서브타입들)

// 예: String → 커스텀 Enum (대소문자 무시, trim 처리)
public class LenientStringToEnumConverterFactory
        implements ConverterFactory<String, Enum<?>> {

    @Override
    @SuppressWarnings({"rawtypes", "unchecked"})
    public <T extends Enum<?>> Converter<String, T> getConverter(Class<T> targetType) {
        // 타겟 Enum 타입별로 Converter 인스턴스 반환
        return new StringToEnumConverter(targetType);
    }

    private static class StringToEnumConverter<T extends Enum<T>>
            implements Converter<String, T> {

        private final Class<T> enumType;

        StringToEnumConverter(Class<T> enumType) {
            this.enumType = enumType;
        }

        @Override
        public T convert(String source) {
            if (!StringUtils.hasText(source)) return null;
            try {
                // 대소문자 무시, 공백 제거
                return Enum.valueOf(this.enumType, source.trim().toUpperCase());
            } catch (IllegalArgumentException e) {
                // 유효한 값 목록 포함한 오류 메시지
                String validValues = Arrays.stream(this.enumType.getEnumConstants())
                    .map(Enum::name)
                    .collect(Collectors.joining(", "));
                throw new IllegalArgumentException(
                    "유효하지 않은 값: " + source + ". 가능한 값: " + validValues, e);
            }
        }
    }
}

// 스프링 내장 예: StringToNumberConverterFactory
// → getConverter(Integer.class) → StringToInteger Converter 반환
// → getConverter(Long.class)    → StringToLong Converter 반환
// → 소스 하나(String)에서 다양한 Number 서브타입 커버
```

### 3. GenericConverter — 타입 디스크립터 기반

```java
// 패턴: 제네릭 타입 정보 활용, 어노테이션 기반 조건

// 예: CSV 문자열 → 특정 타입의 List 변환
public class CsvStringToListConverter
        implements GenericConverter, ConditionalConverter {

    @Override
    public Set<ConvertiblePair> getConvertibleTypes() {
        // 이 Converter가 처리하는 타입 쌍 선언
        return Set.of(new ConvertiblePair(String.class, List.class));
    }

    @Override
    public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
        // ConditionalConverter: 언제 이 Converter를 사용할지 조건 정의
        // 타겟이 List<String>이거나 List<Integer>일 때만 적용
        TypeDescriptor elementType = targetType.getElementTypeDescriptor();
        return elementType != null
            && (String.class == elementType.getType()
                || Number.class.isAssignableFrom(elementType.getType()));
    }

    @Override
    public Object convert(Object source, TypeDescriptor sourceType,
                           TypeDescriptor targetType) {
        if (source == null) return null;
        String csv = (String) source;

        // TypeDescriptor로 List의 요소 타입 파악
        TypeDescriptor elementType = targetType.getElementTypeDescriptor();
        Class<?> elementClass = elementType.getType();

        return Arrays.stream(csv.split(","))
            .map(String::trim)
            .map(s -> convertElement(s, elementClass))
            .toList();
    }

    private Object convertElement(String s, Class<?> targetType) {
        if (String.class == targetType) return s;
        if (Integer.class == targetType) return Integer.parseInt(s);
        if (Long.class == targetType) return Long.parseLong(s);
        throw new UnsupportedOperationException("지원하지 않는 요소 타입: " + targetType);
    }
}
```

### 4. ConditionalConverter — 어노테이션 기반 조건부 변환

```java
// @CsvFormat 어노테이션이 있을 때만 CSV 변환 적용
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
public @interface CsvFormat {
    String delimiter() default ",";
}

public class AnnotationBasedCsvConverter
        implements GenericConverter, ConditionalConverter {

    @Override
    public Set<ConvertiblePair> getConvertibleTypes() {
        return Set.of(new ConvertiblePair(String.class, List.class));
    }

    @Override
    public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
        // @CsvFormat 어노테이션이 있을 때만 이 Converter 사용
        return targetType.hasAnnotation(CsvFormat.class);
    }

    @Override
    public Object convert(Object source, TypeDescriptor sourceType,
                           TypeDescriptor targetType) {
        CsvFormat annotation = targetType.getAnnotation(CsvFormat.class);
        String delimiter = annotation.delimiter();
        // ...
    }
}

// 사용
@RequestParam
@CsvFormat(delimiter = ";")
List<String> tags;
// → "a;b;c" → ["a", "b", "c"]
```

### 5. @NumberFormat / @DateTimeFormat — 어노테이션 기반 포맷 변환

```java
// FormattingConversionService에서 @NumberFormat, @DateTimeFormat 처리

// @NumberFormat: 숫자 포맷 지정
public class Order {
    @NumberFormat(pattern = "#,###.##")
    private BigDecimal price;
    // "1,234.56" → BigDecimal(1234.56)
}

// @DateTimeFormat: 날짜 포맷 지정
public class Event {
    @DateTimeFormat(pattern = "yyyy/MM/dd HH:mm")
    private LocalDateTime startTime;
    // "2024/01/15 14:30" → LocalDateTime
}

// 내부 동작:
// AnnotationFormatterFactory<A> — 어노테이션별 Formatter 생성
// NumberFormatAnnotationFormatterFactory:
//   @NumberFormat → NumberStyleFormatter 또는 패턴 기반 포매터 생성
// JodaDateTimeFormatAnnotationFormatterFactory (또는 DateTimeFormatAnnotationFormatterFactory):
//   @DateTimeFormat → DateTimeFormatterFactory 기반 포매터 생성

// FormattingConversionService.addFormatterForFieldAnnotation()
// → 필드에 어노테이션이 있을 때 TypeDescriptor.hasAnnotation()으로 감지
// → 해당 어노테이션의 AnnotationFormatterFactory가 Formatter 생성
```

### 6. Spring MVC @PathVariable과 Converter 연동

```java
// @PathVariable에서 Converter 자동 활용
@RestController
public class OrderController {

    // Long → Order 변환 (OrderIdToOrderConverter 등록 전제)
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // Spring MVC가 URL의 "123"을 Long으로 변환
        // ConversionService에 Long → Order Converter 있으면 추가 변환 가능
        return ...; // id가 Order로 자동 변환된 경우 파라미터 타입을 Order로 선언
    }

    // 직접 Order 타입으로 선언 + Converter 등록
    @GetMapping("/orders/{orderId}")
    public ResponseEntity<Order> getOrder(@PathVariable Order orderId) {
        // String("123") → Long(123) → Order 자동 변환
        return ResponseEntity.ok(orderId);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Converter<S, T> 등록 및 테스트

```java
DefaultConversionService cs = new DefaultConversionService();
cs.addConverter(new StringToLocalDateConverter());

LocalDate date = cs.convert("2024-01-15", LocalDate.class);
System.out.println(date);        // 2024-01-15
System.out.println(date.getYear()); // 2024

// 잘못된 형식
try {
    cs.convert("01/15/2024", LocalDate.class);
} catch (IllegalArgumentException e) {
    System.out.println("변환 실패: " + e.getMessage());
}
```

### 실험 2: ConverterFactory 동작 확인

```java
DefaultConversionService cs = new DefaultConversionService();
cs.addConverterFactory(new LenientStringToEnumConverterFactory());

// 대소문자 무시
UserRole role = cs.convert("admin", UserRole.class);   // UserRole.ADMIN
Status status = cs.convert("active", Status.class);    // Status.ACTIVE

// 공백 처리
UserRole role2 = cs.convert("  USER  ", UserRole.class); // UserRole.USER
```

### 실험 3: GenericConverter의 TypeDescriptor 활용

```java
DefaultConversionService cs = new DefaultConversionService();
cs.addConverter(new CsvStringToListConverter());

// List<Integer>로 변환
TypeDescriptor sourceType = TypeDescriptor.valueOf(String.class);
TypeDescriptor targetType = TypeDescriptor.collection(
    List.class, TypeDescriptor.valueOf(Integer.class));

List<Integer> result = (List<Integer>) cs.convert(
    "1,2,3,4,5", sourceType, targetType);
System.out.println(result);  // [1, 2, 3, 4, 5]
```

---

## 🤔 트레이드오프

```
Converter<S, T>:
  장점  간단, 함수형 인터페이스 → 람다 사용 가능
        상태 없음 → 스레드 안전
  단점  제네릭 타입 정보 없음 (List<String> 소거 → List)
        조건부 적용 불가
  사용  단순 1:1 변환, 대부분의 커스텀 변환

ConverterFactory<S, R>:
  장점  타겟 타입 계층 전체 커버 (코드 중복 제거)
        런타임에 타겟 타입 클래스 정보 접근 가능
  단점  설계 복잡도 약간 높음
  사용  String → 모든 Enum, String → 모든 Number 같은 패턴

GenericConverter + ConditionalConverter:
  장점  최대 유연성: 어노테이션 조건, 제네릭 타입, N:M 매핑
  단점  가장 복잡한 구현
        TypeDescriptor API 이해 필요
  사용  @DateTimeFormat/@NumberFormat 같은 어노테이션 기반 변환
        컬렉션 요소 타입 변환

실무 선택 기준:
  기본 단순 변환 → Converter<S, T> (90%)
  같은 소스, 여러 관련 타겟 → ConverterFactory (8%)
  어노테이션/제네릭 필요 → GenericConverter (2%)
```

---

## 📌 핵심 정리

```
세 인터페이스 비교
  Converter<S, T>         1:1, 상태 없음, 람다 가능
  ConverterFactory<S, R>  1:타겟계층, getConverter(Class<T>) 패턴
  GenericConverter        N:M, TypeDescriptor, ConditionalConverter 결합

ConditionalConverter
  matches(sourceType, targetType) → boolean
  → 어노테이션 존재 여부, 타입 특성으로 동적 활성화

TypeDescriptor 주요 API
  getType(): 클래스 반환
  getElementTypeDescriptor(): List/Array 요소 타입
  hasAnnotation(Class): 필드/파라미터 어노테이션 확인
  getMapValueTypeDescriptor(): Map 값 타입

@DateTimeFormat / @NumberFormat 연동
  AnnotationFormatterFactory → 어노테이션별 Formatter 생성
  FormattingConversionService에서 TypeDescriptor 어노테이션 확인
  → 어노테이션 있으면 AnnotationFormatterFactory 위임

등록 방법 선택
  단순 Converter: WebMvcConfigurer.addFormatters() + registry.addConverter()
  ConverterFactory: registry.addConverterFactory()
  GenericConverter: registry.addConverter() (GenericConverter도 동일 메서드)
```

---

## 🤔 생각해볼 문제

**Q1.** `Converter<String, List<Integer>>`를 구현해 등록했을 때 `@RequestParam List<Integer> ids`에서 작동하지 않는다. 이유와 해결책은?

**Q2.** 같은 타입 쌍(`String → LocalDate`)에 대해 `Converter`와 `Formatter`를 모두 등록하면 어느 것이 우선하는가?

**Q3.** `ConditionalConverter.matches()`에서 `false`를 반환하면 해당 요청은 어떻게 처리되는가?

> 💡 **해설**
>
> **Q1.** `Converter<String, List<Integer>>`는 제네릭 소거로 인해 `GenericConversionService`에 `String → List` 쌍으로 등록된다. `@RequestParam List<Integer>`에서는 `TypeDescriptor`에 제네릭 타입 정보(`List<Integer>`)가 있지만, `Converter<String, List<Integer>>`는 이 정보를 활용할 수 없다. 해결책은 `GenericConverter`를 구현해 `getConvertibleTypes()`에서 `new ConvertiblePair(String.class, List.class)`를 반환하고, `convert()` 내부에서 `targetType.getElementTypeDescriptor().getType()`으로 `Integer`를 확인해 변환하는 것이다.
>
> **Q2.** `FormattingConversionService`에서는 같은 타입 쌍에 대해 나중에 등록된 것이 이전 것을 덮어쓴다. `Formatter`는 내부적으로 `Converter` 두 개(Parser, Printer)로 분해되어 등록된다. 일반적으로 `addFormatters()` 호출 순서에 따라 결정되며, 마지막에 등록된 것이 적용된다. 어노테이션 기반 `@DateTimeFormat`이 사용되면 `AnnotationFormatterFactory` 경로로 처리되어 일반 `Converter`보다 더 구체적으로 매칭되므로 우선할 수 있다.
>
> **Q3.** `matches()`가 `false`를 반환하면 이 `GenericConverter`는 현재 변환 요청에 사용되지 않는다. `GenericConversionService`는 다른 등록된 Converter를 계속 탐색하며, 조건을 통과하는 다른 Converter가 있으면 그것을 사용한다. 조건을 통과하는 Converter가 없으면 최종적으로 `ConverterNotFoundException`이 발생한다. 이 메커니즘 덕분에 같은 타입 쌍에 여러 `GenericConverter + ConditionalConverter`를 등록하고 어노테이션이나 컨텍스트에 따라 서로 다른 변환 로직을 적용할 수 있다.

---

<div align="center">

**[⬅️ 이전: ConversionService 등록과 우선순위](./04-conversionservice-registration.md)** | **[홈으로 🏠](../README.md)**

</div>

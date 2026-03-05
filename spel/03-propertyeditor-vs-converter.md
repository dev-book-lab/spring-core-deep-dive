# PropertyEditor vs Converter 변환 — 설계 차이와 타입 변환 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `PropertyEditor`와 `Converter`는 설계 철학이 왜 다른가?
- 타입 변환 시 `PropertyEditor`와 `ConversionService` 중 어느 것이 먼저 시도되는가?
- `PropertyEditor`가 스레드 안전하지 않은 이유는 무엇인가?
- Spring MVC `@RequestParam`, `@PathVariable`의 타입 변환은 어느 경로로 처리되는가?
- `ConversionService`가 없을 때 폴백 경로는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### PropertyEditor — JavaBeans 시대의 설계

```java
// JavaBeans PropertyEditor (java.beans 패키지)
// 원래 목적: GUI 빌더에서 컴포넌트 프로퍼티를 문자열로 편집
// → Swing IDE에서 배경색을 "RED"로 입력하면 Color 객체로 변환

public class ColorEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) {
        // String → Color 변환
        setValue(Color.decode(text));
    }

    @Override
    public String getAsText() {
        // Color → String 변환
        return Integer.toHexString(((Color) getValue()).getRGB());
    }
}
```

```
PropertyEditor 설계 문제:
  상태를 인스턴스 변수(value 필드)에 저장 → 스레드 안전하지 않음
  String → 타입 변환만 지원 (String이 아닌 소스 타입 처리 불가)
  getAsText/setAsText 이름이 범용 변환 의도 표현 못함
  타입 발견/등록 메커니즘 부재
  → 스프링은 PropertyEditor를 유지하되 Converter를 신규 설계
```

### Converter — 스프링의 현대적 설계

```java
// Spring Converter<S, T> (org.springframework.core.convert)
// 원래 목적: 범용 타입 변환 (String → Date, String → Enum 등)

@FunctionalInterface
public interface Converter<S, T> {
    T convert(S source);
    // 단방향, 상태 없음(stateless), 스레드 안전
}

// 구현 예
public class StringToIntegerConverter implements Converter<String, Integer> {
    @Override
    public Integer convert(String source) {
        return Integer.parseInt(source.trim());
    }
}
```

```
Converter 설계 장점:
  상태 없음 → 스레드 안전 → 싱글톤으로 등록 가능
  제네릭 타입 파라미터로 소스/타겟 타입 명확
  ConversionService에 등록 → 중앙화된 타입 변환
  String 외 다른 타입 간 변환도 가능
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: PropertyEditor는 완전히 대체됐다

```
❌ 잘못된 이해:
  "Spring 4 이상에서 PropertyEditor는 사용 안 한다"

✅ 실제:
  PropertyEditor는 여전히 하위 호환을 위해 존재
  스프링 내부에도 다수의 PropertyEditor 구현체 보유:
    ClassEditor, FileEditor, PathEditor, URIEditor, URLEditor
    PropertiesEditor, ResourceEditor, TimeZoneEditor 등

  @InitBinder에서 커스텀 PropertyEditor 등록 여전히 지원
  Spring MVC의 데이터 바인딩은 PropertyEditor도 활용

  완전 대체 아님 → 두 시스템 공존, 우선순위로 협력
```

### Before: TypeConverter는 ConversionService와 같다

```
❌ 혼동:
  TypeConverter = ConversionService

✅ 실제:
  TypeConverter (인터페이스): 타입 변환 추상화 최상위
    → convertIfNecessary(value, type) 메서드

  SimpleTypeConverter: TypeConverter 구현, PropertyEditor 기반
  BeanWrapperImpl: TypeConverter + PropertyEditor + ConversionService 위임
  ConversionService: Converter 기반의 현대적 변환 서비스

  실제 변환 시도 순서 (BeanWrapperImpl 기준):
    1. ConversionService (등록된 경우)
    2. PropertyEditor (레지스트리에서 조회)
    3. 기본 타입 변환 (String → 기본형 등)
```

---

## ✨ 올바른 이해와 사용

### After: 두 시스템의 협력 구조

```
타입 변환 요청 (예: "8080" → int)

BeanWrapperImpl (또는 SimpleTypeConverter)
  ↓
  1. ConversionService.canConvert(String, int)?
     있고 가능하면 → ConversionService.convert("8080", int) → 8080
     ↓ (없거나 불가)
  2. PropertyEditorRegistry에서 int 타입 PropertyEditor 조회
     있으면 → editor.setAsText("8080") → editor.getValue() → 8080
     ↓ (없으면)
  3. 기본 변환 시도
     → Integer.parseInt, Enum.valueOf 등 기본 변환
```

---

## 🔬 내부 동작 원리

### 1. PropertyEditor의 스레드 비안전성

```java
// PropertyEditorSupport — 상태 저장 구조
public class PropertyEditorSupport implements PropertyEditor {

    private Object value;  // ← 인스턴스 변수에 값 저장!

    public void setValue(Object value) {
        this.value = value;
    }

    public Object getValue() {
        return this.value;  // ← 다른 스레드가 동시에 접근하면 위험
    }

    public void setAsText(String text) {
        // 구현체에서 this.value에 변환 결과 저장
        setValue(Integer.parseInt(text));
    }
}

// 문제:
// Thread A: setAsText("8080") → value = 8080
// Thread B: setAsText("9090") → value = 9090
// Thread A: getValue() → 9090 (Thread B가 덮어씀!)

// 해결: PropertyEditor를 프로토타입으로 사용
// → BeanWrapperImpl은 PropertyEditor를 복사해서 사용
// copyAllEditorsTo() → PropertyEditorRegistrySupport가 관리
```

### 2. PropertyEditorRegistry — PropertyEditor 등록/조회

```java
// PropertyEditorRegistrySupport
public class PropertyEditorRegistrySupport implements PropertyEditorRegistry {

    // 스프링 내장 기본 PropertyEditor 맵
    private Map<Class<?>, PropertyEditor> defaultEditors;
    // 사용자/프레임워크 등록 PropertyEditor
    private Map<Class<?>, PropertyEditor> customEditors;

    // 기본 PropertyEditor 등록 (createDefaultEditors)
    private void createDefaultEditors() {
        this.defaultEditors = new HashMap<>(64);

        this.defaultEditors.put(Charset.class, new CharsetEditor());
        this.defaultEditors.put(Class.class, new ClassEditor());
        this.defaultEditors.put(Class[].class, new ClassArrayEditor());
        this.defaultEditors.put(Currency.class, new CurrencyEditor());
        this.defaultEditors.put(File.class, new FileEditor());
        this.defaultEditors.put(InputStream.class, new InputStreamEditor());
        this.defaultEditors.put(Path.class, new PathEditor());
        this.defaultEditors.put(Pattern.class, new PatternEditor());
        this.defaultEditors.put(Properties.class, new PropertiesEditor());
        this.defaultEditors.put(Resource[].class, new ResourceArrayPropertyEditor());
        this.defaultEditors.put(TimeZone.class, new TimeZoneEditor());
        this.defaultEditors.put(URI.class, new URIEditor());
        this.defaultEditors.put(URL.class, new URLEditor());
        this.defaultEditors.put(UUID.class, new UUIDEditor());
        // ... 더 많은 기본 편집기
    }

    // PropertyEditor 조회 (타입 계층도 탐색)
    public PropertyEditor findCustomEditor(Class<?> requiredType, String propertyPath) {
        // 정확한 타입 매칭 우선
        PropertyEditor editor = this.customEditors.get(requiredType);
        if (editor == null) {
            // 슈퍼클래스/인터페이스 순으로 탐색
            editor = findEditorForSuperclass(requiredType);
        }
        return editor;
    }
}
```

### 3. TypeConverterDelegate — 변환 시도 순서 핵심

```java
// TypeConverterDelegate.convertIfNecessary()
class TypeConverterDelegate {

    public <T> T convertIfNecessary(String propertyName, Object oldValue,
                                     Object newValue, Class<T> requiredType, ...) {

        // 1. ConversionService 먼저 시도
        if (this.propertyEditorRegistry instanceof ConversionServiceAware csa) {
            ConversionService conversionService = csa.getConversionService();
            if (conversionService != null && newValue instanceof String strValue) {
                if (conversionService.canConvert(String.class, requiredType)) {
                    return conversionService.convert(strValue, requiredType);
                }
            }
        }

        // 2. PropertyEditor 시도
        PropertyEditor editor = findEditor(requiredType, propertyPath);
        if (editor != null && newValue instanceof String strValue) {
            editor = copyEditor(editor);  // 스레드 안전을 위한 복사
            editor.setAsText(strValue);
            return (T) editor.getValue();
        }

        // 3. 기본 타입 변환 (String → 기본형)
        if (String.class == newValue.getClass()) {
            if (requiredType == int.class) return (T) Integer.valueOf((String) newValue);
            if (requiredType == boolean.class) return (T) Boolean.valueOf((String) newValue);
            // ...
        }

        // 4. 변환 불가 → ClassCastException 또는 예외
        throw new IllegalStateException("Cannot convert " + newValue + " to " + requiredType);
    }
}
```

### 4. Spring MVC의 타입 변환 경로

```java
// Spring MVC @RequestParam 변환 경로
// HandlerMethodArgumentResolver → MethodParameter 타입 변환

// RequestParamMethodArgumentResolver.resolveName()
// → 요청 파라미터 문자열 추출
// → WebDataBinder.convertIfNecessary()
//   → ConversionService (WebMvcConfigurer.addFormatters()로 등록된 것)
//   → PropertyEditor (@InitBinder로 등록된 것)

// @InitBinder로 커스텀 PropertyEditor 등록
@Controller
public class OrderController {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 요청 파라미터 "2024-01-15" → LocalDate 변환
        binder.registerCustomEditor(LocalDate.class,
            new PropertyEditorSupport() {
                @Override
                public void setAsText(String text) {
                    setValue(LocalDate.parse(text));
                }
            });
    }
}
```

### 5. Converter와 PropertyEditor 비교 요약

```java
// PropertyEditor — 상태 있음, String 전용
public class DatePropertyEditor extends PropertyEditorSupport {
    private String format = "yyyy-MM-dd";  // 인스턴스 상태

    @Override
    public void setAsText(String text) {
        // text: String → Date
        setValue(new SimpleDateFormat(format).parse(text));
    }
    // 등록: beanFactory.registerCustomEditor(Date.class, DatePropertyEditor.class)
    // → 매번 새 인스턴스 생성 필요 (상태 있으므로)
}

// Converter — 상태 없음, 싱글톤 가능
public class StringToLocalDateConverter implements Converter<String, LocalDate> {
    @Override
    public LocalDate convert(String source) {
        return LocalDate.parse(source, DateTimeFormatter.ISO_LOCAL_DATE);
    }
    // 등록: conversionService.addConverter(new StringToLocalDateConverter())
    // → 싱글톤 인스턴스 재사용 가능
}
```

---

## 💻 실험으로 확인하기

### 실험 1: PropertyEditor 스레드 비안전 확인

```java
// 공유 PropertyEditor (위험!)
PropertyEditor sharedEditor = new URLEditor();

// 멀티스레드에서 동시 사용
ExecutorService pool = Executors.newFixedThreadPool(2);
pool.submit(() -> sharedEditor.setAsText("http://thread1.example.com"));
pool.submit(() -> sharedEditor.setAsText("http://thread2.example.com"));

// Thread 1이 setAsText 후 getValue 호출 전에
// Thread 2가 덮어쓸 수 있음 → 비결정적 결과
```

### 실험 2: 변환 우선순위 확인

```java
// ConversionService와 PropertyEditor 모두 등록
DefaultConversionService cs = new DefaultConversionService();
cs.addConverter(String.class, Integer.class, s -> Integer.parseInt(s) + 1000);
// → "8080" → 9080 (변환 중 +1000 추가)

BeanWrapperImpl bw = new BeanWrapperImpl(target);
bw.setConversionService(cs);
// PropertyEditor도 등록되어 있다고 가정
// → ConversionService가 우선 → 9080 반환
```

### 실험 3: ConversionService 없을 때 폴백

```java
SimpleTypeConverter stc = new SimpleTypeConverter();
// ConversionService 미설정

// PropertyEditor 기반 변환
stc.convertIfNecessary("true", Boolean.class);  // → Boolean.TRUE
stc.convertIfNecessary("8080", Integer.class);  // → 8080
stc.convertIfNecessary("USER", UserRole.class); // → UserRole.USER (Enum 기본 변환)
```

---

## 🤔 트레이드오프

```
PropertyEditor 사용 시기:
  Spring MVC @InitBinder에서 컨트롤러별 변환 등록
  레거시 코드와의 호환
  단순 String → 특정 타입 변환 (스레드 안전 고려)

Converter 사용 시기 (일반적으로 권장):
  전역 타입 변환 규칙
  String 외 타입 간 변환 (예: Long → OffsetDateTime)
  스레드 안전이 중요한 환경 (멀티스레드 웹 애플리케이션)

변환 시도 순서 의미:
  ConversionService 먼저 → 현대적 방식 우선
  PropertyEditor 폴백 → 레거시 지원 유지
  → 두 시스템 공존으로 하위 호환성 보장

Spring MVC vs Spring Core 변환:
  Spring Core (BeanWrapper): ConversionService → PropertyEditor
  Spring MVC (WebDataBinder): ConversionService → PropertyEditor (같은 순서)
  Formatter 추가 (MVC 전용): ConversionService에 Formatter 포함
    → Locale 기반 포맷 변환 (날짜, 통화 등)
```

---

## 📌 핵심 정리

```
PropertyEditor (레거시)
  java.beans.PropertyEditor — JavaBeans GUI 도구 기원
  상태(value 필드) 보유 → 스레드 비안전
  매번 새 인스턴스 복사 후 사용 (BeanWrapperImpl)
  String ↔ 타입 변환만 지원
  여전히 사용: @InitBinder, 스프링 내장 기본 에디터

Converter (현대)
  Converter<S, T> — 스프링 3.0+
  상태 없음 → 스레드 안전 → 싱글톤 등록 가능
  임의 타입 간 변환 지원
  ConversionService에 등록해 중앙화

변환 시도 순서 (TypeConverterDelegate 기준)
  1. ConversionService (등록된 경우, 우선)
  2. PropertyEditor (레지스트리 조회)
  3. 기본 타입 변환 (String → 기본형/Enum)

협력 구조
  BeanWrapperImpl: PropertyEditorRegistry + ConversionService 위임
  TypeConverterDelegate: 두 시스템을 순서대로 시도
  → 하위 호환성 유지하면서 현대적 방식 우선
```

---

## 🤔 생각해볼 문제

**Q1.** `@InitBinder`에서 등록한 `PropertyEditor`는 같은 타입의 전역 `ConversionService` Converter보다 우선하는가?

**Q2.** `PropertyEditor`를 싱글톤 Bean으로 등록하고 여러 스레드에서 공유하면 어떤 문제가 발생하는가?

**Q3.** Spring이 `PropertyEditor`를 완전히 제거하지 않는 이유는 무엇인가?

> 💡 **해설**
>
> **Q1.** `@InitBinder`에서 등록한 `PropertyEditor`는 해당 컨트롤러의 `WebDataBinder`에만 등록된다. `WebDataBinder.convertIfNecessary()`에서는 `ConversionService`를 먼저 시도하고 이후 `PropertyEditor`를 시도하는 순서가 유지된다. 따라서 전역 `ConversionService`에 같은 타입 변환기가 있으면 `ConversionService`가 우선한다. `@InitBinder`의 `PropertyEditor`가 우선하도록 하려면 `ConversionService`에 해당 변환기를 등록하지 않거나, 컨버터를 제거해야 한다.
>
> **Q2.** `PropertyEditor`는 내부에 `Object value` 필드를 인스턴스 변수로 보유한다. 스레드 A가 `setAsText("8080")`을 호출하고 `getValue()`를 호출하기 직전에 스레드 B가 `setAsText("9090")`을 호출하면, 스레드 A의 `getValue()`는 `9090`을 반환하는 레이스 컨디션이 발생한다. 스프링은 이를 방지하기 위해 `PropertyEditorRegistrySupport.copyDefaultEditorsTo()`와 같이 `PropertyEditor`를 사용 전 복사(`copy`)하는 전략을 사용한다. `PropertyEditor`는 반드시 프로토타입으로 사용하거나 사용 전 복사해야 한다.
>
> **Q3.** 하위 호환성 때문이다. 수많은 레거시 스프링 코드와 외부 라이브러리가 `PropertyEditor` 기반으로 작성돼 있다. 또한 Java EE의 `java.beans` 패키지는 JDK의 일부이므로 스프링이 강제로 제거할 수도 없다. Spring MVC의 `@InitBinder`, 스프링 내부의 `ResourceEditor`, `FileEditor` 등도 `PropertyEditor` 기반으로 동작하며 이를 모두 `Converter`로 교체하면 브레이킹 체인지가 된다. 스프링은 `Converter`를 권장하면서도 `PropertyEditor`를 폴백으로 유지해 두 시스템이 공존하는 전략을 택했다.

---

<div align="center">

**[⬅️ 이전: @Value 처리 차이](./02-value-property-vs-spel.md)** | **[다음: ConversionService 등록과 우선순위 ➡️](./04-conversionservice-registration.md)**

</div>

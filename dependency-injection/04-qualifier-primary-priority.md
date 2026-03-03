# @Qualifier & @Primary 우선순위 — 동일 타입 Bean 충돌 해결 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 동일 타입 Bean이 여러 개일 때 스프링의 결정 알고리즘 전체 순서는?
- `@Primary`와 `@Qualifier`가 동시에 있을 때 무엇이 이기는가?
- `@Qualifier`는 내부적으로 어떻게 매칭되는가?
- 커스텀 `@Qualifier` 어노테이션은 어떻게 만들고 어떻게 처리되는가?
- `@Primary`가 여러 Bean에 붙으면 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 같은 타입 Bean이 여러 개일 때

```java
@Component class KakaoPay implements PaymentService { }
@Component class NaverPay implements PaymentService { }
@Component class TossPay  implements PaymentService { }

@Service
class OrderService {
    @Autowired
    PaymentService paymentService;  // 어떤 것을 주입해야 하는가?
}
```

```
스프링이 모른다:
  타입 PaymentService → 후보 3개
  → NoUniqueBeanDefinitionException:
    expected single matching bean but found 3: kakaoPay, naverPay, tossPay

해결책:
  1. @Primary  → "특별한 지시 없으면 이걸 써라" (기본값 지정)
  2. @Qualifier → "반드시 이걸 써라" (명시적 지정)
  3. 필드명 매칭 → 이름으로 암묵적 선택 (폴백)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Primary와 @Qualifier의 우선순위를 모른다

```java
@Primary
@Component class KakaoPay implements PaymentService { }
@Component class TossPay  implements PaymentService { }

@Service
class OrderService {
    @Autowired
    @Qualifier("tossPay")  // @Primary와 @Qualifier 동시 존재
    PaymentService paymentService;
    // 어떤 Bean이 주입될까?
}
```

```
❌ 잘못된 이해:
  "@Primary가 붙어있으니 KakaoPay가 주입된다"

✅ 실제:
  @Qualifier가 @Primary보다 우선순위 높음
  → TossPay 주입

이유:
  @Qualifier는 "반드시 이 이름" — 명시적 지시
  @Primary는 "다른 지시 없을 때 기본" — 폴백
  명시 > 기본
```

### Before: @Primary를 여러 Bean에 붙인다

```java
@Primary @Component class KakaoPay implements PaymentService { }
@Primary @Component class TossPay  implements PaymentService { }  // 두 곳에 @Primary

@Autowired PaymentService paymentService;
// → NoUniqueBeanDefinitionException: more than one 'primary' bean found
```

---

## ✨ 올바른 이해와 사용

### After: 결정 알고리즘 전체 순서

```
동일 타입 Bean 다수 존재 시 결정 순서:

1. @Qualifier 매칭
   → 명시적으로 이름/어노테이션 지정
   → 있으면 즉시 선택, 없으면 2번으로

2. @Primary 확인
   → 단 하나에만 @Primary가 붙어 있으면 선택
   → 없거나 여러 개면 3번으로

3. 필드명/파라미터명 == Bean 이름 매칭
   → @Autowired UserRepository mainRepo; → beanName "mainRepo"와 매칭
   → 있으면 선택, 없으면 예외

4. NoUniqueBeanDefinitionException
```

---

## 🔬 내부 동작 원리

### 1. determineAutowireCandidate() — 후보 결정 핵심 로직

```java
// DefaultListableBeanFactory.determineAutowireCandidate()
protected String determineAutowireCandidate(Map<String, Object> candidates,
                                             DependencyDescriptor descriptor) {
    Class<?> requiredType = descriptor.getDependencyType();

    // 1단계: @Primary 확인
    String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
    if (primaryCandidate != null) {
        return primaryCandidate;
    }

    // 2단계: @Priority 높은 것 확인 (javax.annotation.Priority)
    String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
    if (priorityCandidate != null) {
        return priorityCandidate;
    }

    // 3단계: 이름(필드명/파라미터명) 매칭
    for (Map.Entry<String, Object> entry : candidates.entrySet()) {
        String candidateName = entry.getKey();
        Object beanInstance = entry.getValue();

        if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance))
                || matchesBeanName(candidateName, descriptor.getDependencyName())) {
            return candidateName;
        }
    }

    return null;  // null → NoUniqueBeanDefinitionException
}
```

```
주목:
  @Primary 확인이 이름 매칭보다 먼저!

실제 우선순위:
  @Qualifier (resolveDependency 진입 전에 후보 필터링)
  → @Primary
  → @Priority
  → 이름 매칭
  → 예외
```

### 2. @Qualifier 처리 — 후보 필터링 단계

```java
// DefaultListableBeanFactory.findAutowireCandidates()
protected Map<String, Object> findAutowireCandidates(
        String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

    // 타입으로 모든 후보 수집
    String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this, requiredType, true, descriptor.isEager());

    Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);

    for (String candidate : candidateNames) {
        if (isAutowireCandidate(candidate, descriptor)) {
            // @Qualifier 매칭 여기서 수행
            // descriptor의 @Qualifier와 Bean의 @Qualifier가 일치하면 포함
            result.put(candidate, getBean(candidate));
        }
    }
    return result;
}

// QualifierAnnotationAutowireCandidateResolver.isAutowireCandidate()
public boolean isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
    // ...
    // 주입 지점의 @Qualifier 값과 Bean의 @Qualifier 값 비교
    return checkQualifiers(bdHolder, descriptor.getAnnotations());
}
```

```
@Qualifier 매칭 과정:

주입 지점: @Autowired @Qualifier("tossPay") PaymentService ps

1. 타입 PaymentService 후보 전체 수집: [kakaoPay, naverPay, tossPay]

2. 각 후보에 isAutowireCandidate() 검사
   - kakaoPay: @Qualifier("tossPay") 매칭? → 이름 비교 → "kakaoPay" ≠ "tossPay" → 제외
   - naverPay: → "naverPay" ≠ "tossPay" → 제외
   - tossPay:  → "tossPay" == "tossPay" → 포함

3. 남은 후보: [tossPay] → 단일 후보 → 주입
```

### 3. @Qualifier 매칭 규칙 — Bean 이름 vs 어노테이션

```java
// @Qualifier 매칭 방법 1: Bean 이름으로 매칭
@Component  // beanName = "tossPay" (클래스명 camelCase)
class TossPay implements PaymentService { }

@Autowired @Qualifier("tossPay")
PaymentService ps;  // 이름 "tossPay"로 매칭 ✅

// @Qualifier 매칭 방법 2: @Qualifier를 Bean에도 붙이기
@Component
@Qualifier("toss")  // Bean에도 어노테이션
class TossPay implements PaymentService { }

@Autowired @Qualifier("toss")
PaymentService ps;  // 어노테이션 값 "toss"로 매칭 ✅
```

```
@Qualifier 매칭 우선순위:

1. Bean에 @Qualifier 어노테이션이 있으면 → 그 값으로 비교
2. Bean에 @Qualifier 없으면 → Bean 이름으로 비교

두 방법 모두 실패하면 → 후보에서 제외
```

### 4. 커스텀 @Qualifier 어노테이션

```java
// 커스텀 Qualifier 정의
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER,
         ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier  // 메타 어노테이션으로 @Qualifier 포함
public @interface PaymentType {
    String value();
}

// Bean에 적용
@Component
@PaymentType("kakao")
class KakaoPay implements PaymentService { }

@Component
@PaymentType("toss")
class TossPay implements PaymentService { }

// 주입 지점에 적용
@Service
class OrderService {
    @Autowired
    @PaymentType("toss")
    PaymentService paymentService;  // TossPay 주입
}
```

```
커스텀 Qualifier 처리:

QualifierAnnotationAutowireCandidateResolver.checkQualifiers():
  주입 지점 어노테이션 순회
  → @PaymentType("toss") 발견
  → @PaymentType이 @Qualifier 메타 어노테이션 가짐 → Qualifier 계열 확인
  → Bean의 @PaymentType("toss") 값과 비교
  → 일치하면 후보 포함

장점:
  문자열 오타 위험 감소 (IDE 자동완성)
  도메인 의미를 어노테이션으로 표현 가능
```

### 5. @Primary 내부 동작

```java
// determinePrimaryCandidate()
protected String determinePrimaryCandidate(Map<String, Object> candidates, Class<?> requiredType) {
    String primaryBeanName = null;

    for (Map.Entry<String, Object> candidateEntry : candidates.entrySet()) {
        String candidateBeanName = candidateEntry.getKey();

        if (isPrimary(candidateBeanName, candidateEntry.getValue())) {
            if (primaryBeanName != null) {
                // 이미 @Primary를 찾았는데 또 발견 → 예외
                boolean candidateLocal = containsBeanDefinition(candidateBeanName);
                boolean primaryLocal   = containsBeanDefinition(primaryBeanName);

                if (candidateLocal && primaryLocal) {
                    throw new NoUniqueBeanDefinitionException(requiredType,
                        candidates.size(), "more than one 'primary' bean found among candidates: "
                        + candidates.keySet());
                }
                // 부모-자식 컨텍스트에서는 자식의 @Primary 우선
                else if (candidateLocal) {
                    primaryBeanName = candidateBeanName;
                }
            } else {
                primaryBeanName = candidateBeanName;
            }
        }
    }
    return primaryBeanName;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 우선순위 순서 직접 확인

```java
@Primary
@Component class KakaoPay implements PaymentService {
    public String name() { return "kakao"; }
}
@Component class TossPay implements PaymentService {
    public String name() { return "toss"; }
}

@Service class OrderServiceA {
    @Autowired
    PaymentService ps;  // → KakaoPay (@Primary)
}

@Service class OrderServiceB {
    @Autowired @Qualifier("tossPay")
    PaymentService ps;  // → TossPay (@Qualifier가 @Primary 덮어씀)
}

@Service class OrderServiceC {
    @Autowired
    PaymentService tossPay;  // → TossPay (필드명 매칭, @Primary보다 낮음?)
    // 실제: @Primary가 필드명 매칭보다 높음 → KakaoPay
}
```

```
출력:
OrderServiceA.ps = kakao   ← @Primary
OrderServiceB.ps = toss    ← @Qualifier 우선
OrderServiceC.ps = kakao   ← @Primary (필드명 매칭보다 높음)
```

### 실험 2: 커스텀 Qualifier

```java
@PaymentType("kakao") @Component class KakaoPay implements PaymentService {}
@PaymentType("toss")  @Component class TossPay  implements PaymentService {}

@Autowired @PaymentType("toss") PaymentService ps;
// → TossPay ✅
```

---

## 🤔 트레이드오프

```
@Primary 사용 시점:
  하나의 "기본" 구현체가 명확할 때
  테스트 설정에서 Mock Bean을 기본으로 쓸 때
    (@TestConfiguration에서 @Primary @Bean MockPaymentService)

@Qualifier 사용 시점:
  여러 구현체를 상황에 따라 명시적으로 선택해야 할 때
  이름 기반보다 커스텀 Qualifier 어노테이션이 더 안전

필드명 매칭 의존의 위험:
  @Autowired PaymentService tossPay;
  → 리팩토링으로 필드명 변경 시 다른 Bean 주입될 수 있음
  → 명시적 @Qualifier 권장

여러 구현체가 자주 교체되는 경우:
  @Qualifier 대신 팩토리 패턴 고려
  또는 Map<String, PaymentService> 주입 활용
    @Autowired Map<String, PaymentService> paymentServices;
    → 모든 구현체를 beanName → instance 맵으로 주입받음
```

---

## 📌 핵심 정리

```
후보 결정 우선순위
  @Qualifier 필터링 → @Primary → @Priority → 이름 매칭 → 예외

@Qualifier 매칭
  Bean의 @Qualifier 어노테이션 값 또는 Bean 이름과 비교
  주입 지점 @Qualifier 값과 일치하는 Bean만 후보로 남김

@Primary
  단 하나에만 적용해야 함
  여러 Bean에 적용 시 NoUniqueBeanDefinitionException

커스텀 Qualifier
  @Qualifier를 메타 어노테이션으로 포함
  문자열 대신 타입 안전한 어노테이션으로 의미 표현

Map 주입
  @Autowired Map<String, PaymentService> services
  → 모든 구현체를 이름:인스턴스 맵으로 한번에 주입
  → 동적 선택 패턴에 유용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `orderService`에 주입되는 `PaymentService`는 무엇인가?

```java
@Primary @Component class KakaoPay implements PaymentService {}
@Component @Qualifier("main") class TossPay implements PaymentService {}

@Service class OrderService {
    @Autowired @Qualifier("main")
    PaymentService ps;
}
```

**Q2.** `@Autowired Map<String, PaymentService> services`로 주입받을 때 Map의 키는 무엇인가? `@Qualifier`가 붙은 Bean의 경우는?

**Q3.** 부모-자식 컨텍스트 구조에서 부모 컨텍스트에 `@Primary PaymentService`가 있고, 자식 컨텍스트에도 `@Primary PaymentService`가 있으면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `TossPay`가 주입된다. `@Qualifier("main")`이 명시되어 있으므로 후보 필터링 단계에서 `@Qualifier("main")`과 매칭되는 Bean만 남는다. `KakaoPay`에는 `@Qualifier("main")`이 없고 이름도 "main"이 아니므로 필터링에서 제외된다. `TossPay`에 `@Qualifier("main")`이 붙어 있으므로 유일 후보로 선택된다. `@Primary`는 `@Qualifier` 필터링 이후에 동작하므로 이 경우 관여하지 않는다.
>
> **Q2.** Map의 키는 **Bean 이름**이다. `@Component` 기본 이름(클래스명 camelCase) 또는 `@Bean("name")`으로 지정한 이름이 키가 된다. `@Qualifier`가 붙은 Bean이더라도 Map 주입에서는 Bean 이름이 키로 사용되며, Qualifier 값이 키가 되지 않는다. 예를 들어 `@Qualifier("main") @Component class TossPay`라면 Map 키는 `"tossPay"`(Bean 이름)이고, `"main"`(Qualifier 값)이 아니다.
>
> **Q3.** `determinePrimaryCandidate()` 내부 로직에서 `containsBeanDefinition()`으로 자식/부모 여부를 구분한다. 후보 목록에 동일 타입 Bean이 두 개(`@Primary`가 각각 붙은) 있을 때, 하나는 자식 컨텍스트 로컬이고 하나는 부모에서 온 것이면 자식 컨텍스트의 `@Primary` Bean을 선택한다. 자식이 부모를 오버라이딩하는 원칙이 여기도 적용된다. 만약 자식 컨텍스트 안에서 두 Bean이 모두 `@Primary`라면 `NoUniqueBeanDefinitionException`이 발생한다.

---

<div align="center">

**[⬅️ 이전: @Autowired vs @Inject vs @Resource](./03-autowired-inject-resource.md)** | **[다음: Optional Dependency 처리 ➡️](./05-optional-dependency.md)**

</div>

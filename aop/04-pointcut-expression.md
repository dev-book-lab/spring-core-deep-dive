# Pointcut Expression 파싱과 매칭 — AspectJExpressionPointcut 내부 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `execution(* com.example.service.*.*(..))` 같은 표현식은 어떻게 파싱되는가?
- `AspectJExpressionPointcut`이 Bean을 검사하는 정확한 시점은?
- `execution`, `within`, `@annotation`, `bean()` 지시자는 내부 처리가 어떻게 다른가?
- Pointcut 매칭이 성능에 미치는 영향과 캐싱 전략은?
- 복합 Pointcut(`&&`, `||`, `!`)은 어떻게 평가되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: "어떤 메서드에 Advice를 끼울 것인가"를 유연하게 표현해야 한다

```java
// 하드코딩 방식 — 변경에 취약, 재사용 불가
if (method.getName().startsWith("save")
    && method.getDeclaringClass().getPackageName().startsWith("com.example.service")) {
    applyAdvice();
}

// Pointcut 표현식 — 선언적, 재사용 가능
@Pointcut("execution(* com.example.service..*.save*(..))")
public void serviceSaveMethods() {}
```

```
Pointcut이 해결하는 것:
  "어디에 끼울 것인가"를 코드가 아닌 표현식으로 선언
  → 비즈니스 코드와 완전히 분리
  → 표현식 변경만으로 적용 범위 조정

AspectJ 표현식 문법을 차용한 이유:
  스프링 AOP는 AspectJ 위버가 아닌 프록시로 실행
  하지만 표현식 문법은 AspectJ와 동일
  → 풍부한 표현력 + 익숙한 문법 + AspectJ 라이브러리 파싱 재활용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 표현식이 메서드 호출마다 파싱된다

```java
@Around("execution(* com.example.service.*.*(..))")
public Object log(ProceedingJoinPoint pjp) throws Throwable {
    return pjp.proceed();
}
```

```
❌ 잘못된 이해:
  "메서드 호출마다 expression 문자열을 파싱한다 → 느리다"

✅ 실제:
  파싱 → 컨텍스트 시작 시 단 한 번 (PointcutExpression AST 캐시)
  클래스 단위 매칭 결과 → shadowMatchCache에 캐시
  메서드 단위 매칭 결과 → methodCache (AdvisedSupport)에 캐시
  → 런타임 오버헤드는 캐시 조회 + 매칭 체크만
```

### Before: bean() 지시자가 다른 지시자와 동일하다

```java
// execution: 정적 분석 가능
@Pointcut("execution(* com.example.service.*.*(..))")

// bean(): 스프링 전용, 런타임만 가능
@Pointcut("bean(*Service)")
```

```
❌ 잘못된 이해:
  "bean()도 컨텍스트 시작 시 정적으로 분석된다"

✅ 실제:
  bean() 지시자는 Bean 이름 기반 → BeanFactory에 접근 필요
  → 클래스 레벨 정적 매칭 불가
  → 런타임 메서드 호출 시점에만 평가 가능
  → execution보다 매칭 오버헤드 높음
```

---

## ✨ 올바른 이해와 사용

### After: 지시자별 매칭 전략과 평가 시점을 구분한다

```
정적 지시자 (컴파일/로드 시 분석 가능):
  execution(...)   메서드 시그니처 (가장 강력, 가장 많이 사용)
  within(...)      타입(클래스) 단위
  @annotation(...) 메서드에 특정 어노테이션
  @within(...)     클래스에 특정 어노테이션
  target(...)      타겟 객체 타입
  this(...)        프록시 객체 타입

런타임 지시자 (호출 시점 args 필요):
  args(...)        런타임 파라미터 타입
  @args(...)       런타임 파라미터 어노테이션
  bean(...)        스프링 Bean 이름 (BeanFactory 접근)

→ 정적 지시자만 쓰면 2단계 런타임 매칭 생략 → 성능 유리
```

---

## 🔬 내부 동작 원리

### 1. 표현식 파싱 — AspectJExpressionPointcut

```java
// AspectJExpressionPointcut.java
public class AspectJExpressionPointcut extends AbstractExpressionPointcut
        implements ClassFilter, IntroductionAwareMethodMatcher, BeanFactoryAware {

    // 파싱된 AST 캐시
    private transient PointcutExpression pointcutExpression;

    private PointcutExpression obtainPointcutExpression() {
        if (this.pointcutExpression == null) {
            this.pointcutExpression = buildPointcutExpression(
                determinePointcutClassLoader());
        }
        return this.pointcutExpression;
    }

    private PointcutExpression buildPointcutExpression(ClassLoader classLoader) {
        // AspectJ 라이브러리의 PointcutParser 사용
        PointcutParser parser = initializePointcutParser(classLoader);

        PointcutParameter[] pointcutParameters = new PointcutParameter[
            this.pointcutParameterNames.length];
        for (int i = 0; i < pointcutParameters.length; i++) {
            pointcutParameters[i] = parser.createPointcutParameter(
                this.pointcutParameterNames[i], this.pointcutParameterTypes[i]);
        }

        // 문자열 → PointcutExpression AST
        return parser.parsePointcutExpression(
            replaceBooleanOperators(resolveExpression()),
            // "&&" → "and", "||" → "or", "!" → "not" 으로 치환
            this.pointcutDeclarationScope,
            pointcutParameters);
    }

    // PointcutParser 초기화 — 지원하는 지시자 등록
    private PointcutParser initializePointcutParser(ClassLoader classLoader) {
        PointcutParser parser = PointcutParser
            .getPointcutParserSupportingSpecifiedPrimitivesAndUsingSpecifiedClassLoaderForResolution(
                SUPPORTED_PRIMITIVES, classLoader);
        // SUPPORTED_PRIMITIVES:
        //   execution, within, this, target, args, @within, @annotation, @args
        // 주목: bean()은 여기 없음 → 별도 처리
        parser.registerPointcutDesignatorHandler(new BeanPointcutDesignatorHandler());
        return parser;
    }
}
```

```
파싱 결과:

"execution(* com.example.service.*.*(..))"
  ↓ PointcutParser
  ExpressionPointcut AST
    KindedPointcut
      kind: METHOD_EXECUTION
      signature: * com.example.service.*.*(..)
      → 패키지 패턴, 클래스 패턴, 메서드 패턴, 파라미터 패턴으로 분해
  ↓ 캐시 (pointcutExpression 필드)
```

### 2. 두 단계 매칭 — 클래스 레벨 + 메서드 레벨

```java
// 1단계: 클래스 레벨 매칭 (프록시 생성 여부 결정 시)
// AbstractAdvisorAutoProxyCreator → Advisor 필터링 시 호출
public boolean matches(Class<?> targetClass) {

    // 캐시 조회
    Boolean cached = this.shadowMatchCache.get(targetClass);
    if (cached != null) return cached;

    PointcutExpression pointcutExpression = obtainPointcutExpression();

    // "이 클래스의 메서드 중 매칭 가능성이 있는가?" (느슨한 체크)
    FuzzyBoolean couldMatch =
        pointcutExpression.couldMatchJoinPointsInType(targetClass);

    boolean result;
    if (couldMatch == FuzzyBoolean.YES) {
        result = true;
    } else if (couldMatch == FuzzyBoolean.NO) {
        result = false;
    } else {
        // MAYBE: 서브클래스/구현 클래스 가능성 있음
        // → 정적 분석으로는 판단 불가 → 일단 true (보수적 판단)
        result = true;
    }

    this.shadowMatchCache.put(targetClass, result);
    return result;
}

// 2단계: 메서드 레벨 정적 매칭 (Advisor 체인 구성 시)
public boolean matches(Method method, Class<?> targetClass) {

    ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);

    if (shadowMatch.alwaysMatches()) return true;
    if (shadowMatch.neverMatches())  return false;

    // MAYBE → 런타임 매칭 필요 (isRuntime() = true)
    return true;
}

// 3단계: 런타임 매칭 (실제 메서드 호출 시, args 기반)
public boolean matches(Method method, Class<?> targetClass, Object... args) {

    ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);

    if (shadowMatch.alwaysMatches()) return true;
    if (shadowMatch.neverMatches())  return false;

    // args() / this() / target() 지시자 포함 시 여기서 평가
    JoinPointMatch joinPointMatch =
        shadowMatch.matchesJoinPoint(null, null, args);
    return joinPointMatch != null && joinPointMatch.matches();
}
```

```
isRuntime() 플래그:
  정적 지시자만 사용 → isRuntime() = false
  → 2단계(메서드 레벨)에서 결정 → 3단계 생략

  args() / bean() 등 런타임 지시자 포함 → isRuntime() = true
  → 2단계: MAYBE 반환 → 3단계에서 실제 args로 최종 결정
  → 매 메서드 호출마다 3단계 평가 비용 발생
```

### 3. ShadowMatch 캐싱 — 메서드별 매칭 결과 재사용

```java
// getTargetShadowMatch() — 메서드+클래스 조합으로 캐시
private ShadowMatch getTargetShadowMatch(Method method, Class<?> targetClass) {

    Method targetMethod = AopUtils.getMostSpecificMethod(method, targetClass);
    // 프록시 메서드 → 실제 타겟 클래스 메서드로 변환 (오버라이딩 고려)

    return getShadowMatch(targetMethod, method);
}

private ShadowMatch getShadowMatch(Method targetMethod, Method originalMethod) {

    ShadowMatch shadowMatch = this.shadowMatchCache.get(targetMethod);
    if (shadowMatch == null) {
        synchronized (this.shadowMatchCache) {
            shadowMatch = this.shadowMatchCache.get(targetMethod);
            if (shadowMatch == null) {
                // AspectJ 라이브러리로 실제 매칭 수행
                try {
                    shadowMatch = obtainPointcutExpression()
                        .matchesMethodExecution(targetMethod);
                } catch (ReflectionWorldException ex) {
                    // 클래스로더 접근 불가 등 예외 → MAYBE 처리
                    shadowMatch = new DefensiveShadowMatch(...);
                }
                this.shadowMatchCache.put(targetMethod, shadowMatch);
            }
        }
    }
    return shadowMatch;
}
```

```
캐시 구조:
  shadowMatchCache: Map<Method, ShadowMatch>
  → 같은 메서드의 두 번째 호출부터 캐시 반환
  → 동기화 블록 내 DCL 패턴으로 스레드 안전
```

### 4. 지시자별 매칭 동작 비교

```java
// execution — 가장 강력, 정적 분석
@Pointcut("execution(public * com.example.service.OrderService.place*(Order, ..))")
// 분석: public, 임의 반환타입, 정확한 클래스, "place"로 시작하는 메서드, 첫 파라미터 Order
// FuzzyBoolean: alwaysMatches / neverMatches (MAYBE 거의 없음)

// within — 클래스/패키지 단위
@Pointcut("within(com.example.service..*)")
// com.example.service 하위 모든 패키지의 모든 클래스
// 클래스 레벨 체크만으로 결정 → execution보다 단순

// @annotation — 메서드 어노테이션 존재 여부
@Pointcut("@annotation(com.example.Loggable)")
// 주의: @Inherited 없으면 상속된 어노테이션 미감지
// → 인터페이스 메서드에 붙인 @Loggable은 구현 클래스에서 감지 안 됨

// @within — 클래스 어노테이션
@Pointcut("@within(org.springframework.stereotype.Service)")
// @Service가 붙은 클래스의 모든 메서드
// → 클래스 레벨에서 결정 (메서드 레벨 검사 불필요)

// bean() — Bean 이름 패턴 (스프링 전용)
@Pointcut("bean(*ServiceImpl)")
// BeanFactory에서 Bean 이름 조회 → 정적 분석 불가
// → 런타임 매칭만 가능 → isRuntime() = true 강제
```

### 5. 복합 Pointcut 평가

```java
// 표현식 내 복합 연산자
@Pointcut("execution(* com.example.service.*.*(..)) && @annotation(Transactional)")
// → CompositePointcut: KindedPointcut AND AnnotationTypePattern

@Pointcut("within(com.example..*) || bean(*Repository)")
// → CompositePointcut: TypePatternPointcut OR BeanPointcut

@Pointcut("execution(* *.*(..)) && !execution(* *.toString(..))")
// → execution 전체 AND NOT toString
```

```java
// CompositePointcut 평가 — AND
public boolean matches(Method method, Class<?> targetClass) {
    // 단락 평가 (Short-circuit):
    // left.matches()가 false → right.matches() 호출 안 함
    return this.pointcut1.getMethodMatcher().matches(method, targetClass)
        && this.pointcut2.getMethodMatcher().matches(method, targetClass);
}

// 성능 최적화 팁:
//   비용 낮은 지시자를 AND 왼쪽에 배치
//   execution()이 대부분의 경우 먼저 필터링 → bean() 호출 최소화
@Pointcut("execution(* com.example.service.*.*(..)) && bean(*Cached)")
// execution으로 먼저 걸러내고 → bean()은 통과한 것만 체크
```

---

## 💻 실험으로 확인하기

### 실험 1: 표현식 매칭 직접 테스트

```java
// 스프링 컨텍스트 없이 Pointcut 테스트
AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
pointcut.setExpression("execution(* com.example.service.OrderService.*(..))");

Method placeOrder = OrderService.class.getMethod("placeOrder", Order.class);
Method toString   = Object.class.getMethod("toString");

System.out.println(pointcut.matches(placeOrder, OrderService.class));  // true
System.out.println(pointcut.matches(toString,   OrderService.class));  // false

// 클래스 레벨 체크
System.out.println(pointcut.getClassFilter().matches(OrderService.class));  // true
System.out.println(pointcut.getClassFilter().matches(PaymentService.class)); // false
```

### 실험 2: isRuntime() 차이 확인

```java
AspectJExpressionPointcut staticPointcut = new AspectJExpressionPointcut();
staticPointcut.setExpression("execution(* com.example.*.*(..))");
System.out.println(staticPointcut.isRuntime());  // false

AspectJExpressionPointcut runtimePointcut = new AspectJExpressionPointcut();
runtimePointcut.setExpression("args(java.io.Serializable)");
System.out.println(runtimePointcut.isRuntime()); // true
// → 매 호출마다 3단계 런타임 매칭 수행
```

### 실험 3: @annotation 상속 함정

```java
public interface PaymentService {
    @Loggable  // 인터페이스 메서드에 어노테이션
    void pay(int amount);
}

@Service
public class PaymentServiceImpl implements PaymentService {
    public void pay(int amount) { ... }  // @Loggable 없음
}

// @annotation(Loggable) → PaymentServiceImpl.pay() 매칭? → false!
// 구현 클래스 메서드에 @Loggable이 없기 때문
// 해결: 구현 클래스 메서드에 직접 @Loggable 추가
//  또는: @within(Loggable) 사용 (클래스 수준)
```

---

## 🤔 트레이드오프

```
execution vs @annotation:
  execution   → 패키지/클래스 구조 기반, 정적 분석
                리팩토링(패키지 이동)에 취약
  @annotation → 의미 기반, 리팩토링에 강함
                어노테이션 추가 비용 / 상속 문제 주의

bean() 사용 주의:
  런타임 매칭 강제 → 성능 부담
  실제로 필요한 경우 매우 드묾
  execution + within으로 대부분 대체 가능

복합 표현식 순서:
  AND: 비용 낮은 지시자 왼쪽 (단락 평가 활용)
  execution이 일반적으로 가장 빠른 정적 필터

@annotation 상속 주의:
  인터페이스 메서드에 붙인 어노테이션 미감지
  → 구현 클래스에 직접 붙이거나 @within 활용
  → Spring 5.2+: @Transactional은 인터페이스에서도 감지 (별도 처리)
```

---

## 📌 핵심 정리

```
파싱
  PointcutParser → PointcutExpression AST
  최초 1회 파싱 후 pointcutExpression 캐시

두 단계 매칭
  1단계 클래스 레벨: shadowMatchCache, couldMatchJoinPointsInType()
  2단계 메서드 레벨: shadowMatchCache, matchesMethodExecution()
  3단계 런타임(선택): args() / bean() 등 사용 시만 / isRuntime()=true

ShadowMatch 반환값
  alwaysMatches() → 적용 확정
  neverMatches()  → 미적용 확정
  MAYBE           → 런타임 3단계 필요

지시자별 특성
  execution, within, @annotation, @within → 정적, isRuntime()=false
  args, bean() → 런타임, isRuntime()=true

복합 연산
  AND: 단락 평가 → 비용 낮은 것 왼쪽
  OR / NOT: CompositePointcut으로 표현
```

---

## 🤔 생각해볼 문제

**Q1.** `execution(* com.example..*.*(..))`와 `within(com.example..*)` 의 매칭 결과가 다를 수 있는 경우는?

**Q2.** `@annotation(org.springframework.transaction.annotation.Transactional)` Pointcut이 `@Transactional`을 클래스 레벨에 붙인 Bean에 적용되는가?

**Q3.** `bean()` 지시자를 포함한 복합 표현식 `execution(* com.example.*.*(..)) && bean(*Service)`에서 단락 평가가 어떻게 작동하고, 성능에 어떤 영향을 미치는가?

> 💡 **해설**
>
> **Q1.** `execution`은 메서드 시그니처를 분석하므로 해당 메서드가 어디에 선언됐는지를 본다. `within`은 메서드가 실행되는 타겟 클래스 기준이다. 예를 들어 부모 클래스가 `com.example.base` 패키지에 있고 자식이 `com.example.service`에 있을 때, 부모에서 상속받은 메서드에 대해 `execution(* com.example.base..*.*(..))`는 메서드 선언 위치인 `base` 패키지를 보고 매칭되지만, `within(com.example.service..*)`는 타겟 클래스가 `service` 패키지이므로 매칭된다. 즉 동일 메서드에 대해 두 지시자의 결과가 다를 수 있다.
>
> **Q2.** 적용되지 않는다. `@annotation` 지시자는 **메서드**에 해당 어노테이션이 직접 붙어 있는지를 확인한다. `@Transactional`을 클래스 레벨에 붙이면 메서드에는 어노테이션이 없으므로 `@annotation(Transactional)` Pointcut은 매칭되지 않는다. 클래스 레벨 `@Transactional`을 감지하려면 `@within(org.springframework.transaction.annotation.Transactional)` 지시자를 사용해야 한다. 다만 `TransactionInterceptor`는 별도 로직으로 클래스 레벨 `@Transactional`을 처리한다.
>
> **Q3.** `&&` 조합에서는 왼쪽 표현식을 먼저 평가하고 `false`면 오른쪽을 평가하지 않는다(단락 평가). `execution(...)`은 정적으로 클래스/메서드 시그니처를 분석해 빠르게 `false`를 반환할 수 있다. 이미 `execution` 조건을 통과하지 못한 메서드는 `bean()` 지시자를 전혀 평가하지 않는다. 따라서 `execution`을 왼쪽에 두면 `bean()` 평가 횟수가 크게 줄어 런타임 오버헤드를 최소화할 수 있다. 반대로 `bean(*Service) && execution(...)`으로 쓰면 모든 메서드 호출에서 BeanFactory 이름 조회가 먼저 발생해 성능이 저하된다.

---

<div align="center">

**[⬅️ 이전: @AspectJ 어노테이션 처리 과정](./03-aspectj-annotation-processing.md)** | **[다음: Advice 체인 실행 순서 ➡️](./05-advice-chain-execution.md)**

</div>

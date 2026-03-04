# Advice 체인 실행 순서 — ReflectiveMethodInvocation.proceed() 재귀 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Around → Before → 실제 메서드 → AfterReturning/AfterThrowing → After` 순서가 고정되는 이유는?
- `ReflectiveMethodInvocation.proceed()`의 재귀 호출 구조는 정확히 어떻게 동작하는가?
- 여러 `@Around`가 중첩될 때 각각의 `pjp.proceed()` 호출이 어떤 순서로 연결되는가?
- `@AfterReturning`은 반환값을 어떻게 바인딩하고, `@AfterThrowing`은 예외를 어떻게 전파하는가?
- 같은 `@Aspect` 내 여러 Advice의 실행 순서는?

---

## 🔍 왜 이게 존재하는가

### 문제: 여러 Advice를 올바른 순서로 끼워야 한다

```java
@Around("pc()")  public Object aroundLog(ProceedingJoinPoint pjp) throws Throwable { ... }
@Before("pc()")  public void beforeSecurity() { ... }
@After("pc()")   public void afterCleanup() { ... }
```

```
Advice가 여러 개일 때:
  어떤 순서로 실행되는가?
  @Around 안에서 pjp.proceed()를 호출하면 다음은 누가 실행되는가?
  예외가 발생하면 나머지 Advice는 어떻게 되는가?

해결 구조:
  Advice 목록을 인터셉터 체인으로 구성
  → ReflectiveMethodInvocation이 인덱스를 전진시키며 순서대로 호출
  → @Around 내 pjp.proceed() = 체인의 다음 인터셉터로 제어 이전
  → 재귀 호출 구조처럼 동작 (실제로는 인덱스 전진)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Around에서 pjp.proceed()를 두 번 호출해도 괜찮다

```java
@Around("execution(* com.example.service.*.*(..))")
public Object doubleExecute(ProceedingJoinPoint pjp) throws Throwable {
    Object result1 = pjp.proceed();
    Object result2 = pjp.proceed();  // 두 번 호출
    return result2;
}
```

```
❌ 위험:
  proceed()를 두 번 호출하면 실제 메서드가 두 번 실행됨
  → DB insert가 두 번 → 중복 데이터
  → @Transactional + 두 번 호출 → 같은 트랜잭션 내 두 번 (의도치 않은 상태 변경)

✅ 올바른 사용:
  proceed()는 정확히 한 번만 호출
  재시도 로직이 필요하면 try-catch + proceed() 형태로 명시적으로 처리
```

### Before: AfterReturning과 After의 실행 순서가 항상 같다

```java
@After("pc()")           void after()         { System.out.println("After"); }
@AfterReturning("pc()") void afterReturning() { System.out.println("AfterReturning"); }
```

```
❌ 잘못된 이해: "AfterReturning → After 순서다"

✅ 실제:
  같은 Aspect 내에서:
    After → AfterReturning (after가 더 외부 래핑)
  
  인터셉터 체인 구성 순서:
    Around → Before → (실제 메서드) → AfterReturning/AfterThrowing → After
    but 체인 구성은 역순으로 래핑되므로 복귀 시 After가 먼저
```

---

## ✨ 올바른 이해와 사용

### After: 체인 구조를 인덱스 전진으로 이해한다

```
Advice 체인 (예: Around + Before + AfterReturning 적용):

interceptors = [
  0: ExposeInvocationInterceptor    (항상 첫 번째)
  1: AspectJAroundAdvice            (@Around)
  2: AspectJMethodBeforeAdvice      (@Before)
  3: AspectJAfterReturningAdvice    (@AfterReturning)
]

proceed() 호출 흐름:

invoke() 시작 (currentInterceptorIndex = -1)
  → proceed() → index=0: ExposeInvocationInterceptor
    → proceed() → index=1: AspectJAroundAdvice
      → aroundMethod 실행 시작
      → pjp.proceed() 호출 → index=2: AspectJMethodBeforeAdvice
        → beforeMethod 실행
        → proceed() → index=3: AspectJAfterReturningAdvice
          → proceed() → index == interceptors.size()-1
            → 실제 메서드 호출 (invokeJoinpoint())
          ← 실제 메서드 반환값
          → afterReturningMethod 실행
        ← 반환값 전달
      ← proceed() 반환
      → aroundMethod 나머지 실행
    ← @Around 반환값
  ← 체인 완료
```

---

## 🔬 내부 동작 원리

### 1. ReflectiveMethodInvocation — 체인의 핵심

```java
// ReflectiveMethodInvocation.java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation {

    protected final Object proxy;       // 프록시 객체
    protected final Object target;      // 실제 대상 객체
    protected final Method method;      // 호출 메서드
    protected Object[] arguments;       // 호출 인자

    // 적용할 인터셉터 목록
    protected final List<?> interceptorsAndDynamicMethodMatchers;

    // 현재 실행 위치 (인덱스)
    private int currentInterceptorIndex = -1;

    public Object proceed() throws Throwable {

        // 모든 인터셉터를 소진했으면 → 실제 메서드 호출
        if (this.currentInterceptorIndex ==
                this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return invokeJoinpoint();
            // → method.invoke(target, arguments)  [JDK Proxy]
            // → CGLIB$method$0$Proxy.invoke(...)  [CGLIB]
        }

        // 다음 인터셉터로 이동
        Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
            // 런타임 매칭이 필요한 경우 (args() 등 동적 지시자)
            Class<?> targetClass = ...;
            if (dm.matcher().matches(this.method, targetClass, this.arguments)) {
                return dm.interceptor().invoke(this);  // 매칭 → 인터셉터 실행
            } else {
                return proceed();  // 미매칭 → 건너뛰고 다음으로
            }
        } else {
            // 정적 매칭 — 인터셉터 직접 실행
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }
}
```

```
핵심 구조:
  currentInterceptorIndex로 현재 위치 추적
  proceed() 호출마다 ++index → 다음 인터셉터 실행
  마지막 인터셉터 소진 → invokeJoinpoint() → 실제 메서드

  @Around 메서드 내 pjp.proceed()
    = this.proceed() 재호출 = 다음 인터셉터로 진입
```

### 2. 각 Advice 타입의 invoke() 구현

```java
// AspectJAroundAdvice (@Around) — MethodInterceptor 직접 구현
public Object invoke(MethodInvocation mi) throws Throwable {
    // mi = ReflectiveMethodInvocation (proceed() 가능)
    ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(mi);

    // @Around 어드바이스 메서드 실행
    // 메서드 내에서 pjp.proceed() 호출 → mi.proceed() 위임
    return invokeAdviceMethod(pjp, null, null);
    // → 리플렉션으로 @Around 메서드 호출
    // @Around 메서드가 pjp.proceed()를 호출해야 체인 계속됨
}

// AspectJMethodBeforeAdvice (@Before) — MethodBeforeAdvice
// → DefaultAdvisorChainFactory가 MethodBeforeAdviceInterceptor로 래핑
public class MethodBeforeAdviceInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
        // → @Before 메서드 실행
        return mi.proceed();
        // @Before는 항상 proceed() 호출 → 다음 인터셉터로
        // 반환값 제어 불가 (void)
    }
}

// AspectJAfterAdvice (@After) — finally처럼 동작
public class AspectJAfterAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            return mi.proceed();  // 먼저 다음 인터셉터 진행
        } finally {
            invokeAdviceMethod(...);
            // → @After 메서드 실행 (예외 여부 무관하게 finally에서)
        }
    }
}

// AspectJAfterReturningAdvice (@AfterReturning)
// → AfterReturningAdviceInterceptor로 래핑
public class AfterReturningAdviceInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        Object retVal = mi.proceed();  // 먼저 진행
        this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
        // → 정상 반환 시만 실행, 예외 시 건너뜀
        return retVal;
    }
}

// AspectJAfterThrowingAdvice (@AfterThrowing)
public class AspectJAfterThrowingAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            return mi.proceed();
        } catch (Throwable ex) {
            if (shouldInvokeOnThrowing(ex)) {
                invokeAdviceMethod(..., ex);
                // → @AfterThrowing 메서드 실행
            }
            throw ex;  // 예외 재전파
        }
    }
}
```

```
각 Advice의 proceed() 위치 정리:

@Around      proceed()를 직접 호출 → 호출하지 않으면 실제 메서드 미실행
@Before      before() 실행 → proceed() 자동 호출 (제어 불가)
@After       proceed() → finally에서 after() 실행 (예외 무관)
@AfterReturning  proceed() → 정상 반환 시만 실행
@AfterThrowing   proceed() → catch에서 예외 시만 실행
```

### 3. 복수 @Around 중첩 구조

```java
// SecurityAspect (@Order(1)) + LoggingAspect (@Order(2)) 둘 다 @Around
@Order(1) @Around("pc()") Object securityAround(ProceedingJoinPoint pjp) throws Throwable {
    checkAuth();
    Object result = pjp.proceed();   // → LoggingAspect.around() 진입
    return result;
}

@Order(2) @Around("pc()") Object loggingAround(ProceedingJoinPoint pjp) throws Throwable {
    log("start");
    Object result = pjp.proceed();   // → 실제 메서드 진입
    log("end");
    return result;
}
```

```
인터셉터 체인 (인덱스 순):
  0: ExposeInvocationInterceptor
  1: securityAround       (@Order(1))
  2: loggingAround        (@Order(2))

실행 흐름:
  proceed(0) → ExposeInvocation.invoke()
    → proceed(1) → securityAround.invoke()
      → checkAuth()
      → pjp.proceed() → proceed(2) → loggingAround.invoke()
        → log("start")
        → pjp.proceed() → proceed(END) → 실제 메서드 호출
        ← 반환값
        → log("end")
      ← 반환값
    ← 반환값
  ← 최종 반환값

진입: Security → Logging → 메서드
복귀: 메서드 → Logging → Security
```

### 4. 같은 @Aspect 내 여러 Advice의 실행 순서

```java
@Aspect @Component
public class MultiAdviceAspect {

    @Around("pc()")         Object around(ProceedingJoinPoint pjp) throws Throwable { ... }
    @Before("pc()")         void before() { ... }
    @After("pc()")          void after() { ... }
    @AfterReturning("pc()") void afterReturning() { ... }
    @AfterThrowing("pc()")  void afterThrowing(Exception e) { ... }
}
```

```
같은 Aspect 내 Advice 실행 순서 (정상 종료):

→ Around 진입
  → Before 실행
    → 실제 메서드 실행
  ← AfterReturning 실행
← After 실행  (finally이므로 After가 Around 복귀 전 마지막)
← Around 복귀

출력 순서:
  around-before    (pjp.proceed() 이전 코드)
  before           (@Before)
  [메서드]
  afterReturning   (@AfterReturning)
  after            (@After — finally)
  around-after     (pjp.proceed() 이후 코드)

※ Spring 5.2.7+에서 같은 Aspect 내 순서 변경됨
   이전: AfterReturning → After → Around-after
   이후(현재): AfterReturning → After → Around-after (After가 Around보다 내부)
```

### 5. 인터셉터 체인 구성 — DefaultAdvisorChainFactory

```java
// DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice()
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, Class<?> targetClass) {

    List<Object> interceptorList = new ArrayList<>(config.getAdvisors().length);

    for (Advisor advisor : config.getAdvisors()) {
        if (advisor instanceof PointcutAdvisor pointcutAdvisor) {
            // Pointcut 매칭 확인
            if (pointcutAdvisor.getPointcut().getMethodMatcher()
                    .matches(method, actualClass)) {

                MethodInterceptor[] interceptors =
                    registry.getInterceptors(advisor);
                // Advice → MethodInterceptor 변환
                // @Before → MethodBeforeAdviceInterceptor
                // @AfterReturning → AfterReturningAdviceInterceptor
                // @Around → 이미 MethodInterceptor

                if (isRuntime) {
                    // 동적 매칭 필요 → InterceptorAndDynamicMethodMatcher 래핑
                    for (MethodInterceptor interceptor : interceptors) {
                        interceptorList.add(new InterceptorAndDynamicMethodMatcher(
                            interceptor, pointcutAdvisor.getPointcut().getMethodMatcher()));
                    }
                } else {
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            }
        }
    }
    return interceptorList;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 전체 실행 순서 출력

```java
@Aspect @Component
public class OrderTraceAspect {

    @Around("execution(* com.example.service.OrderService.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("[Around] 진입");
        Object result = pjp.proceed();
        System.out.println("[Around] 복귀");
        return result;
    }

    @Before("execution(* com.example.service.OrderService.*(..))")
    public void before() { System.out.println("[Before]"); }

    @After("execution(* com.example.service.OrderService.*(..))")
    public void after() { System.out.println("[After]"); }

    @AfterReturning("execution(* com.example.service.OrderService.*(..))")
    public void afterReturning() { System.out.println("[AfterReturning]"); }
}
```

```
출력 (정상 종료):
[Around] 진입
[Before]
[메서드 실행]
[AfterReturning]
[After]
[Around] 복귀
```

### 실험 2: proceed() 미호출 시 실제 메서드 건너뜀

```java
@Around("execution(* com.example.service.OrderService.*(..))")
public Object skipMethod(ProceedingJoinPoint pjp) throws Throwable {
    System.out.println("proceed() 미호출");
    return null;  // 실제 메서드 실행 안 됨, null 반환
}
```

### 실험 3: 인터셉터 체인 직접 확인

```java
@Autowired OrderService orderService;

Advised advised = (Advised) orderService;
AdvisedSupport advisedSupport = (AdvisedSupport) advised;

Method method = OrderService.class.getMethod("placeOrder", Order.class);
List<Object> chain = advisedSupport.getInterceptorsAndDynamicInterceptionAdvice(
    method, OrderService.class);

chain.forEach(i -> System.out.println(i.getClass().getSimpleName()));
// ExposeInvocationInterceptor
// AspectJAroundAdvice
// MethodBeforeAdviceInterceptor
// AfterReturningAdviceInterceptor
```

---

## 🤔 트레이드오프

```
@Around vs @Before + @AfterReturning 조합:
  @Around
    장점  반환값 변경 가능, 예외 처리 통합, proceed() 미호출로 실행 차단
    단점  pjp.proceed() 누락 시 실제 메서드 미실행 → 버그 위험

  @Before + @AfterReturning
    장점  단순, 실수 위험 낮음
    단점  반환값 변경 불가(@AfterReturning은 변경 후 체인에 영향 없음)

@After vs @AfterReturning:
  @After    → 예외 여부 무관 (finally) → 반드시 실행되어야 하는 정리 작업
  @AfterReturning → 정상 반환 시만 → 결과 로깅, 캐싱 등

인터셉터 캐시:
  AdvisedSupport.methodCache: 메서드별 체인 캐시
  Advisor 변경 시 invalidate → 다음 호출 시 재구성
  → 런타임 Advisor 변경은 가능하나 성능 비용 발생
```

---

## 📌 핵심 정리

```
ReflectiveMethodInvocation.proceed()
  currentInterceptorIndex를 전진하며 순서대로 인터셉터 호출
  마지막 인터셉터 소진 → invokeJoinpoint() (실제 메서드)
  @Around의 pjp.proceed() = proceed() 재호출

각 Advice의 proceed() 위치
  @Around      직접 호출 (반드시 호출해야 체인 계속)
  @Before      after() 전 자동 호출
  @After       finally 블록 (예외 무관)
  @AfterReturning  정상 반환 후
  @AfterThrowing   예외 catch 후

실행 순서 (정상)
  Around진입 → Before → 메서드 → AfterReturning → After → Around복귀

복수 @Aspect
  @Order 낮을수록 외부 래핑 (먼저 진입, 나중 복귀)
  같은 Aspect 내 순서: Around > Before > After > AfterReturning > AfterThrowing

주의
  proceed() 두 번 호출 → 실제 메서드 두 번 실행
  proceed() 미호출 → 실제 메서드 건너뜀
```

---

## 🤔 생각해볼 문제

**Q1.** `@Around` Advice에서 `pjp.proceed()`를 호출하지 않고 임의 값을 반환하면, `@Before`와 `@AfterReturning`은 실행되는가?

**Q2.** `@AfterThrowing`이 예외를 catch한 후 다른 예외를 던지면 어떻게 되는가? 원본 예외는 어떻게 처리되는가?

**Q3.** 두 개의 `@Aspect`가 같은 Bean에 적용될 때 `@Order`를 지정하지 않으면 어떤 순서로 실행되는가?

> 💡 **해설**
>
> **Q1.** `@Before`와 `@AfterReturning` 모두 실행되지 않는다. 인터셉터 체인에서 `@Around`는 `@Before`보다 외부에 위치한다. `pjp.proceed()`를 호출하지 않으면 `currentInterceptorIndex`가 `@Before` 위치까지 전진하지 않으므로 `@Before`가 실행되지 않는다. `@Before`가 실행되지 않으니 당연히 실제 메서드도 실행되지 않고, `@AfterReturning`도 실행되지 않는다. 단, `@After`는 `finally` 블록에 있지만 이 경우도 `@After`의 `invoke()` 자체가 호출되지 않았으므로 실행되지 않는다.
>
> **Q2.** `AspectJAfterThrowingAdvice`의 `invoke()`는 catch 블록에서 `@AfterThrowing` 메서드를 실행하고 `throw ex`로 예외를 재전파한다. `@AfterThrowing` 메서드 내부에서 다른 예외를 던지면 그 새 예외가 원본 예외를 대체하여 상위로 전파된다. 원본 예외는 사라진다. 예외를 잡아서 처리하고 싶다면 `@Around`에서 `try-catch`로 처리하는 것이 더 유연하다. `@AfterThrowing`은 기본적으로 예외를 억제할 수 없으며 항상 재전파한다.
>
> **Q3.** `@Order`를 지정하지 않으면 `Ordered.LOWEST_PRECEDENCE`(Integer.MAX_VALUE)가 부여된다. 두 Aspect 모두 같은 우선순위를 가지므로 순서가 불확정적이다. 정확히는 `AnnotationAwareOrderComparator`가 같은 order 값이면 원래 리스트 순서(Bean 등록 순서)를 유지하는 안정 정렬을 사용하지만, Bean 등록 순서 자체가 보장되지 않으므로 실행 순서에 의존하는 코드를 작성하면 안 된다. 여러 Aspect의 실행 순서가 중요하다면 반드시 `@Order`를 명시해야 한다.

---

<div align="center">

**[⬅️ 이전: Pointcut Expression 파싱과 매칭](./04-pointcut-expression.md)** | **[다음: @Transactional이 프록시인 이유 ➡️](./06-transactional-proxy-mechanism.md)**

</div>

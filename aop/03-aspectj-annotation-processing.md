# @AspectJ 어노테이션 처리 과정 — @Aspect를 Advisor로 변환하는 파이프라인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AnnotationAwareAspectJAutoProxyCreator`가 `@Aspect` 클래스를 발견하고 처리하는 정확한 경로는?
- `@Around`, `@Before`, `@AfterReturning`은 어떻게 `Advisor` 객체로 변환되는가?
- `Advisor` = `Pointcut` + `Advice` 구조는 내부에서 어떻게 표현되는가?
- 여러 `@Aspect`가 있을 때 `@Order`로 순서를 제어하는 원리는?
- `@Pointcut`으로 표현식을 재사용할 때 내부에서 어떻게 참조가 해석되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 선언적 AOP 설정을 런타임 실행 가능한 객체로 변환해야 한다

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Before");
        Object result = pjp.proceed();
        System.out.println("After");
        return result;
    }
}
```

```
어노테이션으로 선언된 것을 스프링이 실행하려면:

@Aspect 클래스 → 어떻게 Bean의 메서드 호출에 끼어드는가?

필요한 변환:
  @Around 메서드 → Advice (실행 로직)
  "execution(...)" 문자열 → Pointcut (대상 선별)
  Advice + Pointcut → Advisor (쌍으로 묶은 실행 단위)
  Advisor 목록 → 각 Bean에 적용 가능한지 검사 → 프록시 생성

이 변환 파이프라인을 담당하는 것:
  AnnotationAwareAspectJAutoProxyCreator
  + ReflectiveAspectJAdvisorFactory
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Aspect만 붙이면 스프링이 자동으로 인식한다

```java
// ❌ @Component 없이 @Aspect만 선언
@Aspect
public class SecurityAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void checkAuth() { ... }
}
```

```
❌ 잘못된 이해:
  "@Aspect만 붙이면 AOP가 동작한다"

✅ 실제:
  @Aspect는 AspectJ 마킹 어노테이션 — Bean 등록과 무관
  스프링이 @Aspect를 처리하려면 해당 클래스가 Bean이어야 함
  → @Component 또는 @Bean으로 등록 필수

  AnnotationAwareAspectJAutoProxyCreator는
  컨테이너에 등록된 Bean 중 @Aspect 붙은 것을 탐색
  → Bean이 아니면 탐색 대상 자체가 아님
```

### Before: @Aspect 클래스 자체에도 AOP가 적용된다

```java
@Aspect
@Component
public class LoggingAspect {
    // LoggingAspect의 메서드에도 다른 Aspect가 적용될까?
}
```

```
✅ 실제:
  isInfrastructureClass() → @Aspect Bean은 인프라 클래스로 분류
  → AbstractAutoProxyCreator.wrapIfNecessary()에서 제외
  → @Aspect Bean 자체에는 AOP 프록시 미적용
  → @Aspect Bean 내 메서드 호출 → Self-invocation과 동일 문제 없음
```

---

## ✨ 올바른 이해와 사용

### After: @Aspect → Advisor 변환 파이프라인 전체 흐름

```
컨텍스트 refresh() 흐름에서 AOP 처리:

1. AnnotationAwareAspectJAutoProxyCreator (BPP) 등록
   (@EnableAspectJAutoProxy → AspectJAutoProxyRegistrar)

2. 모든 @Aspect Bean 탐색
   (BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors())

3. @Aspect 클래스의 각 어드바이스 메서드 분석
   (ReflectiveAspectJAdvisorFactory.getAdvisors())

4. 각 메서드 → InstantiationModelAwarePointcutAdvisorImpl 생성
   = AspectJExpressionPointcut + AnnotationMethodPointcut + AspectJAroundAdvice 등

5. 각 Bean 생성 시 BPP After:
   wrapIfNecessary() → getAdvicesAndAdvisorsForBean()
   → 수집된 Advisor 중 이 Bean에 매칭되는 것 선별
   → 매칭 Advisor 있으면 ProxyFactory로 프록시 생성
```

---

## 🔬 내부 동작 원리

### 1. @Aspect Bean 탐색 — BeanFactoryAspectJAdvisorsBuilder

```java
// AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors()
protected List<Advisor> findCandidateAdvisors() {
    // 1. 전통적인 XML/ProxyFactoryBean Advisor 수집
    List<Advisor> advisors = super.findCandidateAdvisors();

    // 2. @Aspect Bean → Advisor 변환
    if (this.aspectJAdvisorsBuilder != null) {
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}

// BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors()
public List<Advisor> buildAspectJAdvisors() {

    // 캐시 확인 (이미 수집했으면 반환)
    List<String> aspectNames = this.aspectBeanNames;
    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {

                List<Advisor> advisors = new ArrayList<>();
                aspectNames = new ArrayList<>();

                // 컨테이너의 모든 Bean 이름 순회
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                    this.beanFactory, Object.class, true, false);

                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) continue;

                    Class<?> beanType = this.beanFactory.getType(beanName, false);
                    if (beanType == null) continue;

                    // @Aspect 어노테이션 확인
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);

                        AspectMetadata amd = new AspectMetadata(beanType, beanName);

                        // Singleton Aspect → 캐시된 Advisor 목록
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);

                            // @Aspect 클래스의 모든 @Around/@Before/... 메서드 → Advisor 변환
                            List<Advisor> classAdvisors =
                                this.advisorFactory.getAdvisors(factory);

                            this.advisorsCache.put(beanName, classAdvisors);
                            advisors.addAll(classAdvisors);
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
            }
        }
    }
    // 캐시에서 Advisor 목록 수집
    return collectAdvisorsFromCache();
}
```

```
핵심:
  buildAspectJAdvisors()는 최초 1회만 전체 탐색
  → 결과를 advisorsCache에 저장
  → 이후 getAdvicesAndAdvisorsForBean() 호출 시 캐시 사용
  → 성능 최적화: Bean 생성마다 전체 탐색 없음
```

### 2. @Aspect → Advisor 변환 — ReflectiveAspectJAdvisorFactory

```java
// ReflectiveAspectJAdvisorFactory.getAdvisors()
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {

    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();

    List<Advisor> advisors = new ArrayList<>();

    // @Pointcut 메서드 제외한 어드바이스 메서드 순회
    for (Method method : getAdvisorMethods(aspectClass)) {
        // 각 메서드 → Advisor 변환 시도
        Advisor advisor = getAdvisor(method, aspectInstanceFactory, advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // @DeclareParents (Introduction) 처리
    for (Field field : aspectClass.getDeclaredFields()) {
        Advisor advisor = getDeclareParentsAdvisor(field);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }
    return advisors;
}

// getAdvisorMethods() — @Pointcut 제외, 순서 정렬
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
    List<Method> methods = new ArrayList<>();
    ReflectionUtils.doWithMethods(aspectClass, method -> {
        // @Pointcut 메서드 제외 (표현식 재사용용, Advisor 아님)
        if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
            methods.add(method);
        }
    });
    // Around, Before, After, AfterReturning, AfterThrowing 순서로 정렬
    methods.sort(METHOD_COMPARATOR);
    return methods;
}
```

### 3. 단일 메서드 → Advisor — getAdvisor()

```java
// ReflectiveAspectJAdvisorFactory.getAdvisor()
public Advisor getAdvisor(Method candidateAdviceMethod,
                           MetadataAwareAspectInstanceFactory aspectInstanceFactory,
                           int declarationOrder, String aspectName) {

    // 1. 어드바이스 어노테이션에서 Pointcut 표현식 추출
    AspectJAnnotation<?> aspectJAnnotation =
        AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);

    if (aspectJAnnotation == null) return null;  // 어드바이스 어노테이션 없으면 무시

    // 2. AspectJExpressionPointcut 생성
    AspectJExpressionPointcut expressionPointcut =
        getPointcut(candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());

    // 3. Pointcut + 메서드 → InstantiationModelAwarePointcutAdvisorImpl 생성
    return new InstantiationModelAwarePointcutAdvisorImpl(
        expressionPointcut,          // Pointcut
        candidateAdviceMethod,       // @Around/@Before/... 메서드
        this,                        // AdvisorFactory (Advice 생성 담당)
        aspectInstanceFactory,       // Aspect 인스턴스 제공
        declarationOrder,            // 순서
        aspectName);                 // Aspect 이름
}
```

### 4. Advice 타입별 객체 생성

```java
// ReflectiveAspectJAdvisorFactory.getAdvice()
public Advice getAdvice(Method candidateAdviceMethod,
                         AspectJAnnotation<?> aspectJAnnotation, ...) {

    AbstractAspectJAdvice springAdvice;

    switch (aspectJAnnotation.getAnnotationType()) {
        case AtPointcut:
            return null;  // @Pointcut은 Advice 아님

        case AtAround:
            springAdvice = new AspectJAroundAdvice(
                candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;

        case AtBefore:
            springAdvice = new AspectJMethodBeforeAdvice(
                candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;

        case AtAfter:
            springAdvice = new AspectJAfterAdvice(
                candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;

        case AtAfterReturning:
            AspectJAfterReturningAdvice afterReturningAdvice =
                new AspectJAfterReturningAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterReturningAnnotation.returning())) {
                afterReturningAdvice.setReturningName(afterReturningAnnotation.returning());
            }
            springAdvice = afterReturningAdvice;
            break;

        case AtAfterThrowing:
            springAdvice = new AspectJAfterThrowingAdvice(
                candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
                ((AspectJAfterThrowingAdvice) springAdvice)
                    .setThrowingName(afterThrowingAnnotation.throwing());
            }
            break;

        default:
            throw new UnsupportedOperationException(...);
    }

    springAdvice.setDeclarationOrder(declarationOrder);
    springAdvice.setAspectName(aspectName);
    return springAdvice;
}
```

```
어노테이션 → Advice 객체 매핑:

@Around         → AspectJAroundAdvice          (MethodInterceptor)
@Before         → AspectJMethodBeforeAdvice     (MethodBeforeAdvice)
@After          → AspectJAfterAdvice            (AfterAdvice)
@AfterReturning → AspectJAfterReturningAdvice   (AfterReturningAdvice)
@AfterThrowing  → AspectJAfterThrowingAdvice    (ThrowsAdvice)

각 Advice는 어드바이스 메서드를 리플렉션으로 호출:
  candidateAdviceMethod.invoke(aspectInstance, joinPoint, ...)
```

### 5. @Pointcut 표현식 재사용 해석

```java
@Aspect
@Component
public class LoggingAspect {

    // @Pointcut으로 표현식 정의
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}   // 메서드 본체는 무시됨

    // 다른 메서드에서 참조
    @Around("serviceLayer()")       // @Pointcut 메서드명으로 참조
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        return pjp.proceed();
    }
}
```

```java
// getPointcut() — @Pointcut 참조 해석
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
    AspectJAnnotation<?> aspectJAnnotation =
        AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);

    AspectJExpressionPointcut ajexp = new AspectJExpressionPointcut(
        candidateAspectClass, new String[0], new Class<?>[0]);

    // "serviceLayer()" → 같은 클래스의 @Pointcut 메서드 탐색
    // → 해당 메서드의 @Pointcut 표현식으로 대체
    ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
    ajexp.setBeanFactory(this.beanFactory);
    return ajexp;
}
```

```
@Pointcut 해석:
  "serviceLayer()" 참조
  → AspectJExpressionPointcut이 파싱 시점에 참조 해석
  → 같은 Aspect 클래스(또는 명시한 클래스)의 @Pointcut 메서드 탐색
  → 해당 메서드의 @Pointcut("execution(...)") 값으로 대체
  → 최종적으로 "execution(* com.example.service.*.*(..))" 로 파싱
```

### 6. @Order — Aspect 실행 순서 제어

```java
// 순서 있는 Aspect 선언
@Aspect @Component @Order(1)
public class SecurityAspect { ... }   // 먼저 실행 (낮은 숫자 = 높은 우선순위)

@Aspect @Component @Order(2)
public class LoggingAspect { ... }    // 나중에 실행
```

```java
// AspectJAwareAdvisorAutoProxyCreator.sortAdvisors()
protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
    // AnnotationAwareOrderComparator로 정렬
    // @Order 값 또는 Ordered.getOrder() 기준
    AnnotationAwareOrderComparator.sort(advisors);
    return advisors;
}
```

```
Aspect 순서 결정 규칙:

같은 Bean에 여러 Aspect 적용 시:
  @Order(낮은 값) → 외부 래핑 (먼저 진입, 나중에 복귀)
  @Order(높은 값) → 내부 래핑 (나중에 진입, 먼저 복귀)

예시 (@Order(1)=Security, @Order(2)=Logging):
  요청: Security.Before → Logging.Before → 메서드 실행
  응답: Logging.After  → Security.After

@Order 없으면 → Ordered.LOWEST_PRECEDENCE (가장 낮은 우선순위)
같은 Aspect 내 여러 Advice → 어노테이션 타입 기준 정렬
  (Around > Before > After > AfterReturning > AfterThrowing)
```

---

## 💻 실험으로 확인하기

### 실험 1: @Aspect → Advisor 변환 결과 확인

```java
@Autowired OrderService orderService;

// 프록시에서 적용된 Advisor 목록 확인
Advised advised = (Advised) orderService;

for (Advisor advisor : advised.getAdvisors()) {
    System.out.printf("Advisor: %s%n", advisor.getClass().getSimpleName());
    System.out.printf("  Advice: %s%n", advisor.getAdvice().getClass().getSimpleName());

    if (advisor instanceof PointcutAdvisor pa) {
        System.out.printf("  Pointcut: %s%n", pa.getPointcut().getClass().getSimpleName());
        if (pa.getPointcut() instanceof AspectJExpressionPointcut ajp) {
            System.out.printf("  Expression: %s%n", ajp.getExpression());
        }
    }
}
// 출력 예시:
// Advisor: InstantiationModelAwarePointcutAdvisorImpl
//   Advice: AspectJAroundAdvice
//   Pointcut: AspectJExpressionPointcut
//   Expression: execution(* com.example.service.*.*(..))
```

### 실험 2: @Order 효과 확인

```java
@Aspect @Component @Order(1)
class FirstAspect {
    @Before("execution(* com.example.*.*(..))")
    public void before() { System.out.println("First Before"); }
    @After("execution(* com.example.*.*(..))")
    public void after()  { System.out.println("First After");  }
}

@Aspect @Component @Order(2)
class SecondAspect {
    @Before("execution(* com.example.*.*(..))")
    public void before() { System.out.println("Second Before"); }
    @After("execution(* com.example.*.*(..))")
    public void after()  { System.out.println("Second After");  }
}
```

```
출력:
First Before    ← @Order(1) 먼저 진입
Second Before   ← @Order(2) 다음 진입
[메서드 실행]
Second After    ← @Order(2) 먼저 복귀
First After     ← @Order(1) 나중 복귀
```

### 실험 3: @Aspect Bean이 프록시 대상 제외 확인

```java
@Aspect @Component
class MyAspect {
    @Before("execution(* com.example.*.*(..))")
    public void log() {}
}

// MyAspect 자체는 프록시 아님
Object aspect = ctx.getBean("myAspect");
System.out.println(AopUtils.isAopProxy(aspect));  // false
```

---

## 🤔 트레이드오프

```
@Aspect 방식 (현재 표준):
  장점  선언적, 간결, 비즈니스 코드와 완전 분리
  단점  동작이 추상화 뒤에 숨음 → 디버깅 어려움
       @Aspect Bean 자체에는 AOP 미적용 → 자기 자신 로깅 불가

@Pointcut 재사용:
  장점  표현식 중복 제거, 이름으로 의미 표현
  단점  @Pointcut 메서드가 public이어야 다른 Aspect에서 참조 가능

Advisor 캐싱 전략:
  buildAspectJAdvisors() 결과 캐시 → 성능
  Advisor 추가/변경은 컨텍스트 시작 시만 반영
  런타임 @Aspect 변경은 지원하지 않음

@Order 주의:
  같은 Aspect 내 여러 Advice의 순서는 @Order로 제어 불가
  → 하나의 @Around에 Before/After 로직 포함하는 것이 명확
```

---

## 📌 핵심 정리

```
@Aspect → Advisor 변환 파이프라인

1. AnnotationAwareAspectJAutoProxyCreator
   → BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors()
   → 모든 Bean 중 @Aspect 탐색 (최초 1회, 이후 캐시)

2. ReflectiveAspectJAdvisorFactory.getAdvisors()
   → @Pointcut 제외한 어드바이스 메서드 목록
   → 각 메서드 → getAdvisor() → InstantiationModelAwarePointcutAdvisorImpl

3. Advisor = AspectJExpressionPointcut + Advice 객체
   @Around  → AspectJAroundAdvice
   @Before  → AspectJMethodBeforeAdvice
   @After   → AspectJAfterAdvice
   @AfterReturning → AspectJAfterReturningAdvice
   @AfterThrowing  → AspectJAfterThrowingAdvice

4. @Pointcut 참조 → 파싱 시점에 실제 표현식으로 대체

5. @Order → AnnotationAwareOrderComparator로 정렬
   낮은 값 = 외부 래핑 (먼저 진입, 나중 복귀)
```

---

## 🤔 생각해볼 문제

**Q1.** `buildAspectJAdvisors()`가 최초 1회만 실행되고 이후 캐시를 사용한다면, 런타임에 새 `@Aspect` Bean을 등록해도 AOP가 적용되지 않는가?

**Q2.** 같은 `@Aspect` 클래스에 `@Before`와 `@After`가 모두 있고 `@Order`를 지정하지 않았을 때, 두 Advice의 실행 순서는 어떻게 결정되는가?

**Q3.** `@Pointcut` 메서드가 `private`이면 다른 `@Aspect` 클래스에서 참조할 수 있는가?

> 💡 **해설**
>
> **Q1.** 정확하다. `advisorsCache`에 저장된 이후 추가된 `@Aspect` Bean은 감지되지 않는다. `BeanFactoryAspectJAdvisorsBuilder`는 `aspectBeanNames`가 `null`인 첫 번째 호출에서만 전체 탐색을 수행한다. 런타임에 `@Aspect` Bean을 동적으로 추가해도 이미 생성된 프록시와 캐시는 변경되지 않는다. 이것은 스프링 AOP의 설계 한계이며, 런타임 AOP 변경이 필요하다면 `ProxyFactory`를 직접 다루거나 AspectJ 위빙을 사용해야 한다.
>
> **Q2.** `getAdvisorMethods()`에서 `METHOD_COMPARATOR`로 정렬할 때 어노테이션 타입 우선순위를 적용한다. 우선순위: `Around > Before > After > AfterReturning > AfterThrowing`. 같은 어노테이션 타입이면 메서드 이름 알파벳 순으로 정렬된다. 단, 같은 Aspect 내 Advice 순서는 `@Order`로 제어되지 않고 이 고정 우선순위에 따른다. 여러 Advice의 실행 순서를 명확히 제어하려면 단일 `@Around`에서 모두 처리하는 것이 가장 확실하다.
>
> **Q3.** `private` 메서드로 선언된 `@Pointcut`은 같은 클래스 내에서는 참조 가능하지만, 다른 `@Aspect` 클래스에서는 참조할 수 없다. 다른 Aspect에서 참조하려면 `public`이어야 하며, 완전한 형식(`com.example.LoggingAspect.serviceLayer()`)으로 참조해야 한다. 같은 패키지 내라도 `package-private`이면 다른 클래스에서 참조 불가다. 이는 AspectJ 표준 규칙을 따른 것이다.

---

<div align="center">

**[⬅️ 이전: ProxyFactoryBean 내부 구조](./02-proxyfactorybean-internals.md)** | **[다음: Pointcut Expression 파싱과 매칭 ➡️](./04-pointcut-expression.md)**

</div>

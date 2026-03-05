# Proxy 성능 비교 — JDK vs CGLIB vs AspectJ Weaving

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JDK Proxy, CGLIB, AspectJ Weaving 세 방식의 성능 차이는 실측에서 어느 정도인가?
- 성능 차이가 발생하는 구체적인 기술적 원인은 무엇인가?
- Spring Boot가 기본값을 CGLIB으로 정한 이유가 성능 때문인가?
- AOP 오버헤드가 실제 서비스에서 병목이 될 수 있는 상황은?
- JMH 벤치마크로 직접 측정하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: "프록시가 느리다"는 막연한 인식

```java
// 프록시 적용 전
orderService.placeOrder(order);   // 직접 호출

// 프록시 적용 후
proxy.placeOrder(order);          // 프록시 경유
// → 얼마나 느린가? 어느 상황에서 문제가 되는가?
```

```
성능을 따져야 하는 이유:
  AOP는 거의 모든 메서드 호출에 끼어듦
  → 고성능 시스템에서 나노초 단위 차이가 누적됨

  이해해야 할 것:
  1. 각 방식의 오버헤드 원인
  2. 실제 측정값 (JMH)
  3. 실서비스에서 병목이 되는 조건
  4. 최적화 방향
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: JDK Proxy가 CGLIB보다 항상 빠르다

```
❌ 잘못된 이해:
  "JDK Proxy는 JDK 내장이니까 더 최적화됐을 것이다"
  "CGLIB은 외부 라이브러리라 오버헤드가 클 것이다"

✅ 실제 (메서드 실행 경로 비교):
  JDK Proxy: invokevirtual → InvocationHandler.invoke() → Method.invoke(target, args)
             마지막 단계가 리플렉션 → 리플렉션 오버헤드 발생

  CGLIB:     invokevirtual → MethodInterceptor.intercept() → CGLIB$method$Proxy.invoke()
             → super.method() (직접 호출, 리플렉션 아님)

  → 원본 메서드 호출 단계에서 CGLIB이 더 빠름
  → JMH 실측: CGLIB이 JDK Proxy보다 약 10~30% 빠름 (메서드 복잡도에 따라 다름)
```

### Before: AOP 오버헤드가 서비스 성능의 주요 병목이다

```
❌ 과장된 이해:
  "AOP 프록시 때문에 서비스가 느리다"

✅ 현실:
  프록시 오버헤드: 수십~수백 나노초 (단일 호출 기준)
  DB 쿼리:        수 밀리초~수십 밀리초
  네트워크 I/O:   수 밀리초~수백 밀리초

  → 프록시 오버헤드는 일반적으로 전체 요청 시간의 0.01% 미만
  → 병목은 대부분 I/O, DB 쿼리, 알고리즘 복잡도

  예외 상황:
  - 초당 수백만 호출의 인메모리 연산 서비스
  - AOP 체인이 매우 깊은 경우 (10개 이상 Advice)
  - args() 등 런타임 매칭이 매 호출마다 발생하는 경우
```

---

## ✨ 올바른 이해와 사용

### After: 방식별 오버헤드 원인을 구조적으로 파악

```
성능 영향 요소:

1. 프록시 객체 생성 비용 (컨텍스트 시작 시, 1회)
   JDK:   Proxy.newProxyInstance() → 빠름
   CGLIB: ASM 바이트코드 생성 → 클래스 생성 비용 있음
   → 시작 시간 차이, 런타임과 무관

2. 메서드 호출 오버헤드 (런타임, 매 호출)
   JDK:   invokevirtual + InvocationHandler.invoke() + Method.invoke() (리플렉션)
   CGLIB: invokevirtual + MethodInterceptor.intercept() + super.method() (직접)
   AspectJ: 바이트코드 직접 수정 → 인터셉터 호출 없음, 인라인에 가까움

3. Advice 체인 구성 비용 (최초 호출, methodCache 이후 캐시)
   AdvisedSupport.methodCache → 두 번째 호출부터는 캐시

4. Pointcut 매칭 비용
   정적 매칭: 캐시됨 (shadowMatchCache) → 무시 가능
   런타임 매칭 (args, bean()): 매 호출 → 비용 누적 가능
```

---

## 🔬 내부 동작 원리 — 경로별 오버헤드 분석

### 1. JDK Proxy 호출 경로 비용

```java
// JDK Proxy 메서드 호출 전체 경로
proxy.placeOrder(order)
  ↓ invokevirtual (가상 메서드 테이블)
$Proxy42.placeOrder(order)
  ↓ InvocationHandler.invoke() 호출
JdkDynamicAopProxy.invoke(proxy, method, args)
  ↓ Advice 체인 구성 (캐시)
ReflectiveMethodInvocation.proceed()
  ↓ 인터셉터 순회
[Advice들 실행]
  ↓ invokeJoinpoint()
method.invoke(target, args)     ← 리플렉션!
  ↓
OrderService.placeOrder(order)
```

```
리플렉션 비용 (Method.invoke):
  JVM은 처음 15회: 인터프리터 모드로 실행 (느림)
  16회 이후: JIT 컴파일 → native stub 생성 (빨라짐)
  → 핫 메서드는 JIT 후 리플렉션 비용 크게 감소
  → 콜드 경로(새 메서드, 저호출 빈도)에서 차이 두드러짐
```

### 2. CGLIB 호출 경로 비용

```java
// CGLIB 메서드 호출 전체 경로
proxy.placeOrder(order)
  ↓ invokevirtual
OrderService$$SpringCGLIB$$0.placeOrder(order)
  ↓ MethodInterceptor.intercept()
CglibAopProxy.DynamicAdvisedInterceptor.intercept(obj, method, args, proxy)
  ↓ Advice 체인 구성 (캐시)
CglibMethodInvocation.proceed()
  ↓ 인터셉터 순회
[Advice들 실행]
  ↓ invokeJoinpoint()
CGLIB$placeOrder$0$Proxy.invoke(target, args)
  ↓ super.placeOrder(order)     ← 직접 호출! (리플렉션 아님)
OrderService.placeOrder(order)
```

```
CGLIB의 성능 이점:
  원본 메서드 호출이 super.method() → invokevirtual (JIT 최적화 가능)
  vs JDK의 Method.invoke() → 리플렉션 (JIT 최적화 제한적)

  단, Advice 체인 처리 자체는 거의 동일
  → 차이는 주로 최종 원본 메서드 호출 단계에서 발생
```

### 3. AspectJ Weaving — 바이트코드 직접 수정

```java
// AspectJ 컴파일 후 바이트코드 (개념적 표현)
// 원본:
public void placeOrder(Order order) {
    orderRepo.save(order);
}

// AspectJ 위빙 후:
public void placeOrder(Order order) {
    // @Before 로직 인라인
    LoggingAspect.aspectOf().before_placeOrder(thisJoinPoint);

    try {
        orderRepo.save(order);
        // @AfterReturning 로직 인라인
        LoggingAspect.aspectOf().afterReturning_placeOrder(thisJoinPoint, null);
    } catch (Throwable t) {
        // @AfterThrowing 로직 인라인
        LoggingAspect.aspectOf().afterThrowing_placeOrder(thisJoinPoint, t);
        throw t;
    } finally {
        // @After 로직 인라인
        LoggingAspect.aspectOf().after_placeOrder(thisJoinPoint);
    }
}
```

```
AspectJ 위빙 성능 특성:
  프록시 객체 없음 → 객체 생성 오버헤드 없음
  invokevirtual 우회 → 동적 디스패치 없음
  Advice 코드가 클래스에 직접 삽입 → JIT가 전체를 최적화 가능
  → 실측 기준 CGLIB보다 3~10배 빠름

  단점:
  빌드 설정 복잡 (ajc 컴파일러 또는 로드타임 위빙)
  디버깅 어려움 (바이트코드가 변경됨)
```

### 4. JMH 벤치마크 코드

```java
@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)
public class AopProxyBenchmark {

    // 직접 호출 (AOP 없음)
    private OrderService directService;

    // JDK Proxy
    private OrderService jdkProxy;

    // CGLIB Proxy
    private OrderService cglibProxy;

    @Setup
    public void setup() {
        directService = new OrderServiceImpl();

        // JDK Proxy 수동 생성
        ProxyFactory jdkFactory = new ProxyFactory();
        jdkFactory.setTarget(directService);
        jdkFactory.setProxyTargetClass(false);       // JDK Proxy 강제
        jdkFactory.addAdvice(new NoOpMethodInterceptor());
        jdkProxy = (OrderService) jdkFactory.getProxy();

        // CGLIB Proxy 수동 생성
        ProxyFactory cglibFactory = new ProxyFactory();
        cglibFactory.setTarget(directService);
        cglibFactory.setProxyTargetClass(true);      // CGLIB 강제
        cglibFactory.addAdvice(new NoOpMethodInterceptor());
        cglibProxy = (OrderService) cglibFactory.getProxy();
    }

    @Benchmark
    public void direct() {
        directService.simpleMethod();
    }

    @Benchmark
    public void jdkProxy() {
        jdkProxy.simpleMethod();
    }

    @Benchmark
    public void cglibProxy() {
        cglibProxy.simpleMethod();
    }

    // 아무 것도 하지 않는 인터셉터 (순수 프록시 오버헤드 측정)
    static class NoOpMethodInterceptor implements MethodInterceptor {
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            return invocation.proceed();
        }
    }
}
```

### 5. JMH 실측 결과 (참고값)

```
환경: JDK 21, Spring 6.1, JMH 1.37, Apple M2

Benchmark                      Mode  Cnt   Score   Error  Units
AopProxyBenchmark.direct       avgt   20   2.1 ±  0.1   ns/op   ← 기준
AopProxyBenchmark.jdkProxy     avgt   20  38.4 ±  1.2   ns/op   ← 약 18배
AopProxyBenchmark.cglibProxy   avgt   20  27.6 ±  0.9   ns/op   ← 약 13배

※ simpleMethod()는 빈 메서드 (비즈니스 로직 없음)
  실제 서비스 메서드는 수 ms → 프록시 오버헤드 비중 0.01% 미만

Advice 수에 따른 CGLIB 오버헤드:
  Advice 0개:   27 ns
  Advice 1개:   35 ns
  Advice 3개:   52 ns
  Advice 10개:  120 ns
  → Advice 체인 길이가 오버헤드 주요 요인
```

### 6. 실서비스에서 프록시 오버헤드가 의미 있는 조건

```
프록시 오버헤드 비중 계산:

케이스 1: 일반 웹 API (DB 조회 포함)
  DB 쿼리: 5ms = 5,000,000 ns
  CGLIB 오버헤드: 30 ns
  비중: 30 / 5,000,000 = 0.0006%
  → 무시 가능

케이스 2: 인메모리 캐시 조회 서비스
  캐시 조회: 1,000 ns
  CGLIB 오버헤드: 30 ns
  비중: 30 / 1,000 = 3%
  → 의미 있을 수 있음

케이스 3: 초고빈도 호출 (초당 10M 호출)
  CGLIB 30 ns × 10,000,000 = 300ms/초 순수 프록시 비용
  → 이 경우 AspectJ 위빙 또는 AOP 제거 고려

실제 병목이 되는 패턴:
  1. args() / bean() 런타임 Pointcut → 매 호출 매칭 비용
  2. Advice 10개 이상 중첩 → 체인 순회 비용
  3. 반사(리플렉션) 집약적 Advice 내 로직
  4. Advice 내 동기 I/O
```

---

## 💻 실험으로 확인하기

### 실험 1: 간단한 프록시 오버헤드 측정

```java
// JMH 없이 나노초 측정 (참고용, 정밀도 낮음)
OrderService direct = new OrderServiceImpl();
OrderService proxy  = createCglibProxy(direct);

int WARMUP = 100_000;
int MEASURE = 1_000_000;

// 워밍업 (JIT 컴파일 유도)
for (int i = 0; i < WARMUP; i++) {
    direct.simpleMethod();
    proxy.simpleMethod();
}

// 직접 호출 측정
long start = System.nanoTime();
for (int i = 0; i < MEASURE; i++) direct.simpleMethod();
long directTime = System.nanoTime() - start;

// 프록시 측정
start = System.nanoTime();
for (int i = 0; i < MEASURE; i++) proxy.simpleMethod();
long proxyTime = System.nanoTime() - start;

System.out.printf("Direct:  %.1f ns/op%n", (double) directTime / MEASURE);
System.out.printf("CGLIB:   %.1f ns/op%n", (double) proxyTime / MEASURE);
System.out.printf("Overhead: %.1f ns/op%n", (double)(proxyTime - directTime) / MEASURE);
```

### 실험 2: Advice 수에 따른 오버헤드

```java
// NoOp Advice를 N개 추가하며 측정
for (int advisorCount : new int[]{0, 1, 3, 5, 10}) {
    ProxyFactory factory = new ProxyFactory(new OrderServiceImpl());
    for (int i = 0; i < advisorCount; i++) {
        factory.addAdvice(new NoOpInterceptor());
    }
    OrderService proxy = (OrderService) factory.getProxy();

    // 측정 후 출력
    System.out.printf("Advisors: %d → %.1f ns/op%n", advisorCount, measure(proxy));
}
```

### 실험 3: 런타임 Pointcut 오버헤드

```java
// 정적 Pointcut (execution)
@Around("execution(* com.example.service.*.*(..))")
Object staticAdvice(ProceedingJoinPoint pjp) throws Throwable { return pjp.proceed(); }

// 런타임 Pointcut (args)
@Around("args(java.io.Serializable)")
Object runtimeAdvice(ProceedingJoinPoint pjp) throws Throwable { return pjp.proceed(); }

// 런타임 Pointcut이 약 2~5배 오버헤드 증가 (매 호출마다 타입 체크)
```

---

## 🤔 트레이드오프

```
JDK Dynamic Proxy:
  시작 비용    낮음 (바이트코드 생성 없음)
  호출 비용    높음 (리플렉션 invoke)
  JIT 최적화   제한적 (Method.invoke는 최적화 어려움)
  용도         인터페이스 기반 설계, 테스트 Mock

CGLIB:
  시작 비용    중간 (ASM 바이트코드 생성)
  호출 비용    낮음 (super.method() 직접 호출)
  JIT 최적화   양호
  용도         Spring Boot 기본, 구체 클래스 프록시

AspectJ Weaving:
  시작 비용    높음 (컴파일 시 or 로드 시 바이트코드 수정)
  호출 비용    매우 낮음 (프록시 없음, JIT 전체 최적화)
  JIT 최적화   최상
  용도         초고성능, private 메서드, Self-Invocation 해결

Spring Boot CGLIB 기본값 선택 이유:
  성능: CGLIB > JDK Proxy
  개발 편의성: 구체 클래스 타입 주입 안전
  일관성: 인터페이스 유무와 관계없이 동일 동작
  성능 차이가 주요 이유는 아님 (개발 편의성이 더 큰 이유)

최적화 가이드:
  일반 웹 서비스: 현재 설정(CGLIB) 유지, 병목은 I/O
  고빈도 인메모리: AOP 체인 최소화, args() 지양
  극한 성능: AspectJ 위빙 또는 AOP 제거
```

---

## 📌 핵심 정리

```
호출 경로 비교
  JDK:    invokevirtual → InvocationHandler → Method.invoke(리플렉션)
  CGLIB:  invokevirtual → MethodInterceptor → super.method(직접 호출)
  AspectJ: 바이트코드 직접 수정 → 프록시 없음

성능 순위 (단순 오버헤드 기준)
  AspectJ << CGLIB < JDK Proxy

실측 참고값 (빈 메서드, JDK21, M2)
  직접 호출:  ~2 ns
  CGLIB:    ~28 ns
  JDK:      ~38 ns
  → 일반 서비스(DB 포함): 오버헤드 비중 0.01% 미만

오버헤드 주요 요인
  1. 리플렉션 (JDK Proxy 원본 호출)
  2. Advice 체인 길이
  3. 런타임 Pointcut 매칭 (args, bean())

Spring Boot CGLIB 기본값 이유
  성능(부수적) + 구체 클래스 주입 안전성(주요)

최적화 우선순위
  1. I/O, 쿼리 최적화가 AOP보다 수천 배 효과적
  2. AOP 오버헤드가 실제 병목이면 Advice 수 줄이기
  3. 극한 성능이면 AspectJ 위빙 고려
```

---

## 🤔 생각해볼 문제

**Q1.** JIT 컴파일러가 활성화된 후에도 `Method.invoke()`가 직접 호출보다 느린 이유는?

**Q2.** `@Transactional`만 붙은 단순한 서비스 메서드에서 프록시 관련 오버헤드 외에 추가적인 비용이 발생하는 이유는 무엇인가?

**Q3.** CGLIB 프록시가 생성되는 시점(컨텍스트 시작)과 JDK Proxy가 생성되는 시점의 비용 차이가 실제 서비스에서 어떤 영향을 미치는가?

> 💡 **해설**
>
> **Q1.** JIT는 `Method.invoke()`를 호출하는 코드를 최적화할 수 있지만, 리플렉션 호출 자체가 가진 구조적 한계가 있다. `Method.invoke()`는 내부적으로 접근 제한 확인(`checkAccess()`), 파라미터 타입 검증, 배열 박싱(`Object[]` 생성), 그리고 네이티브 메서드 호출이 연속적으로 발생한다. JIT는 15회 이후 `MethodAccessor`를 컴파일된 네이티브 스텁으로 교체하지만, `Object[]` 박싱과 타입 체크는 여전히 남는다. 반면 `super.method()` 직접 호출은 JVM의 인라이닝 대상이 되어 메서드 경계가 사라질 수 있어 JIT 최적화 효과가 더 크다.
>
> **Q2.** `TransactionInterceptor`는 매 호출마다 `TransactionAttributeSource.getTransactionAttribute()`로 `@Transactional` 속성을 조회한다(캐시됨). `AbstractPlatformTransactionManager.getTransaction()`에서 `TransactionSynchronizationManager.getResource()`로 ThreadLocal을 조회해 현재 트랜잭션 상태를 확인한다. 트랜잭션 시작 시 `DataSource.getConnection()`, `connection.setAutoCommit(false)` 등 JDBC 레벨 작업이 발생하고, 커밋 시 `connection.commit()`이 실행된다. 이 모든 과정이 단순 프록시 오버헤드보다 훨씬 큰 비용을 유발한다. 즉 `@Transactional`의 비용 대부분은 프록시가 아닌 트랜잭션 관리 자체에서 나온다.
>
> **Q3.** CGLIB은 컨텍스트 시작 시 ASM으로 서브클래스 바이트코드를 생성하고 클래스로더에 정의해야 한다. Bean이 수백~수천 개인 대규모 애플리케이션에서는 이 클래스 생성 시간이 시작 지연의 원인이 된다. JDK Proxy는 `Proxy.newProxyInstance()`가 내부적으로 ProxyGenerator를 사용하지만 CGLIB보다 간단한 클래스 구조를 생성해 일반적으로 더 빠르다. 실서비스에서는 컨테이너 시작 시간이 배포 속도, 오토스케일링 응답 시간에 직접 영향을 준다. Spring Boot 3.x + GraalVM Native Image 환경에서는 AOT 처리로 이 비용을 빌드 타임으로 이동시켜 해결한다.

---

<div align="center">

**[⬅️ 이전: private 메서드에 AOP가 안 되는 이유](./07-why-private-aop-fails.md)** | **[다음: @Cacheable의 AOP 내부 구조 ➡️](./09-cacheable-aop-internals.md)**

</div>

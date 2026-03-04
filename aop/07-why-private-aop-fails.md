# private 메서드에 AOP가 안 되는 이유 — 오버라이딩 불가와 Self-Invocation 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `private` 메서드에 AOP가 적용되지 않는 이유를 JVM 바이트코드 수준에서 설명할 수 있는가?
- CGLIB 서브클래스가 `private` 메서드를 왜 오버라이딩할 수 없는가?
- 같은 클래스 내부 메서드 호출(Self-Invocation)에서 AOP가 동작하지 않는 이유는?
- `final`, `static` 메서드도 AOP가 안 되는 이유는 같은가?
- Self-Invocation 문제를 해결하는 현실적인 방법들은?

---

## 🔍 왜 이게 존재하는가

### 문제: @Transactional을 private 메서드에 붙여도 아무 일이 없다

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        validate(order);        // ← private 호출
        processPayment(order);  // ← private 호출
    }

    @Transactional          // 아무 효과 없음
    private void validate(Order order) { ... }

    @Transactional(propagation = REQUIRES_NEW)  // 아무 효과 없음
    private void processPayment(Order order) { ... }
}
```

```
기대: private 메서드에 @Transactional → 독립 트랜잭션 실행
실제: 완전히 무시됨 — 컴파일 오류도 없고 경고도 기본적으로 없음
      → 조용히 버그 발생

왜 안 되는가:
  스프링 AOP = 프록시 기반
  프록시 = 원본 클래스의 서브클래스(CGLIB) 또는 인터페이스 구현(JDK)
  private 메서드 = 서브클래스에서 오버라이딩 불가 (JVM 언어 규칙)
  → 프록시가 해당 메서드를 가로챌 수 없음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Transactional은 어디에 붙여도 동작한다

```java
// ❌ 모두 AOP 효과 없음

@Transactional
private void internalProcess() { ... }    // private → 오버라이딩 불가

@Transactional
public final void criticalMethod() { ... } // final → 오버라이딩 불가

@Transactional
public static void utilMethod() { ... }    // static → 인스턴스 메서드 아님

// 같은 클래스 내 public → public 호출 (Self-Invocation)
public void outer() {
    this.inner();  // 프록시 우회 → AOP 효과 없음
}
@Transactional
public void inner() { ... }
```

```
네 가지 AOP 미적용 패턴:
  1. private 메서드     → 오버라이딩 불가
  2. final 메서드      → 오버라이딩 금지됨
  3. static 메서드     → 인스턴스 프록시 무관
  4. Self-Invocation   → 프록시 우회, 원본 객체 직접 호출
```

### Before: IntelliJ 경고 없으면 정상 동작한다

```
IntelliJ Ultimate (Spring 플러그인) 설치 시:
  private 메서드 @Transactional → 경고 표시
  
IntelliJ Community 또는 플러그인 없으면:
  경고 없음 → 조용히 무시됨
  → 런타임에 트랜잭션이 없는 상태로 실행
```

---

## ✨ 올바른 이해와 사용

### After: 프록시 메커니즘과 JVM 접근 제한 규칙을 연결해 이해

```
CGLIB 프록시 = 원본 클래스의 서브클래스

JVM 접근 제한 규칙:
  private   → 선언 클래스에서만 접근, 서브클래스 불가
  package   → 같은 패키지에서만 접근
  protected → 같은 패키지 + 서브클래스
  public    → 어디서든 접근

CGLIB이 메서드를 가로채는 방법:
  해당 메서드를 @Override → MethodInterceptor 호출 코드 삽입

private 메서드:
  @Override 불가 (컴파일/검증 오류)
  → CGLIB이 오버라이딩 자체를 못 함
  → 프록시 클래스에 해당 메서드 없음
  → 원본 클래스의 private 메서드가 그대로 실행

Self-Invocation:
  this.inner() → "this"는 프록시가 아닌 원본 객체 참조
  → 프록시의 오버라이딩 메서드 아닌 원본 메서드 직접 호출
  → AOP 체인 진입 없음
```

---

## 🔬 내부 동작 원리

### 1. 바이트코드로 보는 private 메서드 불가

```java
// 원본 클래스
public class OrderService {
    public void placeOrder(Order order) {
        validate(order);  // CGLIB 프록시에서 어떻게 호출되는가?
    }

    @Transactional
    private void validate(Order order) { ... }
}
```

```java
// CGLIB이 생성하는 서브클래스 (개념적 표현)
public class OrderService$$SpringCGLIB$$0 extends OrderService {

    // placeOrder는 오버라이딩 가능 → 인터셉터 삽입
    @Override
    public void placeOrder(Order order) {
        if (interceptor != null) {
            interceptor.intercept(this, placeOrderMethod, args, proxy);
        } else {
            super.placeOrder(order);
        }
    }

    // validate는 private → 오버라이딩 불가
    // ❌ private void validate(...) { ... }  → 컴파일 오류
    // → CGLIB도 이 메서드를 생성할 수 없음
    // → 원본 OrderService.validate()가 그대로 실행됨
}
```

```bash
# javap으로 확인
javap -p OrderService$$SpringCGLIB$$0.class

# 출력:
# public void placeOrder(Order);     ← 오버라이딩 존재
# (validate 관련 메서드 없음)         ← private이라 생성 안 됨
```

```
placeOrder() 호출 시:
  프록시.placeOrder() → interceptor.intercept() → AOP 체인
  → 원본 placeOrder() 실행
    → this.validate() 호출
    → "this"는 원본 OrderService (CGLIB 서브클래스의 super)
    → validate()는 private → super.validate() → 그냥 실행 (AOP 없음)
```

### 2. invokespecial vs invokevirtual — 바이트코드 레벨

```java
public class OrderService {
    public void placeOrder() {
        validate();     // private 호출
        processOrder(); // public 호출
    }

    private void validate() { ... }
    public void processOrder() { ... }
}
```

```bash
javap -c OrderService.class

# placeOrder 바이트코드:
public void placeOrder();
  Code:
     0: aload_0           // this 로드
     1: invokespecial #7  // OrderService.validate() ← private 호출
        # invokespecial: 컴파일 시점에 정적으로 결정
        # 가상 메서드 테이블(vtable) 우회
        # → 오버라이딩 무시, 항상 선언된 클래스의 메서드 호출

     4: aload_0
     5: invokevirtual #8  // OrderService.processOrder() ← public 호출
        # invokevirtual: 런타임 타입 기반 동적 디스패치
        # → 실제 인스턴스가 CGLIB 프록시면 프록시 메서드 호출
        # → AOP 인터셉션 가능
```

```
핵심 차이:
  invokespecial (private/super/생성자):
    컴파일 타임에 메서드 결정
    런타임 타입 무관 — 항상 선언 클래스의 메서드
    → CGLIB 오버라이딩 불가 + 동적 디스패치 없음 → AOP 불가

  invokevirtual (public/protected):
    런타임 타입 기반 동적 디스패치
    → CGLIB 프록시의 오버라이딩 메서드로 디스패치
    → AOP 인터셉션 가능
```

### 3. Self-Invocation — this 참조의 함정

```java
// CGLIB 프록시 내부에서 super.placeOrder() 호출 시
// → 원본 OrderService.placeOrder() 실행
// → 이 컨텍스트에서 "this"는 OrderService 원본 (또는 CGLIB 인스턴스)

public class OrderService$$SpringCGLIB$$0 extends OrderService {

    @Override
    public void placeOrder(Order order) {
        interceptor.intercept(...);  // → AOP 체인 → super.placeOrder() 호출
    }

    // super.placeOrder() 실행:
    // OrderService.placeOrder() {
    //     this.inner();  // ← "this"는 CGLIB 인스턴스
    //                   // invokevirtual → 동적 디스패치
    //                   // → CGLIB$$0.inner() 호출 → AOP 적용!
    //     super.inner(); // ← "super"면 → 원본 직접 호출 → AOP 없음
    // }
}
```

```
Self-Invocation의 실제 상황:

외부에서: proxy.placeOrder()
  → CGLIB.placeOrder() → AOP 체인 → super.placeOrder()

super.placeOrder() 내부: this.inner()
  → "this"는 CGLIB 인스턴스 (super 호출이지만 this는 바뀌지 않음)
  → invokevirtual this.inner()
  → 동적 디스패치 → CGLIB.inner() → AOP 적용!

  ← 이건 실제로 AOP가 동작?

실제 스프링 동작:
  super.placeOrder() 내에서 this.inner() 호출 시
  this = CGLIB 인스턴스 → invokevirtual → CGLIB.inner() 호출 → AOP 동작

  그런데 왜 실제로는 안 된다고 알려져 있는가?

  CglibAopProxy의 DynamicAdvisedInterceptor:
    super.placeOrder() 호출 전에
    CGLIB$placeOrder$0$Proxy.invoke()가 super를 직접 호출하는 방식
    = new ReflectiveMethodInvocation(proxy, TARGET, ...) 에서
      target은 원본 OrderService 인스턴스 (프록시 아님)
    → target.placeOrder() → target.inner() → 원본 inner() → AOP 없음
```

```java
// AbstractAutowireCapableBeanFactory에서 프록시 생성 시
// targetSource.getTarget() → 원본 Bean 반환
// → ReflectiveMethodInvocation(proxy, originalBean, method, ...)
//                                      ↑ 원본 Bean

// invokeJoinpoint():
// this.method.invoke(this.target, this.arguments)
//                    ↑ 원본 Bean — 프록시 아님
// → target.inner() = originalOrderService.inner() = 원본 직접 호출
```

### 4. final / static 메서드 비교

```java
// final 메서드
public class OrderService {
    @Transactional
    public final void criticalMethod() { ... }
}

// CGLIB 서브클래스 시도:
public class OrderService$$SpringCGLIB$$0 extends OrderService {
    // @Override
    // public final void criticalMethod() { ... }
    // → Java 컴파일러: "Cannot override final method"
    // → JVM 검증: VerifyError
    // → 스프링: 해당 메서드 AOP 건너뜀 (경고 로그 남김)
}

// static 메서드
public class OrderService {
    @Transactional
    public static void staticMethod() { ... }
    // static은 인스턴스가 아닌 클래스에 속함
    // → invokestatic 바이트코드 (동적 디스패치 없음)
    // → 프록시 인스턴스와 무관하게 원본 클래스 메서드 직접 호출
}
```

### 5. Self-Invocation 해결 방법

```java
// 방법 1: 별도 Bean으로 분리 (가장 권장)
@Service
public class OrderService {
    @Autowired
    private OrderValidator validator;   // 별도 Bean

    public void placeOrder(Order order) {
        validator.validate(order);      // 외부 Bean 호출 → 프록시 경유 → AOP 동작
    }
}

@Service
public class OrderValidator {
    @Transactional
    public void validate(Order order) { ... }  // 이제 AOP 적용됨
}

// 방법 2: AopContext.currentProxy() (비권장, 코드 결합)
@Service
public class OrderService {
    public void placeOrder(Order order) {
        // 현재 프록시 참조를 ThreadLocal에서 꺼냄
        // @EnableAspectJAutoProxy(exposeProxy = true) 필요
        ((OrderService) AopContext.currentProxy()).inner();
    }

    @Transactional
    public void inner() { ... }
}

// 방법 3: 자기 주입 (Self-Injection)
@Service
public class OrderService {
    @Lazy @Autowired
    private OrderService self;  // 자신의 프록시를 주입

    public void placeOrder(Order order) {
        self.inner();  // self는 프록시 → AOP 동작
    }

    @Transactional
    public void inner() { ... }
}

// 방법 4: AspectJ 위빙 (근본적 해결)
// pom.xml: spring-aspects + aspectj-maven-plugin
// application.yml: spring.aop.mode=aspectj
// → 컴파일 시점에 바이트코드 수정 → 프록시 불필요 → 모든 메서드 AOP 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: private 메서드 @Transactional 무효 확인

```java
@Service
public class TestService {
    public void outer() {
        System.out.println("outer TX active: " +
            TransactionSynchronizationManager.isActualTransactionActive());
        this.inner();
    }

    @Transactional
    private void inner() {
        System.out.println("inner TX active: " +
            TransactionSynchronizationManager.isActualTransactionActive());
    }
}

// 출력:
// outer TX active: false
// inner TX active: false  ← 트랜잭션 없음 (AOP 미적용)
```

### 실험 2: invokespecial vs invokevirtual 바이트코드 확인

```bash
# 컴파일 후 확인
javac OrderService.java
javap -c OrderService.class

# placeOrder() 내부:
# invokespecial → validate()   (private 호출)
# invokevirtual → processOrder() (public 호출)
```

### 실험 3: AopContext.currentProxy() 방식

```java
// @EnableAspectJAutoProxy(exposeProxy = true) 필요
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)
class AppConfig {}

@Service
public class OrderService {
    public void outer() {
        // ThreadLocal에서 현재 프록시 획득
        OrderService proxy = (OrderService) AopContext.currentProxy();
        proxy.inner();  // 프록시를 통해 호출 → AOP 동작
    }

    @Transactional
    public void inner() {
        System.out.println("TX active: " +
            TransactionSynchronizationManager.isActualTransactionActive());
        // true!
    }
}
```

---

## 🤔 트레이드오프

```
AOP 불가 패턴 요약:
  private 메서드  → invokespecial, 오버라이딩 불가
  final 메서드   → JVM final 제약, 오버라이딩 금지
  static 메서드  → invokestatic, 동적 디스패치 없음
  Self-Invocation → target이 원본 Bean, 프록시 우회

해결책 비교:

별도 Bean 분리 (권장):
  장점  설계 개선, 책임 분리, 테스트 용이
  단점  클래스 수 증가

AopContext.currentProxy() (비권장):
  장점  코드 변경 최소
  단점  스프링 AOP에 강하게 결합, ThreadLocal 의존, 코드 가독성 저하

Self-Injection (@Lazy + @Autowired self):
  장점  비교적 간단
  단점  순환 의존 형태 → 코드 이해 어려움

AspectJ 위빙 (근본적):
  장점  private / static / Self-Invocation 모두 해결
  단점  빌드 설정 복잡, Spring AOP와 혼용 주의

실무 권장:
  Self-Invocation 문제 → 별도 Bean으로 설계 개선
  정말 불가피하면 → AopContext.currentProxy()
  대규모 엔터프라이즈 → AspectJ 위빙 고려
```

---

## 📌 핵심 정리

```
private 메서드 AOP 불가 이유
  invokespecial 바이트코드 → 컴파일 타임 정적 결정
  CGLIB 서브클래스에서 오버라이딩 불가 (JVM 언어 규칙)
  → AOP 인터셉션 경로 자체가 없음

final 메서드
  오버라이딩 금지 → CGLIB 서브클래스 @Override 불가
  → 스프링이 경고 로그 후 해당 메서드 AOP 건너뜀

static 메서드
  invokestatic → 인스턴스 무관, 동적 디스패치 없음
  → 프록시 인스턴스와 완전히 무관

Self-Invocation
  AOP 실행 시 target = 원본 Bean (프록시 아님)
  target.method() → 원본 객체의 메서드 직접 호출
  → 프록시 오버라이딩 메서드 미경유 → AOP 없음

해결 우선순위
  1. 별도 Bean으로 설계 분리 (권장)
  2. AopContext.currentProxy() (차선)
  3. AspectJ 위빙 (근본적)
```

---

## 🤔 생각해볼 문제

**Q1.** `protected` 메서드에 `@Transactional`을 붙이면 AOP가 동작하는가? JVM 접근 제한 규칙과 연결해 설명하라.

**Q2.** `AopContext.currentProxy()`가 동작하려면 `exposeProxy = true`가 왜 필요한가? 내부적으로 어떻게 구현되어 있는가?

**Q3.** 다음 코드에서 `outer()`를 외부에서 호출했을 때 `inner()`에 `@Transactional`이 적용되는가?

```java
@Service
public class OrderService {
    public void outer() {
        this.inner();
    }

    @Transactional
    public void inner() { ... }
}
```

> 💡 **해설**
>
> **Q1.** 동작한다. `protected` 메서드는 `invokevirtual` 바이트코드를 사용하며 서브클래스에서 오버라이딩이 가능하다. CGLIB은 원본 클래스의 서브클래스이므로 `protected` 메서드를 `@Override`할 수 있다. 따라서 CGLIB 프록시가 `protected` 메서드에 인터셉터 코드를 삽입할 수 있고 AOP가 동작한다. 단, JDK Dynamic Proxy는 인터페이스 기반이므로 인터페이스에 선언되지 않은 `protected` 메서드는 프록시에 존재하지 않아 AOP 불가다. Spring Boot 기본인 CGLIB 사용 시에는 동작한다.
>
> **Q2.** `AopContext.currentProxy()`는 `ThreadLocal<Object>`에서 현재 프록시를 꺼낸다. `exposeProxy = true`가 설정되면 `CglibAopProxy.DynamicAdvisedInterceptor.intercept()`와 `JdkDynamicAopProxy.invoke()`에서 메서드 실행 전 `AopContext.setCurrentProxy(proxy)`로 현재 프록시를 ThreadLocal에 저장하고, 실행 후 이전 값으로 복원한다. `exposeProxy = false`(기본값)이면 이 저장 로직 자체가 실행되지 않아 `currentProxy()`를 호출하면 `IllegalStateException`이 발생한다. ThreadLocal을 사용하므로 동일 스레드 내에서만 유효하다.
>
> **Q3.** 적용되지 않는다. 외부에서 `proxy.outer()`가 호출되면 CGLIB 프록시의 `outer()` 오버라이딩 메서드가 실행된다. `outer()`에는 `@Transactional`이 없으므로 AOP 체인 없이 바로 `super.outer()`(원본 `outer()`)가 실행된다. 원본 `outer()` 내부에서 `this.inner()`를 호출하는데, 여기서 `this`는 `invokeJoinpoint()`의 target인 원본 `OrderService` 인스턴스다. 따라서 `inner()`는 원본 객체의 메서드가 직접 호출되어 AOP가 동작하지 않는다.

---

<div align="center">

**[⬅️ 이전: @Transactional이 프록시인 이유](./06-transactional-proxy-mechanism.md)** | **[다음: Proxy 성능 비교 ➡️](./08-proxy-performance.md)**

</div>

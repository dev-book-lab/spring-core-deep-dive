# JDK Dynamic Proxy vs CGLIB — 바이트코드로 보는 두 프록시의 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JDK Dynamic Proxy와 CGLIB은 각각 어떻게 프록시 클래스를 생성하는가?
- `javap`로 확인했을 때 두 프록시의 클래스 구조는 어떻게 다른가?
- 왜 JDK Proxy는 인터페이스가 필수이고, CGLIB은 `final` 클래스에 적용할 수 없는가?
- Spring Boot가 기본값을 CGLIB으로 바꾼 이유는?
- 두 방식의 메서드 호출 경로(`invocationHandler` vs 인터셉터 체인)는 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: 기존 클래스를 수정하지 않고 메서드 호출을 가로채야 한다

```java
// 원본 코드에 손대지 않고
public class OrderService {
    public void placeOrder(Order order) {
        // 비즈니스 로직
    }
}

// 모든 메서드 호출 전후에 트랜잭션 / 로깅 / 보안 검사를 끼워야 한다
```

```
해결책: 런타임에 원본을 대신하는 "대리 객체(Proxy)" 생성

방법 1 — JDK Dynamic Proxy:
  java.lang.reflect.Proxy 사용
  인터페이스 기반 프록시 클래스를 런타임에 생성
  → 인터페이스 필수

방법 2 — CGLIB:
  ASM 바이트코드 조작 라이브러리 사용
  원본 클래스의 서브클래스를 런타임에 생성
  → 인터페이스 불필요, 단 final 제약
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 인터페이스만 있으면 JDK Proxy가 더 낫다

```java
public interface PaymentService {
    void pay(int amount);
}

@Service
public class PaymentServiceImpl implements PaymentService {
    @Transactional
    public void pay(int amount) { ... }
}
```

```
❌ 잘못된 이해:
  "인터페이스 있으면 JDK Proxy, 없으면 CGLIB — 각각 최적"

✅ Spring Boot 2.x 이후 기본:
  spring.aop.proxy-target-class=true (기본값)
  → 인터페이스 있어도 CGLIB 사용

이유:
  JDK Proxy는 인터페이스 타입으로만 주입 가능
  → @Autowired PaymentServiceImpl svc; → 실패 (구체 클래스 주입 불가)
  → 개발자 혼란 유발
  CGLIB은 구체 클래스 타입으로도 주입 가능 → 예측 가능
```

### Before: CGLIB 프록시는 원본 클래스와 무관한 별도 객체다

```java
// ❌ 잘못된 이해
// "CGLIB 프록시는 완전히 새로운 객체 — 원본과 관계없다"

// ✅ 실제:
// CGLIB 프록시는 원본 클래스의 서브클래스
// → instanceof OrderService → true
// → 원본 클래스의 필드, 메서드 상속
// → 오버라이딩 가능한 메서드에만 인터셉션 적용
```

---

## ✨ 올바른 이해와 사용

### After: 두 프록시의 생성 방식과 구조를 명확히 구분

```
JDK Dynamic Proxy:
  java.lang.reflect.Proxy.newProxyInstance() 호출
  → JVM이 런타임에 인터페이스 구현 클래스 생성
  → 모든 메서드 호출 → InvocationHandler.invoke() 위임
  → 생성된 클래스: $Proxy0, $Proxy1, ...

CGLIB:
  Enhancer.create() 호출 (내부적으로 ASM 사용)
  → 원본 클래스를 상속한 서브클래스 바이트코드 생성
  → 오버라이딩 가능한 메서드 → MethodInterceptor.intercept() 위임
  → 생성된 클래스: OrderService$$SpringCGLIB$$0
```

---

## 🔬 내부 동작 원리

### 1. JDK Dynamic Proxy 생성 과정

```java
// JdkDynamicAopProxy.getProxy()
public Object getProxy(ClassLoader classLoader) {
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(
        this.advised, true);
    // 프록시할 인터페이스 목록 수집

    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);

    // java.lang.reflect.Proxy로 프록시 클래스 생성
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
    // this = JdkDynamicAopProxy (InvocationHandler 구현체)
}

// JdkDynamicAopProxy.invoke() — 모든 메서드 호출의 진입점
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    // equals, hashCode 등 특수 메서드 처리
    if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
        return equals(args[0]);
    }

    // Advice 체인 실행
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
        method, targetClass);

    MethodInvocation invocation = new ReflectiveMethodInvocation(
        proxy, target, method, args, targetClass, chain);

    return invocation.proceed();  // Advice 체인 → 실제 메서드
}
```

```
JDK Proxy 바이트코드 구조 (javap 출력 단순화):

public final class $Proxy42 extends java.lang.reflect.Proxy
        implements PaymentService {

    private static Method m3;  // PaymentService.pay() 메서드 참조

    static {
        m3 = Class.forName("com.example.PaymentService")
                  .getMethod("pay", int.class);
    }

    public final void pay(int amount) {
        // 모든 메서드 → InvocationHandler.invoke() 위임
        this.h.invoke(this, m3, new Object[]{amount});
    }
}

특징:
  Proxy 클래스를 상속 → 다른 클래스 상속 불가
  메서드마다 static Field에 Method 객체 캐시
  모든 메서드가 h.invoke()로 위임 (단일 진입점)
```

### 2. CGLIB 생성 과정

```java
// CglibAopProxy.getProxy()
public Object getProxy(ClassLoader classLoader) {
    // CGLIB Enhancer 설정
    Enhancer enhancer = createEnhancer();
    enhancer.setSuperclass(proxySuperClass);    // 원본 클래스를 superclass로
    enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

    // 콜백 설정: 메서드별로 다른 인터셉터 적용
    Callback[] callbacks = getCallbacks(rootClass);
    enhancer.setCallbacks(callbacks);
    enhancer.setCallbackFilter(new ProxyCallbackFilter(
        this.advised.getConfigurationOnlyCopy(), ...));

    // ASM으로 서브클래스 바이트코드 생성 + 인스턴스화
    return enhancer.create();
}
```

```java
// CGLIB 생성 클래스 구조 (javap 단순화)
public class OrderService$$SpringCGLIB$$0 extends OrderService {

    // CGLIB 내부 필드
    private boolean CGLIB$BOUND;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;

    // 원본 메서드를 직접 호출하는 bridge 메서드
    final void CGLIB$placeOrder$0(Order order) {
        super.placeOrder(order);  // 원본 메서드 직접 호출
    }

    // 오버라이딩된 메서드 — 인터셉터 체인으로 위임
    public void placeOrder(Order order) {
        MethodInterceptor interceptor = this.CGLIB$CALLBACK_0;
        if (interceptor == null) {
            CGLIB$BIND_CALLBACKS(this);
            interceptor = this.CGLIB$CALLBACK_0;
        }
        if (interceptor != null) {
            // MethodInterceptor.intercept() 호출
            interceptor.intercept(this, CGLIB$placeOrder$0$Method,
                new Object[]{order}, CGLIB$placeOrder$0$Proxy);
        } else {
            super.placeOrder(order);  // 인터셉터 없으면 직접 호출
        }
    }
}
```

```
CGLIB 특징:
  OrderService 상속 → instanceof OrderService = true
  각 메서드별로 개별 오버라이딩
  CGLIB$method$0: 원본 메서드 직접 호출용 bridge
  CallbackFilter: 메서드별로 다른 Callback(인터셉터) 적용 가능
```

### 3. 바이트코드 레벨 비교 — javap

```bash
# JDK Proxy 클래스 덤프
-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true
# 또는 Spring Boot에서:
System.setProperty("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");

javap -p -c $Proxy42.class
```

```
# JDK Proxy 메서드 (pay):
public final void pay(int);
  Code:
     0: aload_0
     1: getfield      #4   // Field h:Ljava/lang/reflect/InvocationHandler;
     4: aload_0
     5: getstatic     #8   // Field m3:Ljava/lang/reflect/Method;
     8: iload_1
     9: invokestatic  #9   // Integer.valueOf(int)
    12: aastore
    13: invokeinterface #10 // InvocationHandler.invoke()
    ↑ 핵심: invokeinterface → h.invoke() 단일 경로
```

```bash
# CGLIB 클래스 덤프
-Dcglib.debugLocation=/tmp/cglib-classes

javap -p -c OrderService$$SpringCGLIB$$0.class
```

```
# CGLIB 오버라이딩 메서드 (placeOrder):
public void placeOrder(Order);
  Code:
     0: aload_0
     1: getfield      #15  // CGLIB$CALLBACK_0 (MethodInterceptor)
     4: ifnonnull     17
     7: aload_0
     8: invokestatic  #16  // CGLIB$BIND_CALLBACKS
    17: aload_0
    18: getfield      #15
    21: ifnull        42
    24: aload_0
    25: getstatic     #17  // CGLIB$placeOrder$0$Method
    28: aload_1
    29: invokevirtual #18  // MethodInterceptor.intercept()
    ↑ 핵심: 메서드별 직접 invokevirtual → 더 직접적인 경로
```

### 4. final 제약 — 바이트코드 이유

```java
// CGLIB이 final 클래스에 실패하는 이유
public final class PaymentServiceImpl { ... }
// ↑ final → 서브클래스 생성 불가 → CGLIB 프록시 생성 불가

// CGLIB이 final 메서드에 실패하는 이유
public class PaymentServiceImpl {
    public final void pay(int amount) { ... }
    // ↑ final 메서드 → 오버라이딩 불가 → CGLIB 인터셉션 불가
    // 호출해도 super.pay() 바로 실행됨 (인터셉터 체인 우회)
}
```

```
JVM final 의미:
  final 메서드: 서브클래스에서 override 금지
  → CGLIB 서브클래스가 override하려 하면 → VerifyError
  → 스프링은 이를 감지하고 해당 메서드 인터셉션 포기
  → AOP가 적용되지 않는 메서드로 남음 (경고 로그)
```

### 5. 메서드 호출 경로 비교

```
JDK Proxy 호출 경로:
  호출자
  → $Proxy42.pay()
  → InvocationHandler.invoke()
  → ReflectiveMethodInvocation.proceed()
  → Advice 체인 (Around → Before → After)
  → Method.invoke(target, args)  ← 리플렉션으로 원본 호출
  → PaymentServiceImpl.pay()

CGLIB 호출 경로:
  호출자
  → OrderService$$CGLIB.placeOrder()
  → MethodInterceptor.intercept()
  → CglibMethodInvocation.proceed()
  → Advice 체인 (Around → Before → After)
  → CGLIB$placeOrder$0$Proxy.invoke()
  → super.placeOrder()  ← 직접 호출 (리플렉션 아님)
  → OrderService.placeOrder()

차이:
  JDK: 원본 호출이 Method.invoke() → 리플렉션 오버헤드
  CGLIB: 원본 호출이 super.method() → 직접 invokevirtual
  → CGLIB이 원본 메서드 호출 단계에서 더 빠름
```

---

## 💻 실험으로 확인하기

### 실험 1: 프록시 타입 확인

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder() {}
}

// 컨텍스트에서 꺼낸 후 확인
OrderService svc = ctx.getBean(OrderService.class);
System.out.println(svc.getClass().getName());
// Spring Boot 기본 (CGLIB):
// com.example.OrderService$$SpringCGLIB$$0

System.out.println(svc instanceof OrderService);  // true
System.out.println(AopUtils.isCglibProxy(svc));   // true
System.out.println(AopUtils.isJdkDynamicProxy(svc)); // false
```

### 실험 2: JDK Proxy로 전환

```yaml
# application.yml
spring:
  aop:
    proxy-target-class: false  # CGLIB → JDK Proxy로 전환
```

```java
// 인터페이스 있는 경우
PaymentService svc = ctx.getBean(PaymentService.class);
System.out.println(svc.getClass().getName());
// com.sun.proxy.$Proxy42

// 구체 클래스로 주입 시도
PaymentServiceImpl impl = ctx.getBean(PaymentServiceImpl.class);
// → BeanNotOfRequiredTypeException
//   JDK Proxy는 PaymentService 타입, PaymentServiceImpl 아님
```

### 실험 3: CGLIB 클래스 파일 덤프

```java
// JVM 옵션으로 CGLIB 생성 클래스 저장
System.setProperty("cglib.debugLocation", "/tmp/cglib");

// 이후 /tmp/cglib에 생성된 .class 파일 확인
// javap -p -c /tmp/cglib/com/example/OrderService$$SpringCGLIB$$0.class
```

---

## 🤔 트레이드오프

```
JDK Dynamic Proxy:
  장점  JDK 내장 (추가 의존성 없음), 인터페이스 기반 설계 강제
  단점  인터페이스 필수, 구체 클래스 타입 주입 불가
       원본 메서드 호출 시 리플렉션 사용

CGLIB:
  장점  인터페이스 없어도 가능, 구체 클래스 타입 주입 가능
       원본 메서드 직접 호출 (super.method())
  단점  final 클래스/메서드 불가, 기본 생성자 필요(일부 버전)
       생성된 클래스 파일 크기 더 큼

Spring Boot 기본이 CGLIB인 이유:
  @Autowired 구체 클래스 주입 실수 방지
  인터페이스 유무와 관계없이 일관된 동작
  Spring Boot 2.0부터 spring.aop.proxy-target-class=true 기본값

AspectJ 위빙 (참고):
  컴파일/로드 시점에 바이트코드 직접 수정
  런타임 프록시 오버헤드 없음
  private 메서드도 가능
  → 별도 설정 복잡, 스프링 기본 방식 아님
```

---

## 📌 핵심 정리

```
JDK Dynamic Proxy
  java.lang.reflect.Proxy → 인터페이스 구현 클래스 런타임 생성
  $Proxy0 extends Proxy implements 인터페이스
  모든 메서드 → InvocationHandler.invoke() 단일 경로
  원본 호출: Method.invoke() (리플렉션)
  인터페이스 필수

CGLIB
  ASM 바이트코드 → 원본 클래스의 서브클래스 생성
  OrderService$$SpringCGLIB$$0 extends OrderService
  각 메서드 오버라이딩 → MethodInterceptor.intercept()
  원본 호출: super.method() (직접 호출)
  final 클래스/메서드 불가

Spring Boot 기본값
  proxy-target-class=true → CGLIB
  인터페이스 있어도 CGLIB 사용
  구체 클래스 타입 @Autowired 안전

메서드 호출 경로
  JDK: → InvocationHandler → 리플렉션
  CGLIB: → MethodInterceptor → super 직접 호출
```

---

## 🤔 생각해볼 문제

**Q1.** JDK Proxy로 생성된 Bean을 `@Autowired PaymentServiceImpl svc`로 주입받으려 하면 어떤 예외가 발생하는가? 이유를 바이트코드 관점에서 설명하라.

**Q2.** CGLIB 프록시 클래스에서 `equals()`와 `hashCode()`는 어떻게 처리되는가? 원본 클래스의 것을 그대로 쓰는가?

**Q3.** `spring.aop.proxy-target-class=true`(CGLIB)이 기본값임에도 왜 인터페이스 기반 설계가 여전히 권장되는가?

> 💡 **해설**
>
> **Q1.** `BeanNotOfRequiredTypeException`이 발생한다. JDK Proxy로 생성된 `$Proxy42`는 `Proxy`를 상속하고 `PaymentService` 인터페이스를 구현한다. `PaymentServiceImpl`과는 상속 관계가 없다. JVM의 타입 체크 기준에서 `$Proxy42`는 `PaymentServiceImpl` 타입이 아니므로 캐스팅 자체가 불가능하다. 스프링은 Bean 타입이 요청 타입(`PaymentServiceImpl`)에 할당 가능한지 확인하고, 불가능하면 예외를 던진다. CGLIB을 쓰면 `OrderService$$CGLIB`이 `OrderService`의 서브클래스이므로 구체 클래스 타입 주입이 가능하다.
>
> **Q2.** CGLIB은 기본적으로 `Object.equals()`와 `hashCode()`를 오버라이딩하지 않는다. 원본 클래스가 이 메서드를 정의했다면 상속으로 그대로 사용하고, 정의하지 않았다면 `Object`의 구현을 사용한다. 단, 스프링 AOP의 CGLIB 설정(`DelegatingIntroductionInterceptor` 등)에 따라 특정 메서드를 프록시 자체에서 처리하도록 설정할 수 있다. JDK Proxy는 `equals()`와 `hashCode()`를 `InvocationHandler.invoke()`로 라우팅해 `JdkDynamicAopProxy` 내부에서 처리한다.
>
> **Q3.** 인터페이스는 구현의 계약을 명시하고 의존 역전 원칙(DIP)을 지원하기 때문이다. `@Autowired PaymentService svc`로 주입받으면 나중에 `PaymentServiceImpl`을 `MockPaymentService`로 교체할 수 있어 테스트 용이성이 높아진다. 또한 여러 구현체를 `@Primary`나 `@Qualifier`로 선택하는 전략 패턴 적용이 자연스럽다. CGLIB이 기본값이더라도 인터페이스 기반 설계는 테스트·확장성·명시적 계약의 이점을 그대로 제공한다.

---

<div align="center">

**[다음: ProxyFactoryBean 내부 구조 ➡️](./02-proxyfactorybean-internals.md)**

</div>

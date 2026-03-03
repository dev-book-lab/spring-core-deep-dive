# 생성자 vs 필드 vs 세터 주입 — 바이트코드로 보는 진짜 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 생성자 주입, 필드 주입, 세터 주입은 바이트코드 레벨에서 어떻게 다른가?
- 왜 스프링 공식 문서는 생성자 주입을 권장하는가? 단순한 스타일 문제인가?
- 필드 주입이 `final` 키워드를 쓸 수 없는 기술적 이유는 무엇인가?
- 생성자 주입이 순환 참조를 컨테이너 시작 시점에 감지하는 이유는?
- 테스트에서 생성자 주입이 유리한 기술적 근거는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 의존성을 전달하는 방법이 여러 가지인 이유

```java
// 방법 1: 생성자 주입
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// 방법 2: 필드 주입
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}

// 방법 3: 세터 주입
@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

```
세 방법 모두 동작한다.
그렇다면 차이는 단순히 "취향" 문제인가?

답: 아니다.
바이트코드 레벨과 객체 불변성, 테스트 용이성에서
근본적인 차이가 존재한다.
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 필드 주입이 편하니까 아무 문제 없다

```java
// ❌ 필드 주입 남용
@Service
public class UserService {
    @Autowired private UserRepository userRepository;
    @Autowired private EmailService emailService;
    @Autowired private CacheService cacheService;
    @Autowired private AuditService auditService;
    // ... 더 추가해도 눈에 띄지 않음
}
```

```
문제 1: 불변성 없음
  private PaymentService paymentService;
  → final 불가 → 누군가 나중에 바꿀 수 있음

문제 2: 의존성 폭발 감지 불가
  필드만 늘리면 되므로 의존성이 10개가 돼도 눈에 안 띔
  생성자라면: 파라미터 10개 → 즉시 "이 클래스 너무 많은 걸 한다" 신호

문제 3: 스프링 없이 테스트 불가
  UserService svc = new UserService();
  // userRepository = null → NullPointerException
  → 순수 단위 테스트에서 반드시 스프링 컨텍스트 필요

문제 4: 순환 참조 런타임 감지
  A → B → A 순환이 있어도 컨테이너 시작 시 감지 안 됨
  → 첫 getBean() 호출 시 BeanCurrentlyInCreationException
```

---

## ✨ 올바른 이해와 사용

### After: 생성자 주입이 권장되는 기술적 근거

```java
// ✅ 생성자 주입
@Service
public class OrderService {
    private final PaymentService paymentService;  // final 가능 → 불변 보장
    private final InventoryService inventoryService;

    // @Autowired 생략 가능 (단일 생성자면 스프링이 자동 인식)
    public OrderService(PaymentService paymentService,
                        InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
}
```

```
생성자 주입의 기술적 장점:
  1. final 필드 → 불변 객체 (컴파일러 보장)
  2. 생성 시점에 모든 의존성 검증 (null 불가)
  3. 순환 참조 컨테이너 시작 시 즉시 감지
  4. new OrderService(mockPayment, mockInventory) 로 순수 단위 테스트
  5. 파라미터 증가 → 클래스 책임 과다 신호 (리팩토링 유도)
```

---

## 🔬 내부 동작 원리

### 1. 바이트코드 비교 — javap로 직접 확인

```java
// 소스 코드
public class OrderService {
    // 생성자 주입 버전
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

```bash
javap -c OrderService.class
```

```
// 생성자 주입 바이트코드
public OrderService(com.example.PaymentService);
  Code:
     0: aload_0                     // this 로드
     1: invokespecial #1            // Object.<init>() 호출
     4: aload_0                     // this 로드
     5: aload_1                     // 파라미터(paymentService) 로드
     6: putfield #2                 // this.paymentService = paymentService
     9: return

핵심:
  - putfield: 생성자 내부에서 필드에 직접 대입
  - final 필드도 생성자 안에서는 putfield 가능
  - JVM 보장: 생성자 완료 후 필드는 변경 불가 (final semantics)
```

```java
// 필드 주입 버전
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // final 불가
}
```

```
// 필드 주입 바이트코드 (스프링이 리플렉션으로 수행)
// 스프링 내부 AutowiredAnnotationBeanPostProcessor:

Field field = OrderService.class.getDeclaredField("paymentService");
field.setAccessible(true);           // private 접근 허용
field.set(orderServiceInstance, paymentServiceBean);
```

```
// JVM 레벨:
// setAccessible(true) → AccessController.checkAccess() 우회
// Field.set() → Unsafe.putObject() 또는 직접 메모리 쓰기

// final 필드에 리플렉션으로 set() 시도:
final Field f = ...;
f.setAccessible(true);
f.set(obj, value);  // Java 9 이후 경고, Java 17+ 모듈 시스템에서 실패 가능
→ 그래서 @Autowired 필드는 final 선언 자체가 컴파일 에러는 아니지만
  스프링이 주입 시 IllegalAccessException 발생
```

```java
// 세터 주입 버전
public class OrderService {
    private PaymentService paymentService;  // final 불가

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

```
// 세터 주입 바이트코드 (스프링이 리플렉션으로 메서드 호출)
Method setter = OrderService.class.getMethod("setPaymentService", PaymentService.class);
setter.invoke(orderServiceInstance, paymentServiceBean);

// JVM 레벨:
// invokevirtual → setPaymentService(PaymentService)
// 내부에서 putfield → this.paymentService = paymentService
```

### 2. 스프링이 주입을 수행하는 위치 — AbstractAutowireCapableBeanFactory

```java
// AbstractAutowireCapableBeanFactory.populateBean()
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // ...
    // InstantiationAwareBeanPostProcessor.postProcessProperties() 호출
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        // AutowiredAnnotationBeanPostProcessor가 여기서 @Autowired 처리
    }
}
```

```java
// AutowiredAnnotationBeanPostProcessor.postProcessProperties()
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {

    // @Autowired 대상 메타데이터 조회 (캐시된 것)
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);

    metadata.inject(bean, beanName, pvs);  // 실제 주입
    return pvs;
}
```

```java
// AutowiredFieldElement.inject() — 필드 주입 핵심
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;

    // 타입으로 Bean 후보 탐색
    Object value = resolveFieldValue(field, bean, beanName);

    if (value != null) {
        // 리플렉션으로 private 필드에 접근
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);  // ← 여기서 실제 주입
    }
}
```

```
주입 시점 비교:

생성자 주입:
  createBeanInstance() → 인스턴스 생성 시
  → 생성자 파라미터를 먼저 resolve하고 생성자 호출
  → 인스턴스 생성과 주입이 동시에 발생

필드/세터 주입:
  createBeanInstance() → 인스턴스 생성 (의존성 없이)
  populateBean()       → 생성된 인스턴스에 리플렉션으로 주입
  → 생성과 주입이 분리됨 → 일시적으로 불완전한 객체 존재
```

### 3. 순환 참조 감지 시점 차이

```java
// A → B → A 순환 참조

// 생성자 주입: 컨테이너 시작 시 즉시 감지
@Service
class A {
    A(B b) { }  // B가 필요
}
@Service
class B {
    B(A a) { }  // A가 필요
}

// refresh() 시:
// A 생성 시도 → B 필요 → B 생성 시도 → A 필요
// → A가 이미 "생성 중" → BeanCurrentlyInCreationException
// → 컨테이너 시작 실패 (명확한 오류)
```

```java
// 필드/세터 주입: 3단계 캐시로 해결
@Service
class A {
    @Autowired B b;
}
@Service
class B {
    @Autowired A a;
}

// refresh() 시:
// A 생성 → A 인스턴스(미완성) singletonFactories에 등록
// A의 B 주입 → B 생성 시작
// B의 A 주입 → singletonFactories에서 A(미완성) 조회 → 주입 성공
// B 완성 → A 완성
// → 순환 참조 해결됨 (경고 없이)
// Spring Boot 2.6+ 기본: allowCircularReferences=false로 변경
```

### 4. 테스트 용이성 비교

```java
// 생성자 주입 — 스프링 없이 단위 테스트
class OrderServiceTest {
    @Test
    void placeOrder_success() {
        // Mock 직접 생성, 스프링 컨텍스트 불필요
        PaymentService mockPayment = Mockito.mock(PaymentService.class);
        InventoryService mockInventory = Mockito.mock(InventoryService.class);

        OrderService svc = new OrderService(mockPayment, mockInventory);
        // 순수 단위 테스트 가능
    }
}

// 필드 주입 — 스프링 없이 테스트하려면 리플렉션 필요
class OrderServiceTest {
    @Test
    void placeOrder_success() throws Exception {
        OrderService svc = new OrderService();

        // 리플렉션으로 private 필드 주입
        Field f = OrderService.class.getDeclaredField("paymentService");
        f.setAccessible(true);
        f.set(svc, Mockito.mock(PaymentService.class));
        // 또는 @SpringBootTest + @MockBean 사용 → 컨텍스트 로드 비용
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: final 필드 차이 확인

```java
// 생성자 주입 → final 가능
@Service
public class ServiceA {
    private final DepService dep;  // ✅ final

    public ServiceA(DepService dep) {
        this.dep = dep;
    }
}

// 필드 주입 → final 불가
@Service
public class ServiceB {
    @Autowired
    private final DepService dep;  // ❌ 컴파일 오류 또는 스프링 주입 실패
    // "Cannot @Autowired a final field"
}
```

### 실험 2: 순환 참조 감지 시점

```java
// Spring Boot 2.6+ 기본 설정 (allowCircularReferences=false)
@Service class CircularA { CircularA(CircularB b) {} }
@Service class CircularB { CircularB(CircularA a) {} }

// 컨텍스트 로드 시 즉시:
// The dependencies of some of the beans in the application context
// form a cycle:
// circularA → circularB → circularA
```

### 실험 3: 테스트 코드 간결성 비교

```java
// 생성자 주입 테스트 — 단 3줄
@Test
void test() {
    var svc = new OrderService(mock(PaymentService.class), mock(InventoryService.class));
    assertThat(svc.placeOrder(new Order())).isNotNull();
}

// 필드 주입 테스트 — @SpringBootTest 필요
@SpringBootTest  // 전체 컨텍스트 로드 (수 초)
class OrderServiceTest {
    @Autowired OrderService svc;        // 실제 Bean
    @MockBean PaymentService payment;   // Mock Bean
    // ...
}
```

---

## 🤔 트레이드오프

```
생성자 주입:
  장점  final 불변성, 순환 참조 조기 감지, 순수 단위 테스트
  단점  의존성 많으면 생성자 파라미터 길어짐 → SRP 위반 신호로 활용

세터 주입이 유효한 경우:
  선택적 의존성 (주입 안 해도 동작 가능)
  → @Autowired(required=false)
  기존 레거시 클래스에 의존성 추가 시
  → 기본 생성자 유지하면서 선택적 주입

필드 주입을 피해야 하는 이유 요약:
  final 불가 (불변성 없음)
  순수 단위 테스트 어려움
  의존성 과다 시 경고 신호 없음
  스프링 리플렉션 의존 (모듈 시스템 강화로 미래 호환성 불확실)
```

---

## 📌 핵심 정리

```
생성자 주입 바이트코드
  생성자 파라미터 → 바로 putfield
  final 필드에도 쓰기 가능 (생성자 내부)
  생성과 주입이 동시 발생

필드 주입 바이트코드
  리플렉션: field.setAccessible(true) + field.set()
  final 필드 주입 불가 (Java 모듈 시스템 강화 이후)
  생성과 주입이 분리 → 일시적 불완전 객체

세터 주입 바이트코드
  리플렉션: method.invoke()
  내부에서 putfield
  final 불가, 선택적 의존성에 적합

순환 참조 감지
  생성자 주입 → 컨테이너 시작 시 즉시 BeanCurrentlyInCreationException
  필드/세터 주입 → 3단계 캐시로 해결 (감지 안 됨)

권장 방향
  기본: 생성자 주입
  선택적 의존성: 세터 주입
  필드 주입: 테스트 코드 내 @InjectMocks 제외 사용 지양
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드가 컴파일은 되지만 스프링 컨텍스트 로드 시 실패하는 이유는?

```java
@Service
public class MyService {
    @Autowired
    private final MyRepository repository;
}
```

**Q2.** 생성자 주입에서 `@Autowired`를 생략할 수 있는 조건은 무엇인가? 내부적으로 어떻게 판단하는가?

**Q3.** 세터 주입이 생성자 주입보다 적합한 실제 사례를 하나 들고, 기술적 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** `final` 필드는 반드시 생성자 또는 필드 초기화 시점에 값이 할당돼야 한다. `@Autowired` 필드 주입은 객체 생성 이후 리플렉션(`field.set()`)으로 값을 넣는 방식인데, `final` 필드에 대해 `setAccessible(true)` 후 `set()` 시도 시 `IllegalAccessException`이 발생한다. Java 9 이후 모듈 시스템이 강화되면서 이 제한이 더욱 엄격해졌다. 스프링은 이를 `BeanCreationException`으로 감싸서 던진다.
>
> **Q2.** 스프링 4.3부터 단일 생성자가 있는 경우 `@Autowired` 없이도 스프링이 자동으로 생성자 주입을 수행한다. `AutowiredAnnotationBeanPostProcessor`가 Bean 클래스의 생성자를 분석할 때, `@Autowired`가 없어도 생성자가 딱 하나뿐이면 그 생성자를 주입 대상으로 선택한다. 생성자가 두 개 이상이면 `@Autowired`를 명시해야 어느 생성자를 사용할지 알 수 있다.
>
> **Q3.** 플러그인 아키텍처나 선택적 기능에서 세터 주입이 적합하다. 예를 들어 `EmailService`가 있을 때 `NotificationService`가 반드시 이메일 서비스 없이도 동작해야 하는 경우 `@Autowired(required=false) public void setEmailService(EmailService es) { ... }`로 선언하면, `EmailService` Bean이 없어도 `NotificationService`가 정상 생성된다. 생성자 주입이었다면 `EmailService` Bean이 없을 때 컨텍스트 로드 자체가 실패한다.

---

<div align="center">

**[다음: 순환 참조 해결 — 3단계 캐시 ➡️](./02-circular-dependency-3cache.md)**

</div>

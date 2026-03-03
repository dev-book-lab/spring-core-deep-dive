# 순환 참조 해결 — 3단계 캐시의 동작 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스프링의 3단계 캐시(`singletonObjects`, `earlySingletonObjects`, `singletonFactories`)는 각각 무슨 역할인가?
- 필드 주입에서 순환 참조가 해결되는 정확한 단계는?
- 생성자 주입에서 순환 참조가 해결되지 않는 이유는?
- AOP 프록시가 적용된 Bean에서 순환 참조 시 `earlySingletonObjects`가 필요한 이유는?
- Spring Boot 2.6 이후 순환 참조 기본값이 바뀐 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: A가 B를 필요로 하고, B도 A를 필요로 할 때

```java
@Service class OrderService {
    @Autowired PaymentService paymentService;
}

@Service class PaymentService {
    @Autowired OrderService orderService;
}
```

```
단순한 생성 순서:

OrderService 생성 시도
  → PaymentService 필요
  → PaymentService 생성 시도
    → OrderService 필요
    → OrderService 생성 시도 (이미 진행 중!)
    → 무한 루프 → StackOverflow 또는 오류

해결 아이디어:
  "완성되지 않은 객체라도 미리 노출하면 된다"
  A를 생성하기 시작하면, 아직 완성 전이더라도
  B가 A를 참조할 수 있도록 참조를 미리 등록
```

이것이 3단계 캐시의 핵심 아이디어다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 3단계 캐시가 모든 순환 참조를 해결한다

```java
// ❌ 잘못된 이해
// "순환 참조는 스프링이 다 알아서 해결해준다"

// 생성자 주입 순환 참조
@Service class A { A(B b) {} }
@Service class B { B(A a) {} }

// → BeanCurrentlyInCreationException 발생
// → 3단계 캐시로 해결 불가
```

```
3단계 캐시가 해결하는 것:
  Singleton Bean + 필드/세터 주입 조합

3단계 캐시가 해결 못 하는 것:
  생성자 주입 순환 참조
  Prototype scope 순환 참조
  Spring Boot 2.6+ (기본 비활성화)
```

### Before: 순환 참조는 나쁘지 않다

```java
// ❌ "스프링이 해결해주니까 괜찮다"
@Service class UserService {
    @Autowired OrderService orderService;  // 왜 UserService가 OrderService를?
}
@Service class OrderService {
    @Autowired UserService userService;    // 왜 OrderService가 UserService를?
}
```

```
순환 참조가 의미하는 것:
  두 클래스가 서로를 강하게 결합
  → 단일 책임 원칙(SRP) 위반 신호
  → 분리해야 할 책임이 얽혀 있음

스프링이 해결해주더라도:
  아키텍처 문제는 그대로 존재
  → 순환 참조 발생 시 먼저 설계를 재검토
```

---

## ✨ 올바른 이해와 사용

### After: 3단계 캐시의 정확한 역할을 이해하고, 순환 참조 자체를 제거한다

```java
// 순환 참조 해소 방법 1: 공통 의존성 추출
@Service class UserOrderMediator {
    // UserService와 OrderService가 공통으로 필요한 로직
}

@Service class UserService {
    @Autowired UserOrderMediator mediator;  // 순환 없음
}

@Service class OrderService {
    @Autowired UserOrderMediator mediator;  // 순환 없음
}

// 순환 참조 해소 방법 2: @Lazy로 지연 주입
@Service class A {
    @Autowired @Lazy B b;  // B를 프록시로 주입, 실제 사용 시 resolve
}
```

---

## 🔬 내부 동작 원리

### 1. 3단계 캐시 구조

```java
// DefaultSingletonBeanRegistry.java
public class DefaultSingletonBeanRegistry {

    // 1단계: 완성된 Singleton Bean 저장소
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 2단계: 조기 노출된 Bean (아직 완성 전, 프록시 적용 전)
    //        순환 참조 발생 시 여기에 임시 저장
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    // 3단계: ObjectFactory — Bean을 나중에 만들어주는 람다
    //        AOP 프록시가 있을 때 early reference도 프록시로 감싸기 위해 존재
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    // 현재 생성 중인 Bean 이름 집합 (순환 감지용)
    private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
}
```

```
캐시별 역할:

singletonObjects (1단계)
  완전히 초기화된 Bean
  모든 주입, 초기화 콜백, BPP 처리 완료
  getBean() 호출 시 여기서 먼저 탐색

earlySingletonObjects (2단계)
  생성은 됐지만 아직 완전히 초기화 안 된 Bean
  singletonFactories에서 꺼내서 보관
  순환 참조 시 다른 Bean에 임시 제공

singletonFactories (3단계)
  Bean을 만들어줄 ObjectFactory 람다 저장
  AOP 프록시 있으면: () -> getEarlyBeanReference(beanName, mbd, bean)
  AOP 없으면:       () -> bean (원본 그대로)
```

### 2. 순환 참조 해결 단계별 시나리오

```
시나리오: OrderService(A) ←→ PaymentService(B) 순환 참조 (필드 주입)

Step 1: OrderService(A) 생성 시작
  singletonsCurrentlyInCreation.add("orderService")
  
  createBeanInstance("orderService")
  → new OrderService()  (paymentService 아직 null)
  
  → singletonFactories.put("orderService", () -> orderServiceInstance)
    (3단계 캐시에 ObjectFactory 등록)

Step 2: OrderService(A)의 paymentService 주입 시작
  populateBean → @Autowired paymentService 처리
  → getBean("paymentService") 호출

Step 3: PaymentService(B) 생성 시작
  singletonsCurrentlyInCreation.add("paymentService")
  
  createBeanInstance("paymentService")
  → new PaymentService()  (orderService 아직 null)
  
  → singletonFactories.put("paymentService", () -> paymentServiceInstance)

Step 4: PaymentService(B)의 orderService 주입 시작
  populateBean → @Autowired orderService 처리
  → getBean("orderService") 호출

Step 5: getSingleton("orderService") 캐시 탐색
  1단계 singletonObjects     → 없음 (아직 완성 안 됨)
  2단계 earlySingletonObjects → 없음 (아직 2단계 진입 안 함)
  3단계 singletonFactories    → 있음! ObjectFactory 발견
  
  ObjectFactory.getObject() 호출
  → orderServiceInstance 반환
  
  earlySingletonObjects.put("orderService", orderServiceInstance)
  singletonFactories.remove("orderService")

Step 6: PaymentService(B) 완성
  orderService 필드에 Step 5의 (미완성) OrderService 주입
  initializeBean(paymentService) 실행
  singletonObjects.put("paymentService", paymentServiceInstance)  ← 1단계 등록

Step 7: 다시 OrderService(A)로 돌아옴
  paymentService 필드에 완성된 PaymentService 주입
  initializeBean(orderService) 실행
  singletonObjects.put("orderService", orderServiceInstance)  ← 1단계 등록
  earlySingletonObjects.remove("orderService")
```

### 3. AOP 프록시가 있을 때 — earlySingletonObjects의 진짜 역할

```java
// AOP 프록시가 적용된 순환 참조
@Service
@Transactional  // ← CGLIB 프록시 생성 예정
class OrderService {
    @Autowired PaymentService paymentService;
}

@Service
class PaymentService {
    @Autowired OrderService orderService;
}
```

```
문제:
  PaymentService가 주입받아야 하는 것: OrderService의 CGLIB 프록시
  But: 프록시는 BPP After(initializeBean 이후)에서 생성됨
  순환 참조 시 initializeBean 이전에 early reference가 노출됨
  → 프록시 없는 원본이 PaymentService에 주입될 위험!

해결: singletonFactories에 ObjectFactory로 등록

// AbstractAutowireCapableBeanFactory.doCreateBean():
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));

// getEarlyBeanReference():
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
        exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        // AbstractAutoProxyCreator.getEarlyBeanReference() 가 여기서 프록시 미리 생성
    }
    return exposedObject;
}
```

```
AOP 있을 때 흐름:

Step 3(확장): singletonFactories에 등록 시
  () -> getEarlyBeanReference("orderService", mbd, rawOrderService)
       → AbstractAutoProxyCreator가 프록시 미리 생성
       → 프록시를 earlySingletonObjects에 저장

Step 5(확장): PaymentService가 OrderService 요청 시
  earlySingletonObjects에서 OrderService의 프록시 반환
  → PaymentService는 처음부터 프록시를 주입받음

결론:
  earlySingletonObjects가 없으면
  AOP + 순환 참조 조합에서 원본과 프록시 불일치 발생
  → 2단계 캐시의 존재 이유
```

### 4. 생성자 주입이 해결 안 되는 이유

```java
@Service class A { A(B b) {} }
@Service class B { B(A a) {} }
```

```
생성자 주입 시도:

Step 1: A 생성 시도
  singletonsCurrentlyInCreation.add("a")
  createBeanInstance("a") 호출
  → 생성자 파라미터 B가 필요
  → getBean("b") 호출

Step 2: B 생성 시도
  singletonsCurrentlyInCreation.add("b")
  createBeanInstance("b") 호출
  → 생성자 파라미터 A가 필요
  → getBean("a") 호출

Step 3: A 조회
  singletonsCurrentlyInCreation에 "a" 있음 → 현재 생성 중!
  → BeanCurrentlyInCreationException 즉시 발생

이유:
  필드/세터 주입: 인스턴스 생성 후 주입
  → new A() 는 성공 → 3단계 캐시에 early ref 등록 가능

  생성자 주입: 인스턴스 생성 = 주입
  → new A(b) 에서 B가 필요 → B 생성 시도
  → B 생성에서 A 필요 → A는 아직 new 도 못 함
  → singletonFactories에 등록할 인스턴스 자체가 없음
```

---

## 💻 실험으로 확인하기

### 실험 1: 3단계 캐시 상태 추적

```java
// 디버그 로그로 캐시 상태 확인
// application.yml
logging:
  level:
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry: TRACE

// 출력 예시:
// Creating shared instance of singleton bean 'orderService'
// Creating shared instance of singleton bean 'paymentService'
// Returning cached instance of singleton bean 'orderService'  ← earlySingletonObjects에서
```

### 실험 2: Spring Boot 2.6+ 순환 참조 비활성화

```yaml
# application.yml
spring:
  main:
    allow-circular-references: true  # 기본 false → 명시적으로 허용해야 함
```

```
Spring Boot 2.6 변경 이유:
  순환 참조는 설계 문제 신호
  기본적으로 허용하면 개발자가 인지 못할 수 있음
  → 기본값 false로 변경
  → 순환 참조 발생 시 명시적으로 허용하거나 설계 개선 유도
```

### 실험 3: @Lazy로 순환 참조 해소

```java
// 생성자 주입에서도 @Lazy로 해결 가능
@Service
class A {
    private final B b;

    A(@Lazy B b) {  // B를 프록시로 감싸서 주입 (실제 B 생성 지연)
        this.b = b;
    }
}

@Service
class B {
    private final A a;

    B(A a) {
        this.a = a;
    }
}
// A 생성 시: B 프록시 주입 (실제 B 아직 생성 안 됨)
// B 생성 시: A는 이미 완성 → 정상 주입
// A에서 b.method() 호출 시: 그때 실제 B 생성
```

---

## 🤔 트레이드오프

```
순환 참조 허용 vs 금지:
  허용 (Spring Boot 2.5 이하 기본)
    기존 레거시 코드 동작 유지
    단, 설계 문제를 숨김

  금지 (Spring Boot 2.6+ 기본)
    순환 참조 발생 즉시 명확한 오류
    설계 개선 강제

@Lazy 해결책의 트레이드오프:
  장점: 생성자 주입 + 순환 참조 동시 해결
  단점: 실제 Bean 대신 프록시가 주입됨
        → 첫 메서드 호출 시 실제 Bean 생성 (지연)
        → 프록시 오버헤드 존재

3단계 캐시의 한계:
  Singleton scope만 지원
  Prototype scope 순환 참조는 해결 불가
  → 매번 새 인스턴스를 만들어야 해서 early reference 재사용 불가
```

---

## 📌 핵심 정리

```
3단계 캐시 역할
  1단계 singletonObjects     완전히 완성된 Bean
  2단계 earlySingletonObjects 생성 완료, 초기화 전 (AOP 프록시 포함)
  3단계 singletonFactories    ObjectFactory 람다 (프록시 생성 지점)

순환 참조 해결 흐름
  A 생성 시작 → 3단계 캐시 등록
  B 생성 시 A 요청 → 3단계 → 2단계로 이동하여 반환
  B 완성 → 1단계 등록
  A에 B 주입 → A 완성 → 1단계 등록

AOP + 순환 참조
  getEarlyBeanReference() 에서 프록시 미리 생성
  2단계 캐시에 프록시 저장 → 원본/프록시 불일치 방지

생성자 주입 해결 불가 이유
  인스턴스 생성 전 의존성 필요 → early reference 등록 자체 불가
  → BeanCurrentlyInCreationException 즉시 발생

Spring Boot 2.6+ 기본값
  allowCircularReferences=false
  순환 참조 발생 시 설계 재검토 유도
```

---

## 🤔 생각해볼 문제

**Q1.** `singletonFactories`가 `ObjectFactory` 람다를 저장하는 이유는 무엇인가? 그냥 원본 Bean을 바로 저장하면 안 되는가?

**Q2.** Prototype scope Bean에서 순환 참조가 해결되지 않는 이유를 3단계 캐시 동작과 연결해 설명하라.

**Q3.** 다음 코드에서 `PaymentService`에 최종 주입되는 `OrderService`는 원본인가, CGLIB 프록시인가?

```java
@Service @Transactional
class OrderService {
    @Autowired PaymentService paymentService;
}

@Service
class PaymentService {
    @Autowired OrderService orderService;
}
```

> 💡 **해설**
>
> **Q1.** `@Transactional` 등 AOP 프록시가 적용된 Bean의 경우, early reference도 프록시여야 한다. 원본 Bean을 직접 저장하면 순환 참조 시 상대 Bean이 원본을 주입받게 되고, 이후 BPP After에서 프록시로 교체돼도 이미 원본을 가진 Bean은 프록시 없이 동작한다. `ObjectFactory` 람다로 저장하면, 실제로 early reference가 필요한 순간에 `getEarlyBeanReference()`를 호출해 AOP 프록시를 즉시 생성하고 2단계 캐시에 보관할 수 있다. 프록시가 없는 Bean은 람다가 원본을 그대로 반환한다.
>
> **Q2.** 3단계 캐시는 Singleton Bean에만 동작한다. Prototype scope는 `getBean()` 호출마다 새 인스턴스를 생성하므로, "현재 생성 중인 인스턴스"를 캐시에 보관하는 개념 자체가 없다. A(Prototype)가 B를 필요로 하고 B가 A를 필요로 할 때, A를 캐시에 등록할 수 없으므로 B가 A를 요청하면 새 A 생성을 시도하고, 그 새 A도 B를 필요로 해 무한 재귀가 발생한다. 스프링은 `BeanCreationException`으로 이를 감지해 실패시킨다.
>
> **Q3.** CGLIB 프록시가 주입된다. `OrderService`에 `@Transactional`이 있으므로 `AbstractAutoProxyCreator`가 동작한다. `doCreateBean()` 단계에서 `singletonFactories`에 `() -> getEarlyBeanReference("orderService", mbd, rawBean)` 람다가 등록된다. `PaymentService`가 `OrderService`를 요청할 때 이 람다가 실행되고, `AbstractAutoProxyCreator.getEarlyBeanReference()`에서 CGLIB 프록시가 생성되어 2단계 캐시에 저장된다. `PaymentService`에는 이 프록시가 주입된다.

---

<div align="center">

**[⬅️ 이전: 생성자 vs 필드 vs 세터 주입](./01-constructor-vs-field-injection-bytecode.md)** | **[다음: @Autowired vs @Inject vs @Resource ➡️](./03-autowired-inject-resource.md)**

</div>

# @Transactional이 프록시인 이유 — TransactionInterceptor 내부 동작

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Transactional`이 프록시로 구현된 이유와 대안(AspectJ 위빙)의 차이는?
- `TransactionInterceptor`가 Advice 체인에 들어가는 과정은?
- `TransactionInterceptor.invoke()`에서 트랜잭션 시작 → 커밋/롤백이 이루어지는 정확한 경로는?
- `PlatformTransactionManager`에 위임하는 구조가 가져다주는 이점은?
- `@Transactional`의 `propagation`, `isolation`, `rollbackFor` 속성은 어디서 처리되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 비즈니스 로직에 트랜잭션 코드를 섞으면 안 된다

```java
// ❌ 트랜잭션 코드 직접 작성
public void placeOrder(Order order) {
    TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());
    try {
        orderRepo.save(order);
        paymentRepo.charge(order.amount());
        txManager.commit(status);
    } catch (Exception ex) {
        txManager.rollback(status);
        throw ex;
    }
}

// ✅ @Transactional 선언
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    paymentRepo.charge(order.amount());
    // 트랜잭션 시작/커밋/롤백 코드 없음
}
```

```
해결 원리:
  @Transactional 메서드를 가진 Bean → 프록시로 교체
  프록시의 해당 메서드 호출 → TransactionInterceptor 실행
  TransactionInterceptor → 트랜잭션 시작 → 원본 메서드 호출 → 커밋/롤백

  비즈니스 로직과 트랜잭션 관리 완전 분리
  PlatformTransactionManager를 통해 DB 구현에서도 독립
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 같은 클래스의 메서드를 this로 호출하면 트랜잭션이 적용된다

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        // ...
        this.sendNotification(order);  // ← this 호출
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(Order order) {
        // REQUIRES_NEW로 별도 트랜잭션에서 실행되길 기대
        // 실제로는 같은 트랜잭션에서 실행됨!
    }
}
```

```
❌ 잘못된 이해:
  "같은 클래스라도 @Transactional 붙어 있으면 트랜잭션이 제어된다"

✅ 실제 (Self-Invocation 함정):
  this.sendNotification() → 프록시를 거치지 않고 원본 객체 직접 호출
  → TransactionInterceptor 미실행
  → propagation = REQUIRES_NEW 무시됨
  → placeOrder의 트랜잭션 그대로 참여

해결:
  1. 별도 Bean으로 분리 → 프록시를 통한 외부 호출
  2. ApplicationContext.getBean()으로 프록시 획득 (비권장)
  3. @Scope("prototype") + @Lazy 자기 주입
  4. AspectJ 위빙 (컴파일/로드 타임)
```

### Before: @Transactional(readOnly=true)는 SELECT를 보장한다

```java
@Transactional(readOnly = true)
public List<Order> getOrders() {
    orderRepo.save(new Order());  // readOnly인데 저장?
    return orderRepo.findAll();
}
```

```
❌ 잘못된 이해:
  "readOnly=true면 쓰기 작업이 차단된다"

✅ 실제:
  readOnly=true → JDBC Connection에 setReadOnly(true) 힌트 전달
  일부 DB 드라이버/DB가 이 힌트를 무시할 수 있음
  JPA에서는 FlushMode.MANUAL로 설정 (flush 방지)
  → 실수로 save() 호출해도 예외 발생하지 않을 수 있음
  → 최적화 힌트이지 보안 장벽이 아님
```

---

## ✨ 올바른 이해와 사용

### After: TransactionInterceptor가 Advice 체인에 들어가는 경로

```
@EnableTransactionManagement 등록 경로:

@EnableTransactionManagement
  → @Import(TransactionManagementConfigurationSelector)
  → ProxyTransactionManagementConfiguration 등록

ProxyTransactionManagementConfiguration:
  @Bean BeanFactoryTransactionAttributeSourceAdvisor advisor
    → Pointcut:  TransactionAttributeSourcePointcut
                 (@Transactional 있는 메서드만 매칭)
    → Advice:    TransactionInterceptor
                 (트랜잭션 시작/커밋/롤백 실행)

  → AbstractAutoProxyCreator가 advisor 수집
  → @Transactional Bean → 프록시 생성 + TransactionInterceptor 체인 삽입
```

---

## 🔬 내부 동작 원리

### 1. TransactionAttributeSourcePointcut — 대상 선별

```java
// TransactionAttributeSourcePointcut
// → @Transactional 붙은 메서드만 매칭하는 Pointcut
class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut {

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        TransactionAttributeSource tas = getTransactionAttributeSource();
        // tas = AnnotationTransactionAttributeSource
        // → @Transactional 어노테이션을 읽어 TransactionAttribute 반환
        // → null이면 매칭 안 됨 (트랜잭션 미적용)
        return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
    }
}

// AnnotationTransactionAttributeSource.getTransactionAttribute()
public TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass) {

    // 캐시 조회
    Object cacheKey = getCacheKey(method, targetClass);
    TransactionAttribute cached = this.attributeCache.get(cacheKey);
    if (cached != null) return cached == NULL_TRANSACTION_ATTRIBUTE ? null : cached;

    // @Transactional 어노테이션 탐색 순서:
    // 1. 메서드 직접 선언
    // 2. 메서드의 인터페이스 선언
    // 3. 클래스 레벨
    // 4. 클래스의 인터페이스 레벨
    TransactionAttribute txAttr = determineTransactionAttribute(method, targetClass);

    this.attributeCache.put(cacheKey, txAttr != null ? txAttr : NULL_TRANSACTION_ATTRIBUTE);
    return txAttr;
}
```

### 2. TransactionInterceptor.invoke() — 트랜잭션 핵심 로직

```java
// TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor
public Object invoke(MethodInvocation invocation) throws Throwable {
    // 타겟 클래스 확인
    Class<?> targetClass = (invocation.getThis() != null
        ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // TransactionAspectSupport.invokeWithinTransaction()에 위임
    return invokeWithinTransaction(invocation.getMethod(), targetClass,
        new CoroutinesInvocationCallback() {
            @Override
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();  // 다음 인터셉터 또는 실제 메서드
            }
        });
}

// TransactionAspectSupport.invokeWithinTransaction()
protected Object invokeWithinTransaction(Method method, Class<?> targetClass,
                                          InvocationCallback invocation) throws Throwable {

    // 1. @Transactional 속성 조회
    TransactionAttributeSource tas = getTransactionAttributeSource();
    TransactionAttribute txAttr = (tas != null
        ? tas.getTransactionAttribute(method, targetClass) : null);

    // 2. TransactionManager 결정
    TransactionManager tm = determineTransactionManager(txAttr);

    // 리액티브 vs 명령형 분기
    if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager rtm) {
        // 리액티브 트랜잭션 처리 (Mono/Flux)
        return new ReactiveTransactionSupport(rtm).invokeWithinTransaction(...);
    }

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // 3. 선언적 트랜잭션 처리
    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // 트랜잭션 시작 (propagation 처리 포함)
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // 4. 실제 메서드(또는 다음 인터셉터) 실행
            retVal = invocation.proceedWithInvocation();

        } catch (Throwable ex) {
            // 5. 예외 발생 → 롤백 여부 결정
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // 6. TransactionInfo 정리 (스레드로컬 복원)
            cleanupTransactionInfo(txInfo);
        }

        // 7. 정상 종료 → 커밋
        if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            // Vavr Try 타입 특수 처리
            TransactionStatus status = txInfo.getTransactionStatus();
            if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
            }
        }
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
    // ...
}
```

### 3. createTransactionIfNecessary() — propagation 처리

```java
// TransactionAspectSupport.createTransactionIfNecessary()
protected TransactionInfo createTransactionIfNecessary(
        PlatformTransactionManager ptm, TransactionAttribute txAttr, String joinpointId) {

    TransactionStatus status = null;
    if (txAttr != null) {
        // AbstractPlatformTransactionManager.getTransaction()
        // → propagation 정책에 따라 신규 트랜잭션 or 기존 참여 결정
        status = ptm.getTransaction(txAttr);
    }
    return prepareTransactionInfo(ptm, txAttr, joinpointId, status);
}

// AbstractPlatformTransactionManager.getTransaction() — propagation 핵심
public final TransactionStatus getTransaction(TransactionDefinition definition) {

    Object transaction = doGetTransaction();  // 현재 스레드 트랜잭션 상태 조회

    if (isExistingTransaction(transaction)) {
        // 기존 트랜잭션 존재
        return handleExistingTransaction(definition, transaction, ...);
        // REQUIRED     → 기존 참여
        // REQUIRES_NEW → 기존 중단, 새 트랜잭션 시작
        // NESTED       → SavePoint 생성
        // NOT_SUPPORTED → 기존 중단, 트랜잭션 없이 실행
        // NEVER        → 예외 발생
    }

    // 기존 트랜잭션 없음
    if (definition.getPropagationBehavior() == PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(...);
    }
    if (definition.getPropagationBehavior() == PROPAGATION_REQUIRED
            || definition.getPropagationBehavior() == PROPAGATION_REQUIRES_NEW
            || definition.getPropagationBehavior() == PROPAGATION_NESTED) {
        // 새 트랜잭션 시작
        doBegin(transaction, definition);
        // → JDBC: connection.setAutoCommit(false)
        // → JPA: entityManager.getTransaction().begin()
    }
    return prepareTransactionStatus(...);
}
```

### 4. completeTransactionAfterThrowing() — 롤백 결정

```java
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
    if (txInfo.hasTransaction()) {
        if (txInfo.transactionAttribute != null
                && txInfo.transactionAttribute.rollbackOn(ex)) {
            // rollbackOn() 판단:
            // 기본: RuntimeException, Error → 롤백
            //       Checked Exception → 커밋 (!)
            // rollbackFor 속성으로 재정의 가능
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        } else {
            // rollbackOn()이 false → 예외여도 커밋
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}

// RuleBasedTransactionAttribute.rollbackOn()
public boolean rollbackOn(Throwable ex) {
    // @Transactional(rollbackFor = ..., noRollbackFor = ...) 규칙 적용
    for (RollbackRuleAttribute rule : this.rollbackRules) {
        if (rule.getDepth(ex) >= 0) {
            return !(rule instanceof NoRollbackRuleAttribute);
        }
    }
    // 규칙 없으면 기본: RuntimeException / Error → true
    return super.rollbackOn(ex);
}
```

```
Checked Exception 주의:
  @Transactional
  public void process() throws IOException {  // Checked Exception
      throw new IOException("...");
  }
  → 기본적으로 커밋 시도! (Checked Exception은 rollbackOn=false 기본)
  → 의도한 동작인지 반드시 확인

  해결:
  @Transactional(rollbackFor = IOException.class)
  public void process() throws IOException { ... }
```

### 5. 트랜잭션 동기화 — TransactionSynchronizationManager

```java
// 트랜잭션 관련 상태는 ThreadLocal에 저장
public abstract class TransactionSynchronizationManager {

    // 현재 스레드의 트랜잭션 리소스 (ConnectionHolder 등)
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    // 현재 스레드의 트랜잭션 이름
    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

    // 현재 스레드가 트랜잭션 중인지 여부
    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
}
```

```
ThreadLocal 기반 트랜잭션 동기화:
  트랜잭션 시작 → 스레드로컬에 Connection 바인딩
  Repository/JPA → 같은 스레드에서 같은 Connection 재사용
  트랜잭션 종료 → 스레드로컬 정리

@Async 함정:
  새 스레드로 전환 → 스레드로컬 없음 → 새 트랜잭션
  → @Transactional + @Async 메서드 호출 주의
```

---

## 💻 실험으로 확인하기

### 실험 1: Self-Invocation 함정 확인

```java
@Service
public class OrderService {

    @Transactional
    public void outer() {
        System.out.println("outer TX: " +
            TransactionSynchronizationManager.getCurrentTransactionName());
        this.inner();  // Self-Invocation
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void inner() {
        System.out.println("inner TX: " +
            TransactionSynchronizationManager.getCurrentTransactionName());
        // outer와 같은 트랜잭션 이름 출력 → REQUIRES_NEW 무시됨
    }
}
```

### 실험 2: Checked Exception 롤백 기본 동작

```java
@Transactional
public void process() throws IOException {
    orderRepo.save(order);  // 저장
    throw new IOException("네트워크 오류");
    // 기본 설정: Checked Exception → 롤백 안 됨 → save가 커밋됨!
}

// 해결
@Transactional(rollbackFor = Exception.class)
public void process() throws IOException {
    orderRepo.save(order);
    throw new IOException("네트워크 오류");  // 이제 롤백됨
}
```

### 실험 3: TransactionInterceptor 적용 확인

```java
@Autowired OrderService orderService;

// TransactionInterceptor가 체인에 있는지 확인
Advised advised = (Advised) orderService;
boolean hasTransactionInterceptor = Arrays.stream(advised.getAdvisors())
    .anyMatch(a -> a.getAdvice() instanceof TransactionInterceptor);
System.out.println(hasTransactionInterceptor);  // true
```

---

## 🤔 트레이드오프

```
프록시 기반 @Transactional:
  장점  설정 간단, 선언적, 비즈니스 코드 분리
  단점  Self-Invocation 불가
       private 메서드 불가 (다음 문서에서 상세)
       프록시 오버헤드 (미미)

AspectJ 위빙 (컴파일/로드 타임):
  장점  Self-Invocation 가능, private 가능, 오버헤드 없음
  단점  빌드 도구 설정 복잡, 학습 곡선

Propagation 선택 가이드:
  REQUIRED      → 기본, 대부분의 비즈니스 메서드
  REQUIRES_NEW  → 독립 실행 필요 (감사 로그, 알림 등)
  NESTED        → 부분 롤백 (SavePoint)
  MANDATORY     → 트랜잭션 없이 호출하면 안 되는 내부 메서드

readOnly=true 최적화:
  JPA: FlushMode.MANUAL → 변경 감지(Dirty Checking) 비용 절약
  DB: read replica 라우팅 힌트 (DataSource routing 설정 시)
  → 읽기 전용 메서드에 항상 설정 권장
```

---

## 📌 핵심 정리

```
@EnableTransactionManagement 등록 경로
  ProxyTransactionManagementConfiguration
    → BeanFactoryTransactionAttributeSourceAdvisor 등록
    → Pointcut: TransactionAttributeSourcePointcut (@Transactional 감지)
    → Advice: TransactionInterceptor

TransactionInterceptor.invoke()
  txAttr = getTransactionAttribute(method)   @Transactional 속성 읽기
  ptm.getTransaction(txAttr)                 propagation 처리, 트랜잭션 시작
  invocation.proceed()                       실제 메서드 실행
  commitTransactionAfterReturning()          정상 → 커밋
  completeTransactionAfterThrowing()         예외 → rollbackOn() 판단 → 롤백/커밋

ThreadLocal 동기화
  트랜잭션 시작 → Connection을 스레드로컬에 바인딩
  같은 스레드의 Repository → 같은 Connection 재사용
  스레드 전환(@Async) → 트랜잭션 전파 안 됨

Checked Exception 기본값
  RuntimeException/Error → 롤백
  Checked Exception → 커밋 (주의!)
  rollbackFor로 재정의 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional(propagation = REQUIRES_NEW)` 메서드가 Self-Invocation으로 호출됐을 때의 동작을 TransactionInterceptor 실행 경로로 설명하라.

**Q2.** `@Transactional readOnly=true` 메서드 내에서 JPA `save()`를 호출하면 어떻게 되는가?

**Q3.** 두 개의 `@Transactional` 메서드 A(REQUIRED) → B(REQUIRES_NEW)가 서로 다른 Bean에 있을 때, B에서 예외가 발생하면 A의 트랜잭션은 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `this.sendNotification()`처럼 Self-Invocation이 발생하면 프록시를 거치지 않고 원본 객체의 메서드가 직접 호출된다. 따라서 `TransactionInterceptor.invoke()`가 실행되지 않는다. `createTransactionIfNecessary()`도 호출되지 않으므로 `REQUIRES_NEW`로 새 트랜잭션을 시작하는 로직 자체가 동작하지 않는다. 호출자의 트랜잭션이 이미 활성화돼 있으면 그 트랜잭션에 그대로 참여하고, 트랜잭션이 없으면 트랜잭션 없이 실행된다.
>
> **Q2.** `readOnly=true`는 JPA의 `FlushMode`를 `MANUAL`로 설정한다. `FlushMode.MANUAL`에서는 명시적으로 `flush()`를 호출하지 않는 한 영속성 컨텍스트의 변경 사항이 DB에 반영되지 않는다. `save()`는 JPA 영속성 컨텍스트에 엔티티를 등록하지만(`persist()`), 트랜잭션 커밋 시 자동 flush가 발생하지 않아 실제 INSERT SQL이 실행되지 않는다. 단, `saveAndFlush()`처럼 명시적 flush가 있으면 실행될 수 있다. 또한 DB 드라이버 수준의 `setReadOnly(true)` 힌트가 쓰기를 차단할 수도 있지만 이는 드라이버/DB 구현에 따라 다르다.
>
> **Q3.** B가 `REQUIRES_NEW`이므로 A의 트랜잭션은 일시 중단되고 B는 독립된 새 트랜잭션을 시작한다. B에서 예외가 발생하면 B의 트랜잭션이 롤백된다. 이 예외가 A로 전파되면 A의 `TransactionInterceptor`에서 `completeTransactionAfterThrowing()`이 호출돼 `rollbackOn(ex)`를 평가한다. `RuntimeException`이면 A도 롤백, Checked Exception이면 A는 커밋(기본값)된다. 따라서 B의 롤백과 A의 트랜잭션은 독립적이지만, B의 예외가 A로 전파되어 A도 롤백될 수 있다. B의 예외를 A에서 catch하면 A는 B의 실패와 무관하게 커밋 가능하다.

---

<div align="center">

**[⬅️ 이전: Advice 체인 실행 순서](./05-advice-chain-execution.md)** | **[다음: private 메서드에 AOP가 안 되는 이유 ➡️](./07-why-private-aop-fails.md)**

</div>

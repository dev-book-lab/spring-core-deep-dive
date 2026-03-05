# @Cacheable의 AOP 내부 구조 — CacheInterceptor와 @Transactional 패턴 재사용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Cacheable`이 AOP 프록시로 동작하는 과정이 `@Transactional`과 어떻게 동일한가?
- `CacheInterceptor`가 Advice 체인에 들어가는 정확한 경로는?
- `@Cacheable`의 `key` SpEL 표현식은 어느 시점에 어떻게 평가되는가?
- `@CachePut`과 `@CacheEvict`는 `@Cacheable`과 실행 순서가 어떻게 다른가?
- `@Cacheable(sync = true)`가 필요한 이유와 내부 동작은?

---

## 🔍 왜 이게 존재하는가

### 문제: 메서드 결과를 캐시하려면 호출마다 조건 분기가 필요하다

```java
// 캐시 없음 — 매번 DB 조회
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

// 수동 캐시 — 중복 코드, 캐시 키 관리, 조건 분기 직접 처리
public Order findOrder(Long id) {
    String key = "order:" + id;
    Order cached = cacheManager.getCache("orders").get(key, Order.class);
    if (cached != null) return cached;

    Order order = orderRepository.findById(id).orElseThrow();
    cacheManager.getCache("orders").put(key, order);
    return order;
}

// @Cacheable — AOP가 위 로직을 대신 처리
@Cacheable(cacheNames = "orders", key = "#id")
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
    // 캐시 히트 시 이 메서드 자체가 실행되지 않음
}
```

```
@Cacheable이 AOP로 구현된 이유:
  캐시 조회/저장 로직은 비즈니스 로직과 완전히 분리된 횡단 관심사
  → Advice로 메서드 실행 전후를 가로채 캐시 처리
  → @Transactional과 동일한 ProxyFactory + Interceptor 구조 재사용
  → CacheManager 교체만으로 Redis, Caffeine, EhCache 전환 가능
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Cacheable은 항상 메서드를 실행한 후 결과를 저장한다

```java
// ❌ 잘못된 이해
@Cacheable("orders")
public Order findOrder(Long id) {
    System.out.println("DB 조회 실행");
    return orderRepository.findById(id).orElseThrow();
}

// 두 번 호출:
findOrder(1L);  // "DB 조회 실행" 출력
findOrder(1L);  // "DB 조회 실행" 다시 출력? → ❌
```

```
✅ 실제:
  캐시 히트 시 메서드 자체가 호출되지 않음
  → CacheInterceptor.invoke() 에서 캐시 조회 → 값 있으면 즉시 반환
  → 프록시가 원본 메서드 호출을 건너뜀 (Around Advice 구조)

두 번째 호출: "DB 조회 실행" 출력 없음 → 캐시에서 반환
```

### Before: @CachePut은 @Cacheable과 같다

```java
// ❌ 잘못된 이해:
// "@CachePut도 캐시 히트 시 메서드를 건너뛴다"

// ✅ 실제:
// @CachePut: 항상 메서드를 실행하고, 결과를 캐시에 저장/갱신
// @Cacheable: 캐시 미스 시에만 메서드 실행, 결과 저장

// 사용 목적:
// @Cacheable → 읽기 (캐시 우선 조회)
// @CachePut  → 쓰기 (캐시 강제 갱신, 항상 실행)
// @CacheEvict → 삭제 (캐시 무효화)
```

---

## ✨ 올바른 이해와 사용

### After: @Transactional과 @Cacheable의 AOP 구조 비교

```
@Transactional 구조:
  @EnableTransactionManagement
    → BeanFactoryTransactionAttributeSourceAdvisor 등록
       Pointcut: AnnotationTransactionAttributeSource (@Transactional 탐색)
       Advice:   TransactionInterceptor (트랜잭션 시작/커밋/롤백)
    → AbstractAutoProxyCreator가 프록시 생성

@Cacheable 구조:
  @EnableCaching
    → BeanFactoryCacheOperationSourceAdvisor 등록
       Pointcut: AnnotationCacheOperationSource (@Cacheable/@CachePut/@CacheEvict 탐색)
       Advice:   CacheInterceptor (캐시 조회/저장/삭제)
    → AbstractAutoProxyCreator가 프록시 생성 (동일한 인프라 재사용)

1:1 대응:
  TransactionAttributeSource   ↔  CacheOperationSource
  TransactionInterceptor       ↔  CacheInterceptor
  PlatformTransactionManager   ↔  CacheManager
  TransactionStatus            ↔  Cache (캐시 저장소 추상화)
```

---

## 🔬 내부 동작 원리

### 1. @EnableCaching — 인프라 등록

```java
// @EnableCaching
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)  // ImportSelector
public @interface EnableCaching {
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
}

// CachingConfigurationSelector → ProxyCachingConfiguration 선택 (PROXY 모드)
// ProxyCachingConfiguration (AutoProxyRegistrar + BeanFactoryCacheOperationSourceAdvisor)

@Configuration(proxyBeanMethods = false)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

    // Advisor: Pointcut + Advice 묶음
    @Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
    public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor(
            CacheOperationSource cacheOperationSource,
            CacheInterceptor cacheInterceptor) {

        BeanFactoryCacheOperationSourceAdvisor advisor =
            new BeanFactoryCacheOperationSourceAdvisor();
        advisor.setCacheOperationSource(cacheOperationSource);  // Pointcut
        advisor.setAdvice(cacheInterceptor);                    // Advice
        advisor.setOrder(this.enableCaching.getNumber("order"));
        return advisor;
    }

    // Advice
    @Bean
    public CacheInterceptor cacheInterceptor(CacheOperationSource cacheOperationSource) {
        CacheInterceptor interceptor = new CacheInterceptor();
        interceptor.configure(this.errorHandler, this.keyGenerator,
            this.cacheResolver, this.cacheManager);
        interceptor.setCacheOperationSource(cacheOperationSource);
        return interceptor;
    }

    // Pointcut (메타데이터 소스)
    @Bean
    public CacheOperationSource cacheOperationSource() {
        return new AnnotationCacheOperationSource();
        // @Cacheable, @CachePut, @CacheEvict 어노테이션 파싱
    }
}
```

### 2. CacheInterceptor.invoke() — 캐시 처리 핵심

```java
// CacheInterceptor extends CacheAspectSupport implements MethodInterceptor
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();

        // 캐시 작업 소스에서 메서드의 캐시 어노테이션 파싱
        CacheOperationInvoker aopAllianceInvoker = () -> {
            try {
                return invocation.proceed();  // 원본 메서드 호출
            } catch (Throwable ex) {
                throw new CacheOperationInvoker.ThrowableWrapper(ex);
            }
        };

        Object target = invocation.getThis();
        // CacheAspectSupport.execute()로 실제 캐시 로직 위임
        return execute(aopAllianceInvoker, target, method, invocation.getArguments());
    }
}
```

### 3. CacheAspectSupport.execute() — @Cacheable / @CachePut / @CacheEvict 분기

```java
// CacheAspectSupport.execute() 핵심 흐름
protected Object execute(CacheOperationInvoker invoker, Object target,
                          Method method, Object[] args) {

    CacheOperationContexts contexts = getOperationContexts(target, method, args);

    // 1. @CacheEvict (beforeInvocation = true) — 메서드 실행 전 캐시 삭제
    processCacheEvicts(contexts.get(CacheEvictOperation.class), true, CacheOperationExpressionEvaluator.NO_RESULT);

    // 2. @Cacheable — 캐시 조회
    Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

    // 3. 캐시 히트? → 원본 메서드 건너뜀
    List<CachePutRequest> cachePutRequests = new ArrayList<>();
    if (cacheHit == null) {
        collectPutRequests(contexts.get(CacheableOperation.class),
            CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
    }

    Object cacheValue;
    Object returnValue;

    if (cacheHit != null && !hasCachePut(contexts)) {
        // 캐시 히트 → 원본 메서드 호출 없이 캐시값 반환
        cacheValue = cacheHit.get();
        returnValue = wrapCacheValue(method, cacheValue);
    } else {
        // 캐시 미스 또는 @CachePut → 원본 메서드 호출
        returnValue = invokeOperation(invoker);
        cacheValue = unwrapReturnValue(returnValue);
    }

    // 4. @CachePut — 항상 캐시 저장
    collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

    // 5. 캐시 저장 실행 (@Cacheable 미스 결과 + @CachePut 결과)
    for (CachePutRequest cachePutRequest : cachePutRequests) {
        cachePutRequest.apply(cacheValue);
    }

    // 6. @CacheEvict (beforeInvocation = false, 기본) — 메서드 실행 후 캐시 삭제
    processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

    return returnValue;
}
```

```
실행 순서 정리:
  @CacheEvict(beforeInvocation=true)  → 메서드 실행 전 삭제
  @Cacheable 캐시 조회
    히트 → 메서드 건너뜀, 캐시값 반환
    미스 → 메서드 실행
  @CachePut → 항상 메서드 실행, 결과 저장
  @CacheEvict(beforeInvocation=false) → 메서드 실행 후 삭제 (기본)
```

### 4. SpEL 캐시 키 평가 — CacheOperationExpressionEvaluator

```java
// @Cacheable(key = "#id")
// @Cacheable(key = "#order.customerId + ':' + #order.status")
// @Cacheable(condition = "#id > 0", unless = "#result == null")

// CacheOperationExpressionEvaluator
class CacheOperationExpressionEvaluator extends CachedExpressionEvaluator {

    // SpEL EvaluationContext 구성
    public EvaluationContext createEvaluationContext(Collection<? extends Cache> caches,
                                                      Method method, Object[] args,
                                                      Object target, Class<?> targetClass,
                                                      Method targetMethod,
                                                      Object result, ...) {

        CacheExpressionRootObject rootObject = new CacheExpressionRootObject(
            caches, method, args, target, targetClass);

        MethodBasedEvaluationContext evaluationContext =
            new MethodBasedEvaluationContext(rootObject, targetMethod, args, ...);

        // #result 변수 — unless에서 반환값 접근
        if (result == RESULT_UNAVAILABLE) {
            evaluationContext.addUnavailableVariable(RESULT_VARIABLE);
        } else {
            evaluationContext.setVariable(RESULT_VARIABLE, result);
        }
        return evaluationContext;
    }
}

// SpEL 컨텍스트에서 사용 가능한 변수:
// #root.method     → 메서드 객체
// #root.target     → 대상 객체
// #root.caches     → 캐시 컬렉션
// #root.args       → 인수 배열
// #root.methodName → 메서드 이름
// #파라미터명       → 메서드 파라미터 (예: #id, #order)
// #result          → 반환값 (unless, @CachePut에서 사용)
// #a0, #p0         → 첫 번째 파라미터 (인덱스 접근)
```

```java
// 키 표현식 예시
@Cacheable(cacheNames = "orders", key = "#id")
public Order findOrder(Long id) { ... }

@Cacheable(cacheNames = "orders", key = "#root.methodName + ':' + #id")
public Order findOrderByMethod(Long id) { ... }

@Cacheable(cacheNames = "products",
    condition = "#category != null",       // 실행 전 평가 (캐시 여부 결정)
    unless = "#result.stock == 0")         // 실행 후 평가 (#result 접근 가능)
public Product findProduct(String category) { ... }
```

### 5. @Cacheable(sync = true) — 동시성 문제 해결

```java
// 문제: 캐시 미스 시 여러 스레드가 동시에 원본 메서드 호출
// Thread A: 캐시 미스 → DB 조회 시작
// Thread B: 캐시 미스 → DB 조회 시작 (A가 아직 저장 전)
// Thread A: DB 조회 완료 → 캐시 저장
// Thread B: DB 조회 완료 → 캐시 저장 (중복 실행!)
// → "Cache Stampede" / "Thundering Herd" 문제

// 해결: sync = true
@Cacheable(cacheNames = "expensiveData", key = "#id", sync = true)
public ExpensiveData computeData(Long id) { ... }
// → 캐시 미스 시 단 하나의 스레드만 원본 메서드 실행
// → 나머지 스레드는 대기 → 첫 번째 스레드의 결과 재사용

// 내부 동작: CacheAspectSupport.findCachedItem() 분기
if (contexts.isSynchronized()) {
    // Cache.get(key, Callable) — Cache 구현체가 동기화 책임
    // Caffeine, Redis(Lettuce) 등 지원 여부 확인 필요
    return cache.get(generateKey(context), () -> invokeOperation(invoker));
}
```

### 6. @CacheEvict — 단건 vs 전체 삭제

```java
// 단건 삭제 (기본)
@CacheEvict(cacheNames = "orders", key = "#id")
public void deleteOrder(Long id) { ... }
// → "orders" 캐시에서 key="id값" 항목만 제거

// 전체 삭제
@CacheEvict(cacheNames = "orders", allEntries = true)
public void clearOrderCache() { ... }
// → "orders" 캐시 전체 비움

// 메서드 실행 전 삭제 (기본: 실행 후)
@CacheEvict(cacheNames = "orders", key = "#id", beforeInvocation = true)
public void updateOrder(Long id, Order order) { ... }
// beforeInvocation=true: 메서드 예외 발생해도 캐시 삭제됨
// beforeInvocation=false(기본): 메서드 성공 시에만 캐시 삭제

// 여러 어노테이션 조합
@Caching(
    put    = @CachePut(cacheNames = "orders", key = "#result.id"),
    evict  = @CacheEvict(cacheNames = "orderSummary", allEntries = true)
)
public Order createOrder(OrderRequest request) { ... }
```

### 7. Self-Invocation 함정 — @Transactional과 동일

```java
@Service
public class OrderService {

    // ❌ 같은 클래스 내부 호출 → 프록시 우회 → 캐시 미적용
    public Order processAndFind(Long id) {
        return findOrder(id);  // this.findOrder() → 프록시 아닌 원본 호출
    }

    @Cacheable(cacheNames = "orders", key = "#id")
    public Order findOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }

    // ✅ 해결 1: ApplicationContext.getBean() 으로 프록시 직접 획득
    // ✅ 해결 2: OrderService를 별도 Bean으로 분리
    // ✅ 해결 3: @EnableCaching의 mode=AspectJ (컴파일 타임 위빙)
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 캐시 히트 시 메서드 미실행 확인

```java
@Cacheable(cacheNames = "orders", key = "#id")
public Order findOrder(Long id) {
    System.out.println("DB 조회: " + id);
    return orderRepository.findById(id).orElseThrow();
}

// 테스트
findOrder(1L);  // 출력: "DB 조회: 1"
findOrder(1L);  // 출력 없음 → 캐시에서 반환
findOrder(2L);  // 출력: "DB 조회: 2" (다른 키)
findOrder(1L);  // 출력 없음 → 여전히 캐시
```

### 실험 2: @CachePut vs @Cacheable 차이

```java
@Cacheable(cacheNames = "orders", key = "#id")
public Order findOrder(Long id) {
    System.out.println("@Cacheable 실행: " + id);
    return orderRepository.findById(id).orElseThrow();
}

@CachePut(cacheNames = "orders", key = "#result.id")
public Order updateOrder(Order order) {
    System.out.println("@CachePut 실행: " + order.getId());
    return orderRepository.save(order);
}

findOrder(1L);   // "DB 조회: 1" → 캐시 저장
findOrder(1L);   // 출력 없음 → 캐시 히트
updateOrder(order1);  // "@CachePut 실행: 1" → 항상 실행, 캐시 갱신
findOrder(1L);   // 출력 없음 → 갱신된 캐시 히트
```

### 실험 3: SpEL 키 조합

```java
// 복합 키
@Cacheable(cacheNames = "orders",
    key = "#customerId + ':' + #status.name()")
public List<Order> findOrders(Long customerId, OrderStatus status) { ... }
// 키: "123:PENDING"

// unless로 null 결과 캐시 방지
@Cacheable(cacheNames = "orders",
    key = "#id",
    unless = "#result == null")
public Order findOptionalOrder(Long id) {
    return orderRepository.findById(id).orElse(null);
}
// null 반환 시 캐시에 저장하지 않음
```

---

## 🤔 트레이드오프

```
@Cacheable vs 수동 캐시:
  @Cacheable  선언적, 간결, AOP 인프라 재사용, Self-Invocation 함정
  수동 캐시   명시적, 조건 분기 자유롭지만 반복 코드 많음
  → 표준 CRUD 캐시: @Cacheable 권장
  → 복잡한 조건 / 부분 캐시: 수동 또는 커스텀 Advisor

sync = true:
  장점  Cache Stampede 방지 → DB 과부하 감소
  단점  CacheManager 구현체 지원 여부 확인 필요
       (Caffeine: 지원, 기본 ConcurrentMapCache: 미지원)
       단일 노드에서만 동기화 → 분산 환경에서 완전하지 않음

@CacheEvict beforeInvocation:
  false(기본)  메서드 성공 시에만 삭제 → 실패해도 캐시 유지 (안전)
  true         메서드 예외 시에도 삭제 → 강제 무효화 (더 공격적)

condition vs unless:
  condition  메서드 실행 전 평가 (#result 사용 불가)
             조건 false → 캐시 조회도 저장도 안 함
  unless     메서드 실행 후 평가 (#result 사용 가능)
             조건 true → 결과를 캐시에 저장하지 않음

@Transactional + @Cacheable 조합:
  @Cacheable → 캐시 히트 → @Transactional 메서드 호출 안 됨 → 트랜잭션 없음
  → 캐시 히트 경로에서 트랜잭션이 필요한 로직은 피할 것
  → Advice 체인 순서: @Transactional 먼저, @Cacheable 이후 (or반대, @Order 설정)
```

---

## 📌 핵심 정리

```
@Transactional과 동일한 AOP 구조
  @EnableCaching → BeanFactoryCacheOperationSourceAdvisor 등록
  Pointcut: AnnotationCacheOperationSource (@Cacheable 등 어노테이션 탐색)
  Advice: CacheInterceptor → CacheAspectSupport.execute() 위임

CacheAspectSupport.execute() 실행 순서
  @CacheEvict(beforeInvocation=true) → 캐시 조회(@Cacheable) →
  캐시 히트? → 원본 메서드 건너뜀
  캐시 미스? → 원본 메서드 실행 → 결과 저장 →
  @CachePut 저장 → @CacheEvict(beforeInvocation=false)

SpEL 키 평가
  CacheOperationExpressionEvaluator + MethodBasedEvaluationContext
  #파라미터명, #root.method, #result (unless/@CachePut)

@CachePut vs @Cacheable
  @Cacheable  캐시 미스 시에만 메서드 실행
  @CachePut   항상 메서드 실행 → 캐시 강제 갱신

Self-Invocation 함정
  @Transactional과 동일: 같은 클래스 내부 호출 → 프록시 우회 → 캐시 미적용
  → 별도 Bean 분리 또는 AspectJ 모드로 해결
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`과 `@Cacheable`이 같은 메서드에 모두 선언됐을 때 두 Interceptor의 실행 순서는 어떻게 결정되는가?

**Q2.** 캐시에서 반환된 객체를 수정하면 어떤 문제가 생기는가?

**Q3.** `@Cacheable(unless = "#result == null")`과 `@Cacheable(condition = "#result != null")`은 동작이 같은가?

> 💡 **해설**
>
> **Q1.** 두 `Advisor`(`BeanFactoryTransactionAttributeSourceAdvisor`, `BeanFactoryCacheOperationSourceAdvisor`)가 같은 프록시의 Advice 체인에 들어간다. 실행 순서는 `Ordered` 인터페이스의 `getOrder()` 값으로 결정된다. `@EnableTransactionManagement(order=)`와 `@EnableCaching(order=)`로 각각 지정할 수 있으며, 기본값은 `Ordered.LOWEST_PRECEDENCE`로 동일하다. 일반적으로 `@Transactional`을 외부(먼저 진입)에 두는 것이 권장된다. 캐시 히트 시 트랜잭션 시작 없이 캐시값 반환을 원하면 `@Cacheable`이 외부여야 하고, 캐시 조회도 트랜잭션 안에서 하려면 `@Transactional`이 외부여야 한다.
>
> **Q2.** `ConcurrentMapCache`(기본 인메모리 캐시)는 객체 참조를 그대로 저장한다. 반환된 객체를 수정하면 캐시에 저장된 객체도 함께 수정된다(동일 참조). 이후 캐시 히트 시 수정된 객체가 반환되어 데이터 오염이 발생한다. Redis, Caffeine 등은 직렬화/역직렬화 과정에서 새 객체를 생성하므로 이 문제가 없다. 안전한 패턴은 캐시에서 반환된 객체를 불변으로 설계하거나 방어적 복사를 사용하는 것이다.
>
> **Q3.** 다르다. `condition`은 메서드 실행 전에 평가되며, `condition = "#result != null"`은 `#result`를 사용하기 때문에 이 시점에 `RESULT_UNAVAILABLE`이 설정돼 `SpelEvaluationException`이 발생한다. `condition`에서는 `#result`를 사용할 수 없다. 반면 `unless`는 메서드 실행 후에 평가되므로 `#result`에 접근 가능하다. "결과가 null이면 캐시에 저장하지 않는다"는 의도라면 반드시 `unless = "#result == null"`을 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: Proxy 성능 비교](./08-proxy-performance.md)** | **[홈으로 🏠](../README.md)**

</div>

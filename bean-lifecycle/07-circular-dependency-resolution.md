# 순환 의존 내부 해결 과정 — 3단계 캐시 + ObjectFactory 코드 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `singletonObjects` / `earlySingletonObjects` / `singletonFactories` 세 캐시가 순환 참조에서 협력하는 전체 흐름은?
- `ObjectFactory` 람다가 `singletonFactories`에 등록되는 정확한 시점은?
- AOP 프록시가 있을 때 `earlySingletonObjects`가 반드시 필요한 이유는?
- `getSingleton(beanName, true)`와 `getSingleton(beanName, false)`의 차이는?
- 순환 참조가 해결된 후 최종 등록 과정은 어떻게 이루어지는가?

---

## 🔍 왜 이게 존재하는가

### Chapter 2-02와의 관계

> Chapter 2-02에서 3단계 캐시의 개념과 시나리오 흐름을 다뤘다.  
> 이 문서는 같은 주제를 **소스 코드 레벨**에서 메서드 호출 스택을 따라 완전히 추적한다.

```
이 문서의 목표:
  getSingleton(), doCreateBean(), addSingletonFactory(),
  getEarlyBeanReference() 각 메서드가 정확히 어떤 순서로
  어떤 인자를 가지고 호출되는지 코드로 확인

  "개념 이해" → "코드 추적"으로 내재화
```

---

## 🔬 내부 동작 원리 전체 추적

### 1. getSingleton() 두 버전 — 핵심 분기점

```java
// DefaultSingletonBeanRegistry.java

// 버전 1: allowEarlyReference = true (doGetBean에서 호출)
// → earlySingletonObjects / singletonFactories까지 탐색
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {

    // 1단계: singletonObjects (완성된 Bean)
    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // Bean이 없고 현재 생성 중인 경우에만 하위 캐시 탐색

        // 2단계: earlySingletonObjects (조기 노출 Bean)
        singletonObject = this.earlySingletonObjects.get(beanName);

        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                // DCL (Double-Checked Locking)
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {

                        // 3단계: singletonFactories (ObjectFactory 람다)
                        ObjectFactory<?> singletonFactory =
                            this.singletonFactories.get(beanName);

                        if (singletonFactory != null) {
                            // 람다 실행 → early reference 생성
                            singletonObject = singletonFactory.getObject();
                            // 2단계로 승격 (이후 동일 early ref 재사용)
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            // 3단계 제거 (2단계에 이미 있으므로)
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}

// 버전 2: 완성된 Singleton 생성 흐름 (createBean 호출 포함)
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {

    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);

        if (singletonObject == null) {
            beforeSingletonCreation(beanName);
            // singletonsCurrentlyInCreation.add(beanName)

            try {
                singletonObject = singletonFactory.getObject();
                // → doCreateBean() 실행
            } finally {
                afterSingletonCreation(beanName);
                // singletonsCurrentlyInCreation.remove(beanName)
            }

            // 1단계 캐시 등록
            addSingleton(beanName, singletonObject);
        }
        return singletonObject;
    }
}

protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);      // 1단계 등록
        this.singletonFactories.remove(beanName);                  // 3단계 제거
        this.earlySingletonObjects.remove(beanName);               // 2단계 제거
        this.registeredSingletons.add(beanName);
    }
}
```

### 2. addSingletonFactory() — 3단계 캐시 등록 시점

```java
// AbstractAutowireCapableBeanFactory.doCreateBean()
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {

    // 인스턴스 생성 완료 (필드는 아직 null)
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();

    // ── 3단계 캐시 등록 ────────────────────────────────────────────
    boolean earlySingletonExposure = (
        mbd.isSingleton()
        && this.allowCircularReferences
        && isSingletonCurrentlyInCreation(beanName)
        // singletonsCurrentlyInCreation에 있는지 → getSingleton(버전2) 시 등록됨
    );

    if (earlySingletonExposure) {
        addSingletonFactory(beanName,
            () -> getEarlyBeanReference(beanName, mbd, bean));
        // ObjectFactory 람다 등록
        // 람다는 호출 시(getSingleton 3단계 탐색)에 getEarlyBeanReference() 실행
    }

    // 주입 + 초기화
    populateBean(beanName, mbd, instanceWrapper);
    Object exposedObject = initializeBean(beanName, bean, mbd);

    // ── 순환 참조 후처리 ───────────────────────────────────────────
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        // false: singletonFactories 탐색 안 함 (이미 2단계로 승격됐으면 반환)

        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                // BPP After에서 프록시로 교체 안 됐으면
                // → early ref와 최종 객체를 통일
                exposedObject = earlySingletonReference;
            } else {
                // BPP After에서 프록시로 교체됐는데
                // 누군가 이미 원본(early ref)을 주입받았다면 → 불일치 경고
                if (hasDependentBean(beanName)) {
                    logger.warn("...");
                }
            }
        }
    }

    return exposedObject;
}
```

### 3. getEarlyBeanReference() — AOP 프록시 조기 생성

```java
// AbstractAutowireCapableBeanFactory.getEarlyBeanReference()
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;

    if (!mbd.isSynthetic()) {
        for (SmartInstantiationAwareBeanPostProcessor bp
                : getBeanPostProcessorCache().smartInstantiationAware) {

            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
            // AbstractAutoProxyCreator.getEarlyBeanReference():
            //   @Transactional 등 AOP 대상이면 → 프록시 생성 후 반환
            //   대상 아니면 → bean 그대로 반환
        }
    }
    return exposedObject;
}
```

```
getEarlyBeanReference() 역할:

AOP 없는 경우:
  () -> bean  (원본 그대로)
  → 2단계: 원본 객체

AOP 있는 경우 (@Transactional):
  () -> AbstractAutoProxyCreator.getEarlyBeanReference(bean)
       → 프록시 생성 (BPP After보다 먼저!)
  → 2단계: 프록시 객체

이유:
  B가 A를 주입받을 때 A의 early ref를 가져감
  A가 나중에 BPP After에서 프록시로 교체되면
  B는 원본을 갖고 있는 상황 → 불일치
  → early ref 단계부터 프록시로 만들어 B에 전달
  → B, 컨테이너 모두 같은 프록시를 바라봄
```

### 4. 전체 흐름 — A(@Transactional) ↔ B 순환 참조 완전 추적

```
초기 상태:
  singletonObjects:    {}
  earlySingletonObjects: {}
  singletonFactories:  {}
  singletonsCurrentlyInCreation: {}

─────────────────────────────────────────────────────────
STEP 1: getBean("a") 호출
─────────────────────────────────────────────────────────
getSingleton("a", true)
  → singletonObjects["a"] = null
  → isSingletonCurrentlyInCreation("a") = false
  → 하위 캐시 탐색 안 함 → null

getSingleton("a", ObjectFactory) 호출
  beforeSingletonCreation("a")
  singletonsCurrentlyInCreation: {"a"}

─────────────────────────────────────────────────────────
STEP 2: doCreateBean("a") 실행
─────────────────────────────────────────────────────────
createBeanInstance("a") → new A() (b 필드 null)

earlySingletonExposure = true
  (isSingleton + allowCircularReferences + inCreation)

addSingletonFactory("a",
  () -> getEarlyBeanReference("a", mbd, rawA))
  → A는 @Transactional → AbstractAutoProxyCreator가 프록시 예약

singletonFactories: {"a": () -> proxyOf(rawA)}

─────────────────────────────────────────────────────────
STEP 3: populateBean("a") → B 필요 → getBean("b")
─────────────────────────────────────────────────────────
getSingleton("b", true) → null (아직 없음)

getSingleton("b", ObjectFactory) 호출
  beforeSingletonCreation("b")
  singletonsCurrentlyInCreation: {"a", "b"}

─────────────────────────────────────────────────────────
STEP 4: doCreateBean("b") 실행
─────────────────────────────────────────────────────────
createBeanInstance("b") → new B() (a 필드 null)

addSingletonFactory("b", () -> rawB)
  (B는 AOP 대상 아님 → 원본 그대로)

singletonFactories: {"a": λ_proxyA, "b": λ_rawB}

─────────────────────────────────────────────────────────
STEP 5: populateBean("b") → A 필요 → getBean("a")
─────────────────────────────────────────────────────────
getSingleton("a", true)
  singletonObjects["a"] = null
  isSingletonCurrentlyInCreation("a") = true  ← 생성 중!
  earlySingletonObjects["a"] = null
  singletonFactories["a"] = λ_proxyA  ← 3단계 발견!

  λ_proxyA.getObject()
    → getEarlyBeanReference("a", mbd, rawA)
    → AbstractAutoProxyCreator → CGLIB 프록시 생성 (proxyA)

  earlySingletonObjects["a"] = proxyA  ← 2단계 승격
  singletonFactories.remove("a")

  → proxyA 반환

캐시 상태:
  singletonObjects:    {}
  earlySingletonObjects: {"a": proxyA}
  singletonFactories:  {"b": λ_rawB}

─────────────────────────────────────────────────────────
STEP 6: B의 a 필드에 proxyA 주입 → B 초기화 완료
─────────────────────────────────────────────────────────
initializeBean("b", rawB) → rawB 그대로 (AOP 없음)

addSingleton("b", rawB)
  singletonObjects["b"] = rawB
  singletonFactories.remove("b")
  earlySingletonObjects.remove("b")

singletonsCurrentlyInCreation: {"a"}

캐시 상태:
  singletonObjects:    {"b": rawB}
  earlySingletonObjects: {"a": proxyA}
  singletonFactories:  {}

─────────────────────────────────────────────────────────
STEP 7: A의 b 필드에 rawB 주입 → A 초기화
─────────────────────────────────────────────────────────
A.b = rawB  (populateBean 완료)

initializeBean("a", rawA)
  BPP After → AbstractAutoProxyCreator
    이미 early ref에서 proxyA 생성됨 → createdProxies에 마킹
    → 같은 프록시 재사용 (새 프록시 생성 안 함)
    → BPP After도 proxyA 반환

exposedObject = proxyA

─────────────────────────────────────────────────────────
STEP 8: 후처리 — 일관성 확인
─────────────────────────────────────────────────────────
getSingleton("a", false)  ← false: singletonFactories 탐색 안 함
  earlySingletonObjects["a"] = proxyA  → proxyA 반환

earlySingletonReference = proxyA
exposedObject (BPP After 결과) = proxyA
exposedObject == proxyA  ← 일치 (원본/프록시 불일치 없음)

addSingleton("a", proxyA)
  singletonObjects["a"] = proxyA
  earlySingletonObjects.remove("a")

singletonsCurrentlyInCreation: {}

최종 캐시 상태:
  singletonObjects: {"a": proxyA, "b": rawB}
  earlySingletonObjects: {}
  singletonFactories:  {}

결과:
  B.a → proxyA (CGLIB 프록시) ✅
  컨테이너["a"] → proxyA ✅ (B와 동일한 프록시)
```

### 5. allowCircularReferences 플래그

```java
// AbstractAutowireCapableBeanFactory
private boolean allowCircularReferences = true;  // 기본값

// Spring Boot 2.6+ 기본값 변경
// SpringApplication 내부:
public class SpringApplication {
    private void configureApplication(ConfigurableApplicationContext context) {
        if (!this.allowCircularReferences) {
            ((AbstractAutowireCapableBeanFactory) beanFactory)
                .setAllowCircularReferences(false);
        }
    }
}

// application.yml
// spring.main.allow-circular-references: true  ← 명시적 허용
```

```
allowCircularReferences = false 시:
  earlySingletonExposure = false
  → addSingletonFactory() 미실행
  → 3단계 캐시 미등록
  → 순환 참조 발생 시 singletonsCurrentlyInCreation 감지
  → BeanCurrentlyInCreationException 즉시 발생
  → 순환 참조를 숨기지 않고 설계 개선 유도
```

---

## 💻 실험으로 확인하기

### 실험 1: 캐시 상태 로그로 추적

```yaml
# application.yml
logging:
  level:
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry: TRACE
    org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory: DEBUG
```

```
TRACE 출력 예시:
Creating shared instance of singleton bean 'a'
Creating shared instance of singleton bean 'b'
Returning cached instance of singleton bean 'a'  ← earlySingletonObjects에서 반환
```

### 실험 2: AOP 있는 순환 참조 — 프록시 일치 확인

```java
@Transactional @Service class A { @Autowired B b; }
@Service         class B { @Autowired A a; }

// 컨텍스트 로드 후
A a = ctx.getBean(A.class);
B b = ctx.getBean(B.class);

System.out.println(a.getClass().getSimpleName());  // A$$SpringCGLIB
System.out.println(b.a.getClass().getSimpleName()); // A$$SpringCGLIB
System.out.println(a == b.a);  // true — 동일한 프록시 인스턴스
```

### 실험 3: allowCircularReferences=false 효과

```yaml
spring:
  main:
    allow-circular-references: false  # Boot 2.6+ 기본값
```

```java
@Service class A { @Autowired B b; }
@Service class B { @Autowired A a; }

// 컨텍스트 시작 시:
// The dependencies of some of the beans form a cycle:
// a -> b -> a
// BeanCurrentlyInCreationException
```

---

## 🤔 트레이드오프

```
3단계 캐시 메커니즘:
  장점
    필드/세터 주입 순환 참조 자동 해결
    AOP 프록시와의 불일치 방지
  단점
    이해하기 복잡 (디버깅 어려움)
    잘못 설계된 코드를 숨김

Spring Boot 2.6+ 기본값(false) 의도:
  순환 참조 = 설계 결함 신호
  기본으로 허용하면 개발자가 인지 못하고 방치
  → false로 강제하여 설계 개선 유도
  → 필요 시 명시적으로 true 설정하도록

순환 참조 해소 패턴:
  1. 공통 의존성 추출 (새 서비스 클래스)
  2. 이벤트 기반 분리 (ApplicationEvent)
  3. @Lazy로 초기화 시점 분리
  4. 인터페이스 도입으로 의존 방향 역전
```

---

## 📌 핵심 정리

```
getSingleton(name, allowEarlyReference)
  1단계: singletonObjects
  2단계: earlySingletonObjects (생성 중일 때만)
  3단계: singletonFactories.getObject() → 2단계로 승격

addSingletonFactory() 등록 시점
  doCreateBean() → createBeanInstance() 직후
  singletonsCurrentlyInCreation에 있는 경우

getEarlyBeanReference()
  AOP 없음: 원본 bean 반환
  AOP 있음: 프록시 미리 생성 → 2단계에 프록시 저장

후처리 (doCreateBean 마지막)
  getSingleton(name, false) → 2단계에서 early ref 조회
  exposedObject와 early ref 일치 확인
  addSingleton() → 1단계 등록, 2,3단계 제거

allowCircularReferences = false
  earlySingletonExposure = false
  순환 참조 시 BeanCurrentlyInCreationException 즉시
  Spring Boot 2.6+ 기본값
```

---

## 🤔 생각해볼 문제

**Q1.** `getSingleton(beanName, false)`와 `getSingleton(beanName, true)`의 차이를 순환 참조 후처리 관점에서 설명하라. `doCreateBean()` 마지막에 `false`로 호출하는 이유는?

**Q2.** A → B → C → A 형태의 3개 Bean 순환 참조에서, C가 A를 요청하는 시점의 캐시 상태는 어떻게 되는가?

**Q3.** `AbstractAutoProxyCreator.getEarlyBeanReference()`에서 프록시를 생성한 후, BPP After에서 같은 Bean에 대해 프록시가 다시 생성되지 않는 이유는?

> 💡 **해설**
>
> **Q1.** `getSingleton(name, true)`는 3단계 캐시(`singletonFactories`)까지 탐색하고 `ObjectFactory.getObject()`를 실행해 early ref를 2단계로 승격시킨다. `getSingleton(name, false)`는 1·2단계만 탐색하고 3단계는 탐색하지 않는다. `doCreateBean()` 마지막에 `false`로 호출하는 이유는 3단계 재탐색을 막기 위해서다. 이 시점에는 이미 2단계에 early ref가 올라와 있거나(순환 참조가 발생한 경우) 아예 없다(순환 참조 없는 경우). `false`로 호출해 2단계에 있는 early ref를 조회함으로써, 최종 `exposedObject`와 B가 주입받은 early ref가 같은 객체(프록시)인지 확인하는 일관성 검사를 수행한다.
>
> **Q2.** C가 A를 요청하는 시점에서 A, B, C 모두 `singletonsCurrentlyInCreation`에 있다. `singletonFactories`에는 A, B, C의 `ObjectFactory`가 모두 등록된 상태다(C는 방금 자신의 `addSingletonFactory()`를 호출했고 아직 populate 중). `getSingleton("a", true)` 호출 시 1단계 없음 → 2단계 없음 → 3단계에서 A의 람다 실행 → early ref 생성 → A를 2단계로 승격 → 3단계에서 A 제거. C의 a 필드에 A의 early ref 주입. 이후 C 완성, B 완성, A 완성 순으로 1단계 등록.
>
> **Q3.** `AbstractAutoProxyCreator`는 `earlyProxyReferences`라는 Map을 내부에 유지한다. `getEarlyBeanReference()`에서 프록시를 생성할 때 `earlyProxyReferences.put(cacheKey, bean)`으로 원본 Bean을 기록한다. BPP After(`wrapIfNecessary()`)에서 동일 Bean이 들어오면 `earlyProxyReferences.remove(cacheKey)`로 이미 처리됐음을 확인하고 새 프록시 생성 없이 기존 프록시를 반환한다. 이로써 순환 참조 상황에서 동일 Bean에 대해 두 개의 서로 다른 프록시가 생성되는 것을 방지한다.

---

<div align="center">

**[⬅️ 이전: FactoryBean vs ObjectProvider](./06-factorybean-vs-objectprovider.md)** | **[홈으로 🏠](../README.md)**

</div>

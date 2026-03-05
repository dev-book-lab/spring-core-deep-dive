# BeanDefinition 등록 과정 — 스캔 결과가 레지스트리에 쌓이는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 스캔된 `ScannedGenericBeanDefinition`이 `BeanDefinitionRegistry`에 등록되기까지 거치는 단계는?
- `BeanDefinition`에 `@Lazy`, `@Primary`, `@DependsOn`, `@Role`이 어떻게 반영되는가?
- 중복 등록 충돌 해결 규칙 — 같은 이름의 Bean이 두 번 등록되면 어떻게 되는가?
- `BeanDefinitionRegistry`와 `DefaultListableBeanFactory`의 관계는?
- Bean 이름 생성 규칙 — `@Component("customName")` 없을 때 기본 이름은?

---

## 🔍 왜 이게 존재하는가

### 문제: 스캔된 클래스 메타데이터를 컨테이너가 이해할 수 있는 형태로 변환해야 한다

```
스캔 결과: ScannedGenericBeanDefinition
  → beanClassName: "com.example.service.OrderService" (문자열)
  → 어노테이션 메타데이터: @Service, @Transactional, @Lazy 등

컨테이너가 필요한 것:
  → BeanDefinitionRegistry에 이름으로 등록
  → Scope, 지연 초기화 여부, 의존성 순서 등 설정 완료
  → 충돌 감지 및 해결

이 변환과 등록 과정을 doScan() 내부에서 처리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 같은 이름의 Bean이 두 번 등록되면 항상 예외가 발생한다

```java
// ❌ 잘못된 이해:
// "중복 Bean 이름 → BeanDefinitionStoreException 항상 발생"

// ✅ 실제:
// 중복 등록 결과는 상황에 따라 다름:

// 케이스 1: 완전히 같은 클래스 → 조용히 무시 (호환 가능)
// 케이스 2: 다른 클래스, 같은 이름 → ConflictingBeanDefinitionException
// 케이스 3: XML과 어노테이션 중복 → allowBeanDefinitionOverriding 설정에 따라
// 케이스 4: Spring Boot 기본 → allowBeanDefinitionOverriding=false
//           → 어떤 중복이든 예외 발생
```

### Before: BeanDefinition 등록 = Bean 인스턴스 생성

```
❌ 잘못된 이해:
  "BeanDefinitionRegistry.registerBeanDefinition() 호출 시 Bean 객체가 생성된다"

✅ 실제:
  BeanDefinition 등록 = 설계도(Blueprint) 저장
  Bean 인스턴스 생성 = 나중에 getBean() 또는 refresh() 완료 단계에서

  등록: refresh() → invokeBeanFactoryPostProcessors()
  생성: refresh() → finishBeanFactoryInitialization()
  두 단계는 시간상 완전히 분리됨
```

---

## ✨ 올바른 이해와 사용

### After: 등록 파이프라인을 단계별로 파악

```
ScannedGenericBeanDefinition 생성
  ↓ postProcessBeanDefinition()
AbstractBeanDefinition 기본값 설정
  (autowireMode, dependencyCheck, initMethodName 등)
  ↓ processCommonDefinitionAnnotations()
어노테이션 기반 추가 설정
  @Lazy    → setLazyInit(true)
  @Primary → setPrimary(true)
  @DependsOn → setDependsOn(...)
  @Role    → setRole(...)
  @Description → setDescription(...)
  ↓ generateBeanName()
Bean 이름 결정
  ↓ checkCandidate()
중복 충돌 확인
  ↓ applyScopedProxyMode()
Scope Proxy 필요 시 프록시 BeanDefinition으로 래핑
  ↓ registerBeanDefinition()
DefaultListableBeanFactory.beanDefinitionMap에 저장
```

---

## 🔬 내부 동작 원리

### 1. postProcessBeanDefinition() — AbstractBeanDefinition 기본값 설정

```java
// ClassPathBeanDefinitionScanner.postProcessBeanDefinition()
protected void postProcessBeanDefinition(AbstractBeanDefinition abd, String beanName) {

    // AbstractBeanDefinition 기본값 적용
    abd.applyDefaults(this.beanDefinitionDefaults);
    // → lazyInit, autowireMode, dependencyCheck,
    //   initMethodName, destroyMethodName 기본값

    // Qualifier 처리 (있는 경우)
    if (this.autowireCandidatePatterns != null) {
        abd.setAutowireCandidate(
            PatternMatchUtils.simpleMatch(this.autowireCandidatePatterns, beanName));
    }
}
```

### 2. processCommonDefinitionAnnotations() — 어노테이션 → BeanDefinition 속성

```java
// AnnotationConfigUtils.processCommonDefinitionAnnotations()
public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
    processCommonDefinitionAnnotations(abd, abd.getMetadata());
}

static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd,
                                                AnnotatedTypeMetadata metadata) {
    // @Lazy
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    } else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }

    // @Primary
    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }

    // @DependsOn
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }

    // @Role
    AnnotationAttributes role = attributesFor(metadata, Role.class);
    if (role != null) {
        abd.setRole(role.getNumber("value").intValue());
        // BeanDefinition.ROLE_APPLICATION = 0 (일반 Bean)
        // BeanDefinition.ROLE_SUPPORT     = 1 (보조 Bean)
        // BeanDefinition.ROLE_INFRASTRUCTURE = 2 (인프라 Bean)
    }

    // @Description
    AnnotationAttributes description = attributesFor(metadata, Description.class);
    if (description != null) {
        abd.setDescription(description.getString("value"));
    }
}
```

### 3. Bean 이름 생성 — AnnotationBeanNameGenerator

```java
// AnnotationBeanNameGenerator.generateBeanName()
public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {

    if (definition instanceof AnnotatedBeanDefinition abd) {
        // 어노테이션에서 명시적 이름 확인
        String beanName = determineBeanNameFromAnnotation(abd);
        if (StringUtils.hasText(beanName)) {
            return beanName;  // @Component("customName") → "customName"
        }
    }

    // 기본 이름 생성
    return buildDefaultBeanName(definition, registry);
}

// 어노테이션에서 이름 추출
protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
    AnnotationMetadata amd = annotatedDef.getMetadata();

    // @Component, @Service, @Repository 등에서 value 속성 확인
    Set<String> types = amd.getAnnotationTypes();
    String beanName = null;

    for (String type : types) {
        AnnotationAttributes attributes = AnnotationConfigUtils
            .attributesFor(amd, type);
        if (attributes != null
                && isStereotypeWithNameValue(type, amd.getMetaAnnotationTypes(type), attributes)) {
            Object value = attributes.get("value");
            if (value instanceof String strVal && !strVal.isEmpty()) {
                if (beanName != null && !strVal.equals(beanName)) {
                    throw new IllegalStateException("...");  // 충돌
                }
                beanName = strVal;
            }
        }
    }
    return beanName;
}

// 기본 이름 생성
protected String buildDefaultBeanName(BeanDefinition definition) {
    String beanClassName = definition.getBeanClassName();
    String shortClassName = ClassUtils.getShortName(beanClassName);
    // "com.example.service.OrderService" → "OrderService"

    return Introspector.decapitalize(shortClassName);
    // "OrderService" → "orderService" (첫 글자 소문자)

    // 예외: 첫 두 글자가 모두 대문자면 그대로 유지
    // "URLParser" → "URLParser" (decapitalize 규칙)
}
```

```
Bean 이름 생성 규칙:
  @Component("myBean")   → "myBean"       (명시적)
  @Component             → "orderService" (클래스명 첫 글자 소문자)
  @Service               → "orderService" (동일)

  예외:
  "URLParser"  → "URLParser"  (첫 두 글자 대문자 → 그대로)
  "HTMLParser" → "HTMLParser" (동일)
  "xmlParser"  → "xmlParser"  (이미 소문자 시작)
```

### 4. checkCandidate() — 중복 충돌 확인

```java
// ClassPathBeanDefinitionScanner.checkCandidate()
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) {

    // 레지스트리에 같은 이름이 없으면 → 통과
    if (!this.registry.containsBeanDefinition(beanName)) {
        return true;
    }

    // 같은 이름의 기존 BeanDefinition 조회
    BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);
    BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();
    if (originatingDef != null) {
        existingDef = originatingDef;
    }

    // 호환 가능한지 확인
    if (isCompatible(beanDefinition, existingDef)) {
        return false;  // 조용히 건너뜀 (중복이지만 호환 가능)
    }

    // 호환 불가 → 예외
    throw new ConflictingBeanDefinitionException(
        "Annotation-specified bean name '" + beanName +
        "' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with " +
        "existing, non-compatible bean definition of same name and class [" +
        existingDef.getBeanClassName() + "]");
}

// isCompatible(): 같은 클래스면 호환 가능
protected boolean isCompatible(BeanDefinition newDef, BeanDefinition existingDef) {
    return (!(existingDef instanceof ScannedGenericBeanDefinition)
        // 스캔된 것이 아닌 경우 (수동 등록) → 호환으로 처리 (수동이 우선)
        || (newDef.getSource() != null
            && newDef.getSource().equals(existingDef.getSource()))
        // 같은 소스에서 왔으면 → 호환
        || newDef.equals(existingDef));
        // 완전히 동일한 정의 → 호환
}
```

```
중복 등록 케이스별 결과:

케이스 1: 완전히 같은 클래스, 같은 이름
  isCompatible() = true → 조용히 무시 (기존 유지)

케이스 2: 다른 클래스, 같은 이름
  isCompatible() = false → ConflictingBeanDefinitionException

케이스 3: 스캔된 Bean vs 수동 등록(XML/@Bean) 충돌
  existingDef가 ScannedGenericBeanDefinition 아님
  → isCompatible() = true → 수동 등록이 우선 유지

Spring Boot (allowBeanDefinitionOverriding=false):
  DefaultListableBeanFactory.registerBeanDefinition()에서
  중복 시 BeanDefinitionOverrideException (더 강한 정책)
```

### 5. BeanDefinitionRegistry 구현 — DefaultListableBeanFactory

```java
// DefaultListableBeanFactory implements BeanDefinitionRegistry
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {

    // ... 유효성 검사 ...

    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);

    if (existingDefinition != null) {
        // allowBeanDefinitionOverriding = false (Spring Boot 기본)
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        // allowBeanDefinitionOverriding = true → 덮어쓰기 허용
        this.beanDefinitionMap.put(beanName, beanDefinition);
    } else {
        // 새 등록
        if (isAlreadyInCreation()) {
            // 이미 생성 중인 컨텍스트에서 등록 → 별도 처리 (동적 등록)
            this.beanDefinitionMap.put(beanName, beanDefinition);
        } else {
            // 일반 등록 경로
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // Bean 이름 순서 목록 업데이트
            List<String> updatedDefinitions = new ArrayList<>(
                this.beanDefinitionNames.size() + 1);
            updatedDefinitions.addAll(this.beanDefinitionNames);
            updatedDefinitions.add(beanName);
            this.beanDefinitionNames = updatedDefinitions;
        }
    }

    // singletonObjects에 이미 인스턴스가 있으면 제거 (재생성 유도)
    resetBeanDefinition(beanName);
}
```

```
핵심 저장소:
  beanDefinitionMap: Map<String, BeanDefinition>
    → 이름 → BeanDefinition 매핑 (ConcurrentHashMap)
  beanDefinitionNames: List<String>
    → 등록 순서 보장 (의존성 해결 시 순서 참조)
  manualSingletonNames: Set<String>
    → registerSingleton()으로 직접 등록된 인스턴스 이름
```

### 6. Scope Proxy 래핑

```java
// AnnotationConfigUtils.applyScopedProxyMode()
static BeanDefinitionHolder applyScopedProxyMode(
        ScopeMetadata metadata, BeanDefinitionHolder definition,
        BeanDefinitionRegistry registry) {

    ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();

    if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
        return definition;  // 프록시 없음
    }

    boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
    // TARGET_CLASS → CGLIB 프록시
    // INTERFACES   → JDK Proxy

    // 원본 BeanDefinition을 "scopedTarget.beanName"으로 재등록
    // Scope Proxy BeanDefinition을 원래 이름으로 등록
    return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: BeanDefinition 상세 정보 확인

```java
ConfigurableListableBeanFactory bf = ctx.getBeanFactory();
BeanDefinition bd = bf.getBeanDefinition("orderService");

System.out.println("ClassName:   " + bd.getBeanClassName());
System.out.println("Scope:       " + bd.getScope());
System.out.println("LazyInit:    " + bd.isLazyInit());
System.out.println("Primary:     " + bd.isPrimary());
System.out.println("DependsOn:   " + Arrays.toString(bd.getDependsOn()));
System.out.println("Role:        " + bd.getRole());
System.out.println("Is Scanned:  " + (bd instanceof ScannedGenericBeanDefinition));
```

### 실험 2: 등록 순서 확인

```java
DefaultListableBeanFactory bf =
    (DefaultListableBeanFactory) ctx.getAutowireCapableBeanFactory();

// beanDefinitionNames: 등록 순서 반영
System.out.println("등록 순서:");
bf.getBeanDefinitionNames();
// 스캔 순서 = 패키지명 알파벳 순 → 파일시스템 순서에 따라 다를 수 있음
```

### 실험 3: allowBeanDefinitionOverriding 확인

```yaml
# application.yml (Spring Boot)
spring:
  main:
    allow-bean-definition-overriding: true  # 기본 false → 덮어쓰기 허용
```

---

## 🤔 트레이드오프

```
allowBeanDefinitionOverriding:
  true  → 나중에 등록된 BeanDefinition이 덮어씀 → 유연하지만 버그 위험
  false → 중복 즉시 예외 → 안전하지만 의도적 오버라이드 불가
  Spring Boot 기본 false → 명시적 의도 없는 중복 방지

Bean 이름 충돌 방지:
  @Component("명시적이름") 사용
  또는 패키지 구조로 자연스러운 이름 분리 (같은 이름 다른 패키지 가능)
  멀티모듈에서 이름 충돌 → @ComponentScan + excludeFilter로 제어

등록 순서:
  beanDefinitionNames 리스트가 순서 보장
  @DependsOn으로 명시적 의존성 순서 제어 가능
  같은 타입 여러 Bean → 순서가 @Autowired List<T> 주입에 영향

Scope Proxy:
  @Scope(proxyMode=TARGET_CLASS) → 두 BeanDefinition 등록
  원본: "scopedTarget.beanName"
  프록시: "beanName" (실제 주입되는 것)
```

---

## 📌 핵심 정리

```
등록 파이프라인
  ScannedGenericBeanDefinition 생성
  → postProcessBeanDefinition() : AbstractBeanDefinition 기본값
  → processCommonDefinitionAnnotations() : @Lazy/@Primary/@DependsOn/@Role
  → generateBeanName() : 이름 결정
  → checkCandidate() : 충돌 확인
  → applyScopedProxyMode() : Scope Proxy 래핑
  → registerBeanDefinition() : beanDefinitionMap 저장

Bean 이름 규칙
  @Component("name") → "name"
  기본 → 클래스 short name 첫 글자 소문자 (Introspector.decapitalize)
  URLParser → "URLParser" (두 글자 이상 대문자 시작 시 그대로)

중복 등록 규칙
  같은 클래스 → isCompatible() = true → 조용히 무시
  다른 클래스 → ConflictingBeanDefinitionException
  Spring Boot 기본 → allowBeanDefinitionOverriding=false → 더 강한 정책

BeanDefinitionRegistry 핵심 저장소
  DefaultListableBeanFactory.beanDefinitionMap: Map<String, BeanDefinition>
  beanDefinitionNames: List<String> (등록 순서)
```

---

## 🤔 생각해볼 문제

**Q1.** 스캔으로 등록된 `@Component` Bean과 `@Bean` 메서드로 등록된 Bean이 같은 이름을 가질 때 어떤 규칙이 적용되는가?

**Q2.** `@DependsOn("anotherBean")`이 BeanDefinition에 저장된 후 실제 Bean 생성 시 어떻게 활용되는가?

**Q3.** `beanDefinitionNames` 리스트의 순서가 `@Autowired List<SomeInterface>`로 주입받을 때 어떤 의미를 갖는가?

> 💡 **해설**
>
> **Q1.** `checkCandidate()`의 `isCompatible()` 메서드에서 기존 BeanDefinition이 `ScannedGenericBeanDefinition`이 아니면(`@Bean` 메서드로 등록된 것) 호환 가능으로 처리해 새로 스캔된 것을 건너뛴다. 즉 `@Bean` 메서드로 명시적으로 등록된 Bean이 스캔된 Bean보다 우선한다. 이 규칙은 직접 `@Bean`으로 커스터마이즈한 Bean이 스캔으로 덮어씌워지지 않도록 보호한다. `allowBeanDefinitionOverriding=false`(Spring Boot 기본)에서는 레지스트리 레벨에서 더 강하게 예외를 던진다.
>
> **Q2.** `DefaultSingletonBeanRegistry.getSingleton()`에서 Bean 생성 시 `AbstractBeanFactory.doGetBean()`이 `dependsOn` 배열을 확인해 의존 Bean들을 먼저 `getBean()`으로 초기화한다. 즉 `@DependsOn("anotherBean")`이 있으면 "anotherBean"이 완전히 생성되고 나서야 이 Bean의 생성이 시작된다. 의존성 주입(`@Autowired`)과 달리 실제 객체 참조가 필요 없는 순서 보장 메커니즘이다. DB 초기화 Bean이 서비스 Bean보다 먼저 생성되어야 할 때 활용한다.
>
> **Q3.** `DefaultListableBeanFactory.getBeansOfType()`으로 특정 타입의 Bean 목록을 가져올 때 `beanDefinitionNames`의 순서를 따른다. `@Autowired List<SomeInterface>`는 내부적으로 `getBeansOfType(SomeInterface.class)`를 호출해 Map을 얻고 값 목록으로 변환한다. 따라서 등록 순서(스캔 순서 ≈ 패키지 알파벳 순)가 리스트 주입 순서에 영향을 준다. 명확한 순서가 필요하면 `@Order`나 `Ordered` 인터페이스를 구현해 주입 시점에 정렬되도록 해야 한다.

---

<div align="center">

**[⬅️ 이전: @Conditional 평가 과정](./04-conditional-evaluation.md)** | **[다음: 컴포넌트 인덱스를 통한 스캔 최적화 ➡️](./06-component-index-optimization.md)**

</div>

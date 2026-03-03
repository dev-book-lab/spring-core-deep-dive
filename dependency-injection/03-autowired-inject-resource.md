# @Autowired vs @Inject vs @Resource — 탐색 순서와 처리 경로의 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Autowired`, `@Inject`, `@Resource`는 내부적으로 어떻게 다르게 처리되는가?
- 각 어노테이션의 Bean 탐색 순서는 타입 우선인가, 이름 우선인가?
- `AutowiredAnnotationBeanPostProcessor`가 `@Autowired`를 처리하는 정확한 경로는?
- `@Resource`가 JSR-250이고 `@Inject`가 JSR-330인데, 스프링에서 어떤 차이가 생기는가?
- 동일 타입 Bean이 여러 개일 때 각 어노테이션의 처리 방식 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 의존성 주입 어노테이션이 왜 세 가지인가

```java
// 세 가지 모두 동일하게 보이는데...
@Autowired UserRepository repository;  // 스프링 고유
@Inject    UserRepository repository;  // JSR-330 (javax/jakarta)
@Resource  UserRepository repository;  // JSR-250 (javax/jakarta)
```

```
배경:
  @Autowired: 스프링 독자 어노테이션 (2.5, 2007)
  @Inject:    JSR-330 표준 (Java EE 6, 2009)
              CDI(Contexts and Dependency Injection) 표준
  @Resource:  JSR-250 표준 (Java EE 5, 2006)
              원래 JNDI 리소스 조회용

왜 셋 다 지원하는가:
  자바 EE 표준 호환성
  → CDI를 사용하다 스프링으로 전환 시 코드 변경 최소화
  → 스프링 의존 없이 코드 작성 가능 (JSR 어노테이션 사용 시)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 세 어노테이션이 동일하게 동작한다

```java
// ❌ 잘못된 이해
// "셋 다 @Autowired byType으로 동작한다"

@Component("mainRepo")
class MainUserRepository implements UserRepository { }

@Component("backupRepo")
class BackupUserRepository implements UserRepository { }

// 필드명이 "userRepository"인데 Bean 이름과 매칭 안 됨
@Autowired UserRepository userRepository;  // NoUniqueBeanDefinitionException
@Inject    UserRepository userRepository;  // NoUniqueBeanDefinitionException
@Resource  UserRepository userRepository;  // ← 이름 "userRepository"로 탐색 → 없으면 타입

// 셋이 다르게 동작함!
```

---

## ✨ 올바른 이해와 사용

### After: 탐색 순서와 처리 클래스가 다르다

```
어노테이션별 탐색 전략:

@Autowired / @Inject
  1차: 타입(Type)으로 탐색
  2차: 타입 일치 여러 개 → 필드명/파라미터명으로 Qualifier
  3차: @Qualifier 어노테이션으로 명시적 지정
  → byType 우선

@Resource
  1차: 이름(name 속성 또는 필드명)으로 탐색
  2차: 이름 매칭 실패 → 타입으로 폴백
  → byName 우선
```

---

## 🔬 내부 동작 원리

### 1. 처리 클래스 분리

```java
// @Autowired, @Inject → AutowiredAnnotationBeanPostProcessor
// (동일한 BPP가 처리)
public class AutowiredAnnotationBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

    // 생성자에서 처리할 어노테이션 타입 등록
    public AutowiredAnnotationBeanPostProcessor() {
        this.autowiredAnnotationTypes.add(Autowired.class);
        this.autowiredAnnotationTypes.add(Value.class);

        // JSR-330 @Inject 클래스패스에 있으면 추가
        try {
            this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                ClassUtils.forName("jakarta.inject.Inject", getClass().getClassLoader()));
        } catch (ClassNotFoundException ex) {
            // @Inject 없으면 무시
        }
    }
}

// @Resource → CommonAnnotationBeanPostProcessor
// (별도 BPP가 처리)
public class CommonAnnotationBeanPostProcessor
        extends InitDestroyAnnotationBeanPostProcessor {
    // @Resource, @PostConstruct, @PreDestroy 처리
}
```

```
처리 BPP 분리:

@Autowired, @Inject → AutowiredAnnotationBeanPostProcessor
  → populateBean()의 postProcessProperties() 에서 실행
  → byType 탐색 로직 사용

@Resource → CommonAnnotationBeanPostProcessor
  → 같은 postProcessProperties() 시점
  → byName 탐색 우선 로직 사용

실행 순서:
  CommonAnnotationBeanPostProcessor (PriorityOrdered, order=-3)
  AutowiredAnnotationBeanPostProcessor (PriorityOrdered, order=-2)
  → @Resource가 @Autowired보다 먼저 처리됨
```

### 2. @Autowired 처리 경로 — AutowiredAnnotationBeanPostProcessor 추적

```java
// AutowiredAnnotationBeanPostProcessor.postProcessProperties()
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {

    // 1. 어노테이션 메타데이터 캐시 조회 (없으면 리플렉션으로 수집)
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);

    // 2. 실제 주입 수행
    metadata.inject(bean, beanName, pvs);
    return pvs;
}

// findAutowiringMetadata() 내부
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, ...) {
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);

    if (needsRefresh(metadata, clazz)) {
        // 캐시 미스 → 리플렉션으로 탐색
        metadata = buildAutowiringMetadata(clazz);
        this.injectionMetadataCache.put(cacheKey, metadata);
    }
    return metadata;
}

// buildAutowiringMetadata() — 클래스 계층 순회하며 @Autowired 필드/메서드 수집
private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        // 필드에서 @Autowired 탐색
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            MergedAnnotation<?> ann = findAutowiredAnnotation(field);
            if (ann != null) {
                elements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 메서드에서 @Autowired 탐색
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            // ...
        });

        targetClass = targetClass.getSuperclass();
    } while (targetClass != null && targetClass != Object.class);  // 부모 클래스도 탐색

    return new InjectionMetadata(clazz, elements);
}
```

### 3. @Autowired byType 탐색 — DefaultListableBeanFactory

```java
// AutowiredFieldElement.inject()
protected void inject(Object bean, String beanName, PropertyValues pvs) {
    Field field = (Field) this.member;

    Object value = resolveFieldValue(field, bean, beanName);
    // resolveFieldValue → DefaultListableBeanFactory.resolveDependency()

    ReflectionUtils.makeAccessible(field);
    field.set(bean, value);
}

// DefaultListableBeanFactory.resolveDependency() — 핵심 탐색 로직
public Object resolveDependency(DependencyDescriptor descriptor, ...) {

    // 1. 타입으로 Bean 후보 목록 조회
    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);

    if (matchingBeans.isEmpty()) {
        if (descriptor.isRequired()) {
            throw new NoSuchBeanDefinitionException(type);  // required=true면 예외
        }
        return null;
    }

    if (matchingBeans.size() > 1) {
        // 2. 여러 후보 중 하나 결정
        String autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
        // determineAutowireCandidate():
        //   @Primary Bean 있으면 → 그것 선택
        //   없으면 → 필드명/파라미터명과 Bean 이름 매칭
        //   그래도 없으면 → NoUniqueBeanDefinitionException
    }

    return matchingBeans.values().iterator().next();
}
```

```
@Autowired byType 탐색 순서:

1. 타입으로 후보 목록 조회 (findAutowireCandidates)
2. 후보 1개 → 그것 반환
3. 후보 여러 개:
   a. @Primary 붙은 것 → 선택
   b. @Priority 높은 것 → 선택
   c. 필드명/파라미터명 == Bean 이름 → 선택 (byName 폴백)
   d. 모두 해당 없음 → NoUniqueBeanDefinitionException
```

### 4. @Resource byName 탐색 — CommonAnnotationBeanPostProcessor

```java
// CommonAnnotationBeanPostProcessor.ResourceElement.inject()
protected Object getResourceToInject(Object target, String requestingBeanName) {
    return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
            getResource(this, requestingBeanName));
}

// getResource() 내부
protected Object getResource(LookupElement element, String requestingBeanName) {
    // 1. JNDI 탐색 시도 (웹 환경)
    // 2. 실패하면 → Bean Factory에서 탐색

    return autowireResource(this.resourceFactory, element, requestingBeanName);
}

// autowireResource() — byName 우선
protected Object autowireResource(BeanFactory factory, LookupElement element, ...) {
    String name = element.name;  // @Resource(name="...") 또는 필드명

    if (factory.containsBean(name)) {
        // 1차: 이름으로 직접 조회
        return factory.getBean(name, element.lookupType);
    }
    else {
        // 2차: 이름 매칭 실패 → 타입으로 폴백
        return factory.getBean(element.lookupType);
    }
}
```

```
@Resource 탐색 순서:

1. name 속성 명시: @Resource(name="mainRepo")
   → 이름 "mainRepo"로 직접 getBean()

2. name 없음: @Resource
   → 필드명으로 getBean() 시도
   → 없으면 타입으로 폴백

차이 정리:
  @Autowired: 타입 → (이름 폴백) → 예외
  @Resource:  이름 → (타입 폴백) → 예외
```

### 5. 실제 탐색 결과 비교

```java
@Component("mainRepo")   class MainUserRepository implements UserRepository { }
@Component("backupRepo") class BackupUserRepository implements UserRepository { }

// 케이스 1: 필드명이 타입과 무관한 임의 이름
@Autowired UserRepository repo;
// → 타입 UserRepository로 탐색 → 2개 → 필드명 "repo"로 매칭 시도 → 없음
// → NoUniqueBeanDefinitionException

@Resource UserRepository repo;
// → 이름 "repo"로 탐색 → 없음 → 타입 UserRepository로 폴백 → 2개
// → NoUniqueBeanDefinitionException

// 케이스 2: 필드명이 Bean 이름과 일치
@Autowired UserRepository mainRepo;
// → 타입 UserRepository로 탐색 → 2개 → 필드명 "mainRepo" 매칭 → MainUserRepository 선택 ✅

@Resource UserRepository mainRepo;
// → 이름 "mainRepo"로 탐색 → MainUserRepository 발견 → 즉시 반환 ✅

// 케이스 3: @Resource name 명시
@Resource(name = "backupRepo") UserRepository repository;
// → 이름 "backupRepo"로 직접 탐색 → BackupUserRepository ✅
```

---

## 💻 실험으로 확인하기

### 실험 1: 탐색 순서 차이 확인

```java
@Configuration
class MultiRepoConfig {
    @Bean("fastRepo") UserRepository fastRepo() { return new FastUserRepository(); }
    @Bean("slowRepo") UserRepository slowRepo() { return new SlowUserRepository(); }
}

@Service
class TestService {
    // @Primary 없이 두 Bean 존재

    @Autowired
    UserRepository fastRepo;   // 필드명 "fastRepo" → 타입 2개 → 이름 폴백 → fastRepo 선택 ✅

    @Resource
    UserRepository slowRepo;   // 이름 "slowRepo" → 직접 탐색 → slowRepo 선택 ✅

    @Autowired
    UserRepository someRepo;   // 필드명 "someRepo" → 타입 2개 → 이름 폴백 실패
                               // → NoUniqueBeanDefinitionException ❌
}
```

### 실험 2: @Inject vs @Autowired 동일 동작 확인

```java
// @Inject는 @Autowired와 동일한 BPP(AutowiredAnnotationBeanPostProcessor)로 처리됨
@Service
class ServiceA {
    @Autowired UserRepository repo1;  // 동일하게 동작
    @Inject    UserRepository repo2;  // 동일하게 동작
}

// 차이:
// @Autowired(required=false) → 지원
// @Inject                    → required 속성 없음 (항상 required=true)
// → Optional 처리:
//   @Autowired: required=false 또는 Optional<T>
//   @Inject:    Optional<T> 또는 Provider<T>
```

---

## 🤔 트레이드오프

```
어노테이션 선택 기준:

@Autowired
  스프링 고유 기능(required, @Value 함께 사용)이 필요하면 사용
  스프링 의존성을 코드에 명시적으로 드러냄

@Inject (JSR-330)
  스프링 비의존 코드를 원하면 사용
  CDI와의 호환성 필요 시
  단, required=false 등 스프링 확장 기능 사용 불가

@Resource (JSR-250)
  이름 기반 탐색이 명확히 필요할 때
  "이 Bean은 반드시 이 이름이어야 한다"를 표현할 때
  JNDI 리소스 주입 레거시 코드 호환

실무 권장:
  대부분의 경우 @Autowired + @Qualifier 조합
  혹은 생성자 주입 (어노테이션 불필요)
  여러 팀에서 일관성을 위해 하나로 통일하는 것이 중요
```

---

## 📌 핵심 정리

```
처리 BPP
  @Autowired, @Inject → AutowiredAnnotationBeanPostProcessor
  @Resource           → CommonAnnotationBeanPostProcessor

탐색 순서
  @Autowired/@Inject: 타입 → 이름(필드명) 폴백 → 예외
  @Resource:          이름(필드명/name 속성) → 타입 폴백 → 예외

@Autowired 후보 결정 순서
  타입 매칭 → @Primary → 이름 매칭 → NoUniqueBeanDefinitionException

@Resource 특징
  name 속성 명시 → 이름으로 직접 조회
  name 없음 → 필드명으로 조회 → 실패 시 타입

@Inject vs @Autowired
  동일 BPP 처리, 거의 동일 동작
  @Inject: required 속성 없음 (항상 필수)
  @Autowired: required=false, @Value 함께 사용 가능

실행 순서
  CommonAnnotationBPP (order=-3) 먼저
  AutowiredAnnotationBPP (order=-2) 나중
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 주입되는 Bean은 무엇인가? `@Autowired`와 `@Resource`의 결과가 다른가?

```java
@Component("userRepo")
class UserRepositoryImpl implements UserRepository { }

@Component("adminRepo")
class AdminRepositoryImpl implements UserRepository { }

// 케이스 A
@Autowired UserRepository adminRepo;

// 케이스 B
@Resource  UserRepository adminRepo;
```

**Q2.** `@Inject`는 `required=false`를 지원하지 않는다. `@Inject`를 사용하면서 선택적 의존성을 처리하는 방법 두 가지를 설명하라.

**Q3.** `AutowiredAnnotationBeanPostProcessor`가 `@Inject`를 처리할 때 클래스패스에 `jakarta.inject.Inject`가 없으면 어떻게 동작하는가?

> 💡 **해설**
>
> **Q1.** 둘 다 `AdminRepositoryImpl`을 주입받지만 탐색 경로가 다르다. **케이스 A (`@Autowired`)**: 타입 `UserRepository`로 탐색 → 2개 발견(`userRepo`, `adminRepo`) → 필드명 `adminRepo`와 Bean 이름 매칭 → `AdminRepositoryImpl` 선택. **케이스 B (`@Resource`)**: 필드명 `adminRepo`로 이름 탐색 → `adminRepo` Bean 직접 발견 → `AdminRepositoryImpl` 선택. 결과는 같지만 탐색 경로가 다르다. 만약 Bean 이름이 `adminRepo`가 아닌 다른 이름이었다면 `@Autowired`는 `NoUniqueBeanDefinitionException`, `@Resource`는 이름 탐색 실패 후 타입 폴백으로 동일하게 실패할 수 있다.
>
> **Q2.** ① `Optional<T>` 사용 — `@Inject Optional<MyService> service;` 또는 생성자에서 `@Inject MyService(@Nonnull SomeBean b, Optional<OptionalBean> opt)`. `Optional`이 비어있으면 Bean이 없는 것. ② `Provider<T>` 사용 (JSR-330) — `@Inject Provider<MyService> serviceProvider;` 후 `serviceProvider.get()`으로 필요 시점에 조회. Bean이 없으면 `get()` 호출 시 예외가 발생하므로 존재 여부를 먼저 확인해야 한다. 더 안전한 방법은 `Optional`이다.
>
> **Q3.** `AutowiredAnnotationBeanPostProcessor` 생성자에서 `ClassUtils.forName("jakarta.inject.Inject", ...)`을 `try-catch`로 감싸고 있다. 클래스를 찾지 못하면 `ClassNotFoundException`이 잡히고, 단순히 `autowiredAnnotationTypes`에 `@Inject`가 추가되지 않을 뿐이다. BPP 자체는 정상 생성되며 `@Autowired`, `@Value`만 처리한다. 클래스패스에 `jakarta.inject` 의존성이 없어도 스프링 애플리케이션은 정상 동작한다.

---

<div align="center">

**[⬅️ 이전: 순환 참조 해결 — 3단계 캐시](./02-circular-dependency-3cache.md)** | **[다음: @Qualifier & @Primary 우선순위 ➡️](./04-qualifier-primary-priority.md)**

</div>

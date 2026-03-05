# CGLIB 프록시가 적용되는 이유 — ConfigurationClassEnhancer와 $$SpringCGLIB 클래스 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ConfigurationClassEnhancer`가 CGLIB 서브클래스를 만드는 정확한 과정은?
- 생성된 `$$SpringCGLIB$$0` 클래스에 심어지는 콜백(Callback)은 무엇인가?
- AOP용 CGLIB 프록시와 @Configuration용 CGLIB 서브클래스는 어떻게 다른가?
- `BeanFactory`가 CGLIB 서브클래스 내부에 주입되는 방법은?
- `Objenesis`가 여기서도 사용되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: @Bean 메서드를 일반 Java 메서드로 남겨두면 싱글톤을 보장할 수 없다

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();  // 일반 메서드면 호출마다 새 객체
    }
}

// Spring 컨테이너가 AppConfig를 그대로 사용하면:
// ctx.getBean(DataSource.class) → appConfig.dataSource() → new HikariDataSource()
// orderService() 내부에서 dataSource() → new HikariDataSource() (또 다른 인스턴스)
```

```
해결 아이디어:
  AppConfig 클래스 자체를 서브클래싱해서
  @Bean 메서드를 오버라이딩 → 호출마다 컨테이너에서 기존 Bean을 꺼내 반환

  이를 위해 CGLIB으로 AppConfig의 서브클래스를 동적 생성
  → AppConfig$$SpringCGLIB$$0 extends AppConfig
  → @Bean 메서드 오버라이딩 → BeanMethodInterceptor 연결
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Configuration의 CGLIB = AOP의 CGLIB과 같다

```
❌ 잘못된 이해:
  "CglibAopProxy와 ConfigurationClassEnhancer가 같은 방식으로 프록시를 만든다"

✅ 실제 차이:

AOP CglibAopProxy:
  목적: 메서드 호출 시 Advice 체인 실행
  Callback: DynamicAdvisedInterceptor
  생성: Bean 인스턴스 생성 시 (postProcessAfterInitialization)
  원본 인스턴스: 별도 target으로 분리

@Configuration ConfigurationClassEnhancer:
  목적: @Bean 메서드 호출 시 싱글톤 Bean 반환
  Callback: BeanMethodInterceptor + BeanFactoryAwareMethodInterceptor
  생성: BeanDefinition의 beanClass를 미리 교체 (postProcessBeanFactory)
  원본 인스턴스: CGLIB 서브클래스 자체가 AppConfig 역할
```

### Before: CGLIB 서브클래스는 모든 메서드를 가로챈다

```
❌ 잘못된 이해:
  "AppConfig$$SpringCGLIB$$0의 모든 메서드에 인터셉터가 붙는다"

✅ 실제:
  @Bean 어노테이션이 붙은 메서드만 BeanMethodInterceptor 연결
  setBeanFactory() 메서드만 BeanFactoryAwareMethodInterceptor 연결
  나머지 메서드 → NoOp.INSTANCE (그냥 통과)
```

---

## ✨ 올바른 이해와 사용

### After: 콜백 기반 선택적 인터셉션 구조

```
AppConfig$$SpringCGLIB$$0 구조:

Callback 배열:
  [0] BeanMethodInterceptor       → @Bean 메서드에 적용
  [1] BeanFactoryAwareMethodInterceptor → setBeanFactory() 에 적용
  [2] NoOp.INSTANCE               → 나머지 메서드에 적용

CallbackFilter:
  메서드별로 어떤 Callback을 쓸지 결정
  → isBeanCreationMethod() → BeanMethodInterceptor
  → isBeanFactoryAwareSetter() → BeanFactoryAwareMethodInterceptor
  → 기타 → NoOp
```

---

## 🔬 내부 동작 원리

### 1. ConfigurationClassEnhancer — CGLIB 서브클래스 생성

```java
// ConfigurationClassEnhancer.java
class ConfigurationClassEnhancer {

    // 추가할 인터페이스 목록
    private static final Class<?>[] CALLBACK_TYPES = {
        BeanMethodInterceptor.class,
        BeanFactoryAwareMethodInterceptor.class,
        NoOp.class
    };

    public Class<?> enhance(Class<?> configClass, ClassLoader classLoader) {
        if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
            // 이미 향상된 클래스면 건너뜀
            return configClass;
        }

        // CGLIB Enhancer 설정
        Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
        return enhancedClass;
    }

    private Enhancer newEnhancer(Class<?> configSuperClass, ClassLoader classLoader) {
        Enhancer enhancer = new Enhancer();

        enhancer.setSuperclass(configSuperClass);
        // → AppConfig를 상속

        enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
        // → 향상된 클래스임을 마킹하는 인터페이스
        //   EnhancedConfiguration extends BeanFactoryAware
        //   → setBeanFactory(BeanFactory) 메서드 보유

        enhancer.setUseFactory(false);
        // → CGLIB Factory 패턴 사용 안 함

        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        // → "$$SpringCGLIB$$0" 이름 규칙

        enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
        // → BeanFactory 필드를 클래스에 추가하는 전략

        enhancer.setCallbackFilter(CALLBACK_FILTER);
        // → 메서드별 Callback 선택 필터

        enhancer.setCallbackTypes(CALLBACK_TYPES);
        // → 사용할 Callback 타입 목록

        return enhancer;
    }

    private Class<?> createClass(Enhancer enhancer) {
        Class<?> subclass = enhancer.createClass();
        // → 클래스 생성만 (인스턴스화 아님)

        // Callback 인스턴스를 static thread-local에 등록
        // → 나중에 CGLIB 인스턴스 생성 시 적용됨
        Enhancer.registerStaticCallbacks(subclass, CALLBACKS);

        return subclass;
    }
}
```

### 2. BeanFactoryAwareGeneratorStrategy — $$beanFactory 필드 추가

```java
// BeanFactoryAwareGeneratorStrategy
// → CGLIB 서브클래스 바이트코드 생성 시 추가 필드 삽입

private static class BeanFactoryAwareGeneratorStrategy
        extends ClassLoaderAwareGeneratorStrategy {

    @Override
    protected ClassGenerator transform(ClassGenerator cg) throws Exception {
        // $$beanFactory 필드를 클래스에 추가
        ClassEmitterTransformer transformer = new ClassEmitterTransformer() {
            @Override
            public void end_class() {
                // private BeanFactory $$beanFactory 필드 추가
                declare_field(ACC_PUBLIC, BEAN_FACTORY_FIELD,
                    Type.getType(BeanFactory.class), null);
                super.end_class();
            }
        };
        return new TransformingClassGenerator(cg, transformer);
    }
}
```

```
생성된 CGLIB 서브클래스 (개념적 바이트코드):

public class AppConfig$$SpringCGLIB$$0 extends AppConfig
        implements EnhancedConfiguration {

    // BeanFactoryAwareGeneratorStrategy가 추가한 필드
    public BeanFactory $$beanFactory;

    // EnhancedConfiguration → BeanFactoryAware.setBeanFactory()
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.$$beanFactory = beanFactory;
        // BeanFactoryAwareMethodInterceptor가 처리
    }

    // @Bean 메서드 오버라이딩
    @Override
    public DataSource dataSource() {
        // BeanMethodInterceptor가 처리
        // → 상세 동작은 03 문서에서
    }

    @Override
    public OrderService orderService() {
        // BeanMethodInterceptor가 처리
    }
}
```

### 3. CallbackFilter — 메서드별 Callback 선택

```java
// ConfigurationClassEnhancer 내부 CallbackFilter
private static final CallbackFilter CALLBACK_FILTER = new CallbackFilter() {

    @Override
    public int accept(Method candidateMethod) {
        // setBeanFactory() → BeanFactoryAwareMethodInterceptor (index 1)
        if (isBeanFactoryAwareSetter(candidateMethod)) {
            return 1;
        }
        // @Bean 메서드 → BeanMethodInterceptor (index 0)
        if (isBeanCreationMethod(candidateMethod)) {
            return 0;
        }
        // 나머지 → NoOp (index 2)
        return 2;
    }

    private boolean isBeanCreationMethod(Method method) {
        return method.isAnnotationPresent(Bean.class);
    }

    private boolean isBeanFactoryAwareSetter(Method method) {
        return method.getName().equals("setBeanFactory")
            && method.getParameterCount() == 1
            && BeanFactory.class.isAssignableFrom(method.getParameterTypes()[0]);
    }
};
```

### 4. BeanFactoryAwareMethodInterceptor — BeanFactory 주입 경로

```java
// BeanFactoryAwareMethodInterceptor
// → CGLIB 서브클래스 인스턴스에 BeanFactory를 주입

private static class BeanFactoryAwareMethodInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                             MethodProxy proxy) throws Throwable {

        // setBeanFactory(beanFactory) 호출 시 가로챔
        BeanFactory beanFactory = (BeanFactory) args[0];

        // $$beanFactory 필드에 직접 저장
        Field field = ReflectionUtils.findField(obj.getClass(), BEAN_FACTORY_FIELD);
        ReflectionUtils.makeAccessible(field);
        field.set(obj, beanFactory);

        // BeanFactoryAware.setBeanFactory() 원본도 호출
        if (BeanFactoryAware.class.isAssignableFrom(
                ClassUtils.getUserClass(obj.getClass()))) {
            return proxy.invokeSuper(obj, args);
        }
        return null;
    }
}
```

```
BeanFactory 주입 시점:
  AbstractAutowireCapableBeanFactory.initializeBean()
  → invokeAwareMethods()
  → BeanFactoryAware.setBeanFactory(beanFactory) 호출
  → BeanFactoryAwareMethodInterceptor 가로챔
  → $$beanFactory 필드에 저장

이후 BeanMethodInterceptor에서:
  obj.$$beanFactory.getBean("dataSource")
  → 컨테이너에서 기존 Bean 반환
```

### 5. Objenesis 사용 여부

```java
// ConfigurationClassEnhancer에서의 CGLIB vs AOP의 CGLIB 비교

// AOP CglibAopProxy:
// → ObjenesisCglibAopProxy 사용
// → 기본 생성자 없어도 인스턴스 생성 가능

// ConfigurationClassEnhancer:
// → Objenesis 미사용
// → Enhancer.createClass()로 클래스만 생성
// → 인스턴스는 일반 BeanFactory 생성 경로로 생성
//   (AbstractAutowireCapableBeanFactory.instantiateBean)
// → @Configuration 클래스에는 인수 없는 생성자 필요 (또는 @Autowired 생성자)

// 왜 차이?
// @Configuration 서브클래스는 Bean 생성 경로를 타므로
// 생성자 주입(@Autowired)이 일반적으로 동작함
// Objenesis로 우회할 필요 없음
```

---

## 💻 실험으로 확인하기

### 실험 1: 생성된 CGLIB 클래스 구조 분석

```bash
# 컴파일된 CGLIB 클래스 덤프
# JVM 옵션 추가
-Dcglib.debugLocation=/tmp/cglib-classes

# /tmp/cglib-classes 에 생성된 .class 파일 확인
javap -p /tmp/cglib-classes/com/example/AppConfig\$\$SpringCGLIB\$\$0.class
```

```
출력 예시:
public class AppConfig$$SpringCGLIB$$0 extends AppConfig implements EnhancedConfiguration {
  public BeanFactory $$beanFactory;          ← BeanFactoryAwareGeneratorStrategy가 추가
  public DataSource dataSource();            ← BeanMethodInterceptor 연결
  public OrderService orderService();        ← BeanMethodInterceptor 연결
  public void setBeanFactory(BeanFactory);   ← BeanFactoryAwareMethodInterceptor 연결
  ...
}
```

### 실험 2: EnhancedConfiguration 마킹 확인

```java
@Autowired AppConfig appConfig;

System.out.println(appConfig instanceof EnhancedConfiguration);
// Full Mode → true
// Lite Mode → false

// CGLIB 클래스인지도 확인
System.out.println(appConfig.getClass().getName().contains("SpringCGLIB"));
// Full Mode → true
```

### 실험 3: $$beanFactory 필드 직접 접근

```java
@Autowired AppConfig appConfig;

Field bfField = ReflectionUtils.findField(appConfig.getClass(), "$$beanFactory");
ReflectionUtils.makeAccessible(bfField);
BeanFactory injectedBf = (BeanFactory) bfField.get(appConfig);

System.out.println(injectedBf == ctx.getAutowireCapableBeanFactory());
// true → 컨텍스트의 BeanFactory가 주입됨
```

---

## 🤔 트레이드오프

```
ConfigurationClassEnhancer CGLIB vs AOP CGLIB:

공통점:
  ASM으로 서브클래스 바이트코드 생성
  원본 클래스의 메서드 오버라이딩
  final 클래스/메서드에 사용 불가

차이점:
  @Configuration CGLIB:
    클래스만 미리 생성 (postProcessBeanFactory 단계)
    인스턴스는 일반 Bean 생성 경로
    $$beanFactory 필드 직접 추가
    CallbackFilter로 @Bean 메서드만 선택적 인터셉션

  AOP CGLIB:
    인스턴스 생성 후 postProcessAfterInitialization에서 프록시 래핑
    Objenesis로 기본 생성자 없이도 인스턴스 생성
    target은 별도 원본 Bean
    Advice 체인 연결

BeanFactoryAwareGeneratorStrategy 비용:
  바이트코드에 $$beanFactory 필드 삽입
  → 일반 CGLIB보다 약간 더 복잡한 클래스 생성
  → 미미한 수준
```

---

## 📌 핵심 정리

```
생성 시점
  ConfigurationClassPostProcessor.postProcessBeanFactory()
  → enhanceConfigurationClasses()
  → Full Mode BeanDefinition의 beanClass를 CGLIB 서브클래스로 교체
  → Bean 인스턴스 생성 전에 클래스만 교체됨

Enhancer 설정 핵심
  setSuperclass(AppConfig)         → AppConfig 상속
  setInterfaces(EnhancedConfiguration) → 마킹 인터페이스
  setStrategy(BeanFactoryAwareGeneratorStrategy) → $$beanFactory 필드 추가
  setCallbackFilter(CALLBACK_FILTER) → 메서드별 Callback 선택
  setCallbackTypes(CALLBACK_TYPES) → BeanMethodInterceptor, NoOp 등

Callback 배분 (CallbackFilter)
  @Bean 메서드    → BeanMethodInterceptor
  setBeanFactory() → BeanFactoryAwareMethodInterceptor
  나머지          → NoOp.INSTANCE

BeanFactory 주입 경로
  Bean 초기화 시 BeanFactoryAware.setBeanFactory() 호출
  → BeanFactoryAwareMethodInterceptor 가로챔
  → $$beanFactory 필드에 저장
  → 이후 BeanMethodInterceptor에서 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `EnhancedConfiguration` 인터페이스를 마킹으로 추가하는 이유는 무엇인가?

**Q2.** `enhanceConfigurationClasses()`가 `postProcessBeanFactory()` 단계에서 실행되는 이유는? `postProcessBeanDefinitionRegistry()`에서 바로 처리하면 안 되는가?

**Q3.** `AppConfig$$SpringCGLIB$$0` 클래스를 `Class.forName()`으로 직접 로드하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `EnhancedConfiguration extends BeanFactoryAware`이므로 이 인터페이스를 구현하면 `setBeanFactory()` 메서드가 자동으로 존재하게 된다. 스프링은 Bean 초기화 시 `BeanFactoryAware` 구현체에게 `setBeanFactory()`를 호출하는데, 이 경로를 통해 `BeanFactory`가 `$$beanFactory` 필드에 자동으로 주입된다. 또한 `instanceof EnhancedConfiguration` 체크로 이미 향상된 클래스인지 확인해 중복 향상을 방지하는 가드 역할도 한다.
>
> **Q2.** `postProcessBeanDefinitionRegistry()`는 BeanDefinition 등록 단계이며 이 시점에는 모든 `@Bean` 메서드가 아직 파싱 중일 수 있다. CGLIB 향상은 모든 BeanDefinition이 완전히 등록된 후에 일괄 처리해야 한다. `postProcessBeanFactory()`는 BeanDefinition 등록이 완료된 이후이므로 모든 Full Mode 후보를 한 번에 수집해 향상할 수 있다. 또한 이 단계에서 `beanClass`를 교체해야 이후 `finishBeanFactoryInitialization()`에서 Bean 인스턴스 생성 시 CGLIB 서브클래스가 사용된다.
>
> **Q3.** 런타임에 동적으로 생성된 클래스이므로 `Class.forName("com.example.AppConfig$$SpringCGLIB$$0")`는 `ClassNotFoundException`이 발생한다. 이 클래스는 스프링 컨텍스트가 초기화될 때 CGLIB으로 동적 생성되어 `ClassLoader`에 정의되지만, 일반적인 클래스패스 경로에 `.class` 파일이 없다. 같은 `ClassLoader`를 사용한다면 `Thread.currentThread().getContextClassLoader().loadClass()`로 접근할 수는 있지만 스프링 컨텍스트 없이는 의미가 없다.

---

<div align="center">

**[⬅️ 이전: @Configuration vs @Component](./01-configuration-vs-component.md)** | **[다음: @Bean 메서드 호출 가로채기 ➡️](./03-bean-method-interception.md)**

</div>

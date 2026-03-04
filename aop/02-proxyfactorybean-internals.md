# ProxyFactoryBean 내부 구조 — AopProxy 생성 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ProxyFactoryBean` → `ProxyFactory` → `AopProxy` 생성 체인의 각 역할은?
- `AdvisedSupport`가 관리하는 메타데이터는 무엇인가?
- `DefaultAopProxyFactory`가 JDK Proxy와 CGLIB 중 어떤 기준으로 선택하는가?
- `@EnableAspectJAutoProxy`가 등록하는 컴포넌트는 무엇인가?
- `AbstractAutoProxyCreator`가 Bean에 프록시를 씌우는 정확한 시점은?

---

## 🔍 왜 이게 존재하는가

### 문제: 프록시 생성 로직이 여러 곳에 흩어지면 안 된다

```java
// 직접 프록시를 만들면 — 매번 동일한 결정 로직 반복
if (hasInterface) {
    Proxy.newProxyInstance(...);
} else {
    enhancer.create(...);
}
// 어디에 Advisor를 적용할지, 어떤 프록시를 쓸지 결정 로직이 중복
```

```
스프링의 해결:
  ProxyFactory          → 프록시 생성 파사드 (외부 진입점)
  AdvisedSupport        → Advisor/Advice/Interface 메타데이터 저장
  AopProxyFactory       → JDK/CGLIB 선택 전략
  AopProxy              → 실제 프록시 인스턴스 생성

  각 역할이 명확히 분리됨
  → Bean 생성 파이프라인과 프록시 로직이 느슨하게 결합
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: ProxyFactoryBean이 AOP의 핵심이다

```xml
<!-- Spring 초창기 XML 방식 -->
<bean id="orderService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="orderServiceTarget"/>
    <property name="interceptorNames">
        <list><value>loggingInterceptor</value></list>
    </property>
</bean>
```

```
❌ 잘못된 이해:
  "ProxyFactoryBean이 AOP의 핵심 컴포넌트다"

✅ 실제:
  ProxyFactoryBean은 XML 시대의 수동 방식
  @AspectJ 기반 자동 프록시 시대에는 거의 사용 안 함

  현재 AOP 자동화 경로:
  @EnableAspectJAutoProxy
    → AnnotationAwareAspectJAutoProxyCreator (BPP) 등록
    → Bean 생성 시 BPP After에서 자동으로 프록시 결정
    → 내부적으로 ProxyFactory 사용 (ProxyFactoryBean 아님)
```

---

## ✨ 올바른 이해와 사용

### After: 전체 AOP 프록시 생성 클래스 계층을 파악한다

```
클래스 계층:

ProxyConfig                         (프록시 설정 — proxyTargetClass 등)
    ↑
AdvisedSupport extends ProxyConfig   (Advisor/Interface 메타데이터)
    ↑
ProxyCreatorSupport extends AdvisedSupport (AopProxyFactory 보유)
    ↑
    ├── ProxyFactory                  (수동/프로그래밍 방식)
    └── ProxyFactoryBean              (XML Bean 방식)

자동 프록시 경로 (현재 주류):
AbstractAutoProxyCreator (BPP)
  → wrapIfNecessary()
  → ProxyFactory 직접 생성
  → createAopProxy() → DefaultAopProxyFactory
  → JdkDynamicAopProxy 또는 ObjenesisCglibAopProxy
```

---

## 🔬 내부 동작 원리

### 1. AdvisedSupport — 메타데이터 저장소

```java
// AdvisedSupport.java — 핵심 필드들
public class AdvisedSupport extends ProxyConfig implements Advised {

    // 프록시 대상 (실제 Bean)
    private TargetSource targetSource = EMPTY_TARGET_SOURCE;

    // 프록시할 추가 인터페이스 목록
    private List<Class<?>> interfaces = new ArrayList<>();

    // 적용할 Advisor 목록 (순서 중요)
    private List<Advisor> advisors = new ArrayList<>();

    // 메서드별 인터셉터 체인 캐시
    private transient Map<MethodCacheKey, List<Object>> methodCache;

    // Advisor 추가
    public void addAdvisor(Advisor advisor) {
        this.advisors.add(advisor);
        adviceChanged();  // methodCache 초기화
    }

    // 메서드별 인터셉터 체인 조회 (캐시)
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
            Method method, Class<?> targetClass) {
        MethodCacheKey cacheKey = new MethodCacheKey(method);
        List<Object> cached = this.methodCache.get(cacheKey);
        if (cached == null) {
            cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
            this.methodCache.put(cacheKey, cached);
        }
        return cached;
    }
}
```

```
AdvisedSupport가 관리하는 것:
  targetSource   → 원본 Bean (TargetSource로 래핑)
  interfaces     → JDK Proxy에 구현할 인터페이스 목록
  advisors       → Pointcut + Advice 쌍 목록 (순서 = 실행 순서)
  methodCache    → 메서드별 적용 인터셉터 캐시 (성능 핵심)
```

### 2. DefaultAopProxyFactory — JDK vs CGLIB 선택 로직

```java
// DefaultAopProxyFactory.createAopProxy()
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {

    // CGLIB 선택 조건:
    if (config.isOptimize()              // 최적화 모드 (거의 사용 안 함)
            || config.isProxyTargetClass()  // proxy-target-class=true (Boot 기본)
            || hasNoUserSuppliedProxyInterfaces(config)) {  // 인터페이스 없음

        Class<?> targetClass = config.getTargetClass();

        if (targetClass == null) {
            throw new AopConfigException("...");
        }
        if (targetClass.isInterface()       // 인터페이스 자체면 JDK Proxy
                || Proxy.isProxyClass(targetClass)  // 이미 JDK Proxy면
                || ClassUtils.isLambdaClass(targetClass)) {  // 람다면
            return new JdkDynamicAopProxy(config);
        }

        // 나머지 → CGLIB
        return new ObjenesisCglibAopProxy(config);
    }
    // proxyTargetClass=false + 인터페이스 있음 → JDK Proxy
    return new JdkDynamicAopProxy(config);
}
```

```
선택 흐름 (Spring Boot 기본 proxy-target-class=true):

isProxyTargetClass() = true
  → targetClass가 인터페이스?    → JdkDynamicAopProxy
  → targetClass가 JDK Proxy?    → JdkDynamicAopProxy
  → targetClass가 람다?         → JdkDynamicAopProxy
  → 그 외 (일반 클래스)          → ObjenesisCglibAopProxy

isProxyTargetClass() = false (레거시):
  → 인터페이스 없음              → ObjenesisCglibAopProxy
  → 인터페이스 있음              → JdkDynamicAopProxy
```

### 3. AbstractAutoProxyCreator — 자동 프록시 생성 BPP

```java
// AbstractAutoProxyCreator.postProcessAfterInitialization()
// → BPP After 단계에서 각 Bean에 대해 호출
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            // 순환 참조로 이미 early ref에서 프록시 생성된 경우 제외
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {

    // 이미 프록시거나 인프라 Bean이면 건너뜀
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 이 Bean에 적용할 Advisor 목록 조회
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
        bean.getClass(), beanName, null);

    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 프록시 생성
        Object proxy = createProxy(bean.getClass(), beanName,
            specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;  // 원본 bean 대신 proxy 반환 → 컨테이너에 proxy 등록
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;  // Advisor 없으면 원본 그대로
}
```

### 4. createProxy() — ProxyFactory 사용

```java
// AbstractAutoProxyCreator.createProxy()
protected Object createProxy(Class<?> beanClass, String beanName,
                              Object[] specificInterceptors, TargetSource targetSource) {

    // ProxyFactory 생성 및 설정
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);  // proxyTargetClass 등 설정 복사

    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            // Bean이 구현한 인터페이스 추가
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // Advisor 목록 설정
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);

    // 실제 프록시 생성
    return proxyFactory.getProxy(classLoader);
}

// ProxyFactory.getProxy()
public Object getProxy(ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
    // createAopProxy() → DefaultAopProxyFactory.createAopProxy()
    // → JdkDynamicAopProxy 또는 ObjenesisCglibAopProxy
    // getProxy() → Proxy.newProxyInstance() 또는 enhancer.create()
}
```

### 5. @EnableAspectJAutoProxy — BPP 등록 경로

```java
// @EnableAspectJAutoProxy
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}

// AspectJAutoProxyRegistrar
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                         BeanDefinitionRegistry registry) {
        // AnnotationAwareAspectJAutoProxyCreator BeanDefinition 등록
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        // proxyTargetClass / exposeProxy 속성 적용
        AnnotationAttributes attrs = ...;
        if (attrs.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
    }
}
```

```
등록되는 핵심 BPP:
  AnnotationAwareAspectJAutoProxyCreator
    extends AspectJAwareAdvisorAutoProxyCreator
      extends AbstractAdvisorAutoProxyCreator
        extends AbstractAutoProxyCreator
          extends BeanPostProcessor

  BPP After 단계에서 모든 Bean을 검사
  → @Aspect Bean에서 수집한 Advisor와 매칭
  → 매칭 시 ProxyFactory로 프록시 생성 후 반환
```

---

## 💻 실험으로 확인하기

### 실험 1: ProxyFactory 직접 사용 (테스트/학습용)

```java
// 스프링 컨텍스트 없이 프록시 직접 생성
OrderService target = new OrderService();

ProxyFactory factory = new ProxyFactory();
factory.setTarget(target);
factory.addAdvice(new MethodInterceptor() {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Before: " + invocation.getMethod().getName());
        Object result = invocation.proceed();
        System.out.println("After: " + invocation.getMethod().getName());
        return result;
    }
});

OrderService proxy = (OrderService) factory.getProxy();
proxy.placeOrder(new Order());
// 출력:
// Before: placeOrder
// After: placeOrder
```

### 실험 2: AdvisedSupport 메타데이터 조회

```java
@Autowired OrderService orderService;

// 프록시에서 AdvisedSupport 꺼내기
Advised advised = (Advised) orderService;

System.out.println("Advisors: " + advised.getAdvisors().length);
System.out.println("ProxyTargetClass: " + advised.isProxyTargetClass());
System.out.println("TargetClass: " + advised.getTargetClass());

for (Advisor a : advised.getAdvisors()) {
    System.out.println("  " + a.getClass().getSimpleName()
        + ": " + a.getAdvice().getClass().getSimpleName());
}
```

### 실험 3: AnnotationAwareAspectJAutoProxyCreator 등록 확인

```java
// BPP 목록에서 확인
ConfigurableListableBeanFactory bf = ctx.getBeanFactory();
String[] bppNames = bf.getBeanNamesForType(BeanPostProcessor.class, true, false);
Arrays.stream(bppNames)
      .filter(name -> name.contains("AutoProxy"))
      .forEach(System.out::println);
// org.springframework.aop.config.internalAutoProxyCreator
//   → AnnotationAwareAspectJAutoProxyCreator
```

---

## 🤔 트레이드오프

```
ProxyFactoryBean (수동):
  장점  세밀한 제어, Bean 단위 개별 설정
  단점  설정 verbose, 유지보수 어려움
  사용: 레거시 XML 기반 코드, 특수 프록시 설정

ProxyFactory (프로그래밍):
  장점  스프링 컨텍스트 없이 사용 가능 → 테스트 유용
  단점  각 Bean마다 수동 설정 필요
  사용: 단위 테스트, 프레임워크 개발

AbstractAutoProxyCreator + @EnableAspectJAutoProxy (자동):
  장점  설정 최소화, @Aspect 선언만으로 동작
  단점  동작이 추상화 뒤에 숨어 있음 → 디버깅 어려움
  사용: 현재 스프링 AOP의 표준 방식

ObjenesisCglibAopProxy vs CglibAopProxy:
  Objenesis: 기본 생성자 없이도 인스턴스 생성 가능
  → Spring 4.0부터 기본값 (생성자 주입 + CGLIB 호환성 개선)
```

---

## 📌 핵심 정리

```
생성 체인
  @EnableAspectJAutoProxy
    → AnnotationAwareAspectJAutoProxyCreator (BPP) 등록
  Bean 생성 BPP After
    → wrapIfNecessary()
    → createProxy() → ProxyFactory
    → DefaultAopProxyFactory.createAopProxy()
    → JdkDynamicAopProxy 또는 ObjenesisCglibAopProxy
    → getProxy() → 프록시 인스턴스

AdvisedSupport 역할
  targetSource, interfaces, advisors, methodCache 관리
  메서드별 인터셉터 체인을 캐시 → 성능 핵심

JDK vs CGLIB 선택 (DefaultAopProxyFactory)
  proxyTargetClass=true → 일반 클래스면 CGLIB
  인터페이스 / JDK Proxy / 람다 → JDK Proxy

Advised 인터페이스
  프록시 Bean을 Advised로 캐스팅하면 런타임 Advisor 조회 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `AbstractAutoProxyCreator.wrapIfNecessary()`가 `isInfrastructureClass()` 검사를 먼저 하는 이유는? 어떤 클래스가 인프라 클래스로 분류되는가?

**Q2.** `AdvisedSupport.methodCache`는 언제 초기화되는가? Advisor를 런타임에 추가하면 캐시는 어떻게 되는가?

**Q3.** `ProxyFactory.getProxy()`를 호출할 때 `proxyTargetClass=false`이고 Bean이 인터페이스를 구현하지 않는다면 어떤 일이 벌어지는가?

> 💡 **해설**
>
> **Q1.** 인프라 클래스(Advice, Advisor, AopInfrastructureBean 구현체)에 AOP 프록시를 씌우면 무한 재귀가 발생할 수 있다. 예를 들어 `BeanPostProcessor` 자체에 프록시를 씌우면 BPP가 BPP를 프록시하려 하는 상황이 된다. `isInfrastructureClass()`는 `Advice`, `Pointcut`, `Advisor`, `AopInfrastructureBean`의 구현체를 감지해 프록시 대상에서 제외한다. 이들은 AOP 시스템 자체의 구성 요소이므로 AOP 적용 대상이 되면 안 된다.
>
> **Q2.** `methodCache`는 `AdvisedSupport.adviceChanged()`가 호출될 때마다 초기화된다. `addAdvisor()`, `removeAdvisor()`, `addAdvice()` 등 Advisor/Advice를 변경하는 모든 메서드가 내부적으로 `adviceChanged()`를 호출한다. 런타임에 Advisor를 추가하면 캐시가 즉시 초기화되고 다음 메서드 호출 시 새로 인터셉터 체인을 계산하여 캐싱한다. `Advised` 인터페이스로 캐스팅 후 `addAdvisor()`를 호출하면 런타임 AOP 변경이 가능하다.
>
> **Q3.** `DefaultAopProxyFactory.createAopProxy()`에서 `hasNoUserSuppliedProxyInterfaces(config)` 조건이 true가 된다(인터페이스 없음). 이 경우 `proxyTargetClass`와 무관하게 CGLIB 경로(`ObjenesisCglibAopProxy`)로 진입한다. `proxyTargetClass=false`를 설정해도 인터페이스가 없으면 어쩔 수 없이 CGLIB을 사용한다. JDK Proxy는 반드시 구현할 인터페이스가 하나 이상 있어야 하기 때문이다.

---

<div align="center">

**[⬅️ 이전: JDK Dynamic Proxy vs CGLIB](./01-jdk-proxy-vs-cglib.md)** | **[다음: @AspectJ 어노테이션 처리 과정 ➡️](./03-aspectj-annotation-processing.md)**

</div>

# @PreDestroy vs DisposableBean — 컨테이너 종료와 소멸 콜백의 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@PreDestroy`, `DisposableBean.destroy()`, `destroy-method` 세 가지의 실행 순서는?
- 컨테이너가 종료될 때 내부적으로 어떤 순서로 소멸 콜백이 호출되는가?
- `ShutdownHook`은 어디서 등록되고 어떻게 동작하는가?
- Prototype Bean의 소멸 콜백은 왜 호출되지 않는가?
- 소멸이 **보장되지 않는** 경우는 어떤 상황인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Bean이 사라질 때 리소스를 정리하지 못한다

```java
@Service
public class DatabaseConnectionService {
    private Connection connection;

    @PostConstruct
    public void init() {
        connection = dataSource.getConnection();
    }

    // 컨테이너 종료 시 connection.close()를 누가 호출하는가?
    // 아무도 안 하면 → 커넥션 누수, 파일 핸들 누수
}
```

```
소멸 콜백이 필요한 상황:
  DB 커넥션 / 커넥션 풀 종료
  파일 핸들 / 소켓 닫기
  스레드 풀 정상 종료 (graceful shutdown)
  외부 시스템 연결 해제
  캐시 영구 저장 (flush)

해결: 컨테이너 종료 시 소멸 콜백 호출
  @PreDestroy, DisposableBean.destroy(), destroy-method
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Prototype Bean도 소멸 콜백이 호출된다

```java
@Component @Scope("prototype")
public class PrototypeService {
    @PreDestroy
    public void cleanup() {
        System.out.println("정리 완료");  // 이게 호출될까?
    }
}
```

```
❌ 잘못된 이해: "스프링이 알아서 정리해준다"

✅ 실제:
  Prototype Bean은 생성만 담당, 소멸은 호출자 책임
  → @PreDestroy 절대 호출 안 됨
  → 컨테이너가 Prototype 인스턴스를 추적하지 않음

이유:
  Prototype은 getBean()마다 새 인스턴스 생성
  인스턴스가 몇 개인지, 누가 갖고 있는지 컨테이너가 모름
  → 소멸 추적 자체가 불가능
```

### Before: 소멸 콜백은 항상 실행된다

```java
// ❌ 잘못된 가정
// "JVM이 종료되면 @PreDestroy는 반드시 실행된다"

// 소멸 콜백이 실행되지 않는 경우:
// System.exit()   → ShutdownHook 등록 여부에 따라
// kill -9 (SIGKILL) → 즉시 강제 종료, ShutdownHook 실행 안 됨
// JVM 크래시      → 소멸 보장 없음
// @Scope("prototype") → 소멸 콜백 없음
```

---

## ✨ 올바른 이해와 사용

### After: 소멸 콜백 실행 순서와 보장 범위를 명확히 파악

```
소멸 콜백 실행 순서 (항상 고정):
  1. @PreDestroy        (DestructionAwareBeanPostProcessor)
  2. DisposableBean.destroy()   (직접 호출)
  3. destroy-method     (리플렉션 호출)

ShutdownHook 등록 조건:
  AbstractApplicationContext.registerShutdownHook() 명시 호출
  또는 Spring Boot 자동 등록
  → JVM 종료 시 context.close() 자동 호출

소멸 보장되지 않는 경우:
  Prototype Bean
  kill -9 / JVM 크래시
  registerShutdownHook() 미호출 + 수동 close() 미호출
```

---

## 🔬 내부 동작 원리

### 1. ShutdownHook 등록 — registerShutdownHook()

```java
// AbstractApplicationContext.registerShutdownHook()
public void registerShutdownHook() {
    if (this.shutdownHook == null) {
        this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {
            @Override
            public void run() {
                synchronized (startupShutdownMonitor) {
                    doClose();  // context.close()와 동일
                }
            }
        };
        Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        // JVM 종료 시 이 스레드 실행 보장 (SIGTERM, System.exit() 시)
    }
}
```

```
Spring Boot의 ShutdownHook:
  SpringApplication.run() → ConfigurableApplicationContext 반환 시
  context.registerShutdownHook() 자동 호출
  → 개발자가 명시하지 않아도 ShutdownHook 등록

비-Boot 환경:
  ApplicationContext ctx = new AnnotationConfigApplicationContext(...);
  ctx.registerShutdownHook();  // 명시 필요
  // 또는 try-with-resources / ctx.close() 직접 호출
```

### 2. context.close() → 소멸 콜백 호출 흐름

```java
// AbstractApplicationContext.close()
public void close() {
    synchronized (this.startupShutdownMonitor) {
        doClose();
    }
}

protected void doClose() {
    if (this.active.get() && this.closed.compareAndSet(false, true)) {

        // 1. 라이브사이클 종료 (SmartLifecycle.stop())
        this.lifecycleProcessor.onClose();

        // 2. BeanFactory에 소멸 처리 위임
        destroyBeans();

        // 3. BeanFactory 자체 종료
        closeBeanFactory();

        // 4. 서브클래스 훅
        onClose();

        this.active.set(false);
    }
}

protected void destroyBeans() {
    getBeanFactory().destroySingletons();
    // DefaultSingletonBeanRegistry.destroySingletons()
}
```

### 3. destroySingletons() — 소멸 순서 결정

```java
// DefaultSingletonBeanRegistry.destroySingletons()
public void destroySingletons() {
    // 등록된 순서의 역순으로 소멸 (LIFO)
    String[] disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());

    for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
        destroySingleton(disposableBeanNames[i]);
    }
}

public void destroySingleton(String beanName) {
    // 1. singletonObjects에서 제거
    removeSingleton(beanName);

    // 2. DisposableBean 어댑터 꺼내서 소멸 수행
    DisposableBean disposableBean = this.disposableBeans.remove(beanName);
    destroyBean(beanName, disposableBean);
}
```

```
소멸 순서:
  등록 순서의 역순 (LIFO)
  → 나중에 생성된 Bean이 먼저 소멸
  → 의존 관계 역순으로 정리 (의존하는 쪽이 먼저 소멸)

예시:
  OrderService → PaymentService 의존
  생성 순서: PaymentService → OrderService
  소멸 순서: OrderService → PaymentService (역순)
```

### 4. DisposableBeanAdapter — 세 가지 콜백 통합 실행

```java
// DisposableBeanAdapter.destroy()
public void destroy() {

    // 1. DestructionAwareBeanPostProcessor — @PreDestroy 실행
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
            processor.postProcessBeforeDestruction(this.bean, this.beanName);
            // CommonAnnotationBPP → @PreDestroy 메서드 호출
        }
    }

    // 2. DisposableBean.destroy() 호출
    if (this.invokeDisposableBean) {
        ((DisposableBean) this.bean).destroy();
    }

    // 3. destroy-method 호출 (리플렉션)
    if (!CollectionUtils.isEmpty(this.destroyMethodNames)) {
        for (String destroyMethodName : this.destroyMethodNames) {
            invokeCustomDestroyMethod(destroyMethodName);
        }
    }
}
```

```
DisposableBeanAdapter 등록 시점:
  Bean 생성 완료 후 doCreateBean()에서
  → registerDisposableBeanIfNecessary()
  → @PreDestroy / DisposableBean / destroy-method 중 하나라도 있으면
  → disposableBeans Map에 DisposableBeanAdapter 등록
```

### 5. @PreDestroy 처리 상세

```java
// CommonAnnotationBeanPostProcessor.postProcessBeforeDestruction()
public void postProcessBeforeDestruction(Object bean, String beanName) {
    LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
    metadata.invokeDestroyMethods(bean, beanName);
    // 클래스 계층 순서로 @PreDestroy 실행
    // 부모 클래스 @PreDestroy가 나중에 실행 (초기화와 역순)
}
```

```
@PreDestroy 실행 순서 (초기화 역순):
  초기화: 부모 @PostConstruct → 자식 @PostConstruct
  소멸:   자식 @PreDestroy   → 부모 @PreDestroy

이유:
  buildLifecycleMetadata()에서 destroyMethods 수집 시
  부모 메서드를 뒤에 추가 → 소멸 시 뒤에서부터 실행
```

### 6. Prototype Bean 소멸 콜백이 없는 이유

```java
// AbstractBeanFactory.registerDisposableBeanIfNecessary()
protected void registerDisposableBeanIfNecessary(String beanName, Object bean,
                                                   RootBeanDefinition mbd) {
    if (!mbd.isPrototype()  // ← Prototype이면 등록 안 함
            && requiresDestruction(bean, mbd)) {
        // Singleton인 경우만 disposableBeans에 등록
        registerDisposableBean(beanName,
            new DisposableBeanAdapter(bean, beanName, mbd, ...));
    }
    // Prototype: 생성만 책임, 소멸은 호출자에게 위임
}
```

```
Prototype Bean 소멸 처리 방법:
  직접 소멸:
    PrototypeBean bean = ctx.getBean(PrototypeBean.class);
    try {
        bean.doWork();
    } finally {
        ctx.getBeanFactory().destroyBean(beanName, bean);
        // 명시적으로 소멸 콜백 트리거
    }

  또는 DisposableBeanAdapter 직접 생성:
    new DisposableBeanAdapter(bean, beanName, mbd, ...).destroy();
```

---

## 💻 실험으로 확인하기

### 실험 1: 소멸 콜백 순서 확인

```java
@Component
public class MultiDestroyBean implements DisposableBean {

    @PreDestroy
    public void preDestroy() {
        System.out.println("1. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("2. DisposableBean.destroy()");
    }

    public void customDestroy() {
        System.out.println("3. destroy-method");
    }
}

// @Bean(destroyMethod = "customDestroy")
```

```
컨텍스트 종료 시 출력:
1. @PreDestroy
2. DisposableBean.destroy()
3. destroy-method
```

### 실험 2: 생성 역순 소멸 확인

```java
@Component
class BeanA {
    BeanA() { System.out.println("BeanA 생성"); }
    @PreDestroy void destroy() { System.out.println("BeanA 소멸"); }
}

@Component
class BeanB {  // BeanA 의존
    BeanB(BeanA a) { System.out.println("BeanB 생성"); }
    @PreDestroy void destroy() { System.out.println("BeanB 소멸"); }
}
```

```
출력:
BeanA 생성
BeanB 생성
---컨텍스트 종료---
BeanB 소멸   ← 역순: 나중에 생성된 것 먼저
BeanA 소멸
```

### 실험 3: ShutdownHook 없을 때 소멸 미호출

```java
// ShutdownHook 없이 컨텍스트 생성
var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// ctx.registerShutdownHook(); ← 주석 처리

ctx.getBean(MyService.class).doWork();
System.exit(0);  // @PreDestroy 호출 안 됨
```

---

## 🤔 트레이드오프

```
@PreDestroy (권장):
  장점  JSR-250 표준, 스프링 미의존, 간결
  단점  어노테이션 처리 오버헤드 (미미)

DisposableBean:
  장점  직접 메서드 호출 (리플렉션 없음)
  단점  스프링 인터페이스 구현 → 코드 결합

destroy-method:
  장점  외부 라이브러리 클래스의 close/shutdown 메서드 연결
  단점  문자열 메서드명 → 오타 위험
  특이사항: @Bean의 destroy-method 기본값 = "(inferred)"
    → AutoCloseable / Closeable 구현 시 close() 자동 등록

소멸 보장이 필요할 때:
  try-with-resources 또는 try-finally로 직접 close() 보장
  Prototype Bean의 소멸은 반드시 수동으로 처리
  kill -9 같은 강제 종료에 대비: 외부 상태는 주기적 flush
```

---

## 📌 핵심 정리

```
소멸 콜백 실행 순서
  1. @PreDestroy      (CommonAnnotationBPP.postProcessBeforeDestruction)
  2. destroy()        (DisposableBean 직접 호출)
  3. destroy-method   (리플렉션)

소멸 순서
  등록 역순(LIFO) → 나중에 생성된 Bean이 먼저 소멸
  부모 @PreDestroy는 자식 이후에 실행 (초기화와 역순)

ShutdownHook
  registerShutdownHook() → JVM 종료 시 context.close() 자동
  Spring Boot: 자동 등록
  SIGKILL(kill -9): 실행 안 됨

Prototype Bean
  소멸 콜백 미등록 (컨테이너가 인스턴스 추적 안 함)
  수동 소멸: ctx.getBeanFactory().destroyBean(beanName, bean)

@Bean destroy-method 기본값
  "(inferred)" → AutoCloseable이면 close() 자동 연결
```

---

## 🤔 생각해볼 문제

**Q1.** `@Bean(destroyMethod = "")` 처럼 빈 문자열을 지정하면 어떻게 되는가? 언제 이렇게 설정하는가?

**Q2.** 다음 코드에서 컨텍스트 종료 시 소멸 콜백 실행 순서를 예측하라.

```java
public class Parent {
    @PreDestroy void parentDestroy() { System.out.println("A"); }
}
@Component
public class Child extends Parent implements DisposableBean {
    @PreDestroy void childDestroy()  { System.out.println("B"); }
    @Override public void destroy()  { System.out.println("C"); }
}
```

**Q3.** Spring Boot 애플리케이션에서 Graceful Shutdown을 구현하려면 `@PreDestroy` 외에 어떤 방법을 조합해야 하는가?

> 💡 **해설**
>
> **Q1.** `destroyMethod = ""`로 설정하면 destroy-method 자동 추론(`(inferred)`)이 비활성화된다. 기본값인 `"(inferred)"`는 Bean이 `AutoCloseable`이나 `Closeable`을 구현하면 `close()`를 자동으로 destroy-method로 등록한다. 외부 라이브러리 Bean에서 `close()`가 있지만 컨테이너 종료 시 호출하면 안 되는 경우(예: 공유 리소스, 특정 생명주기 관리 필요)에 `destroyMethod = ""`로 자동 추론을 막아 소멸 콜백이 호출되지 않도록 한다.
>
> **Q2.** 출력 순서: B → A → C. `@PreDestroy`는 `CommonAnnotationBPP`가 처리하며, 소멸 시 자식 메서드 먼저, 부모 메서드 나중 순서로 실행된다(초기화와 역순). 따라서 B(자식 @PreDestroy) → A(부모 @PreDestroy). 이후 `DisposableBean.destroy()`인 C가 실행된다.
>
> **Q3.** Spring Boot 2.3+에서는 `server.shutdown=graceful` 설정으로 내장 Tomcat의 Graceful Shutdown을 활성화할 수 있다. 이렇게 하면 종료 신호 수신 시 새 요청을 거부하고 진행 중인 요청이 완료될 때까지 대기한 후 컨텍스트를 종료한다. `spring.lifecycle.timeout-per-shutdown-phase`로 대기 시간을 설정하고, `SmartLifecycle`을 구현해 외부 리소스(메시지 큐 컨슈머, 스케줄러 등)의 단계별 종료 순서를 제어할 수 있다. `@PreDestroy`는 이 모든 처리가 끝난 뒤 Bean 소멸 단계에서 실행된다.

---

<div align="center">

**[⬅️ 이전: @PostConstruct vs InitializingBean](./02-postconstruct-vs-initializingbean.md)** | **[다음: Aware 인터페이스 체인 ➡️](./04-aware-interfaces.md)**

</div>

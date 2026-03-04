# FactoryBean vs ObjectProvider — 복잡한 Bean 생성과 지연 조회

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `FactoryBean<T>`은 왜 존재하는가? 단순 `@Bean` 메서드로 충분하지 않은가?
- `&beanName`으로 FactoryBean 자체를 꺼내는 방법과 내부 처리 과정은?
- `getObject()`와 `isSingleton()`이 함께 동작하는 원리는?
- `ObjectProvider`와 `FactoryBean`은 각각 어떤 문제를 해결하는가?
- Spring 내부에서 FactoryBean이 실제로 사용되는 사례는?

---

## 🔍 왜 이게 존재하는가

### 문제: @Bean 메서드로 표현하기 어려운 복잡한 생성 로직

```java
// 단순한 경우 → @Bean 충분
@Bean
public DataSource dataSource() {
    return new HikariDataSource(config);
}

// 복잡한 경우 → @Bean으로 표현 한계
// 1. 생성 과정이 매우 복잡하고 조건 분기가 많음
// 2. 타입 파라미터 기반 동적 객체 생성
// 3. 프레임워크가 외부에서 Bean 생성 방식을 교체해야 함
// 4. 생성된 객체와 생성자(FactoryBean) 자체를 모두 Bean으로 노출
```

```
FactoryBean의 역할:
  "Bean을 생성하는 Bean"
  getObject()가 반환하는 객체가 컨테이너에 실제 Bean으로 등록됨
  FactoryBean 자체는 "&" 접두사로 접근 가능

스프링이 내부적으로 쓰는 곳:
  ProxyFactoryBean        → AOP 프록시 생성
  JndiObjectFactoryBean   → JNDI 리소스 조회
  ScopedProxyFactoryBean  → Scope Proxy 생성
  LocalSessionFactoryBean → Hibernate SessionFactory
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: getBean("myFactory")가 FactoryBean을 반환한다

```java
@Component
public class MyFactoryBean implements FactoryBean<Connection> {
    @Override
    public Connection getObject() { return createConnection(); }
    @Override
    public Class<?> getObjectType() { return Connection.class; }
}

// ❌ 잘못된 이해
Object bean = ctx.getBean("myFactoryBean");
// "MyFactoryBean 인스턴스가 반환된다"?

// ✅ 실제:
// ctx.getBean("myFactoryBean")  → Connection 반환 (getObject() 결과)
// ctx.getBean("&myFactoryBean") → MyFactoryBean 반환 (FactoryBean 자체)
```

### Before: isSingleton()=true면 getObject()가 한 번만 호출된다

```java
public class MyFactoryBean implements FactoryBean<HeavyObject> {
    @Override
    public HeavyObject getObject() {
        return new HeavyObject();  // 비싼 생성
    }
    @Override
    public boolean isSingleton() { return true; }
    // "true니까 getObject()는 한 번만 호출되겠지?"
}
```

```
✅ 실제:
  isSingleton() = true → 스프링이 getObject() 결과를 캐시
  → factoryBeanObjectCache에 저장
  → 이후 getBean() 시 캐시에서 반환 (getObject() 재호출 안 함)

  isSingleton() = false → getBean()마다 getObject() 호출 (Prototype처럼)
  → 매번 새 인스턴스
```

---

## ✨ 올바른 이해와 사용

### After: FactoryBean과 ObjectProvider의 역할을 명확히 구분

```
FactoryBean<T>:
  "복잡한 로직으로 Bean을 생성하는 Bean"
  getObject()가 반환하는 T가 실제 Bean
  컨테이너 초기화 시점에 생성 로직 실행
  → 생성 로직 캡슐화, 외부 프레임워크 연동에 유용

ObjectProvider<T>:
  "기존 Bean을 필요할 때 꺼내는 지연 조회기"
  Bean 생성 로직 없음, 컨테이너에서 탐색만
  메서드 호출 시점에 탐색 → 지연 + 선택적 조회
  → Prototype Bean 매번 획득, Optional 처리에 유용
```

---

## 🔬 내부 동작 원리

### 1. FactoryBean 인터페이스

```java
// spring-beans/.../FactoryBean.java
public interface FactoryBean<T> {

    // 실제 Bean으로 노출할 객체 반환
    @Nullable
    T getObject() throws Exception;

    // getObject()가 반환하는 타입 (타입 안전 조회에 사용)
    @Nullable
    Class<?> getObjectType();

    // true: getObject() 결과 캐시 (Singleton처럼)
    // false: getBean()마다 getObject() 재호출 (Prototype처럼)
    default boolean isSingleton() {
        return true;
    }
}
```

### 2. getBean() 시 FactoryBean 처리 경로

```java
// AbstractBeanFactory.doGetBean() — FactoryBean 분기
protected <T> T doGetBean(String name, ...) {

    // "&" 접두사 확인 → FactoryBean 자체 요청 여부
    final String beanName = transformedBeanName(name);
    // "&myFactoryBean" → "myFactoryBean" (& 제거)

    Object beanInstance = getSingleton(beanName, ...);
    // → MyFactoryBean 인스턴스 획득

    if (beanInstance instanceof FactoryBean<?> factoryBean) {
        if (!BeanFactoryUtils.isFactoryDereference(name)) {
            // 이름이 "&"로 시작하지 않음 → getObject() 호출
            object = getObjectFromFactoryBean(factoryBean, beanName, !synthetic);
        } else {
            // "&"로 시작 → FactoryBean 자체 반환
            object = beanInstance;
        }
    }
    return (T) object;
}

// getObjectFromFactoryBean()
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {

    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) {
            // 캐시 확인
            Object object = this.factoryBeanObjectCache.get(beanName);

            if (object == null) {
                // 캐시 미스 → getObject() 호출
                object = doGetObjectFromFactoryBean(factory, beanName);

                // BPP After 적용 (FactoryBean의 getObject() 결과에도 BPP 적용)
                if (shouldPostProcess) {
                    object = postProcessObjectFromFactoryBean(object, beanName);
                }

                // 캐시 저장 (isSingleton=true)
                this.factoryBeanObjectCache.put(beanName, object);
            }
            return object;
        }
    } else {
        // isSingleton=false → 캐시 없이 매번 getObject()
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (shouldPostProcess) {
            object = postProcessObjectFromFactoryBean(object, beanName);
        }
        return object;
    }
}
```

```
FactoryBean 조회 흐름 요약:

getBean("myFactoryBean"):
  1. "myFactoryBean" → MyFactoryBean 인스턴스 획득
  2. FactoryBean 감지 → getObjectFromFactoryBean() 호출
  3. isSingleton=true → factoryBeanObjectCache 확인
  4. 캐시 없음 → getObject() 실행 → Connection 생성
  5. BPP After 적용 → 캐시 저장
  6. Connection 반환

getBean("&myFactoryBean"):
  1. "&myFactoryBean" → isFactoryDereference() = true
  2. MyFactoryBean 인스턴스 그대로 반환 (getObject() 호출 안 함)
```

### 3. 실제 사용 예 — 복잡한 생성 로직 캡슐화

```java
// 암호화 키 기반 동적 객체 생성
public class EncryptedConnectionFactoryBean implements FactoryBean<Connection> {

    private String encryptedUrl;
    private String encryptedPassword;

    @Override
    public Connection getObject() throws Exception {
        // 1. 암호화된 설정값 복호화
        String url      = decrypt(encryptedUrl);
        String password = decrypt(encryptedPassword);

        // 2. SSL 인증서 설정
        Properties sslProps = loadSslCertificates();

        // 3. 커넥션 생성
        return DriverManager.getConnection(url, "user", password, sslProps);
    }

    @Override
    public Class<?> getObjectType() { return Connection.class; }

    @Override
    public boolean isSingleton() { return true; }
}

// 등록
@Bean
public EncryptedConnectionFactoryBean dbConnection() {
    EncryptedConnectionFactoryBean fb = new EncryptedConnectionFactoryBean();
    fb.setEncryptedUrl(encryptedUrl);
    fb.setEncryptedPassword(encryptedPassword);
    return fb;
}

// 사용
@Autowired Connection connection;  // getObject() 결과인 Connection 주입
```

### 4. ObjectProvider와의 역할 비교

```java
// FactoryBean: 생성 로직 담당
public class ReportGeneratorFactoryBean implements FactoryBean<ReportGenerator> {
    @Override
    public ReportGenerator getObject() {
        return new ComplexReportGenerator(loadTemplates(), buildFormatters());
    }
    @Override
    public boolean isSingleton() { return false; }  // 매번 새 인스턴스
}

// ObjectProvider: 조회 담당
@Service
class ReportService {
    @Autowired
    ObjectProvider<ReportGenerator> generatorProvider;

    public void generateReport() {
        ReportGenerator gen = generatorProvider.getObject();
        // → FactoryBean.getObject() 또는 일반 createBean() 결과를 조회
        gen.generate();
    }
}
```

```
역할 분리:
  FactoryBean
    "어떻게 만드는가" 담당
    복잡한 생성 로직을 Bean으로 캡슐화
    주로 프레임워크/라이브러리 통합에 사용

  ObjectProvider
    "언제, 어디서 꺼내는가" 담당
    이미 등록된 Bean을 지연 / 선택적으로 조회
    주로 애플리케이션 코드에서 사용

  조합:
    FactoryBean(isSingleton=false) + ObjectProvider
    → 복잡한 생성 로직 + 지연 조회
```

### 5. SmartFactoryBean — 추가 기능

```java
// SmartFactoryBean: FactoryBean 확장
public interface SmartFactoryBean<T> extends FactoryBean<T> {

    // true: 컨텍스트 시작 시 Eager 초기화
    // false(기본): 첫 getBean() 시 초기화
    default boolean isEagerInit() {
        return false;
    }

    // true: Prototype처럼 매번 새 객체 (isSingleton=false와 유사하나 의미 다름)
    default boolean isPrototype() {
        return false;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: FactoryBean 자체 vs getObject() 결과 조회

```java
@Component
public class NumberFactoryBean implements FactoryBean<Integer> {
    private static int count = 0;

    @Override
    public Integer getObject() { return ++count; }
    @Override
    public Class<?> getObjectType() { return Integer.class; }
    @Override
    public boolean isSingleton() { return true; }
}

// 테스트
Integer num1 = ctx.getBean(Integer.class);            // getObject() 호출 → 1
Integer num2 = ctx.getBean("numberFactoryBean", Integer.class); // 캐시 → 1
NumberFactoryBean fb = ctx.getBean("&numberFactoryBean", NumberFactoryBean.class);

System.out.println(num1);         // 1
System.out.println(num2);         // 1  (캐시, getObject() 재호출 안 함)
System.out.println(fb.getClass()); // NumberFactoryBean (FactoryBean 자체)
```

### 실험 2: isSingleton=false 확인

```java
public class PrototypeFactoryBean implements FactoryBean<byte[]> {
    @Override
    public byte[] getObject() { return new byte[1024]; }
    @Override
    public Class<?> getObjectType() { return byte[].class; }
    @Override
    public boolean isSingleton() { return false; }  // 캐시 안 함
}

byte[] a = ctx.getBean(byte[].class);
byte[] b = ctx.getBean(byte[].class);
System.out.println(a == b);  // false → 매번 새 배열
```

---

## 🤔 트레이드오프

```
FactoryBean vs @Bean 메서드:
  @Bean
    간결, 대부분의 경우 충분
    생성 로직이 메서드 안에 인라인
    테스트 시 메서드 직접 호출 가능

  FactoryBean
    생성 로직이 복잡해 별도 클래스로 분리할 때
    isSingleton() 제어로 Singleton/Prototype 동적 결정
    프레임워크가 생성 방식을 교체해야 할 때
    getObjectType()으로 타입 정보 제공 (제네릭 소거 우회)

현재 스프링 생태계:
  새 코드에서는 @Bean 메서드가 대부분 FactoryBean을 대체
  FactoryBean은 레거시 또는 프레임워크 내부에서 주로 사용
  외부 라이브러리(MyBatis, Hibernate) 통합 시 여전히 활용
```

---

## 📌 핵심 정리

```
FactoryBean<T>
  getObject() → 컨테이너에 등록되는 실제 Bean
  isSingleton() → true: factoryBeanObjectCache에 캐시
                  false: getBean()마다 getObject() 재호출
  "&beanName" → FactoryBean 자체 조회

getBean() 처리 경로
  "name" → FactoryBean 감지 → getObject() → (캐시) → 반환
  "&name" → FactoryBean 자체 반환

스프링 내부 사용
  ProxyFactoryBean, ScopedProxyFactoryBean
  JndiObjectFactoryBean, LocalSessionFactoryBean

ObjectProvider 차이
  FactoryBean: "어떻게 만드는가" (생성 로직)
  ObjectProvider: "언제 꺼내는가" (지연 조회)

현재 권장
  새 코드: @Bean 메서드 (FactoryBean보다 간결)
  프레임워크 통합, 복잡한 생성: FactoryBean
```

---

## 🤔 생각해볼 문제

**Q1.** `FactoryBean.isSingleton() = true`일 때 `getObject()`가 반환한 객체에 `@Transactional`이 적용되는가?

**Q2.** 다음 코드에서 `ctx.getBean(MyService.class)`를 호출했을 때 내부적으로 일어나는 일을 단계별로 설명하라.

```java
public class MyServiceFactoryBean implements FactoryBean<MyService> {
    @Override
    public MyService getObject() { return new MyServiceImpl(); }
    @Override
    public Class<?> getObjectType() { return MyService.class; }
}
```

**Q3.** `FactoryBean`을 사용하는 Bean에 `@Autowired`로 의존성을 주입할 수 있는가? 주입 시점은 언제인가?

> 💡 **해설**
>
> **Q1.** `getObjectFromFactoryBean()` 내부에서 `postProcessObjectFromFactoryBean()`을 호출하는데, 이것이 `applyBeanPostProcessorsAfterInitialization()`을 실행한다. 즉 `getObject()`가 반환한 객체에도 BPP After가 적용된다. `AbstractAutoProxyCreator`가 해당 객체의 타입을 검사하고 `@Transactional`이 있으면 프록시로 감싼다. 따라서 `@Transactional`은 적용된다. 단, `getObject()`가 반환한 객체가 인터페이스를 구현하지 않거나 `final`이면 CGLIB 적용 여부를 확인해야 한다.
>
> **Q2.** ① `AbstractBeanFactory.doGetBean("myServiceFactoryBean")` 호출. ② `getSingleton()`으로 `MyServiceFactoryBean` 인스턴스 획득(없으면 생성). ③ `instanceof FactoryBean` 감지. ④ `isFactoryDereference("myServiceFactoryBean")` = false (& 없음). ⑤ `getObjectFromFactoryBean()` 호출. ⑥ `isSingleton() = true`이므로 `factoryBeanObjectCache` 확인. ⑦ 캐시 없으면 `getObject()` 실행 → `MyServiceImpl` 인스턴스 생성. ⑧ `postProcessObjectFromFactoryBean()` → BPP After 적용. ⑨ `factoryBeanObjectCache.put(beanName, myServiceImpl)`. ⑩ `MyServiceImpl` 반환.
>
> **Q3.** 가능하다. `FactoryBean` 자체도 스프링 Bean이므로 `@Autowired`, 생성자 주입 등 모든 DI 방식을 사용할 수 있다. 주입 시점은 일반 Bean과 동일하게 `populateBean()` 단계다. `FactoryBean` 인스턴스가 먼저 생성되고 의존성이 주입된 후, `getObject()`가 호출된다. 따라서 `getObject()` 내부에서 `@Autowired` 필드를 안전하게 사용할 수 있다.

---

<div align="center">

**[⬅️ 이전: Bean Scope와 프록시](./05-bean-scope-and-proxy.md)** | **[다음: 순환 의존 내부 해결 과정 ➡️](./07-circular-dependency-resolution.md)**

</div>

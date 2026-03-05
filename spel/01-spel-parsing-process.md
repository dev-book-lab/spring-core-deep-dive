# SpEL 파싱 과정 — ExpressionParser에서 AST 평가까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ExpressionParser.parseExpression()`이 문자열을 어떻게 AST로 변환하는가?
- Tokenizer, Parser, AST 세 단계의 역할 분담은 무엇인가?
- `EvaluationContext`가 표현식 평가에 미치는 영향은?
- SpEL 표현식 파싱 결과(`Expression`)를 캐시하면 왜 성능이 향상되는가?
- `StandardEvaluationContext`와 `SimpleEvaluationContext`의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 설정 파일과 어노테이션에서 동적 표현식이 필요하다

```java
// 정적 값 — 컴파일 타임 확정
@Value("5000")
private int timeout;

// SpEL — 런타임 동적 평가
@Value("#{systemProperties['user.timezone'] ?: 'Asia/Seoul'}")
private String timezone;

@Value("#{T(java.lang.Math).random() * 100}")
private double randomValue;

@Value("#{orderService.getDefaultTimeout() * 2}")
private int computedTimeout;
```

```
SpEL이 필요한 이유:
  프로퍼티 참조: #{@beanName.property}
  메서드 호출: #{T(Class).method()}
  연산: #{value * 2 + 100}
  조건부: #{condition ? 'a' : 'b'}
  컬렉션 필터: #{list.?[value > 0]}
  → 런타임 컨텍스트 기반 동적 값 계산
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: SpEL은 항상 매번 파싱한다

```java
// ❌ 성능 낭비 패턴
public class MyService {
    private final ExpressionParser parser = new SpelExpressionParser();

    public Object evaluate(String expr, Object root) {
        // 매번 파싱 → 토크나이저 + AST 생성 반복
        return parser.parseExpression(expr).getValue(root);
    }
}
```

```
✅ 캐시 패턴:
  Expression 객체는 스레드 안전 → 재사용 가능
  Map<String, Expression>으로 파싱 결과 캐시
  → 동일 표현식 문자열: 파싱 1회 → 평가만 반복

  @EventListener condition, @Cacheable key 등
  → 스프링 내부도 MethodBasedEvaluationContext로 캐시 활용
```

### Before: StandardEvaluationContext는 항상 안전하다

```
❌ 잘못된 사용:
  사용자 입력을 StandardEvaluationContext로 직접 평가
  → T(Runtime).getRuntime().exec('rm -rf /') 같은 표현식 실행 가능

✅ 안전 가이드:
  신뢰할 수 없는 입력 → SimpleEvaluationContext 사용
  → 빈 접근, 리플렉션 제한, 타입 참조(T()) 금지
  → @Value, @Cacheable 등 시스템 내부 표현식 → StandardEvaluationContext
```

---

## ✨ 올바른 이해와 사용

### After: 파싱 파이프라인 전체 구조

```
parseExpression("#{orderService.amount * 2}")
  ↓
ExpressionParser (SpelExpressionParser)
  ↓ 1단계: Tokenization
  Tokenizer
    "#{ " → 시작 토큰 무시 (#{...} 래퍼)
    "orderService" → IDENTIFIER 토큰
    "."            → DOT 토큰
    "amount"       → IDENTIFIER 토큰
    "*"            → STAR 토큰 (곱셈)
    "2"            → LITERAL_INT 토큰
  → Token[] 배열 생성

  ↓ 2단계: Parsing (Recursive Descent)
  InternalSpelExpressionParser
    IDENTIFIER + DOT + IDENTIFIER
    → CompoundExpression(
        VariableReference("orderService"),
        PropertyOrFieldReference("amount")
      )
    + STAR + LITERAL_INT
    → OpMultiply(
        CompoundExpression(...),
        IntLiteral(2)
      )
  → SpelExpression (AST 루트)

  ↓ Expression.getValue(evaluationContext, rootObject)
  EvaluationContext
    → TypeLocator: T() 연산자 클래스 탐색
    → BeanResolver: @beanName Bean 탐색
    → PropertyAccessor: .amount 접근
    → TypeConverter: 타입 변환
    → OperatorOverloader: 연산자 오버로딩
  → AST 노드 재귀 평가 → 결과값 반환
```

---

## 🔬 내부 동작 원리

### 1. SpelExpressionParser — 진입점

```java
// SpelExpressionParser (ExpressionParser 구현)
public class SpelExpressionParser extends TemplateAwareExpressionParser {

    private final SpelParserConfiguration configuration;

    public SpelExpressionParser() {
        this(new SpelParserConfiguration());
    }

    @Override
    protected SpelExpression doParseExpression(String expressionString,
                                                ParserContext context) {
        // InternalSpelExpressionParser에 위임
        return new InternalSpelExpressionParser(this.configuration)
            .doParseExpression(expressionString, context);
    }
}
```

```java
// TemplateAwareExpressionParser — 템플릿 표현식 처리
// "Hello #{name}!" → 리터럴 + SpEL 혼합
// → 각 #{...} 부분만 SpEL로 파싱, 나머지는 LiteralExpression
```

### 2. Tokenizer — 문자열 → Token 배열

```java
// Tokenizer (내부 클래스, 직접 사용하지 않음)
class Tokenizer {
    // 입력: "orderService.amount * 2"
    // 출력: [IDENTIFIER("orderService"), DOT, IDENTIFIER("amount"),
    //         STAR, LITERAL_INT(2)]

    // 토큰 종류:
    // IDENTIFIER: 변수명, 메서드명, Bean 이름
    // LITERAL_INT, LITERAL_REAL, LITERAL_STRING: 리터럴 값
    // DOT, COMMA, COLON, SEMICOLON: 구분자
    // LPAREN, RPAREN, LSQUARE, RSQUARE: 괄호
    // PLUS, MINUS, STAR, DIV, MOD: 산술 연산자
    // EQ, NE, LT, GT, LE, GE: 비교 연산자
    // AND, OR, NOT: 논리 연산자
    // QMARK, COLON: 삼항 연산자
    // HASH: # (변수 접근)
    // AT: @ (Bean 참조)
    // T: T() 타입 연산자

    public List<Token> process() {
        while (this.pos < this.max) {
            char ch = this.charsToProcess[this.pos];
            if (isAlphabetic(ch) || ch == '_' || ch == '$') {
                lexIdentifier();
            } else if (isDigit(ch)) {
                lexNumericLiteral(ch == '0');
            } else {
                switch (ch) {
                    case '.' -> pushToken(TokenKind.DOT);
                    case '*' -> pushToken(TokenKind.STAR);
                    case '@' -> pushToken(TokenKind.BEAN_REF);
                    // ...
                }
            }
        }
        return this.tokenStream;
    }
}
```

### 3. InternalSpelExpressionParser — Token → AST

```java
// 재귀 하강 파서 (Recursive Descent Parser)
// 연산자 우선순위에 따라 재귀적으로 AST 구성

class InternalSpelExpressionParser extends Tokenizer {

    // 최상위 표현식 파싱
    private SpelNodeImpl eatExpression() {
        SpelNodeImpl expr = eatLogicalOrExpression();  // 가장 낮은 우선순위부터

        // 삼항 연산자 확인 condition ? a : b
        if (peekToken(TokenKind.QMARK)) {
            nextToken();
            SpelNodeImpl ifTrue = eatExpression();
            SpelNodeImpl ifFalse = eatExpression();
            return new Ternary(expr, ifTrue, ifFalse);
        }
        return expr;
    }

    // 논리 OR (||)
    private SpelNodeImpl eatLogicalOrExpression() {
        SpelNodeImpl expr = eatLogicalAndExpression();
        while (peekToken(TokenKind.SYMBOLIC_OR)) {
            nextToken();
            expr = new OpOr(expr, eatLogicalAndExpression());
        }
        return expr;
    }

    // ... 점진적으로 높은 우선순위 처리
    // 비교 → 덧셈/뺄셈 → 곱셈/나눗셈 → 단항 → 기본 표현식

    // 기본 표현식 (변수, 리터럴, 메서드 호출, 타입 참조 등)
    private SpelNodeImpl eatPrimaryExpression() {
        SpelNodeImpl start = eatStartNode();

        List<SpelNodeImpl> nodes = new ArrayList<>();
        // 체이닝: a.b.c → CompoundExpression([a, b, c])
        while (isPropertyAccess() || isIndexingNode()) {
            nodes.add(eatNode());
        }

        return (nodes.isEmpty() ? start
            : new CompoundExpression(start, nodes.toArray(new SpelNodeImpl[0])));
    }
}
```

### 4. AST 노드 계층 구조

```java
// SpelNode 구현체들 (주요)

// 리터럴 값
IntLiteral, LongLiteral, RealLiteral
StringLiteral          // 'hello'
BooleanLiteral         // true, false
NullLiteral            // null

// 참조
VariableReference      // #variableName
PropertyOrFieldReference // .propertyName
MethodReference        // .methodName(args)
BeanReference          // @beanName
TypeReference          // T(java.lang.String)

// 연산자
OpPlus, OpMinus, OpMultiply, OpDivide, OpModulus
OpEQ, OpNE, OpLT, OpGT, OpLE, OpGE
OpAnd, OpOr, OpNot
Ternary                // condition ? a : b
Elvis                  // value ?: default

// 컬렉션
Projection             // list.![expression]
Selection              // list.?[condition] (전체)
                       // list.^[condition] (첫 번째)
                       // list.$[condition] (마지막)
InlineList             // {1, 2, 3}
InlineMap              // {key: value}

// 기타
Assign                 // variable = value
ConstructorReference   // new Type(args)
FunctionReference      // #function(args)
```

### 5. EvaluationContext — 평가 환경

```java
// StandardEvaluationContext — 풀 기능
public class StandardEvaluationContext implements EvaluationContext {

    private Object rootObject;                    // 루트 객체
    private List<PropertyAccessor> propertyAccessors;  // 프로퍼티 접근
    private List<ConstructorResolver> constructorResolvers;
    private List<MethodResolver> methodResolvers;
    private BeanResolver beanResolver;            // @beanName 해석
    private TypeLocator typeLocator;              // T(Class) 해석
    private TypeConverter typeConverter;          // 타입 변환
    private TypeComparator typeComparator;        // 타입 비교
    private OperatorOverloader operatorOverloader;

    private Map<String, Object> variables = new HashMap<>();
    // → setVariable("name", value) → #name으로 접근

    // BeanResolver 등록 → @beanName 접근 가능
    public void setBeanResolver(BeanResolver beanResolver) {
        this.beanResolver = beanResolver;
    }
}

// SimpleEvaluationContext — 제한된 기능 (보안)
// T() 타입 참조 금지
// @beanName Bean 참조 금지 (BeanResolver 없음)
// Reflection 기반 메서드 호출 제한
// 신뢰할 수 없는 입력 평가 시 사용
SimpleEvaluationContext ctx = SimpleEvaluationContext
    .forReadOnlyDataBinding()   // 읽기 전용
    .withInstanceMethods()      // 인스턴스 메서드만 허용
    .build();
```

### 6. Expression 캐싱 — 성능 최적화

```java
// 스프링 내부 캐시 패턴 (MethodBasedEvaluationContext 등에서 활용)
public class CachedExpressionEvaluator {

    private final SpelExpressionParser parser = new SpelExpressionParser();

    // 파싱 결과 캐시 (표현식 문자열 → Expression)
    private final Map<ExpressionKey, Expression> expressionCache =
        new ConcurrentHashMap<>();

    public Expression getExpression(Map<ExpressionKey, Expression> cache,
                                     AnnotatedElementKey elementKey,
                                     String expression) {
        ExpressionKey expressionKey = createKey(elementKey, expression);
        Expression expr = cache.get(expressionKey);
        if (expr == null) {
            // 파싱 1회 → 이후는 캐시에서 재사용
            expr = this.parser.parseExpression(expression);
            cache.put(expressionKey, expr);
        }
        return expr;
    }
}

// @EventListener condition, @Cacheable key 등에서 이 패턴 사용
// → 동일 표현식 문자열의 파싱 비용 제거
// → Expression.getValue(context)만 반복 실행
```

---

## 💻 실험으로 확인하기

### 실험 1: 기본 SpEL 평가

```java
ExpressionParser parser = new SpelExpressionParser();

// 리터럴
Expression e1 = parser.parseExpression("'Hello, SpEL!'");
System.out.println(e1.getValue(String.class));  // Hello, SpEL!

// 산술
Expression e2 = parser.parseExpression("100 * 2 + 50");
System.out.println(e2.getValue(Integer.class));  // 250

// 루트 객체 프로퍼티
Order order = new Order("ORD-001", 5000);
Expression e3 = parser.parseExpression("amount * 1.1");
System.out.println(e3.getValue(order, Double.class));  // 5500.0
```

### 실험 2: EvaluationContext 활용

```java
StandardEvaluationContext ctx = new StandardEvaluationContext();
ctx.setVariable("discount", 0.1);
ctx.setBeanResolver((context, beanName) ->
    applicationContext.getBean(beanName));

Expression e = parser.parseExpression("amount * (1 - #discount)");
System.out.println(e.getValue(ctx, order, Double.class));  // 4500.0

// Bean 참조
Expression beanExpr = parser.parseExpression("@taxService.calculate(amount)");
System.out.println(beanExpr.getValue(ctx, order, Double.class));
```

### 실험 3: 컬렉션 연산

```java
List<Order> orders = List.of(
    new Order("A", 1000), new Order("B", 3000), new Order("C", 2000)
);

StandardEvaluationContext ctx = new StandardEvaluationContext();
ctx.setVariable("orders", orders);

// 필터: amount > 1500인 것만
Expression filter = parser.parseExpression("#orders.?[amount > 1500]");
List<Order> filtered = (List<Order>) filter.getValue(ctx);
// → [Order("B", 3000), Order("C", 2000)]

// 변환: id만 추출
Expression project = parser.parseExpression("#orders.![id]");
List<String> ids = (List<String>) project.getValue(ctx);
// → ["A", "B", "C"]
```

---

## 🤔 트레이드오프

```
SpEL 표현식 vs 자바 코드:
  SpEL    설정/어노테이션에서 간결하게 표현
          런타임 동적 평가 (조건부 Bean, 동적 캐시 키 등)
          파싱 비용 있음 → 캐시 필수
  자바 코드  컴파일 타임 오류 감지, IDE 지원, 리팩토링 안전
          SpEL보다 빠름 (파싱/평가 오버헤드 없음)

StandardEvaluationContext vs SimpleEvaluationContext:
  Standard  풀 기능: T(), @bean, 리플렉션, 생성자 호출
            시스템 내부 표현식에 사용
  Simple    기능 제한: 읽기 전용 / 인스턴스 메서드만
            사용자 입력 평가 시 보안 필수

표현식 캐시:
  캐시 있음: 파싱 1회 → 평가만 반복 → O(1) 조회
  캐시 없음: 매번 파싱 → 토크나이저 + AST 생성 반복
  → 반복 호출되는 표현식(이벤트 조건, 캐시 키)은 반드시 캐시
```

---

## 📌 핵심 정리

```
파싱 파이프라인
  문자열 → Tokenizer → Token[] → InternalSpelExpressionParser
  → SpelExpression (AST 루트)

Tokenizer 역할
  문자열을 의미 있는 최소 단위(Token)로 분리
  IDENTIFIER, LITERAL, DOT, STAR, BEAN_REF(at) 등

AST 구성 (재귀 하강 파서)
  연산자 우선순위대로 재귀 처리
  OpMultiply, CompoundExpression, PropertyOrFieldReference 등 노드 생성

EvaluationContext 역할
  PropertyAccessor: .property 접근 방식
  BeanResolver: @beanName Bean 반환
  TypeLocator: T(Class) 클래스 반환
  Variables: #varName 변수 저장소

보안
  StandardEvaluationContext: 풀 기능 (신뢰된 표현식만)
  SimpleEvaluationContext: 제한 (사용자 입력 평가 시)

성능
  Expression 객체는 스레드 안전 → ConcurrentHashMap으로 캐시 가능
  동일 표현식 반복 평가 시 파싱 결과 재사용 필수
```

---

## 🤔 생각해볼 문제

**Q1.** `parser.parseExpression("1 + 2 * 3")`의 평가 결과는 `7`인가 `9`인가? 파서가 어떻게 연산자 우선순위를 처리하는가?

**Q2.** `Expression` 객체를 여러 스레드에서 동시에 `getValue()` 호출해도 안전한가?

**Q3.** `@Value("#{@orderService.timeout}")`에서 `@orderService`는 어떻게 해석되는가? `BeanResolver`가 없으면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 결과는 `7`이다. 재귀 하강 파서는 연산자 우선순위를 반영한 문법으로 파싱한다. 곱셈(`*`)이 덧셈(`+`)보다 높은 우선순위를 가지므로 `2 * 3`이 먼저 `OpMultiply(2, 3)` 노드로 구성되고, `1 + result`가 `OpPlus(1, OpMultiply(2, 3))`로 구성된다. AST 평가 시 `OpMultiply`가 먼저 `6`을 반환하고 `OpPlus(1, 6) = 7`이 최종 결과가 된다.
>
> **Q2.** 안전하다. `SpelExpression` 객체는 불변 AST를 보유하며, `getValue()` 호출 시 변경 상태 없이 AST 노드를 재귀 평가한다. 단, `EvaluationContext`는 변경 가능한 상태(`variables` 등)를 포함할 수 있으므로 멀티스레드 환경에서 동일 `EvaluationContext`를 공유하면 안 된다. `Expression`은 공유하되 `EvaluationContext`는 호출마다 새로 생성하거나 ThreadLocal로 관리하는 것이 일반적인 패턴이다.
>
> **Q3.** `@orderService`는 `Tokenizer`에서 `BEAN_REF` 토큰으로 분류되고 `BeanReference` AST 노드로 파싱된다. 평가 시 `EvaluationContext.getBeanResolver().resolve(context, "orderService")`를 호출해 해당 Bean을 반환받는다. `@Value`에서 사용되는 경우 `BeanExpressionContext`가 `BeanFactory`를 기반으로 한 `BeanResolver`를 제공하므로 ApplicationContext의 `orderService` Bean이 반환된다. `BeanResolver`가 설정되지 않은 `EvaluationContext`에서 `@beanName`을 평가하면 `SpelEvaluationException: No bean resolver registered`가 발생한다.

---

<div align="center">

**[다음: @Value("${...}") vs @Value("#{...}") ➡️](./02-value-property-vs-spel.md)**

</div>

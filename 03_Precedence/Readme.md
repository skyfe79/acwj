# 3부: 연산자 우선순위

컴파일러 제작 여정의 이전 단계에서 살펴보았듯이, 파서는 반드시 프로그래밍 언어의 의미론을 강제하지는 않는다. 파서는 단지 문법의 구문과 구조적 규칙만을 강제할 뿐이다.

우리가 작성한 코드는 `2 * 3 + 4 * 5`와 같은 수식의 계산 결과가 틀리게 나왔다. 이는 코드가 다음과 같은 AST를 생성했기 때문이다:

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

하지만 실제로는 다음과 같은 구조여야 한다:

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

이 문제를 해결하려면 파서에 연산자 우선순위를 처리하는 코드를 추가해야 한다. 이를 구현하는 방법은 (최소한) 두 가지가 있다:

  + 언어의 문법에 연산자 우선순위를 명시적으로 정의하는 방법
  + 연산자 우선순위 테이블을 사용하여 기존 파서의 동작에 영향을 주는 방법

## 연산자 우선순위의 명시적 표현

이전 여정에서 우리가 다룬 문법은 다음과 같다:

```grammar
expression: number
          | expression '*' expression
          | expression '/' expression
          | expression '+' expression
          | expression '-' expression
          ;

number:  T_INTLIT
         ;
```

이 문법에서는 네 가지 수학 연산자 사이에 아무런 구분이 없다. 이제 이 문법을 개선하여 연산자들 간의 차이를 만들어보자:

```grammar
expression: additive_expression
    ;

additive_expression:
      multiplicative_expression
    | additive_expression '+' multiplicative_expression
    | additive_expression '-' multiplicative_expression
    ;

multiplicative_expression:
      number
    | number '*' multiplicative_expression
    | number '/' multiplicative_expression
    ;

number:  T_INTLIT
         ;
```

이제 두 종류의 표현식을 가지게 되었다: *덧셈* 표현식과 *곱셈* 표현식이다. 주목할 점은 이 문법이 숫자를 오직 곱셈 표현식의 일부로만 취급한다는 것이다. 이는 '*'와 '/' 연산자가 양쪽의 숫자에 더 강하게 결합하도록 만들어, 더 높은 우선순위를 갖게 한다.

덧셈 표현식은 두 가지 형태를 가질 수 있다. 하나는 곱셈 표현식 자체이고, 다른 하나는 덧셈(즉, 곱셈) 표현식 뒤에 '+'나 '-' 연산자가 오고 그 뒤에 또 다른 곱셈 표현식이 오는 형태이다. 이렇게 함으로써 덧셈 표현식은 곱셈 표현식보다 훨씬 낮은 우선순위를 갖게 된다.

## 재귀 하강 파서에서의 구현 방법

앞서 설명한 문법을 재귀 하강 파서로 어떻게 구현할 수 있을까? `expr2.c` 파일에서 이를 구현했으며, 아래에서 자세히 설명한다.

핵심은 두 개의 함수를 만드는 것이다:
1. `multiplicative_expr()` 함수는 높은 우선순위를 가진 '*'와 '/' 연산자를 처리한다.
2. `additive_expr()` 함수는 상대적으로 낮은 우선순위를 가진 '+'와 '-' 연산자를 처리한다.

두 함수는 모두 하나의 표현식과 연산자를 읽어들인다. 그리고 같은 우선순위의 연산자가 계속 나타나는 동안, 각 함수는 입력을 더 파싱하고 첫 번째 연산자를 사용해 좌측과 우측 부분을 결합한다.

다만, `additive_expr()` 함수는 더 높은 우선순위를 가진 `multiplicative_expr()` 함수에 제어를 넘겨야 한다. 이것이 파서가 연산자 우선순위를 올바르게 처리하는 방법이다.

## 덧셈 표현식 파서 (`additive_expr()`)

이 함수는 '+' 또는 '-' 이항 연산자를 루트로 하는 AST(추상 구문 트리)를 생성한다.

```c
// '+' 또는 '-' 이항 연산자를 루트로 하는 AST 트리를 반환한다
struct ASTnode *additive_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // 현재 연산자보다 높은 우선순위를 가진 왼쪽 서브트리를 가져온다
  left = multiplicative_expr();

  // 남은 토큰이 없으면 왼쪽 노드만 반환한다
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // 현재 우선순위 수준에서 연산자를 처리하는 루프
  while (1) {
    // 다음 정수 리터럴을 가져온다
    scan(&Token);

    // 현재 연산자보다 높은 우선순위를 가진 오른쪽 서브트리를 가져온다
    right = multiplicative_expr();

    // 낮은 우선순위 연산자로 두 서브트리를 결합한다
    left = mkastnode(arithop(tokentype), left, right, 0);

    // 현재 우선순위 수준의 다음 토큰을 가져온다
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // 생성된 트리를 반환한다
  return (left);
}
```

함수의 시작 부분에서는 곧바로 `multiplicative_expr()`를 호출한다. 이는 첫 번째 연산자가 높은 우선순위를 가진 '*' 또는 '/'일 수 있기 때문이다. 이 함수는 낮은 우선순위의 '+' 또는 '-' 연산자를 만날 때만 반환한다.

따라서 `while` 루프에 도달했을 때는 '+' 또는 '-' 연산자를 가지고 있음을 알 수 있다. 입력에서 더 이상 토큰이 없을 때(T_EOF 토큰을 만날 때)까지 루프를 계속한다.

루프 안에서는 다시 `multiplicative_expr()`를 호출한다. 이는 앞으로 나올 연산자들이 현재보다 높은 우선순위를 가질 수 있기 때문이다. 마찬가지로 높은 우선순위 연산자가 아닐 때 반환한다.

왼쪽과 오른쪽 서브트리를 모두 얻으면, 이전 루프에서 얻은 연산자로 이들을 결합할 수 있다. 이 과정이 반복되어, 예를 들어 `2 + 4 + 6`이라는 표현식은 다음과 같은 AST 트리가 된다:

```
       +
      / \
     +   6
    / \
   2   4
```

하지만 `multiplicative_expr()`가 자체적으로 더 높은 우선순위 연산자를 가지고 있다면, 여러 노드를 포함한 서브트리들을 결합하게 된다.

## 곱셈 표현식 파서(`multiplicative_expr()`)

이 함수는 곱셈(*) 또는 나눗셈(/) 연산자를 루트로 하는 AST(추상 구문 트리)를 생성한다.

```c
// '*' 또는 '/' 이진 연산자를 루트로 하는 AST 트리를 반환한다
struct ASTnode *multiplicative_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // 왼쪽의 정수 리터럴을 가져온다
  // 동시에 다음 토큰을 읽어들인다
  left = primary();

  // 남은 토큰이 없으면 왼쪽 노드만 반환한다
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // 토큰이 '*' 또는 '/'인 동안 반복한다
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    // 다음 정수 리터럴을 읽어들인다
    scan(&Token);
    right = primary();

    // 왼쪽 정수 리터럴과 결합한다
    left = mkastnode(arithop(tokentype), left, right, 0);

    // 현재 토큰의 정보를 갱신한다
    // 남은 토큰이 없으면 반복을 중단한다
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // 지금까지 만든 트리를 반환한다
  return (left);
}
```

이 코드는 `additive_expr()` 함수와 비슷한 구조를 가지고 있지만, 실제 정수 리터럴을 얻기 위해 `primary()` 함수를 호출한다는 점이 다르다. 또한 높은 우선순위 수준의 연산자('*'와 '/')가 있을 때만 반복문을 실행한다. 

낮은 우선순위 연산자를 만나면, 그 시점까지 만든 서브트리를 반환한다. 이 서브트리는 다시 `additive_expr()` 함수로 전달되어 낮은 우선순위 연산자를 처리하게 된다.

## 재귀 하향 파싱의 단점

연산자 우선순위를 명시적으로 처리하는 재귀 하향 파서(recursive descent parser)는 비효율적인 면이 있다. 우선순위 단계에 도달하기 위해 많은 함수 호출이 필요하기 때문이다. 또한 각 연산자 우선순위 단계를 처리하기 위한 함수가 별도로 필요하므로, 코드의 양이 많아지는 문제가 발생한다.

## 대안: 프랫 파싱(Pratt Parsing)

문법에서 명시적인 우선순위를 반복해서 작성하는 대신, 각 토큰에 우선순위 값을 지정하는 테이블을 사용하는 [프랫 파서(Pratt parser)](https://en.wikipedia.org/wiki/Pratt_parser)를 활용하면 코드의 양을 크게 줄일 수 있다.

이 시점에서 Bob Nystrom이 작성한 [Pratt Parsers: Expression Parsing Made Easy](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)를 읽어보기를 강력히 권장한다. 프랫 파서는 여전히 이해하기 어려운 개념이므로, 기본 개념에 익숙해질 때까지 최대한 많이 읽고 공부하는 것이 좋다.

## expr.c: 프랫 파싱의 구현

`expr.c` 파일에 프랫 파싱을 구현했다. 이는 `expr2.c`를 대체할 수 있는 구현이다. 구현 내용을 자세히 살펴보자.

우선 각 토큰의 우선순위를 결정하는 코드가 필요하다:

```c
// 각 토큰의 연산자 우선순위
static int OpPrec[] = { 0, 10, 10, 20, 20,    0 };
//                     EOF  +   -   *   /  INTLIT

// 이항 연산자인지 확인하고
// 해당 연산자의 우선순위를 반환한다
static int op_precedence(int tokentype) {
  int prec = OpPrec[tokentype];
  if (prec == 0) {
    fprintf(stderr, "syntax error on line %d, token %d\n", Line, tokentype);
    exit(1);
  }
  return (prec);
}
```

숫자가 클수록(예: 20) 낮은 숫자(예: 10)보다 높은 우선순위를 의미한다.

의문이 들 수 있다. `OpPrec[]`라는 참조 테이블이 있는데 왜 함수가 필요할까? 이는 문법 오류를 찾기 위해서다.

예를 들어 `234 101 + 12`와 같은 입력을 생각해보자. 처음 두 토큰을 읽을 수는 있다. 하지만 단순히 `OpPrec[]`로 두 번째 토큰 `101`의 우선순위를 확인한다면, 이것이 연산자가 아니라는 사실을 알아차리지 못한다. 따라서 `op_precedence()` 함수는 올바른 문법 구조를 강제하는 역할을 한다.

이제 각 우선순위 수준마다 함수를 만드는 대신, 연산자 우선순위 테이블을 사용하는 단일 표현식 함수를 구현했다:

```c
// 이항 연산자를 루트로 하는 AST 트리를 반환한다.
// ptp 매개변수는 이전 토큰의 우선순위다.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // 왼쪽의 정수 리터럴을 가져온다.
  // 동시에 다음 토큰을 읽어온다.
  left = primary();

  // 토큰이 더 없다면 왼쪽 노드만 반환한다
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // 현재 토큰의 우선순위가
  // 이전 토큰의 우선순위보다 높은 동안 반복한다
  while (op_precedence(tokentype) > ptp) {
    // 다음 정수 리터럴을 가져온다
    scan(&Token);

    // 현재 토큰의 우선순위로 binexpr()을 재귀 호출해
    // 서브트리를 구성한다
    right = binexpr(OpPrec[tokentype]);

    // 생성된 트리와 서브트리를 연결한다. 동시에
    // 토큰을 AST 연산으로 변환한다.
    left = mkastnode(arithop(tokentype), left, right, 0);

    // 현재 토큰의 정보를 갱신한다.
    // 토큰이 더 없다면 왼쪽 노드만 반환한다
    tokentype = Token.token;
    if (tokentype == T_EOF)
      return (left);
  }

  // 우선순위가 같거나 낮아지면
  // 지금까지 만든 트리를 반환한다
  return (left);
}
```

이 코드도 이전 파서 함수들처럼 재귀적인 구조를 가진다. 다만 이번에는 이전에 발견한 토큰의 우선순위를 매개변수로 받는다. `main()`은 가장 낮은 우선순위인 0으로 호출하지만, 재귀 호출에서는 더 높은 값을 사용한다.

이 코드가 `multiplicative_expr()` 함수와 매우 비슷하다는 점을 알 수 있다: 정수 리터럴을 읽고, 연산자의 토큰 타입을 가져온 다음, 트리를 구성하는 반복문을 실행한다.

다른 점은 반복문의 조건과 내부 코드에 있다:

```c
multiplicative_expr():
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    scan(&Token); right = primary();

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }

binexpr():
  while (op_precedence(tokentype) > ptp) {
    scan(&Token); right = binexpr(OpPrec[tokentype]);

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }
```

프랫 파서에서는 다음 연산자의 우선순위가 현재 토큰보다 높을 때, 단순히 `primary()`로 다음 정수 리터럴을 가져오는 대신 `binexpr(OpPrec[tokentype])`을 호출해 연산자 우선순위를 높인다.

우리의 우선순위 수준과 같거나 낮은 토큰을 만나면 단순히:

```c
  return (left);
```

를 실행한다. 이는 현재 함수를 호출한 연산자보다 높은 우선순위의 연산자와 노드들로 구성된 서브트리가 될 수도 있고, 동일한 우선순위의 연산자에 대한 단일 정수 리터럴이 될 수도 있다.

이제 표현식 파싱을 위한 단일 함수가 만들어졌다. 이 함수는 연산자 우선순위를 강제하는 작은 헬퍼 함수를 사용하며, 이를 통해 우리 언어의 의미론을 구현한다.

## 두 파서의 실제 구현과 테스트

두 가지 방식의 파서 프로그램을 각각 다음과 같이 생성할 수 있다:

```bash
$ make parser                                        # Pratt 파서
cc -o parser -g expr.c interp.c main.c scan.c tree.c

$ make parser2                                       # 우선순위 등반 파서
cc -o parser2 -g expr2.c interp.c main.c scan.c tree.c
```

이전 단계에서 사용했던 입력 파일들로 두 파서를 모두 테스트할 수 있다:

```bash
$ make test
(./parser input01; \
 ./parser input02; \
 ./parser input03; \
 ./parser input04; \
 ./parser input05)
15                                       # input01 실행 결과
29                                       # input02 실행 결과
syntax error on line 1, token 5          # input03 실행 결과
Unrecognised character . on line 3       # input04 실행 결과
Unrecognised character a on line 1       # input05 실행 결과

$ make test2
(./parser2 input01; \
 ./parser2 input02; \
 ./parser2 input03; \
 ./parser2 input04; \
 ./parser2 input05)
15                                       # input01 실행 결과
29                                       # input02 실행 결과
syntax error on line 1, token 5          # input03 실행 결과
Unrecognised character . on line 3       # input04 실행 결과
Unrecognised character a on line 1       # input05 실행 결과
```

## 결론 및 다음 단계

지금까지 진행한 내용을 한번 정리해보자. 현재 우리가 구현한 것은 다음과 같다:

+ 프로그래밍 언어의 토큰을 인식하고 반환하는 스캐너
+ 문법을 인식하고 구문 오류를 보고하며 추상 구문 트리(Abstract Syntax Tree)를 구축하는 파서
+ 프로그래밍 언어의 의미 체계를 구현하는 파서의 우선순위 테이블
+ 추상 구문 트리를 깊이 우선으로 탐색하며 입력된 수식의 결과를 계산하는 인터프리터

아직 컴파일러는 완성하지 않았다. 하지만 우리는 첫 컴파일러를 만드는 데 매우 가까워졌다!

컴파일러 개발 여정의 다음 단계에서는 인터프리터를 대체할 것이다. 그 자리에 각 연산자를 가진 AST 노드에 대해 x86-64 어셈블리 코드를 생성하는 번역기를 구현할 것이다. 또한 생성기가 출력하는 어셈블리 코드를 지원하기 위한 어셈블리 전처리부와 후처리부도 함께 생성할 것이다. 

[다음 단계](../04_Assembly/Readme.md)

# 2부: 파싱의 기초

컴파일러 제작 여정의 이번 파트에서는 파서(parser)의 기초를 다룬다. 1부에서 언급했듯이, 파서의 주요 임무는 입력된 코드의 구문과 구조적 요소를 인식하고, 이들이 프로그래밍 언어의 *문법*(grammar)을 준수하는지 확인하는 것이다.

현재 우리는 다음과 같은 언어 요소들을 토큰으로 스캔할 수 있다:

+ 4개의 기본 수학 연산자: `*`, `/`, `+`, `-`
+ 1개 이상의 숫자(`0`..`9`)로 구성된 10진수 정수

이제 파서가 인식할 언어의 문법을 정의해보자.

## BNF: 배커스-나우르 표기법(Backus-Naur Form)의 이해

컴퓨터 프로그래밍 언어를 다루다 보면 [BNF(배커스-나우르 표기법)](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)를 접하게 된다. 여기서는 우리가 원하는 문법을 표현하는 데 필요한 BNF 문법의 기본 개념만 다룬다.

이제 정수를 사용한 수식을 표현하는 문법을 만들어보자. BNF로 표현한 문법은 다음과 같다:

```bnf
expression: number
          | expression '*' expression
          | expression '/' expression
          | expression '+' expression
          | expression '-' expression
          ;

number:  T_INTLIT
         ;
```

수직 막대(`|`)는 문법의 여러 선택지를 구분한다. 위 문법은 다음과 같은 의미를 가진다:

- expression(수식)은 다음 중 하나가 될 수 있다:
  - 하나의 숫자
  - 두 수식을 '*' 기호로 연결한 것
  - 두 수식을 '/' 기호로 연결한 것
  - 두 수식을 '+' 기호로 연결한 것
  - 두 수식을 '-' 기호로 연결한 것
- number(숫자)는 항상 T_INTLIT 토큰으로 표현한다

이 BNF 정의에서 주목할 점은 재귀적 구조다. 수식은 다른 수식을 참조하면서 정의된다. 하지만 재귀는 무한히 계속되지 않는다. 수식이 숫자가 되는 순간, 그 숫자는 T_INTLIT 토큰으로 표현되어 재귀가 종료된다.

BNF에서 사용하는 주요 용어는 다음과 같다:

- "expression"과 "number"는 '비단말 기호'다. 문법 규칙으로 생성되는 요소이기 때문이다.
- T_INTLIT는 '단말 기호'다. 문법 규칙으로 정의되지 않고 이미 인식된 토큰이기 때문이다.
- 마찬가지로 네 개의 수학 연산자 토큰도 단말 기호에 속한다.
 
## 재귀 하향 파싱

우리가 만들 언어의 문법이 재귀적 구조를 가지므로, 재귀적 방식으로 파싱하는 것이 합리적이다. 파싱 과정은 먼저 토큰을 읽고 *다음 토큰을 미리 확인*하는 방식으로 진행한다. 다음 토큰의 종류에 따라 입력을 파싱하기 위한 경로를 결정할 수 있다. 이 과정에서 이미 호출된 함수를 다시 재귀적으로 호출해야 할 수도 있다.

우리의 경우, 모든 수식은 숫자로 시작하며 그 뒤에 수학 연산자가 올 수 있다. 연산자 뒤에는 단일 숫자만 올 수도 있고, 완전히 새로운 수식이 시작될 수도 있다. 이런 구조를 어떻게 재귀적으로 파싱할 수 있을까?

다음과 같은 의사코드로 표현할 수 있다:

```python
def expression():
    # 첫 번째 토큰이 숫자인지 확인한다. 숫자가 아니면 오류를 발생시킨다
    # 다음 토큰을 가져온다
    # 입력의 끝에 도달했다면 반환한다 (기본 조건)
    # 그렇지 않다면 expression() 함수를 다시 호출한다
```

이제 이 함수를 `2 + 3 - 5 T_EOF` 입력에 적용해보자. 여기서 `T_EOF`는 입력의 끝을 나타내는 토큰이다. 각 `expression()` 호출에 번호를 매겨 살펴보자.

```python
expression0:
    2를 스캔한다 (숫자임을 확인)
    다음 토큰 +를 가져온다 (T_EOF가 아님)
    expression() 호출

        expression1:
            3을 스캔한다 (숫자임을 확인)
            다음 토큰 -를 가져온다 (T_EOF가 아님)
            expression() 호출

                expression2:
                    5를 스캔한다 (숫자임을 확인)
                    다음 토큰 T_EOF를 가져온다, expression2에서 반환

            expression1에서 반환
    expression0에서 반환
```

이처럼 함수가 `2 + 3 - 5 T_EOF` 입력을 재귀적으로 성공적으로 파싱했다.

물론 아직 입력에 대해 어떤 처리도 하지 않았지만, 그것은 파서의 역할이 아니다. 파서의 주요 임무는 입력을 *인식*하고 문법 오류를 경고하는 것이다. 입력의 *의미 분석*, 즉 입력의 의미를 이해하고 실행하는 것은 다른 부분의 역할이다.

> 나중에 이것이 실제로는 완전히 사실이 아님을 알게 될 것이다. 종종 문법 분석과 의미 분석을 함께 수행하는 것이 더 합리적이다.

## 추상 구문 트리

의미 분석을 수행하려면 인식된 입력을 해석하거나 어셈블리 코드와 같은 다른 형식으로 변환하는 코드가 필요하다. 이번 여정에서는 입력을 해석하는 인터프리터를 구축할 것이다. 하지만 그 전에 먼저 입력을 추상 구문 트리(Abstract Syntax Tree, AST)로 변환하는 과정을 살펴볼 것이다.

AST에 대한 이해를 돕기 위해 다음 글을 읽어보기를 강력히 권장한다:

+ Vaidehi Joshi가 작성한 [AST로 구문 분석 능력 향상하기](https://medium.com/basecs/leveling-up-ones-parsing-game-with-asts-d7a6fc2400ff)

이 글은 AST의 목적과 구조를 명확하게 설명하는 잘 쓰인 글이다. 읽고 오시면 계속 설명을 이어가도록 하겠다.

우리가 구축할 AST의 각 노드 구조는 `defs.h`에 다음과 같이 정의되어 있다:

```c
// AST 노드 타입
enum {
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE, A_INTLIT
};

// 추상 구문 트리 구조체
struct ASTnode {
  int op;                               // 이 트리에서 수행할 "연산"
  struct ASTnode *left;                 // 왼쪽과 오른쪽 자식 트리
  struct ASTnode *right;
  int intvalue;                         // A_INTLIT일 경우의 정수값
};
```

`A_ADD`와 `A_SUBTRACT` 같은 `op` 값을 가진 AST 노드들은 `left`와 `right`가 가리키는 두 개의 자식 AST를 가진다. 이후 단계에서 이 하위 트리들의 값을 더하거나 뺄 것이다.

반면 `A_INTLIT`이라는 `op` 값을 가진 AST 노드는 정수값을 나타낸다. 이 노드는 자식 트리를 가지지 않으며, 단지 `intvalue` 필드에 값을 저장한다.

## AST 노드와 트리 구축하기

`tree.c` 파일은 AST(추상 구문 트리)를 구축하는 함수들을 포함한다. 가장 일반적인 함수인 `mkastnode()`는 AST 노드의 네 가지 필드 값을 모두 받는다. 이 함수는 새로운 노드를 할당하고, 필드 값을 채운 다음 노드의 포인터를 반환한다.

```c
// 일반적인 AST 노드를 생성하고 반환한다
struct ASTnode *mkastnode(int op, struct ASTnode *left,
                         struct ASTnode *right, int intvalue) {
  struct ASTnode *n;

  // 새로운 AST 노드에 메모리를 할당한다
  n = (struct ASTnode *) malloc(sizeof(struct ASTnode));
  if (n == NULL) {
    fprintf(stderr, "mkastnode()에서 메모리 할당 실패\n");
    exit(1);
  }
  // 필드 값들을 복사하고 노드를 반환한다
  n->op = op;
  n->left = left;
  n->right = right;
  n->intvalue = intvalue;
  return (n);
}
```

이 기본 함수를 토대로, 자식 노드가 없는 리프 AST 노드를 만드는 함수와 단일 자식을 가진 AST 노드를 만드는 더 구체적인 함수들을 작성할 수 있다.

```c
// AST 리프 노드를 생성한다
struct ASTnode *mkastleaf(int op, int intvalue) {
  return (mkastnode(op, NULL, NULL, intvalue));
}

// 단항 AST 노드를 생성한다: 자식이 하나만 있다
struct ASTnode *mkastunary(int op, struct ASTnode *left, int intvalue) {
  return (mkastnode(op, left, NULL, intvalue));
}
```

## AST의 목적

AST(추상 구문 트리)는 프로그램이 인식한 각각의 표현식을 저장하는 자료구조다. 이를 통해 나중에 재귀적으로 트리를 순회하면서 표현식의 최종 값을 계산할 수 있다. 이때 수학 연산자의 우선순위를 고려해야 한다. 아래 예제를 통해 자세히 살펴보자.

`2 * 3 + 4 * 5` 라는 수식을 예로 들어보자. 곱셈은 덧셈보다 우선순위가 높다. 따라서 곱셈 연산자와 피연산자를 먼저 묶어서 계산한 후에 덧셈을 수행해야 한다.

AST 트리를 다음과 같이 구성하면:

```text
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

트리를 순회할 때 `2*3`을 먼저 계산하고, 그 다음 `4*5`를 계산한다. 이 두 결과값을 트리의 루트로 전달하여 최종적으로 덧셈을 수행하게 된다.

## 단순한 수식 파서 구현하기

스캐너에서 생성한 토큰 값을 AST 노드의 연산자 값으로 그대로 사용할 수도 있다. 하지만 토큰과 AST 노드의 개념을 분리하는 것이 좋다. 우선 토큰 값을 AST 노드의 연산자 값으로 변환하는 함수부터 구현한다. 이 함수를 포함한 파서의 나머지 부분은 `expr.c` 파일에 있다:

```c
// 토큰을 AST 연산자로 변환한다
int arithop(int tok) {
  switch (tok) {
    case T_PLUS:
      return (A_ADD);
    case T_MINUS:
      return (A_SUBTRACT);
    case T_STAR:
      return (A_MULTIPLY);
    case T_SLASH:
      return (A_DIVIDE);
    default:
      fprintf(stderr, "unknown token in arithop() on line %d\n", Line);
      exit(1);
  }
}
```

switch 문의 default 구문은 주어진 토큰을 AST 노드 타입으로 변환할 수 없을 때 실행된다. 이는 파서의 문법 검사 기능 중 일부가 된다.

다음 토큰이 정수 리터럴인지 확인하고, 리터럴 값을 저장할 AST 노드를 생성하는 함수가 필요하다. 다음과 같이 구현한다:

```c
// 최소 단위 요소(primary factor)를 파싱하고
// 이를 표현하는 AST 노드를 반환한다
static struct ASTnode *primary(void) {
  struct ASTnode *n;

  // INTLIT 토큰의 경우, 이를 위한 리프 AST 노드를 만들고
  // 다음 토큰을 읽는다. 다른 토큰 타입의 경우
  // 문법 오류로 처리한다
  switch (Token.token) {
    case T_INTLIT:
      n = mkastleaf(A_INTLIT, Token.intvalue);
      scan(&Token);
      return (n);
    default:
      fprintf(stderr, "syntax error on line %d\n", Line);
      exit(1);
  }
}
```

이 코드는 전역 변수 `Token`이 존재하고, 이미 입력에서 가장 최근의 토큰을 읽었다고 가정한다. `data.h` 파일에 다음과 같이 선언한다:

```c
extern_ struct token    Token;
```

그리고 `main()` 함수에서는:

```c
  scan(&Token);                 // 입력에서 첫 번째 토큰을 읽는다
  n = binexpr();               // 파일의 수식을 파싱한다
```

이제 파서의 핵심 코드를 작성할 수 있다:

```c
// 이진 연산자를 루트로 하는 AST 트리를 반환한다
struct ASTnode *binexpr(void) {
  struct ASTnode *n, *left, *right;
  int nodetype;

  // 왼쪽의 정수 리터럴을 가져온다
  // 동시에 다음 토큰도 읽는다
  left = primary();

  // 더 이상 토큰이 없으면 왼쪽 노드만 반환한다
  if (Token.token == T_EOF)
    return (left);

  // 토큰을 노드 타입으로 변환한다
  nodetype = arithop(Token.token);

  // 다음 토큰을 읽는다
  scan(&Token);

  // 재귀적으로 오른쪽 트리를 구성한다
  right = binexpr();

  // 두 서브트리로 새로운 트리를 만든다
  n = mkastnode(nodetype, left, right, 0);
  return (n);
}
```

이 간단한 파서 코드에는 연산자 우선순위를 처리하는 부분이 전혀 없다. 현재 상태에서는 모든 연산자의 우선순위가 동일하다고 가정한다. `2 * 3 + 4 * 5`라는 수식을 파싱할 때 코드가 어떻게 동작하는지 따라가보면, 다음과 같은 AST를 만든다:

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

이는 명백히 잘못된 결과다. `4*5`를 먼저 계산해서 20을 얻고, 그 다음 `3+20`을 계산해서 23을 얻게 된다. 올바른 계산 순서는 `2*3`을 먼저 계산해서 6을 얻는 것이다.

그렇다면 왜 이렇게 구현했을까? 간단한 파서를 작성하는 것은 쉽지만, 의미 분석(semantic analysis)까지 올바르게 수행하는 것은 더 어렵다는 점을 보여주기 위해서다.

## 트리 해석하기

(틀린) AST 트리가 준비되었으니, 이제 이 트리를 해석하는 코드를 작성해보자. 여기서도 트리를 순회하는 재귀적 코드를 작성할 것이다. 다음은 의사 코드다:

```python
트리해석:
  먼저, 왼쪽 하위 트리를 해석하여 그 값을 구한다
  다음, 오른쪽 하위 트리를 해석하여 그 값을 구한다
  트리의 루트에 있는 연산자로 두 하위 트리의 값을 계산하고, 그 결과를 반환한다
```

올바른 AST 트리로 돌아가보면:

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

호출 구조는 다음과 같을 것이다:

```python
interpretTree0(+ 연산이 있는 트리):
  interpretTree1(* 연산이 있는 왼쪽 트리) 호출:
     interpretTree2(2가 있는 트리) 호출:
       수학 연산 없음, 2 반환
     interpretTree3(3이 있는 트리) 호출:
       수학 연산 없음, 3 반환
     2 * 3 실행, 6 반환

  interpretTree1(* 연산이 있는 오른쪽 트리) 호출:
     interpretTree2(4가 있는 트리) 호출:
       수학 연산 없음, 4 반환
     interpretTree3(5가 있는 트리) 호출:
       수학 연산 없음, 5 반환
     4 * 5 실행, 20 반환

  6 + 20 실행, 26 반환
```

## 추상 구문 트리 해석 코드

이 코드는 `interp.c` 파일에 구현되어 있으며, 앞서 설명한 의사코드를 따라 작성되었다.

```c
// AST를 입력으로 받아 연산자를 해석하고
// 최종 결과값을 반환한다
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  // 좌측과 우측 서브트리의 값을 계산한다
  if (n->left)
    leftval = interpretAST(n->left);
  if (n->right)
    rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:
      return (leftval + rightval);
    case A_SUBTRACT:
      return (leftval - rightval);
    case A_MULTIPLY:
      return (leftval * rightval);
    case A_DIVIDE:
      return (leftval / rightval);
    case A_INTLIT:
      return (n->intvalue);
    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

switch 문의 default 구문은 해석할 수 없는 AST 노드 타입을 만났을 때 실행된다. 이는 파서의 의미 검사(semantic checking) 부분에서 중요한 역할을 한다.

## 파서 구현하기

`main()` 함수에서 인터프리터를 호출하는 다음과 같은 코드가 있다:

```c
  scan(&Token);                 // 입력에서 첫 번째 토큰을 가져온다
  n = binexpr();               // 파일의 수식을 파싱한다 
  printf("%d\n", interpretAST(n));      // 최종 결과를 계산한다
  exit(0);
```

아래 명령어로 파서를 빌드할 수 있다:

```bash
$ make
cc -o parser -g expr.c interp.c main.c scan.c tree.c
```

파서를 테스트할 수 있는 여러 입력 파일을 제공했지만, 직접 새로운 파일을 만들 수도 있다. 계산된 결과는 정확하지 않을 수 있지만, 파서는 연속된 숫자나 연산자, 입력 끝에 숫자가 없는 경우 등의 입력 오류를 감지한다. 또한 인터프리터에 디버깅 코드를 추가하여 AST 트리 노드가 어떤 순서로 평가되는지 확인할 수 있다:

```bash
$ cat input01
2 + 3 * 5 - 8 / 3

$ ./parser input01
int 2
int 3
int 5
int 8
int 3
8 / 3
5 - 2
3 * 3
2 + 9
11

$ cat input02
13 -6+  4*
5
       +
08 / 3

$ ./parser input02
int 13 
int 6
int 4
int 5
int 8
int 3
8 / 3
5 + 2
4 * 7
6 + 28
13 - 34
-21

$ cat input03
12 34 + -56 * / - - 8 + * 2

$ ./parser input03
unknown token in arithop() on line 1

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./parser input04
Unrecognised character . on line 3

$ cat input05
23 * 456abcdefg

$ ./parser input05
Unrecognised character a on line 1
```

## 결론 및 다음 단계

파서는 프로그래밍 언어의 문법을 인식하고 컴파일러에 입력된 코드가 이 문법을 준수하는지 검사한다. 문법에 맞지 않는 코드를 발견하면 오류 메시지를 출력한다. 우리가 다루는 표현식 문법은 재귀적 구조를 가지고 있으므로, 이를 인식하기 위해 재귀 하향 파서를 구현하기로 결정했다.

현재 파서는 위 출력에서 보듯이 정상적으로 작동한다. 하지만 입력된 코드의 의미를 정확하게 해석하지는 못한다. 다시 말해, 표현식의 정확한 계산 값을 도출하지 못하는 상태이다.

컴파일러 개발 여정의 다음 단계에서는 파서를 개선하여 표현식의 의미 분석 기능을 추가할 것이다. 이를 통해 수학적 연산의 정확한 결과를 얻을 수 있게 된다. 

[다음 단계](../03_Precedence/Readme.md)

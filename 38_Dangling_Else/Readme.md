# 38장: Dangling Else 문제와 그 외

컴파일러 개발 과정에서 이 부분을 시작하며, [Dangling Else 문제](https://en.wikipedia.org/wiki/Dangling_else)를 해결하려고 했다. 하지만 실제로는 몇 가지 파싱 방식을 재구성해야 했는데, 처음부터 파싱이 잘못되었기 때문이다.

이런 상황이 발생한 이유는 기능을 추가하는 데 급급했기 때문이다. 하지만 그 과정에서 전체적인 구조를 충분히 고려하지 못했다.

이제 컴파일러에서 어떤 실수가 있었는지 살펴보자.


## FOR 문법 수정하기

먼저 FOR 루프 구조부터 살펴본다. 현재 동작은 하지만, 더 일반적인 형태로 개선할 필요가 있다.

지금까지 FOR 루프의 BNF 문법은 다음과 같았다:

```
for_statement: 'for' '(' preop_statement ';'
                         true_false_expression ';'
                         postop_statement ')' compound_statement  ;
```

하지만 [C 언어의 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)을 보면 이렇게 정의되어 있다:

```
for_statement:
        | FOR '(' expression_statement expression_statement ')' statement
        | FOR '(' expression_statement expression_statement expression ')' statement
        ;

expression_statement
        : ';'
        | expression ';'
        ;
```

여기서 `expression`은 실제로 쉼표로 구분된 표현식 목록(expression list)을 의미한다.

이것은 FOR 루프의 세 가지 절이 모두 표현식 목록이 될 수 있음을 나타낸다. 만약 완전한 C 컴파일러를 작성한다면 이 부분이 까다로울 수 있다. 하지만 우리는 C 언어의 *부분집합*을 위한 컴파일러를 작성하고 있으므로, C의 전체 문법을 처리할 필요는 없다.

따라서 FOR 루프 파서를 다음과 같이 수정했다:

```
for_statement: 'for' '(' expression_list ';'
                         true_false_expression ';'
                         expression_list ')' compound_statement  ;
```

중간 절은 참 또는 거짓을 반환하는 단일 표현식이어야 한다. 첫 번째와 마지막 절은 표현식 목록이 될 수 있다. 이렇게 하면 `tests/input80.c`에 있는 다음과 같은 FOR 루프를 처리할 수 있다:

```c
    for (x=0, y=1; x < 6; x++, y=y+2)
```


## `expression_list()` 변경 사항

위 작업을 수행하려면 `for_statement()` 파싱 함수를 수정하여 첫 번째와 세 번째 절에 있는 표현식 목록을 파싱하기 위해 `expression_list()`를 호출해야 한다.

하지만 기존 컴파일러에서 `expression_list()`는 표현식 목록을 종료하는 토큰으로 ')'만 허용한다. 따라서 `expr.c` 파일의 `expression_list()`를 수정하여 종료 토큰을 인자로 받도록 변경했다. 그리고 `stmt.c` 파일의 `for_statement()`에는 다음과 같은 코드를 추가했다:

```c
// FOR 문을 파싱하고 AST를 반환
static struct ASTnode *for_statement(void) {
  ...
  // pre_op 표현식과 ';'를 파싱
  preopAST = expression_list(T_SEMI);
  semi();
  ...
  // 조건식과 ';'를 파싱
  condAST = binexpr(0);
  semi();
  ...
  // post_op 표현식과 ')'를 파싱
  postopAST = expression_list(T_RPAREN);
  rparen();
}
```

이제 `expression_list()`의 코드는 다음과 같이 변경되었다:

```c
struct ASTnode *expression_list(int endtoken) {
  ...
  // 종료 토큰이 나올 때까지 반복
  while (Token.token != endtoken) {

    // 다음 표현식을 파싱
    child = binexpr(0);

    // A_GLUE AST 노드 생성 ...
    tree = mkastnode(A_GLUE, P_NONE, tree, NULL, child, NULL, exprcount);

    // 종료 토큰에 도달하면 중단
    if (Token.token == endtoken) break;

    // 이 시점에서 ','가 있어야 함
    match(T_COMMA, ",");
  }

  // 표현식 트리 반환
  return (tree);
}
```


## 단일 문장과 복합 문장

지금까지 우리 컴파일러를 사용하는 프로그래머에게 다음과 같은 경우에 항상 '{' ... '}'를 사용하도록 강제했다:

 + IF 문의 참 부분
 + IF 문의 거짓 부분
 + WHILE 문의 본문
 + FOR 문의 본문
 + 'case' 절 뒤의 본문
 + 'default' 절 뒤의 본문

이 목록에서 처음 네 가지 문장의 경우, 단일 문장일 때는 중괄호가 필요하지 않다. 예를 들어:

```c
  if (x>5)
    x= x - 16;
  else
    x++;
```

하지만 본문에 여러 문장이 있는 경우, 중괄호로 둘러싸인 복합 문장이 필요하다. 예를 들어:

```c
  if (x>5)
    { x= x - 16; printf("not again!\n"); }
  else
    x++;
```

그런데 알 수 없는 이유로 'switch' 문의 'case' 또는 'default' 절 뒤의 코드는 단일 문장의 집합일 수 있으며, 중괄호가 필요하지 않다!! 누가 이런 것을 괜찮다고 생각했을까? 예를 들어:

```c
  switch (x) {
    case 1: printf("statement 1\n");
            printf("statement 2\n");
            break;
    default: ...
  }
```

더 나쁜 것은, 이것도 합법적이다:

```c
  switch (x) {
    case 1: {
      printf("statement 1\n");
      printf("statement 2\n");
      break;
    }
    default: ...
  }
```

따라서 우리는 다음을 파싱할 수 있어야 한다:

 + 단일 문장
 + 중괄호로 둘러싸인 문장 집합
 + '{'로 시작하지 않지만 'case', 'default', 또는 '{'로 시작한 경우 '}'로 끝나는 문장 집합

이를 위해 `stmt.c`의 `compound_statement()`를 수정하여 인자를 받도록 했다:

```c
// 복합 문장을 파싱하고 AST를 반환한다. 
// inswitch가 참이면, 파싱을 종료하기 위해 '}', 'case', 또는 'default' 토큰을 찾는다.
// 그렇지 않으면, 파싱을 종료하기 위해 '}'만 찾는다.
struct ASTnode *compound_statement(int inswitch) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  while (1) {
    // 단일 문장을 파싱한다
    tree = single_statement();
    ...
    // 종료 토큰을 만나면 빠져나간다
    if (Token.token == T_RBRACE) return(left);
    if (inswitch && (Token.token == T_CASE || Token.token == T_DEFAULT)) return(left);
  }
}
```

이 함수가 `inswitch`가 1로 설정된 상태로 호출되면, 'switch' 문을 파싱하는 중이므로 'case', 'default', 또는 '}'를 찾아 복합 문장을 종료한다. 그렇지 않으면, 일반적인 '{' ... '}' 상황에서 파싱을 진행한다.

이제 다음도 허용해야 한다:

 + IF 문 본문 내의 단일 문장
 + WHILE 문 본문 내의 단일 문장
 + FOR 문 본문 내의 단일 문장

현재 이 모든 경우에 `compound_statement(0)`을 호출하고 있지만, 이는 닫는 '}'를 파싱하도록 강제한다. 단일 문장의 경우에는 이런 '}'가 없을 수 있다.

해결책은 IF, WHILE, FOR 파싱 코드가 단일 문장을 파싱하기 위해 `single_statement()`를 호출하도록 하는 것이다. 그리고 `single_statement()`가 여는 중괄호를 만나면 `compound_statement()`를 호출하도록 한다.

따라서 `stmt.c`에 다음과 같은 변경을 했다:

```c
// 단일 문장을 파싱하고 AST를 반환한다.
static struct ASTnode *single_statement(void) {
  ...
  switch (Token.token) {
    case T_LBRACE:
      // '{'를 만나면 복합 문장이다
      lbrace();
      stmt = compound_statement(0);
      rbrace();
      return(stmt);
}
...
static struct ASTnode *if_statement(void) {
  ...
  // 문장의 AST를 얻는다
  trueAST = single_statement();
  ...
  // 'else'가 있으면 건너뛰고 문장의 AST를 얻는다
  if (Token.token == T_ELSE) {
    scan(&Token);
    falseAST = single_statement();
  }
  ...
}
...
static struct ASTnode *while_statement(void) {
  ...
    // 문장의 AST를 얻는다.
  // 이 과정에서 루프 깊이를 업데이트한다
  Looplevel++;
  bodyAST = single_statement();
  Looplevel--;
  ...
}
...
static struct ASTnode *for_statement(void) {
  ...
  // 본문 문장을 얻는다
  // 이 과정에서 루프 깊이를 업데이트한다
  Looplevel++;
  bodyAST = single_statement();
  Looplevel--;
  ...
}
```

이제 컴파일러는 다음과 같은 코드를 받아들일 수 있다:

```c
  if (x>5)
    x= x - 16;
  else
    x++;
```


## "Dangling Else" 문제 해결

아직까지 "dangling else" 문제를 해결하지 못했다. 이 문제는 이번 여정을 시작한 이유 중 하나였다. 하지만 우리가 입력을 파싱하는 방식 덕분에 이 문제가 이미 해결되었다는 사실을 알게 되었다.

다음 프로그램을 살펴보자:

```c
  // Dangling else 테스트
  // x <= 5일 때는 아무것도 출력하지 않아야 함
  for (x=0; x < 12; x++)
    if (x > 5)
      if (x > 10)
        printf("10 < %2d\n", x);
      else
        printf(" 5 < %2d <= 10\n", x);
```

여기서 'else' 코드는 가장 가까운 'if' 문과 짝을 이루어야 한다. 따라서 위의 마지막 `printf` 문은 `x`가 5와 10 사이일 때만 출력되어야 한다. 'else' 코드는 `x > 5`의 반대 조건 때문에 실행되어서는 안 된다.

다행히도, `if_statement()` 파서에서 IF 문의 본문 이후에 'else' 토큰을 탐욕적으로 스캔하도록 구현했다:

```c
  // 문장에 대한 AST를 가져옴
  trueAST = single_statement();

  // 'else'가 있다면 건너뛰고
  // 문장에 대한 AST를 가져옴
  if (Token.token == T_ELSE) {
    scan(&Token);
    falseAST = single_statement();
  }
```

이렇게 하면 'else'가 가장 가까운 'if'와 짝을 이루게 되고, dangling else 문제가 해결된다. 그래서 내가 '{' ... '}'를 강제로 사용하도록 했던 것은 이미 해결된 문제에 대한 걱정이었던 셈이다. 한숨만 나온다.


## 디버그 출력 개선

마지막으로, 디버깅을 개선하기 위해 스캐너를 수정했다. 더 정확히 말하면, 출력되는 디버그 메시지를 개선했다. 지금까지는 오류 메시지에서 토큰의 숫자 값을 출력했었다. 예를 들어:

 + Unexpected token in parameter list: 23
 + Expecting a primary expression, got token: 19
 + Syntax error, token: 44

이런 오류 메시지를 받는 프로그래머에게는 사실상 쓸모가 없었다. `scan.c` 파일에 토큰 문자열 목록을 추가했다:

```c
// 디버깅을 위한 토큰 문자열 목록
char *Tstring[] = {
  "EOF", "=", "||", "&&", "|", "^", "&",
  "==", "!=", ",", ">", "<=", ">=", "<<", ">>",
  "+", "-", "*", "/", "++", "--", "~", "!",
  "void", "char", "int", "long",
  "if", "else", "while", "for", "return",
  "struct", "union", "enum", "typedef",
  "extern", "break", "continue", "switch",
  "case", "default",
  "intlit", "strlit", ";", "identifier",
  "{", "}", "(", ")", "[", "]", ",", ".",
  "->", ":"
};
```

`defs.h` 파일에서는 Token 구조체에 새로운 필드를 추가했다:

```c
// Token 구조체
struct token {
  int token;                    // 토큰 타입, 위의 enum 목록에서 가져옴
  char *tokstr;                 // 토큰의 문자열 버전
  int intvalue;                 // T_INTLIT의 경우, 정수 값
};
```

`scan.c` 파일의 `scan()` 함수에서, 토큰을 반환하기 직전에 해당 토큰의 문자열 버전을 설정한다:

```c
  t->tokstr = Tstring[t->token];
```

마지막으로, 여러 `fatalXX()` 호출을 수정해 현재 토큰의 `intvalue` 필드 대신 `tokstr` 필드를 출력하도록 했다. 이제 다음과 같은 메시지를 볼 수 있다:

 + Unexpected token in parameter list: ==
 + Expecting a primary expression, got token: ]
 + Syntax error, token: >>

이전보다 훨씬 더 나은 디버그 메시지다.


## 결론 및 다음 단계

컴파일러의 "매달린 else" 문제를 해결하려고 시작했지만, 결국 여러 다른 문제들도 함께 수정하게 되었다. 과정에서 "매달린 else" 문제는 실제로 존재하지 않는다는 사실을 알게 되었다.

이제 우리 컴파일러는 스스로 컴파일할 수 있는 모든 핵심 요소를 구현한 단계에 도달했다. 하지만 여전히 많은 작은 문제들을 찾아내고 수정해야 하는 "마무리" 단계에 있다.

앞으로는 컴파일러를 작성하는 방법보다는 고장 난 컴파일러를 고치는 방법에 대해 더 많이 다룰 것이다. 여러분이 앞으로의 여정을 포기하더라도 실망하지 않을 것이다. 지금까지의 여정이 유용했기를 바란다.

다음 단계에서는 현재 작동하지 않지만, 컴파일러를 스스로 컴파일하기 위해 반드시 해결해야 할 문제를 선택해 고쳐 나갈 것이다. [다음 단계](../39_Var_Initialisation_pt1/Readme.md)



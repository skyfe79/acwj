# 8부: 조건문(If Statements) 다루기

값을 비교할 수 있게 되었으니, 이제 우리의 언어에 조건문(IF statements)을 추가할 차례다. 먼저, 조건문의 일반적인 문법과 이를 어셈블리 언어로 어떻게 변환하는지 살펴보자.


## IF 구문

IF 문의 구문은 다음과 같다:

```
  if (조건이 참이면)
    첫 번째 코드 블록을 실행
  else
    다른 코드 블록을 실행
```

이를 일반적으로 어셈블리 언어로 변환하면 어떻게 될까? 실제로는 반대 비교를 수행하고, 반대 비교가 참일 때 점프 또는 분기한다:

```
       반대 비교를 수행
       참이면 L1로 점프
       첫 번째 코드 블록을 실행
       L2로 점프
L1:
       다른 코드 블록을 실행
L2:
   
```

여기서 `L1`과 `L2`는 어셈블리 언어의 레이블이다.


## 컴파일러에서 어셈블리 코드 생성하기

현재 우리는 비교 연산을 기반으로 레지스터를 설정하는 코드를 출력한다. 예를 들어:

```
   int x; x= 7 < 9;         // input04에서
```

이 코드는 다음과 같이 변환된다:

```
        movq    $7, %r8
        movq    $9, %r9
        cmpq    %r9, %r8
        setl    %r9b        // 작을 경우 설정
        andq    $255,%r9
```

하지만 IF 문의 경우, 반대 비교에서 점프해야 한다:

```
   if (7 < 9) 
```

이 코드는 다음과 같이 변환되어야 한다:

```
        movq    $7, %r8
        movq    $9, %r9
        cmpq    %r9, %r8
        jge     L1         // 크거나 같을 경우 점프
        ....
L1:
```

이 부분에서 IF 문을 구현했다. 이 프로젝트는 진행 중인 작업이기 때문에, 과정 중에 몇 가지를 되돌리고 리팩토링해야 했다. 이 과정에서의 변경 사항과 추가된 부분을 최대한 설명하려고 한다.


## 새로운 토큰과 Dangling Else 문제

우리 언어에는 여러 새로운 토큰이 필요하다. 또한 (현재로서는) [dangling else 문제](https://en.wikipedia.org/wiki/Dangling_else)를 피하고 싶다. 이를 위해 모든 문장 그룹을 '{' ... '}' 중괄호로 감싸도록 문법을 변경했다. 나는 이러한 그룹을 "복합문(compound statement)"이라고 부른다. 또한 IF 표현식을 감싸기 위해 '(' ... ')' 괄호와 'if', 'else' 키워드가 필요하다. 따라서 새로운 토큰은 (`defs.h` 파일에서) 다음과 같다:

```c
  T_LBRACE, T_RBRACE, T_LPAREN, T_RPAREN,
  // 키워드
  ..., T_IF, T_ELSE
```


## 토큰 스캐닝

단일 문자 토큰은 명확하므로 코드를 따로 제공하지 않는다. 키워드도 비교적 간단하지만, `scan.c` 파일의 `keyword()` 함수에서 스캐닝 코드를 살펴보자:

```c
  switch (*s) {
    case 'e':
      if (!strcmp(s, "else"))
        return (T_ELSE);
      break;
    case 'i':
      if (!strcmp(s, "if"))
        return (T_IF);
      if (!strcmp(s, "int"))
        return (T_INT);
      break;
    case 'p':
      if (!strcmp(s, "print"))
        return (T_PRINT);
      break;
  }
```


## 새로운 BNF 문법

우리의 문법이 점점 커지고 있어서, 이를 다시 작성했다:

```
 compound_statement: '{' '}'          // 빈 문장, 즉 아무런 문장이 없음
      |      '{' statement '}'
      |      '{' statement statements '}'
      ;

 statement: print_statement
      |     declaration
      |     assignment_statement
      |     if_statement
      ;

 print_statement: 'print' expression ';'  ;

 declaration: 'int' identifier ';'  ;

 assignment_statement: identifier '=' expression ';'   ;

 if_statement: if_head
      |        if_head 'else' compound_statement
      ;

 if_head: 'if' '(' true_false_expression ')' compound_statement  ;

 identifier: T_IDENT ;
```

`true_false_expression`의 정의는 생략했지만, 나중에 몇 가지 연산자를 추가하면 이를 포함할 예정이다.

IF 문의 문법을 주목해보자: `if_head` (else 절 없음)이거나, `if_head` 뒤에 'else'와 `compound_statement`가 오는 형태이다.

모든 다른 문장 타입을 각각의 비터미널 이름으로 분리했다. 또한 이전의 `statements` 비터미널이 이제 `compound_statement` 비터미널로 변경되었고, 이는 '{' ... '}'로 문장들을 감싸야 한다는 것을 의미한다.

이것은 `if_head`의 `compound_statement`가 '{' ... '}'로 둘러싸여 있고, 'else' 키워드 뒤의 `compound_statement`도 마찬가지라는 것을 의미한다. 따라서 중첩된 IF 문이 있다면, 다음과 같이 보일 것이다:

```
  if (condition1 is true) {
    if (condition2 is true) {
      statements;
    } else {
      statements;
    }
  } else {
    statements;
  }
```

그리고 각 'else'가 어느 'if'에 속하는지에 대한 모호함이 없다. 이는 '매달린 else 문제'를 해결한다. 나중에 '{' ... '}'를 선택적으로 만들 예정이다.


## 복합문 파싱

이전의 `void statements()` 함수는 이제 `compound_statement()`로 변경되었으며, 다음과 같은 형태를 가진다:

```c
// 복합문을 파싱하고
// 해당 AST를 반환한다
struct ASTnode *compound_statement(void) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  // 왼쪽 중괄호를 요구한다
  lbrace();

  while (1) {
    switch (Token.token) {
      case T_PRINT:
        tree = print_statement();
        break;
      case T_INT:
        var_declaration();
        tree = NULL;            // 여기서는 AST가 생성되지 않는다
        break;
      case T_IDENT:
        tree = assignment_statement();
        break;
      case T_IF:
        tree = if_statement();
        break;
    case T_RBRACE:
        // 오른쪽 중괄호를 만나면
        // 이를 건너뛰고 AST를 반환한다
        rbrace();
        return (left);
      default:
        fatald("Syntax error, token", Token.token);
    }

    // 각각의 새로운 트리에 대해,
    // left가 비어 있으면 left에 저장하고,
    // 그렇지 않으면 left와 새로운 트리를 결합한다
    if (tree) {
      if (left == NULL)
        left = tree;
      else
        left = mkastnode(A_GLUE, left, NULL, tree, 0);
    }
  }
```

먼저, 코드가 파서에게 복합문의 시작 부분에 있는 '{'를 `lbrace()`로 매칭하도록 강제하고, 끝 부분의 '}'를 `rbrace()`로 매칭했을 때만 빠져나올 수 있게 한다는 점을 주목한다.

둘째, `print_statement()`, `assignment_statement()`, `if_statement()` 모두 AST 트리를 반환하며, `compound_statement()`도 마찬가지이다. 이전 코드에서는 `print_statement()`가 표현식을 평가하기 위해 `genAST()`를 호출한 후 `genprintint()`를 호출했다. 마찬가지로 `assignment_statement()`도 할당을 수행하기 위해 `genAST()`를 호출했다.

이는 여기서는 AST 트리가 있고, 다른 곳에서는 또 다른 AST 트리가 있다는 것을 의미한다. 단일 AST 트리를 생성하고, 이를 위해 `genAST()`를 한 번만 호출해 어셈블리 코드를 생성하는 것이 더 합리적이다.

이것이 반드시 필수는 아니다. 예를 들어, SubC는 표현식에 대해서만 AST를 생성한다. 문장과 같은 언어의 구조적 부분에 대해서는 SubC는 이전 버전의 컴파일러에서 했던 것처럼 코드 생성기에 특정 호출을 한다.

나는 현재로서는 파서가 전체 입력에 대해 단일 AST 트리를 생성하기로 결정했다. 입력이 파싱되면, 하나의 AST 트리에서 어셈블리 출력을 생성할 수 있다.

나중에는 아마 각 함수마다 AST 트리를 생성할 것이다. 나중에.


## IF 문법 파싱

재귀 하강 파서를 사용하기 때문에 IF 문을 파싱하는 것은 크게 어렵지 않다:

```c
// IF 문을 파싱하고
// 선택적인 ELSE 절을 포함하여
// AST를 반환한다
struct ASTnode *if_statement(void) {
  struct ASTnode *condAST, *trueAST, *falseAST = NULL;

  // 'if' '('가 있는지 확인
  match(T_IF, "if");
  lparen();

  // 다음 표현식을 파싱하고
  // 뒤따르는 ')'를 확인한다.
  // 트리의 연산이 비교 연산자인지 확인한다.
  condAST = binexpr(0);

  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("잘못된 비교 연산자");
  rparen();

  // 복합문에 대한 AST를 얻는다
  trueAST = compound_statement();

  // 'else'가 있다면 건너뛰고
  // 복합문에 대한 AST를 얻는다
  if (Token.token == T_ELSE) {
    scan(&Token);
    falseAST = compound_statement();
  }
  // 이 문장에 대한 AST를 만들고 반환한다
  return (mkastnode(A_IF, condAST, trueAST, falseAST, 0));
}
```

현재는 `if (x-2)`와 같은 입력을 처리하고 싶지 않기 때문에, `binexpr()`에서 반환하는 바이너리 표현식의 루트가 A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE 중 하나의 비교 연산자여야 한다는 제한을 두었다.


## 세 번째 자식 노드

지난 `if_statement()` 함수의 마지막 줄에서 다음과 같이 AST 노드를 생성했다:

```c
   mkastnode(A_IF, condAST, trueAST, falseAST, 0);
```

이 코드는 **세 개**의 AST 하위 트리를 사용한다. 여기서 무슨 일이 일어나고 있는 걸까? 보다시피, IF 문은 세 개의 자식 노드를 가진다:

  + 조건을 평가하는 하위 트리
  + 바로 뒤따르는 복합문
  + 'else' 키워드 뒤에 오는 선택적 복합문

따라서 이제 세 개의 자식 노드를 가진 AST 노드 구조가 필요하다 (`defs.h` 파일에 정의됨):

```c
// AST 노드 타입
enum {
  ...
  A_GLUE, A_IF
};

// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  struct ASTnode *left;         // 왼쪽, 중간, 오른쪽 자식 트리
  struct ASTnode *mid;
  struct ASTnode *right;
  union {
    int intvalue;               // A_INTLIT 타입일 때 정수 값
    int id;                     // A_IDENT 타입일 때 심볼 슬롯 번호
  } v;
};
```

따라서 A_IF 트리는 다음과 같은 구조를 가진다:

```
                      IF
                    / |  \
                   /  |   \
                  /   |    \
                 /    |     \
                /     |      \
               /      |       \
      condition   statements   statements
```

이 구조는 IF 문의 조건, 참일 때 실행할 문장, 거짓일 때 실행할 문장을 각각의 자식 노드로 표현한다. 이렇게 하면 IF 문의 동작을 명확히 트리 구조로 나타낼 수 있다.


## AST 노드 연결하기

새로운 A_GLUE AST 노드 타입이 추가되었다. 이 노드는 어떤 용도로 사용될까? 이제 우리는 여러 문장을 포함한 단일 AST 트리를 구성해야 하므로, 이들을 연결할 방법이 필요하다.

`compound_statement()` 루프 코드의 끝 부분을 살펴보자:

```c
      if (left != NULL)
        left = mkastnode(A_GLUE, left, NULL, tree, 0);
```

새로운 서브 트리를 얻을 때마다 기존 트리에 연결한다. 따라서 다음과 같은 문장 시퀀스가 있다고 가정해 보자:

```
    stmt1;
    stmt2;
    stmt3;
    stmt4;
```

최종적으로 다음과 같은 트리가 생성된다:

```
             A_GLUE
              /  \
          A_GLUE stmt4
            /  \
        A_GLUE stmt3
          /  \
      stmt1  stmt2
```

이 트리를 왼쪽에서 오른쪽으로 깊이 우선 탐색하면, 여전히 올바른 순서로 어셈블리 코드가 생성된다.


## 범용 코드 생성기

이제 AST 노드가 여러 자식 노드를 가지게 되면서, 범용 코드 생성기가 조금 더 복잡해진다. 또한 비교 연산자의 경우, IF 문의 일부로 비교를 수행하는지(반대 비교 시 점프), 아니면 일반 표현식의 일부로 비교를 수행하는지(일반 비교 시 레지스터를 1 또는 0으로 설정)를 알아야 한다.

이를 위해 `getAST()`를 수정하여 부모 AST 노드의 연산을 전달할 수 있게 했다:

```c
// AST, 이전 rvalue를 보유하는 레지스터(있는 경우), 부모의 AST 연산을 받아
// 재귀적으로 어셈블리 코드를 생성한다.
// 트리의 최종 값을 가진 레지스터 ID를 반환한다
int genAST(struct ASTnode *n, int reg, int parentASTop) {
   ...
}
```


### 특정 AST 노드 처리하기

`genAST()` 함수는 이제 특정 AST 노드를 처리해야 한다:

```c
  // 이제 상단에서 특정 AST 노드를 처리한다
  switch (n->op) {
    case A_IF:
      return (genIFAST(n));
    case A_GLUE:
      // 각 자식 문장을 실행하고, 각 자식 이후에 레지스터를 해제한다
      genAST(n->left, NOREG, n->op);
      genfreeregs();
      genAST(n->right, NOREG, n->op);
      genfreeregs();
      return (NOREG);
  }
```

리턴하지 않으면 일반적인 이항 연산자 AST 노드를 처리한다. 단, 비교 노드는 예외다:

```c
    case A_EQ:
    case A_NE:
    case A_LT:
    case A_GT:
    case A_LE:
    case A_GE:
      // 부모 AST 노드가 A_IF인 경우, 비교 후 점프를 생성한다.
      // 그렇지 않으면 레지스터를 비교하고, 비교 결과에 따라 하나를 1 또는 0으로 설정한다.
      if (parentASTop == A_IF)
        return (cgcompare_and_jump(n->op, leftreg, rightreg, reg));
      else
        return (cgcompare_and_set(n->op, leftreg, rightreg));
```

새로운 함수 `cgcompare_and_jump()`와 `cgcompare_and_set()`에 대해서는 아래에서 설명한다.


### IF 어셈블리 코드 생성


A_IF AST 노드를 처리하기 위해 특정 함수를 사용하고, 새로운 레이블 번호를 생성하는 함수를 함께 사용한다:

```c
// 새로운 레이블 번호를 생성하고 반환
static int label(void) {
  static int id = 1;
  return (id++);
}

// IF 문과 선택적 ELSE 절에 대한 코드 생성
static int genIFAST(struct ASTnode *n) {
  int Lfalse, Lend;

  // 두 개의 레이블 생성: 하나는 거짓 조건문을 위한 레이블,
  // 다른 하나는 전체 IF 문의 끝을 위한 레이블.
  // ELSE 절이 없을 경우, Lfalse가 바로 끝 레이블이 된다!
  Lfalse = label();
  if (n->right)
    Lend = label();

  // 조건 코드를 생성한 후, 거짓 레이블로 점프하는 명령어 추가.
  // Lfalse 레이블을 레지스터로 전달하는 방식으로 처리.
  genAST(n->left, Lfalse, n->op);
  genfreeregs();

  // 참 조건문에 대한 코드 생성
  genAST(n->mid, NOREG, n->op);
  genfreeregs();

  // 선택적 ELSE 절이 있다면, 끝으로 점프하는 명령어 추가
  if (n->right)
    cgjump(Lend);

  // 거짓 레이블 추가
  cglabel(Lfalse);

  // 선택적 ELSE 절: 거짓 조건문에 대한 코드 생성 후
  // 끝 레이블 추가
  if (n->right) {
    genAST(n->right, NOREG, n->op);
    genfreeregs();
    cglabel(Lend);
  }

  return (NOREG);
}
```

이 코드는 다음과 같은 동작을 수행한다:

```c
  genAST(n->left, Lfalse, n->op);       // 조건 및 Lfalse로 점프
  genAST(n->mid, NOREG, n->op);         // 'if' 이후의 문장
  cgjump(Lend);                         // Lend로 점프
  cglabel(Lfalse);                      // Lfalse: 레이블
  genAST(n->right, NOREG, n->op);       // 'else' 이후의 문장
  cglabel(Lend);                        // Lend: 레이블
```


## x86-64 코드 생성 함수

이제 몇 가지 새로운 x86-64 코드 생성 함수를 살펴보자. 이 함수들은 이전에 작성한 여섯 개의 `cgXXX()` 비교 함수를 대체한다.

일반적인 비교 함수의 경우, AST 연산을 전달하여 해당하는 x86-64 `set` 명령어를 선택한다:

```c
// 비교 명령어 목록
// AST 순서: A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE
static char *cmplist[] =
  { "sete", "setne", "setl", "setg", "setle", "setge" };

// 두 레지스터를 비교하고 조건이 참이면 설정
int cgcompare_and_set(int ASTop, int r1, int r2) {

  // AST 연산의 범위를 확인
  if (ASTop < A_EQ || ASTop > A_GE)
    fatal("Bad ASTop in cgcompare_and_set()");

  fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  fprintf(Outfile, "\t%s\t%s\n", cmplist[ASTop - A_EQ], breglist[r2]);
  fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r2], reglist[r2]);
  free_register(r1);
  return (r2);
}
```

또한 x86-64 명령어 `movzbq`를 발견했다. 이 명령어는 한 레지스터의 최하위 바이트를 가져와 64비트 레지스터에 맞게 확장한다. 이제 이전 코드의 `and $255` 대신 이 명령어를 사용한다.

레이블을 생성하고 해당 레이블로 점프하는 함수도 필요하다:

```c
// 레이블 생성
void cglabel(int l) {
  fprintf(Outfile, "L%d:\n", l);
}

// 레이블로 점프
void cgjump(int l) {
  fprintf(Outfile, "\tjmp\tL%d\n", l);
}
```

마지막으로, 비교를 수행하고 반대 조건에 따라 점프하는 함수가 필요하다. AST 비교 노드 타입을 사용하여 반대 조건으로 점프한다:

```c
// 반전된 점프 명령어 목록
// AST 순서: A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE
static char *invcmplist[] = { "jne", "je", "jge", "jle", "jg", "jl" };

// 두 레지스터를 비교하고 조건이 거짓이면 점프
int cgcompare_and_jump(int ASTop, int r1, int r2, int label) {

  // AST 연산의 범위를 확인
  if (ASTop < A_EQ || ASTop > A_GE)
    fatal("Bad ASTop in cgcompare_and_set()");

  fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  fprintf(Outfile, "\t%s\tL%d\n", invcmplist[ASTop - A_EQ], label);
  freeall_registers();
  return (NOREG);
}
```


## IF 문 테스트

`input05` 파일을 컴파일하기 위해 `make test`를 실행한다:

```c
{
  int i; int j;
  i=6; j=12;
  if (i < j) {
    print i;
  } else {
    print j;
  }
}
```

결과로 생성된 어셈블리 출력은 다음과 같다:

```
        movq    $6, %r8
        movq    %r8, i(%rip)    # i=6;
        movq    $12, %r8
        movq    %r8, j(%rip)    # j=12;
        movq    i(%rip), %r8
        movq    j(%rip), %r9
        cmpq    %r9, %r8        # %r8-%r9 비교, 즉 i-j
        jge     L1              # i >= j이면 L1로 점프
        movq    i(%rip), %r8
        movq    %r8, %rdi       # print i;
        call    printint
        jmp     L2              # else 코드 건너뛰기
L1:
        movq    j(%rip), %r8
        movq    %r8, %rdi       # print j;
        call    printint
L2:
```

그리고 `make test`의 결과는 다음과 같다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c
./comp1 input05
cc -o out out.s
./out
6                   # 6이 12보다 작기 때문에
```


## 결론 및 다음 단계

우리는 IF 문을 통해 언어에 첫 번째 제어 구조를 추가했다. 이 과정에서 기존 코드 일부를 다시 작성해야 했고, 아직 완전한 아키텍처 계획이 없기 때문에 앞으로도 더 많은 부분을 수정해야 할 것이다.

이번 단계에서 주목할 점은 IF 문의 결정을 내리기 위해 일반 비교 연산자와 반대 방향으로 비교를 수행해야 한다는 것이다. 이를 해결하기 위해 각 AST 노드가 부모 노드의 타입을 알 수 있도록 했다. 이제 비교 노드는 부모가 A_IF 노드인지 아닌지를 확인할 수 있다.

Nils Holm이 SubC를 구현할 때는 다른 접근 방식을 선택했다. 같은 문제에 대한 다른 해결책을 보기 위해 그의 코드를 살펴보는 것도 좋다.

컴파일러 작성 여정의 다음 단계에서는 또 다른 제어 구조인 WHILE 루프를 추가할 것이다. [다음 단계](../09_While_Loops/Readme.md)



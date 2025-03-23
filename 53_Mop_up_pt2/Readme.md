# 53부: 마무리 작업, 2편

컴파일러 작성 여정의 이번 편에서는 컴파일러 자체의 소스 코드에서 사용하는 몇 가지 번거로운 문제를 해결한다.


## 연속된 문자열 리터럴

C 언어는 문자열 리터럴을 여러 줄에 걸쳐 선언하거나 여러 문자열로 나눌 수 있다. 예를 들어:

```c
  char *c= "hello " "there, "
           "how " "are " "you?";
```

이 문제를 어휘 분석기에서 해결할 수도 있다. 하지만 이를 구현하려고 많은 시간을 들였다. 문제는 C 전처리기를 다루는 과정에서 코드가 복잡해졌고, 연속된 문자열 리터럴을 허용하는 깔끔한 방법을 찾지 못했다는 점이다.

내가 선택한 해결책은 파서에서 처리하는 것이다. 코드 생성기의 도움을 약간 받아 `expr.c` 파일의 `primary()` 함수에서 문자열 리터럴을 다루는 코드를 다음과 같이 수정했다:

```c
  case T_STRLIT:
    // STRLIT 토큰에 대해 어셈블리 코드를 생성한다.
    id = genglobstr(Text, 0);   // 0은 레이블을 생성하라는 의미

    // 연속된 STRLIT 토큰이 있다면 그 내용을 현재 문자열에 추가한다.
    while (1) {
      scan(&Peektoken);
      if (Peektoken.token != T_STRLIT) break;
      genglobstr(Text, 1);      // 1은 레이블을 생성하지 말라는 의미
      scan(&Token);             // 토큰을 건너뛴다.
    }

    // 이제 리프 AST 노드를 생성한다. id는 문자열의 레이블이다.
    genglobstrend();
    n = mkastleaf(A_STRLIT, pointer_to(P_CHAR), NULL, NULL, id);
    break;
```

`genglobstr()` 함수는 이제 두 번째 인자를 받는다. 이 인자는 이 문자열이 첫 번째 부분인지, 아니면 연속된 부분인지를 알려준다. 또한 `genglobstrend()` 함수는 문자열 리터럴을 NUL로 종료하는 역할을 한다.


## 빈 문장

C 언어는 빈 문장과 빈 복합 문장을 모두 허용한다. 예를 들어:

```c
  while ((c=getc()) != 'x') ;           // ';'는 빈 문장이다

  int fred() { }                        // 빈 본문을 가진 함수
```

컴파일러에서 이 두 가지를 모두 사용하므로, 두 경우를 모두 지원해야 한다. `stmt.c` 파일에서 이제 다음과 같이 처리한다:

```c
static struct ASTnode *single_statement(void) {
  struct ASTnode *stmt;
  struct symtable *ctype;

  switch (Token.token) {
    case T_SEMI:
      // 빈 문장 처리
      semi();
      break;
    ...
  }
  ...
}

struct ASTnode *compound_statement(int inswitch) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  while (1) {
    // 끝 토큰을 만나면 종료한다. 빈 복합 문장을 허용하기 위해 먼저 확인한다
    if (Token.token == T_RBRACE)
      return (left);
    ...
  }
  ...
}
```

이렇게 하면 두 가지 문제점을 모두 해결할 수 있다.


## 재선언된 심볼

C 언어에서는 전역 변수를 나중에 `extern`으로 선언하거나, `extern` 변수를 나중에 전역 변수로 재선언할 수 있다. 하지만 두 선언의 타입은 반드시 일치해야 한다. 또한 심볼 테이블에 하나의 심볼만 남도록 해야 한다. C_GLOBAL과 C_EXTERN 항목이 동시에 존재하면 안 된다.

`stmt.c` 파일에 `is_new_symbol()`이라는 새로운 함수를 추가했다. 이 함수는 변수 이름을 파싱하고 심볼 테이블에서 해당 변수를 찾은 후에 호출한다. 따라서 `sym`은 NULL일 수도 있고(기존 변수가 없는 경우), NULL이 아닐 수도 있다(기존 변수가 존재하는 경우).

심볼이 존재한다면, 안전하게 재선언할 수 있는지 확인하는 과정이 복잡하다.

```c
// 이미 존재할 수 있는 심볼에 대한 포인터가 주어졌을 때,
// 이 심볼이 존재하지 않으면 true를 반환한다. 이 함수를 사용해
// extern을 전역 변수로 변환한다.
int is_new_symbol(struct symtable *sym, int class, 
                  int type, struct symtable *ctype) {

  // 기존 심볼이 없으므로 새로운 심볼이다.
  if (sym==NULL) return(1);

  // 전역 변수와 extern 비교: 두 경우가 일치하면 새로운 심볼이 아니며,
  // 클래스를 전역 변수로 변환할 수 있다.
  if ((sym->class== C_GLOBAL && class== C_EXTERN)
      || (sym->class== C_EXTERN && class== C_GLOBAL)) {

      // 타입이 일치하지 않으면 문제가 발생한다.
      if (type != sym->type)
        fatals("Type mismatch between global/extern", sym->name);

      // 구조체/공용체의 경우, ctype도 비교한다.
      if (type >= P_STRUCT && ctype != sym->ctype)
        fatals("Type mismatch between global/extern", sym->name);

      // 여기까지 도달하면 타입이 일치하므로 심볼을 전역 변수로 표시한다.
      sym->class= C_GLOBAL;
      // 심볼이 새로운 것이 아니라고 반환한다.
      return(0);
  }

  // 여기까지 도달했다면 중복된 심볼이다.
  fatals("Duplicate global variable declaration", sym->name);
  return(-1);   // -Wall 경고를 피하기 위해 반환한다.
}
```

이 코드는 직관적이지만 우아하지는 않다. 또한 재선언된 `extern` 심볼은 전역 심볼로 변환된다. 이는 심볼 테이블에서 심볼을 제거하고 새로운 전역 심볼을 추가할 필요가 없게 한다.


## 논리 연산의 피연산자 타입

다음으로 발견한 버그는 다음과 같은 코드에서 발생했다.

```c
  int *x;
  int y;

  if (x && y > 12) ...
```

컴파일러는 `binexpr()` 함수 내에서 `&&` 연산을 평가한다. 이를 위해 컴파일러는 이항 연산자의 양쪽 피연산자 타입이 호환되는지 확인한다. 만약 위 연산자가 `+`라면, 두 타입은 명백히 호환되지 않는다. 하지만 논리 비교 연산의 경우, 두 값을 *AND* 연산으로 결합할 수 있다.

이 문제를 해결하기 위해 `types.c` 파일의 `modify_type()` 함수 상단에 코드를 추가했다. `&&` 또는 `||` 연산을 수행할 때, 연산자의 양쪽 피연산자는 정수 타입이나 포인터 타입이어야 한다.

```c
struct ASTnode *modify_type(struct ASTnode *tree, int rtype,
                            struct symtable *rctype, int op) {
  int ltype;
  int lsize, rsize;

  ltype = tree->type;

  // A_LOGOR와 A_LOGAND의 경우, 양쪽 타입이 정수 타입이나 포인터 타입이어야 함
  if (op==A_LOGOR || op==A_LOGAND) {
    if (!inttype(ltype) && !ptrtype(ltype))
      return(NULL);
    if (!inttype(ltype) && !ptrtype(rtype))
      return(NULL);
    return (tree);
  }
  ...
}
```

또한 `&&`와 `||` 연산을 잘못 구현했다는 사실을 깨달았기 때문에, 이 부분도 수정해야 한다. 당장은 아니지만, 곧 고칠 예정이다.


## 값을 반환하지 않는 경우

C 언어에는 void 함수에서 값을 반환하지 않고 그냥 빠져나가는 기능이 있다. 하지만 현재 파서는 `return` 키워드 뒤에 괄호와 표현식이 오는 것을 기대한다.

`stmt.c` 파일의 `return_statement()` 함수는 다음과 같이 수정했다:

```c
// return 문을 파싱하고 AST를 반환한다
static struct ASTnode *return_statement(void) {
  struct ASTnode *tree= NULL;

  // 'return' 키워드가 있는지 확인한다
  match(T_RETURN, "return");

  // 반환 값이 있는지 확인한다
  if (Token.token == T_LPAREN) {
    // 괄호와 표현식을 파싱하는 코드
    ...
  } else {
    if (Functionid->type != P_VOID)
      fatal("void 함수가 아닌 경우 반드시 값을 반환해야 한다");
  }

    // A_RETURN 노드를 추가한다
  tree = mkastunary(A_RETURN, P_NONE, NULL, tree, NULL, 0);

  // 세미콜론을 확인한다
  semi();
  return (tree);
}
```

`return` 토큰 뒤에 왼쪽 괄호가 없으면 `tree` 표현식을 NULL로 설정한다. 또한 현재 함수가 void 형식인지 확인하고, 그렇지 않으면 치명적 오류를 출력한다.

이제 `return` 함수를 파싱했으므로 NULL 자식을 가진 A_RETURN AST 노드를 생성할 수 있다. 이제 코드 생성기에서 이를 처리해야 한다. `cg.c` 파일의 `cgreturn()` 함수 상단은 다음과 같다:

```c
// 함수에서 값을 반환하는 코드를 생성한다
void cgreturn(int reg, struct symtable *sym) {

  // 반환할 값이 있는 경우에만 값을 반환한다
  if (reg != NOREG) {
    ..
  }

  cgjump(sym->st_endlabel);
}
```

자식 AST 트리가 없으면 표현식의 값을 담은 레지스터가 없다. 따라서 함수의 종료 레이블로 점프하는 코드만 출력한다.


## 결론 및 다음 단계

컴파일러에서 다섯 가지 사소한 문제를 해결했다. 이 문제들은 컴파일러가 스스로를 컴파일할 수 있도록 하기 위해 반드시 고쳐야 할 것들이다.

`&&`와 `||` 연산자와 관련된 문제도 확인했다. 하지만 이를 해결하기 전에 더 시급한 문제를 먼저 처리해야 한다. 사용 가능한 CPU 레지스터가 제한적이기 때문에, 큰 소스 파일을 컴파일할 때 레지스터가 부족한 상황이 발생한다.

컴파일러 개발 과정의 다음 단계에서는 레지스터 스필(register spills)을 구현해야 한다. 이 작업을 미뤄왔지만, 이제는 컴파일러가 스스로를 컴파일할 때 발생하는 치명적인 오류 대부분이 레지스터 관련 문제다. 따라서 이 문제를 해결할 때가 되었다. [다음 단계](../54_Reg_Spills/Readme.md)



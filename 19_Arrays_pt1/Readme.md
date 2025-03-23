# 19부: 배열, 1부

> *대학교 1학년 때 강의를 해주신 교수님은 스코틀랜드 분이셨고, 상당히 강한 억양을 가지고 계셨다. 1학기 3~4주차 쯤 되자, 수업 시간에 "Hurray!"라는 말을 자주 하시기 시작했다. 약 20분이 지나서야 그분이 "array"를 말하고 계셨다는 것을 깨달았다.*

이번 파트에서는 컴파일러에 배열을 추가하는 작업을 시작한다. 먼저 어떤 기능을 구현해야 할지 파악하기 위해 간단한 C 프로그램을 작성해 보았다:

```c
  int ary[5];               // 5개의 int 엘리먼트를 가진 배열
  int *ptr;                 // int를 가리키는 포인터

  ary[3]= 63;               // ary[3] (lvalue)에 63을 할당
  ptr   = ary;              // ptr을 ary의 시작 주소로 설정
  // ary= ptr;              // 오류: 배열 타입의 표현식에 할당할 수 없음
  ptr   = &ary[0];          // ptr을 ary의 시작 주소로 설정, ary[0]은 lvalue
  ptr[4]= 72;               // ptr을 배열처럼 사용, ptr[4]는 lvalue
```

배열은 포인터와 비슷한 면이 있다. "[ ]" 문법을 사용해 포인터와 배열 모두를 역참조하여 특정 엘리먼트에 접근할 수 있다. 배열의 이름을 "포인터"처럼 사용할 수 있고, 배열의 시작 주소를 포인터에 저장할 수도 있다. 배열 내 특정 엘리먼트의 주소를 얻을 수도 있다. 하지만 한 가지 할 수 없는 것은 포인터로 배열의 시작 주소를 "덮어쓰기"하는 것이다. 배열의 엘리먼트는 변경 가능하지만, 배열의 시작 주소는 변경할 수 없다.

이번 파트에서 구현할 내용은 다음과 같다:

 + 초기화 리스트 없이 고정 크기로 배열을 선언하는 기능
 + 표현식에서 배열 인덱스를 rvalue로 사용하는 기능
 + 할당문에서 배열 인덱스를 lvalue로 사용하는 기능

또한, 각 배열에서 한 차원 이상은 구현하지 않을 것이다.


## 표현식에서 괄호 사용

언젠가 `*(ptr + 2)`와 같은 표현을 시도해 보고 싶다. 이 표현은 `ptr[2]`와 동일한 결과를 가져야 한다. 하지만 아직 표현식에서 괄호를 허용하지 않았기 때문에, 이제 괄호를 추가할 때가 되었다.


### C 문법의 BNF 표현

웹에는 Jeff Lee가 1985년 작성한 [C 언어의 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html) 페이지가 있다. 이 페이지는 아이디어를 얻거나 실수를 줄이는 데 유용한 참고 자료로 활용할 수 있다.

특히, 이 문법은 C 언어의 이항 연산자 우선순위를 직접 구현하는 대신, 재귀적 정의를 사용해 우선순위를 명확히 보여준다. 예를 들어:

```
additive_expression
        : multiplicative_expression
        | additive_expression '+' multiplicative_expression
        | additive_expression '-' multiplicative_expression
        ;
```

위 정의는 `additive_expression`을 파싱할 때 `multiplicative_expression`으로 내려가도록 함으로써, '*'와 '/' 연산자가 '+'와 '-' 연산자보다 높은 우선순위를 가지도록 한다.

표현식 우선순위 계층의 최상위에는 다음과 같은 정의가 있다:

```
primary_expression
        : IDENTIFIER
        | CONSTANT
        | STRING_LITERAL
        | '(' expression ')'
        ;
```

이미 `primary()` 함수가 T_INTLIT와 T_IDENT 토큰을 찾기 위해 호출되고 있으며, 이는 Jeff Lee의 C 문법과 일치한다. 따라서 이곳이 괄호를 포함한 표현식을 파싱하기에 가장 적합한 위치다.

이미 T_LPAREN과 T_RPAREN 토큰이 언어에 존재하므로, 렉서 스캐너를 수정할 필요는 없다.

대신, `primary()` 함수를 약간 수정해 추가적인 파싱을 수행한다:

```c
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  ...

  switch (Token.token) {
  case T_INTLIT:
  ...
  case T_IDENT:
  ...
  case T_LPAREN:
    // 괄호로 시작하는 표현식, '('를 건너뛴다.
    // 표현식과 오른쪽 괄호를 스캔한다.
    scan(&Token);
    n = binexpr(0);
    rparen();
    return (n);

    default:
    fatald("기본 표현식이 필요하지만, 토큰을 발견함", Token.token);
  }

  // 다음 토큰을 스캔하고 리프 노드를 반환한다.
  scan(&Token);
  return (n);
}
```

이렇게 하면 괄호를 포함한 표현식을 처리할 수 있다. 새 코드에서 `rparen()`을 명시적으로 호출하고 `return`을 사용한 이유는, 스위치 문을 벗어나면 마지막 `return` 앞의 `scan(&Token);`이 여는 '(' 토큰과 짝을 이루는 ')' 토큰을 엄격히 강제하지 않기 때문이다.

`test/input19.c` 테스트는 괄호가 제대로 동작하는지 확인한다:

```c
  a= 2; b= 4; c= 3; d= 2;
  e= (a+b) * (c+d);
  printint(e);
```

이 코드는 30, 즉 `6 * 5`를 출력해야 한다.


## 심볼 테이블 변경 사항

현재 심볼 테이블에는 단일 값을 가지는 스칼라 변수와 함수가 있다. 이제 배열을 추가할 차례다. 나중에 `sizeof()` 연산자를 사용해 각 배열의 요소 개수를 가져오고 싶을 것이다. `defs.h` 파일의 변경 사항은 다음과 같다:

```c
// 구조적 타입
enum {
  S_VARIABLE, S_FUNCTION, S_ARRAY
};

// 심볼 테이블 구조체
struct symtable {
  char *name;                   // 심볼 이름
  int type;                     // 심볼의 기본 타입
  int stype;                    // 심볼의 구조적 타입
  int endlabel;                 // S_FUNCTION의 경우, 종료 레이블
  int size;                     // 심볼의 요소 개수
};
```

현재는 배열을 포인터로 처리한다. 따라서 배열의 타입은 "무엇에 대한 포인터"로 정의된다. 예를 들어, 배열 요소가 `int` 타입이라면 "int에 대한 포인터"가 된다. 또한 `sym.c` 파일의 `addglob()` 함수에 인자를 하나 더 추가해야 한다:

```c
int addglob(char *name, int type, int stype, int endlabel, int size) {
  ...
}
```


## 배열 선언 파싱

현재는 크기가 지정된 배열 선언만 허용한다. 변수 선언을 위한 BNF 문법은 다음과 같다:

```
 variable_declaration: type identifier ';'
        | type identifier '[' P_INTLIT ']' ';'
        ;
```

`decl.c` 파일의 `var_declaration()` 함수에서 다음 토큰을 확인하고, 스칼라 변수 선언 또는 배열 선언을 처리해야 한다:

```c
// 주어진 타입으로 스칼라 변수 또는 크기가 지정된 배열을 선언한다.
// 식별자는 이미 스캔되었고 타입은 주어졌다.
void var_declaration(int type) {
  int id;

  // Text 변수에 식별자 이름이 저장되어 있다.
  // 다음 토큰이 '['인 경우
  if (Token.token == T_LBRACKET) {
    // '['를 건너뛴다
    scan(&Token);

    // 배열 크기가 있는지 확인
    if (Token.token == T_INTLIT) {
      // 이 배열을 알려진 배열로 추가하고 어셈블리에서 공간을 생성한다.
      // 배열은 요소 타입에 대한 포인터로 처리한다
      id = addglob(Text, pointer_to(type), S_ARRAY, 0, Token.intvalue);
      genglobsym(id);
    }

    // 다음에 ']'가 있는지 확인
    scan(&Token);
    match(T_RBRACKET, "]");
  } else {
    ...      // 이전 코드
  }
  
    // 끝에 세미콜론이 있는지 확인
  semi();
}
```

이 코드는 상당히 직관적이다. 나중에 배열 선언에 초기화 리스트를 추가할 예정이다.


## 배열 저장소 생성

이제 배열의 크기를 알았으니, `cgglobsym()` 함수를 수정해 어셈블러에서 이 공간을 할당할 수 있다.

```c
void cgglobsym(int id) {
  int typesize;
  // 타입의 크기를 가져옴
  typesize = cgprimsize(Gsym[id].type);

  // 전역 식별자와 레이블 생성
  fprintf(Outfile, "\t.data\n" "\t.globl\t%s\n", Gsym[id].name);
  fprintf(Outfile, "%s:", Gsym[id].name);

  // 공간 생성
  for (int i=0; i < Gsym[id].size; i++) {
    switch(typesize) {
      case 1: fprintf(Outfile, "\t.byte\t0\n"); break;
      case 4: fprintf(Outfile, "\t.long\t0\n"); break;
      case 8: fprintf(Outfile, "\t.quad\t0\n"); break;
      default: fatald("Unknown typesize in cgglobsym: ", typesize);
    }
  }
}
```

이제 다음과 같은 배열을 선언할 수 있다.

```c
  char a[10];
  int  b[25];
  long c[100];
```


## 배열 인덱스 파싱

이 부분에서는 너무 복잡한 내용을 다루지 않는다. 기본적인 배열 인덱스를 rvalue와 lvalue로 처리하는 방법만 알아본다. `test/input20.c` 프로그램은 우리가 구현하려는 기능을 보여준다:

```c
int a;
int b[25];

int main() {
  b[3]= 12; a= b[3];
  printint(a); return(0);
}
```

C 언어의 BNF 문법을 보면, 배열 인덱스는 괄호보다 *약간* 낮은 우선순위를 가진다:

```
primary_expression
        : IDENTIFIER
        | CONSTANT
        | STRING_LITERAL
        | '(' expression ')'
        ;

postfix_expression
        : primary_expression
        | postfix_expression '[' expression ']'
          ...
```

하지만 지금은 배열 인덱스도 `primary()` 함수에서 파싱한다. 의미 분석을 위한 코드가 충분히 커져서 새로운 함수를 만드는 것이 적절했다:

```c
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;


  switch (Token.token) {
  case T_IDENT:
    // 이 부분은 변수, 배열 인덱스, 혹은 함수 호출일 수 있다. 다음 토큰을 스캔해서 확인한다
    scan(&Token);

    // '(' 이면 함수 호출이다
    if (Token.token == T_LPAREN) return (funccall());

    // '[' 이면 배열 참조이다
    if (Token.token == T_LBRACKET) return (array_access());
```

다음은 `array_access()` 함수이다:

```c
// 배열의 인덱스를 파싱하고 이를 위한 AST 트리를 반환한다
static struct ASTnode *array_access(void) {
  struct ASTnode *left, *right;
  int id;

  // 배열로 정의된 식별자인지 확인한 후, 베이스를 가리키는 리프 노드를 만든다
  if ((id = findglob(Text)) == -1 || Gsym[id].stype != S_ARRAY) {
    fatals("Undeclared array", Text);
  }
  left = mkastleaf(A_ADDR, Gsym[id].type, id);

  // '[' 토큰을 읽는다
  scan(&Token);

  // 다음 표현식을 파싱한다
  right = binexpr(0);

  // ']' 토큰을 확인한다
  match(T_RBRACKET, "]");

  // 정수 타입인지 확인한다
  if (!inttype(right->type))
    fatal("Array index is not of integer type");

  // 인덱스를 요소 타입의 크기로 스케일링한다
  right = modify_type(right, left->type, A_ADD);

  // 배열의 베이스에 오프셋을 더하고, 요소를 역참조하는 AST 트리를 반환한다. 이 시점에서는 여전히 lvalue이다
  left = mkastnode(A_ADD, Gsym[id].type, left, NULL, right, 0);
  left = mkastunary(A_DEREF, value_at(left->type), left, 0);
  return (left);
}
```

`int x[20];` 배열과 `x[6]` 배열 인덱스의 경우, 인덱스(6)를 `int`의 크기(4)로 스케일링하고 이를 배열 베이스의 주소에 더해야 한다. 그런 다음 이 요소를 역참조한다. lvalue로 표시된 상태로 남겨두는데, 다음과 같은 작업을 할 수 있기 때문이다:

```c
  x[6] = 100;
```

만약 rvalue가 된다면, `binexpr()`가 A_DEREF AST 노드에서 `rvalue` 플래그를 설정할 것이다.


### 생성된 AST 트리

테스트 프로그램 `tests/input20.c`로 돌아가서, 배열 인덱스가 포함된 AST 트리를 생성하는 코드는 다음과 같다:

```c
  b[3]= 12; a= b[3];
```

`comp1 -T tests/input20.c`를 실행하면 다음과 같은 결과를 얻는다:

```
    A_INTLIT 12
  A_WIDEN
      A_ADDR b
        A_INTLIT 3    # 3은 4로 스케일링됨
      A_SCALE 4
    A_ADD             # 그리고 b의 주소에 더해짐
  A_DEREF             # 이후 역참조됨. 여전히 lvalue임
A_ASSIGN

      A_ADDR b
        A_INTLIT 3    # 위와 동일
      A_SCALE 4
    A_ADD
  A_DEREF rval        # 역참조된 주소는 rvalue가 됨
  A_IDENT a
A_ASSIGN
```


### 기타 파서 수정 사항


`expr.c` 파서에 몇 가지 사소한 변경 사항이 있었다. 이 문제를 디버깅하는 데 꽤 시간이 걸렸다. 연산자 우선순위 조회 함수에 입력 값을 더 엄격하게 확인해야 했다:

```c
// 바이너리 연산자인지 확인하고
// 해당 연산자의 우선순위를 반환한다.
static int op_precedence(int tokentype) {
  int prec;
  if (tokentype >= T_VOID)
    fatald("Token with no precedence in op_precedence:", tokentype);
  ...
}
```

파싱이 제대로 동작할 때까지, 우선순위 테이블에 없는 토큰을 전달했고, `op_precedence()` 함수가 테이블의 끝을 넘어 읽어버렸다. 이런! C 언어의 매력이란!

다른 변경 사항은, 이제 배열 인덱스로 표현식을 사용할 수 있게 되면서 (예: `x[ a+2 ]`), ']' 토큰이 표현식을 종료할 수 있음을 고려해야 한다는 점이다. 따라서 `binexpr()` 함수의 마지막 부분은 다음과 같이 수정했다:

```c
    // 현재 토큰의 세부 정보를 업데이트한다.
    // 세미콜론, ')' 또는 ']'를 만나면 왼쪽 노드만 반환한다
    tokentype = Token.token;
    if (tokentype == T_SEMI || tokentype == T_RPAREN
        || tokentype == T_RBRACKET) {
      left->rvalue = 1;
      return (left);
    }
  }
```


## 코드 생성기 변경 사항

코드 생성기에는 특별한 변경 사항이 없다. 이미 컴파일러에 필요한 모든 구성 요소가 포함되어 있다. 정수 값의 크기 조정, 변수의 주소 가져오기 등이 그 예이다. 다음 테스트 코드를 살펴보자:

```c
  b[3]= 12; a= b[3];
```

이 코드는 다음과 같은 x86-64 어셈블리 코드를 생성한다:

```
        movq    $12, %r8
        leaq    b(%rip), %r9    # b의 주소를 가져옴
        movq    $3, %r10
        salq    $2, %r10        # 3을 2만큼 시프트, 즉 3 * 4
        addq    %r9, %r10       # b의 주소에 더함
        movq    %r8, (%r10)     # 12를 b[3]에 저장

        leaq    b(%rip), %r8    # b의 주소를 다시 가져옴
        movq    $3, %r9
        salq    $2, %r9         # 3을 2만큼 시프트, 즉 3 * 4
        addq    %r8, %r9        # b의 주소에 더함
        movq    (%r9), %r9      # b[3]의 값을 %r9에 로드
        movl    %r9d, a(%rip)   # a에 저장
```


## 결론 및 다음 단계

기본 배열 선언과 배열 표현식을 추가하기 위한 파싱 변경은 문법 처리 측면에서 상대적으로 쉽게 진행했다. 어려웠던 부분은 AST 트리 노드를 올바르게 구성하여 크기 조정, 베이스 주소 추가, 그리고 lvalue/rvalue 설정을 하는 것이었다. 이 부분을 제대로 구현하자, 기존 코드 생성기가 올바른 어셈블리 출력을 생성할 수 있었다.

컴파일러 작성 과정의 다음 단계에서는 문자와 문자열 리터럴을 언어에 추가하고, 이를 출력할 방법을 찾아볼 것이다. [다음 단계](../20_Char_Str_Literals/Readme.md)



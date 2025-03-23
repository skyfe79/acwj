# 21장: 추가 연산자 구현

컴파일러 작성 과정의 이번 파트에서는 비교적 쉽게 구현할 수 있는 여러 표현식 연산자를 추가로 구현하기로 했다. 여기에는 다음 연산자들이 포함된다:

+ `++`와 `--`, 전위 증가/감소와 후위 증가/감소
+ 단항 연산자 `-`, `~`, 그리고 `!`
+ 이항 연산자 `^`, `&`, `|`, `<<`, 그리고 `>>`

또한 선택문과 반복문에서 표현식의 rvalue를 불리언 값으로 처리하는 암시적인 "not zero 연산자"도 구현했다. 예를 들어:

```c
  for (str= "Hello"; *str; str++) ...
```

이렇게 작성하는 대신에:

```c
  for (str= "Hello"; *str != 0; str++) ...
```

이번 장에서는 이러한 연산자들을 어떻게 구현했는지 상세히 다룬다. 각 연산자의 동작 원리와 구현 과정을 통해 컴파일러가 다양한 표현식을 어떻게 처리하는지 이해할 수 있다. 특히 전위와 후위 연산자의 차이점, 단항 연산자의 처리 방식, 그리고 비트 연산자의 구현 방법에 초점을 맞춘다.

또한 "not zero 연산자"의 구현은 코드의 간결성을 높이는 동시에 컴파일러가 불리언 표현식을 어떻게 평가하는지 보여준다. 이는 프로그래머가 더 직관적이고 간결한 코드를 작성할 수 있도록 도와준다.


## 토큰과 스캐닝

언제나 그렇듯, 새로운 언어 요소를 다룰 때는 먼저 토큰부터 살펴본다. 이번에는 몇 가지 새로운 토큰이 추가되었다.

| 스캔된 입력 | 토큰 |
|:-------------:|-------|
|   <code>&#124;&#124;</code>        | T_LOGOR |
|   `&&`        | T_LOGAND |
|   <code>&#124;</code>         | T_OR |
|   `^`         | T_XOR |
|   `<<`        | T_LSHIFT |
|   `>>`        | T_RSHIFT |
|   `++`        | T_INC |
|   `--`        | T_DEC |
|   `~`         | T_INVERT |
|   `!`         | T_LOGNOT |

이 중 일부는 새로운 단일 문자로 구성되어 있어 스캐닝이 간단하다. 다른 경우에는 단일 문자와 서로 다른 문자 쌍을 구별해야 한다. 예를 들어 `<`, `<<`, `<=`가 있다. 이미 `scan.c`에서 이런 스캐닝을 어떻게 처리하는지 살펴봤으므로 여기서는 새로운 코드를 제공하지 않는다. 추가된 내용을 확인하려면 `scan.c` 파일을 살펴보면 된다.


## 바이너리 연산자 파싱 추가

이제 이 연산자들을 파싱해야 한다. 이 중 일부는 바이너리 연산자다: `||`, `&&`, `|`, `^`, `<<`, `>>`. 이미 바이너리 연산자를 위한 우선순위 프레임워크가 준비되어 있다. 새로운 연산자를 이 프레임워크에 추가하기만 하면 된다.

이 작업을 하면서 [C 연산자 우선순위 표](https://en.cppreference.com/w/c/language/operator_precedence)에 따르면 기존 연산자 중 몇 개가 잘못된 우선순위로 설정되어 있음을 발견했다. 또한 AST 노드 연산을 바이너리 연산자 토큰 세트와 맞춰야 한다. 따라서 `defs.h`와 `expr.c`에서 토큰 정의, AST 노드 타입, 그리고 연산자 우선순위 테이블을 다음과 같이 정리했다:

```c
// 토큰 타입
enum {
  T_EOF,
  // 바이너리 연산자
  T_ASSIGN, T_LOGOR, T_LOGAND, 
  T_OR, T_XOR, T_AMPER, 
  T_EQ, T_NE,
  T_LT, T_GT, T_LE, T_GE,
  T_LSHIFT, T_RSHIFT,
  T_PLUS, T_MINUS, T_STAR, T_SLASH,

  // 기타 연산자
  T_INC, T_DEC, T_INVERT, T_LOGNOT,
  ...
};

// AST 노드 타입. 처음 몇 개는 관련 토큰과 일치
enum {
  A_ASSIGN= 1, A_LOGOR, A_LOGAND, A_OR, A_XOR, A_AND,
  A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE, A_LSHIFT, A_RSHIFT,
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE,
  ...
  A_PREINC, A_PREDEC, A_POSTINC, A_POSTDEC,
  A_NEGATE, A_INVERT, A_LOGNOT,
  ...
};

// 각 바이너리 토큰에 대한 연산자 우선순위. defs.h의 토큰 순서와 일치해야 함
static int OpPrec[] = {
  0, 10, 20, 30,                // T_EOF, T_ASSIGN, T_LOGOR, T_LOGAND
  40, 50, 60,                   // T_OR, T_XOR, T_AMPER 
  70, 70,                       // T_EQ, T_NE
  80, 80, 80, 80,               // T_LT, T_GT, T_LE, T_GE
  90, 90,                       // T_LSHIFT, T_RSHIFT
  100, 100,                     // T_PLUS, T_MINUS
  110, 110                      // T_STAR, T_SLASH
};
```


## 새로운 단항 연산자

이제 새로운 단항 연산자인 `++`, `--`, `~`, `!`를 파싱하는 방법을 알아본다. 이 연산자들은 모두 접두사 연산자(즉, 표현식 앞에 위치)지만, `++`와 `--`는 접미사 연산자로도 사용될 수 있다. 따라서 세 가지 접두사 연산자와 두 가지 접미사 연산자를 파싱하고, 각각에 대해 다섯 가지 다른 의미론적 동작을 수행해야 한다.

이 새로운 연산자를 추가하기 위해, [C 언어의 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)을 다시 참고했다. 이 연산자들은 기존의 이항 연산자 프레임워크에 통합할 수 없기 때문에, 재귀 하강 파서에 새로운 함수를 구현해야 한다. 다음은 위 문법에서 **관련된** 부분을 우리의 토큰 이름을 사용해 다시 작성한 것이다:

```
primary_expression
        : T_IDENT
        | T_INTLIT
        | T_STRLIT
        | '(' expression ')'
        ;

postfix_expression
        : primary_expression
        | postfix_expression '[' expression ']'
        | postfix_expression '(' expression ')'
        | postfix_expression '++'
        | postfix_expression '--'
        ;

prefix_expression
        : postfix_expression
        | '++' prefix_expression
        | '--' prefix_expression
        | prefix_operator prefix_expression
        ;

prefix_operator
        : '&'
        | '*'
        | '-'
        | '~'
        | '!'
        ;

multiplicative_expression
        : prefix_expression
        | multiplicative_expression '*' prefix_expression
        | multiplicative_expression '/' prefix_expression
        | multiplicative_expression '%' prefix_expression
        ;

        etc.
```

이항 연산자는 `expr.c`의 `binexpr()` 함수에서 구현되며, 이 함수는 `prefix()`를 호출한다. 위의 BNF 문법에서 `multiplicative_expression`이 `prefix_expression`을 참조하는 것과 마찬가지다. 이미 `primary()`라는 함수가 존재하며, 이제 접미사 표현식을 처리하기 위해 `postfix()` 함수가 필요하다.


## 전위 연산자

우리는 이미 `prefix()` 함수에서 T_AMPER와 T_STAR 토큰을 파싱하고 있다. 여기에 새로운 토큰들(T_MINUS, T_INVERT, T_LOGNOT, T_INC, T_DEC)을 추가하려면 `switch (Token.token)` 문에 더 많은 case 문을 추가하면 된다.

모든 case 문이 비슷한 구조를 가지고 있기 때문에 코드를 여기에 포함하지는 않겠다:

  + `scan(&Token)`을 사용해 토큰을 건너뛴다.
  + `prefix()`를 사용해 다음 표현식을 파싱한다.
  + 의미론적 검사를 수행한다.
  + `prefix()`가 반환한 AST 트리를 확장한다.

하지만 몇 가지 case 문 사이의 차이점은 중요하게 다뤄야 한다. `&`(T_AMPER) 토큰을 파싱할 때, 표현식은 lvalue로 처리되어야 한다. 예를 들어 `&x`를 할 때, 우리는 변수 `x`의 주소를 원한다. `x`의 값의 주소가 아니다. 반면 다른 경우에는 `prefix()`가 반환한 AST 트리를 rvalue로 강제로 변환해야 한다:

  + `-` (T_MINUS)
  + `~` (T_INVERT)
  + `!` (T_LOGNOT)

그리고 전위 증가 및 전위 감소 연산자의 경우, 우리는 실제로 표현식이 lvalue여야 한다. `++x`는 가능하지만 `++3`은 불가능하다. 지금은 단순한 식별자만 요구하도록 코드를 작성했지만, 나중에 `++b[2]`나 `++ *ptr` 같은 경우를 파싱하고 처리할 필요가 있을 것이다.

또한 설계 관점에서, 우리는 `prefix()`가 반환한 AST 트리를 수정하거나(새로운 AST 노드를 추가하지 않고), 하나 이상의 새로운 AST 노드를 트리에 추가할 수 있는 선택지가 있다:

  + T_AMPER는 기존 AST 트리를 수정해 루트를 A_ADDR로 만든다.
  + T_STAR는 트리의 루트에 A_DEREF 노드를 추가한다.
  + T_STAR는 트리를 `int` 값으로 확장한 후 루트에 A_NEGATE 노드를 추가한다. 왜냐하면 트리가 `char` 타입일 수 있는데, 이는 부호 없는 값이므로 부호 없는 값을 부정할 수 없기 때문이다.
  + T_INVERT는 트리의 루트에 A_INVERT 노드를 추가한다.
  + T_LOGNOT는 트리의 루트에 A_LOGNOT 노드를 추가한다.
  + T_INC는 트리의 루트에 A_PREINC 노드를 추가한다.
  + T_DEC는 트리의 루트에 A_PREDEC 노드를 추가한다.


## 후위 연산자 파싱

위에서 링크한 BNF 문법을 살펴보면, 후위 표현식을 파싱하려면 기본 표현식(primary expression)의 파싱을 참조해야 한다. 이를 구현하기 위해 먼저 기본 표현식의 토큰을 가져온 후, 후위 토큰이 뒤따르는지 확인해야 한다.

문법에서는 "postfix"가 "primary"를 호출하는 것으로 보이지만, 실제로는 `primary()`에서 토큰을 스캔한 후, `postfix()`를 호출해 후위 토큰을 파싱하는 방식으로 구현했다.

> 이 방식은 실수였다 — 미래의 Warren이 적음.

위 BNF 문법은 `x++ ++`와 같은 표현식을 허용한다. 문법에 다음과 같은 규칙이 있기 때문이다:

```
postfix_expression:
        postfix_expression '++'
        ;
```

하지만, 표현식 뒤에 후위 연산자가 하나만 오도록 제한할 것이다. 이제 새로운 코드를 살펴보자.

`primary()` 함수는 기본 표현식을 인식한다. 정수 리터럴, 문자열 리터럴, 식별자 등을 처리하며, 괄호로 둘러싸인 표현식도 인식한다. 후위 연산자가 올 수 있는 것은 식별자뿐이다.

```c
static struct ASTnode *primary(void) {
  ...
  switch (Token.token) {
    case T_INTLIT: ...
    case T_STRLIT: ...
    case T_LPAREN: ...
    case T_IDENT:
      return (postfix());
    ...
}
```

함수 호출과 배열 참조 파싱은 `postfix()`로 옮겼다. 이곳에서 후위 `++`와 `--` 연산자를 파싱한다.

```c
// 후위 표현식을 파싱하고 이를 나타내는 AST 노드를 반환한다.
// 식별자는 이미 Text에 들어 있다.
static struct ASTnode *postfix(void) {
  struct ASTnode *n;
  int id;

  // 다음 토큰을 스캔해 후위 표현식이 있는지 확인한다.
  scan(&Token);

  // 함수 호출
  if (Token.token == T_LPAREN)
    return (funccall());

  // 배열 참조
  if (Token.token == T_LBRACKET)
    return (array_access());

  // 변수. 변수가 존재하는지 확인한다.
  id = findglob(Text);
  if (id == -1 || Gsym[id].stype != S_VARIABLE)
    fatals("Unknown variable", Text);

  switch (Token.token) {
      // 후위 증가: 토큰을 건너뛴다.
    case T_INC:
      scan(&Token);
      n = mkastleaf(A_POSTINC, Gsym[id].type, id);
      break;

      // 후위 감소: 토큰을 건너뛴다.
    case T_DEC:
      scan(&Token);
      n = mkastleaf(A_POSTDEC, Gsym[id].type, id);
      break;

      // 단순 변수 참조
    default:
      n = mkastleaf(A_IDENT, Gsym[id].type, id);
  }
  return (n);
}
```

다른 설계 결정도 있었다. `++`의 경우, A_IDENT AST 노드를 만들고 A_POSTINC 부모 노드를 추가할 수도 있었다. 하지만 식별자 이름이 `Text`에 있으므로, 노드 타입과 심볼 테이블에서 식별자의 슬롯 번호를 모두 포함하는 단일 AST 노드를 구성할 수 있었다.


## 정수 표현식을 부울 값으로 변환하기

파싱 부분을 마치고 코드 생성 부분으로 넘어가기 전에, 정수 표현식을 부울 표현식으로 처리할 수 있도록 변경한 내용을 설명한다. 예를 들어:

```
  x= a + b;
  if (x) { printf("x is not zero\n"); }
```

BNF 문법은 표현식을 부울 값으로 제한하는 명시적인 구문 규칙을 제공하지 않는다. 예를 들어:

```
selection_statement
        : IF '(' expression ')' statement
```

따라서 이 작업을 의미적으로 처리해야 한다. IF, WHILE, FOR 루프를 파싱하는 `stmt.c` 파일에 다음 코드를 추가했다:

```c
  // 다음 표현식을 파싱
  // 비교 연산이 아닌 표현식을 부울 값으로 강제 변환
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    condAST = mkastunary(A_TOBOOL, condAST->type, condAST, 0);
```

새로운 AST 노드 타입인 A_TOBOOL을 도입했다. 이 노드는 모든 정수 값을 받아 코드를 생성한다. 값이 0이면 결과는 0이 되고, 그렇지 않으면 결과는 1이 된다.


## 새로운 연산자 코드 생성하기

이제 새로운 연산자에 대한 코드를 생성하는 방법을 살펴본다.  
새로 추가된 AST 노드 타입은 다음과 같다:  
A_LOGOR, A_LOGAND, A_OR, A_XOR, A_AND,  
A_LSHIFT, A_RSHIFT, A_PREINC, A_PREDEC, A_POSTINC, A_POSTDEC,  
A_NEGATE, A_INVERT, A_LOGNOT, A_TOBOOL.  

이 모든 연산자들은 `cg.c` 파일에 있는 플랫폼별 코드 생성기에서 해당 함수를 호출하는 방식으로 구현된다. 따라서 `gen.c` 파일의 `genAST()` 함수에 추가된 새로운 코드는 다음과 같다:

```c
    case A_AND:
      return (cgand(leftreg, rightreg));
    case A_OR:
      return (cgor(leftreg, rightreg));
    case A_XOR:
      return (cgxor(leftreg, rightreg));
    case A_LSHIFT:
      return (cgshl(leftreg, rightreg));
    case A_RSHIFT:
      return (cgshr(leftreg, rightreg));
    case A_POSTINC:
      // 변수의 값을 레지스터에 로드한 후 증가시킨다
      return (cgloadglob(n->v.id, n->op));
    case A_POSTDEC:
      // 변수의 값을 레지스터에 로드한 후 감소시킨다
      return (cgloadglob(n->v.id, n->op));
    case A_PREINC:
      // 변수의 값을 레지스터에 로드하고 증가시킨다
      return (cgloadglob(n->left->v.id, n->op));
    case A_PREDEC:
      // 변수의 값을 레지스터에 로드하고 감소시킨다
      return (cgloadglob(n->left->v.id, n->op));
    case A_NEGATE:
      return (cgnegate(leftreg));
    case A_INVERT:
      return (cginvert(leftreg));
    case A_LOGNOT:
      return (cglognot(leftreg));
    case A_TOBOOL:
      // 부모 AST 노드가 A_IF 또는 A_WHILE인 경우 비교 후 점프를 생성한다.  
      // 그렇지 않으면 레지스터의 값이 0인지 아닌지에 따라 0 또는 1로 설정한다
      return (cgboolean(leftreg, parentASTop, label));
```


## x86-64 특화 코드 생성 함수

이제 실제 x86-64 어셈블리 코드를 생성하는 백엔드 함수를 살펴볼 차례다. 대부분의 비트 연산은 x86-64 플랫폼에서 어셈블리 명령어로 처리할 수 있다.

```c
int cgand(int r1, int r2) {
  fprintf(Outfile, "\tandq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

int cgor(int r1, int r2) {
  fprintf(Outfile, "\torq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

int cgxor(int r1, int r2) {
  fprintf(Outfile, "\txorq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

// 레지스터 값의 부호를 반전
int cgnegate(int r) {
  fprintf(Outfile, "\tnegq\t%s\n", reglist[r]); return (r);
}

// 레지스터 값을 반전
int cginvert(int r) {
  fprintf(Outfile, "\tnotq\t%s\n", reglist[r]); return (r);
}
```

시프트 연산의 경우, 시프트 양을 먼저 `%cl` 레지스터에 로드해야 한다.

```c
int cgshl(int r1, int r2) {
  fprintf(Outfile, "\tmovb\t%s, %%cl\n", breglist[r2]);
  fprintf(Outfile, "\tshlq\t%%cl, %s\n", reglist[r1]);
  free_register(r2); return (r1);
}

int cgshr(int r1, int r2) {
  fprintf(Outfile, "\tmovb\t%s, %%cl\n", breglist[r2]);
  fprintf(Outfile, "\tshrq\t%%cl, %s\n", reglist[r1]);
  free_register(r2); return (r1);
}
```

불리언 표현식을 다루는 연산(결과가 0 또는 1이어야 함)은 조금 더 복잡하다.

```c
// 레지스터 값을 논리적으로 부정
int cglognot(int r) {
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r], reglist[r]);
  fprintf(Outfile, "\tsete\t%s\n", breglist[r]);
  fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r], reglist[r]);
  return (r);
}
```

`test` 명령어는 기본적으로 레지스터를 자기 자신과 AND 연산하여 제로 플래그와 네거티브 플래그를 설정한다. 그런 다음 값이 0이면 레지스터를 1로 설정한다(`sete`). 마지막으로 이 8비트 결과를 64비트 레지스터로 이동한다.

정수를 불리언 값으로 변환하는 코드는 다음과 같다.

```c
// 정수 값을 불리언 값으로 변환. IF 또는 WHILE 연산인 경우 점프
int cgboolean(int r, int op, int label) {
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r], reglist[r]);
  if (op == A_IF || op == A_WHILE)
    fprintf(Outfile, "\tje\tL%d\n", label);
  else {
    fprintf(Outfile, "\tsetnz\t%s\n", breglist[r]);
    fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r], reglist[r]);
  }
  return (r);
}
```

다시 `test`를 통해 레지스터의 값이 0인지 아닌지를 확인한다. 선택문이나 반복문을 처리 중이라면 결과가 거짓일 때 점프한다(`je`). 그렇지 않으면 `setnz`를 사용해 원래 값이 0이 아니면 레지스터를 1로 설정한다.


## 증가 및 감소 연산

`++`와 `--` 연산은 마지막으로 남겨두었다. 여기서 주의할 점은 메모리 위치에서 값을 레지스터로 가져오는 동시에, 별도로 값을 증가시키거나 감소시켜야 한다는 것이다. 그리고 이 작업을 레지스터를 로드하기 전에 할지 후에 할지 선택해야 한다.

이미 전역 변수의 값을 로드하는 `cgloadglob()` 함수가 있으므로, 이를 수정해 필요에 따라 변수를 변경하도록 했다. 코드가 깔끔하지는 않지만 동작은 한다.

```c
// 변수에서 값을 레지스터로 로드한다.
// 레지스터 번호를 반환한다. 만약 연산이 전위 또는 후위 증가/감소라면
// 이 동작도 수행한다.
int cgloadglob(int id, int op) {
  // 새로운 레지스터 할당
  int r = alloc_register();

  // 레지스터 초기화 코드 출력
  switch (Gsym[id].type) {
    case P_CHAR:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincb\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecb\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovzbq\t%s(%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincb\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecb\t%s(\%%rip)\n", Gsym[id].name);
      break;
    case P_INT:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincl\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecl\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovslq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincl\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecl\t%s(\%%rip)\n", Gsym[id].name);
      break;
    case P_LONG:
    case P_CHARPTR:
    case P_INTPTR:
    case P_LONGPTR:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincq\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecq\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincq\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecq\t%s(\%%rip)\n", Gsym[id].name);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}
```

나중에 `x = b[5]++`와 같은 연산을 처리하기 위해 이 코드를 다시 작성해야 할 것 같지만, 일단은 이 정도로 충분하다. 결국, 이 여정의 각 단계에서 작은 발걸음을 내딛는 것이 내가 약속한 바다.


## 새로운 기능 테스트

이 단계에서는 새로운 테스트 입력 파일을 자세히 다루지 않는다. `tests` 디렉토리에 있는 `input22.c`, `input23.c`, `input24.c` 파일을 직접 확인하면 된다. 컴파일러가 이 파일들을 정확히 컴파일하는지 확인할 수 있다:

```
$ make test
...
input22.c: OK
input23.c: OK
input24.c: OK
```


## 결론 및 다음 단계

컴파일러 기능을 확장하는 과정에서 많은 기능을 추가했지만, 추가된 개념적 복잡성은 최소화했기를 바란다.

여러 바이너리 연산자를 추가했으며, 이를 위해 스캐너를 업데이트하고 연산자 우선순위 테이블을 변경했다.

단항 연산자의 경우, `prefix()` 함수에 수동으로 추가했다.

새로운 후위 연산자를 위해 기존의 함수 호출과 배열 인덱스 기능을 새로운 `postfix()` 함수로 분리했다. 그리고 이 함수를 통해 후위 연산자를 추가했다. 여기서는 lvalue와 rvalue에 대해 약간 고민해야 했다. 또한 어떤 AST 노드를 추가할지, 아니면 기존 AST 노드를 재구성할지에 대한 설계 결정도 있었다.

x86-64 아키텍처에는 필요한 연산을 구현하는 명령어가 이미 있기 때문에 코드 생성은 상대적으로 간단했다. 그러나 일부 연산을 위해 특정 레지스터를 설정하거나 원하는 작업을 수행하기 위해 명령어를 조합해야 했다.

증가 및 감소 연산자는 까다로운 부분이었다. 일반 변수에 대해 동작하도록 코드를 작성했지만, 이 부분은 나중에 다시 다뤄야 할 것이다.

컴파일러 작성의 다음 단계에서는 지역 변수를 다룰 예정이다. 지역 변수가 동작하도록 만들면, 이를 함수 매개변수와 인자로 확장할 수 있다. 이 작업은 두 단계 이상이 소요될 것이다. [다음 단계](../22_Design_Locals/Readme.md)



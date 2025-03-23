# 42부: 타입 캐스팅과 NULL

컴파일러 개발 과정의 이번 부분에서는 타입 캐스팅을 구현했다. 처음에는 다음과 같은 코드를 작성할 수 있을 거라 생각했다:

```c
#define NULL (void *)0
```

하지만 `void *`가 제대로 동작하도록 하기에는 아직 충분한 작업을 하지 못했다. 그래서 타입 캐스팅을 추가하고 `void *`도 제대로 동작하도록 만들었다.


## 타입 캐스팅이란?

타입 캐스팅은 표현식의 타입을 강제로 다른 타입으로 변경하는 것을 말한다. 주로 정수 값을 더 작은 범위의 타입으로 줄이거나, 한 타입의 포인터를 다른 타입의 포인터 저장소에 할당할 때 사용한다. 예를 들어:

```c
  int   x= 65535;
  char  y= (char)x;     // y는 이제 255, 하위 8비트
  int  *a= &x;
  char *b= (char *)a;   // b는 x의 주소를 가리킴
  long *z= (void *)0;   // z는 NULL 포인터, 아무것도 가리키지 않음
```

위 예제에서 캐스팅을 할당문에 사용했다. 함수 내부의 표현식에서는 AST 트리에 A_CAST 노드를 추가해 "원래 표현식의 타입을 새로운 타입으로 캐스팅한다"는 것을 명시해야 한다.

전역 변수 할당의 경우, 리터럴 값 앞에 캐스팅을 허용하도록 할당 파서를 수정해야 한다.


## 새로운 함수 `parse_cast()` 추가

`decl.c` 파일에 새로운 함수를 추가했다:

```c
// 캐스트 연산자 안에 나타나는 타입을 파싱한다
int parse_cast(void) {
  int type, class;
  struct symtable *ctype;

  // 괄호 안의 타입을 가져온다
  type= parse_stars(parse_type(&ctype, &class));

  // 에러 검사를 수행한다. 더 많은 검사가 가능할 것이다
  if (type == P_STRUCT || type == P_UNION || type == P_VOID)
    fatal("Cannot cast to a struct, union or void type");
  return(type);
}
```

캐스트 연산자를 감싸는 괄호 '(' ... ')'의 파싱은 다른 곳에서 처리한다. 타입 식별자와 뒤따르는 '*' 토큰을 가져와 캐스트의 타입을 결정한다. 그런 다음 구조체, 공용체, 그리고 `void` 타입으로의 캐스트를 방지한다.

이 함수가 필요한 이유는 표현식과 전역 변수 할당에서 동일한 로직을 반복하지 않기 위해서다. [DRY 코드](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 원칙을 지키기 위해 별도의 함수로 분리했다.


## 표현식에서 형 변환 파싱

우리는 이미 표현식 코드에서 괄호를 파싱하고 있으므로, 이를 수정해야 한다. `expr.c` 파일의 `primary()` 함수에서 이제 다음과 같이 처리한다:

```c
static struct ASTnode *primary(void) {
  int type=0;
  ...
  switch (Token.token) {
  ...
    case T_LPAREN:
    // 괄호로 둘러싸인 표현식의 시작이므로 '('를 건너뛴다.
    scan(&Token);


    // 다음 토큰이 타입 식별자인 경우, 이는 형 변환 표현식이다.
    switch (Token.token) {
      case T_IDENT:
        // 식별자가 typedef와 일치하는지 확인해야 한다.
        // 일치하지 않으면 일반 표현식으로 처리한다.
        if (findtypedef(Text) == NULL) {
          n = binexpr(0); break;
        }
      case T_VOID:
      case T_CHAR:
      case T_INT:
      case T_LONG:
      case T_STRUCT:
      case T_UNION:
      case T_ENUM:
        // 괄호 안의 타입을 가져온다.
        type= parse_cast();

        // 닫는 ')'를 건너뛰고, 다음 표현식을 파싱한다.
        rparen();

      default: n = binexpr(0); // 표현식을 파싱한다.
    }

    // 이 시점에서 n에는 최소한 하나의 표현식이 있고, 형 변환이 있었다면 type은 0이 아닌 값을 가진다.
    // 형 변환이 없었다면 닫는 ')'를 건너뛴다.
    if (type == 0)
      rparen();
    else
      // 형 변환이 있었다면 A_CAST 타입의 단항 AST 노드를 생성한다.
      n= mkastunary(A_CAST, type, n, NULL, 0);
    return (n);
  }
}
```

이 코드는 상당히 복잡하므로 단계별로 살펴보자. 모든 경우에서 `'('` 토큰 다음에 타입 식별자가 있는지 확인한다. `parse_cast()`를 호출하여 형 변환 타입을 가져오고 `')'` 토큰을 파싱한다.

아직 반환할 AST 트리가 없는데, 이는 어떤 표현식을 형 변환하는지 아직 모르기 때문이다. 따라서 기본 케이스로 넘어가 다음 표현식을 파싱한다.

이 시점에서 `type`은 여전히 0일 수도 있고(형 변환 없음), 0이 아닐 수도 있다(형 변환 있음). 형 변환이 없었다면 닫는 괄호를 건너뛰고 괄호 안의 표현식을 반환한다.

형 변환이 있었다면 새로운 `type`과 다음 표현식을 자식으로 하는 `A_CAST` 노드를 생성한다.


## 캐스팅을 위한 어셈블리 코드 생성

우리는 표현식의 값이 레지스터에 저장될 것이므로 운이 좋다. 다음 코드를 살펴보자:

```c
  int   x= 65535;
  char  y= (char)x;     // y는 이제 255, 하위 8비트
```

여기서 65535를 레지스터에 넣을 수 있다. 하지만 y에 저장할 때는 lvalue의 타입이 올바른 크기를 저장하기 위한 코드를 생성한다:

```
        movq    $65535, %r10            # x에 65535 저장
        movl    %r10d, -4(%rbp)
        movslq  -4(%rbp), %r10          # x를 %r10으로 가져옴
        movb    %r10b, -8(%rbp)         # y에 1바이트 저장
```

따라서 `gen.c` 파일의 `genAST()` 함수에는 캐스팅을 처리하기 위한 다음과 같은 코드가 있다:

```c
  ...
  leftreg = genAST(n->left, NOLABEL, NOLABEL, NOLABEL, n->op);
  ...
  switch (n->op) {
    ...
    case A_CAST:
      return (leftreg);         // 특별히 할 일이 없음
    ...
  }
```


## 전역 변수에서의 타입 캐스팅

위의 내용은 변수가 지역 변수일 때는 문제없이 동작한다. 컴파일러가 해당 할당을 표현식으로 처리하기 때문이다. 하지만 전역 변수의 경우, 타입 캐스팅을 직접 파싱하고 그 뒤에 오는 리터럴 값에 적용해야 한다.

예를 들어, `decl.c` 파일의 `scalar_declaration` 함수에서는 다음과 같은 코드가 필요하다:

```c
    // 전역 변수는 리터럴 값으로 초기화해야 한다
    if (class == C_GLOBAL) {
      // 캐스팅이 있는 경우
      if (Token.token == T_LPAREN) {
        // 캐스팅된 타입을 가져온다
        scan(&Token);
        casttype= parse_cast();
        rparen();

        // 두 타입이 호환되는지 확인한다. 아래에서 리터럴을 파싱할 수 있도록
        // 새로운 타입을 변경한다. 'void *' 타입은 모든 포인터 타입에 할당할 수 있다.
        if (casttype == type || (casttype== pointer_to(P_VOID) && ptrtype(type)))
          type= P_NONE;
        else
          fatal("타입 불일치");
      }

      // 변수를 위한 초기값을 하나 생성하고 이 값을 파싱한다
      sym->initlist= (int *)malloc(sizeof(int));
      sym->initlist[0]= parse_literal(type);
      scan(&Token);
    }
```

먼저, 캐스팅이 있을 때 `type= P_NONE`으로 설정하고, `parse_literal()` 함수를 `P_NONE` 타입으로 호출한다. 왜 그럴까? 이전에는 이 함수가 파싱할 리터럴이 인자로 전달된 타입과 정확히 일치해야 했다. 즉, 문자열 리터럴은 `char *` 타입이어야 했고, `char` 타입은 0에서 255 사이의 리터럴 값과 일치해야 했다.

하지만 이제 캐스팅이 있으므로 다음과 같은 코드를 허용할 수 있다:

```c
  char a= (char)65536;
```

따라서 `decl.c` 파일의 `parse_literal()` 함수는 이제 다음과 같이 동작한다:

```c
int parse_literal(int type) {

  // 문자열 리터럴인 경우, 메모리에 저장하고 레이블을 반환한다
  if (Token.token == T_STRLIT) {
    if (type == pointer_to(P_CHAR) || type == P_NONE)
    return(genglobstr(Text));
  }

  // 정수 리터럴인 경우, 범위 검사를 수행한다
  if (Token.token == T_INTLIT) {
    switch(type) {
      case P_CHAR: if (Token.intvalue < 0 || Token.intvalue > 255)
                     fatal("char 타입에 비해 정수 리터럴 값이 너무 큽니다");
      case P_NONE:
      case P_INT:
      case P_LONG: break;
      default: fatal("타입 불일치: 정수 리터럴 vs. 변수");
    }
  } else
    fatal("정수 리터럴 값이 필요합니다");
  return(Token.intvalue);
}
```

여기서 `P_NONE`은 타입 제한을 완화하기 위해 사용된다.


## `void *` 다루기

`void *` 포인터는 다른 모든 포인터 타입 대신 사용할 수 있는 포인터다. 따라서 이 기능을 구현해야 한다.

앞서 전역 변수 할당에서 이미 이 작업을 수행했다:

```c
if (casttype == type || (casttype == pointer_to(P_VOID) && ptrtype(type)))
```

즉, 타입이 동일하거나 `void *` 포인터가 포인터에 할당되는 경우다. 이로 인해 다음과 같은 전역 할당이 가능하다:

```c
char *str = (void *)0;
```

`str`이 `char *` 타입이고 `void *`가 아님에도 불구하고 말이다.

이제 표현식에서 `void *`(그리고 다른 포인터/포인터 연산)를 처리해야 한다. 이를 위해 `types.c` 파일의 `modify_type()` 함수를 수정했다. 이 함수가 무엇을 하는지 다시 살펴보자:

```c
// 주어진 AST 트리와 원하는 타입을 기준으로,
// 트리를 확장하거나 스케일링하여 이 타입과 호환되도록 수정한다.
// 변경이 없으면 원래 트리를 반환하고, 수정된 트리 또는 타입과 호환되지 않으면 NULL을 반환한다.
// 이 작업이 이항 연산의 일부라면 AST 연산자는 0이 아니다.
struct ASTnode *modify_type(struct ASTnode *tree, int rtype, int op);
```

이 코드는 값을 확장한다. 예를 들어 `int x = 'Q';`에서 `x`를 32비트 값으로 만드는 것이다. 또한 스케일링에도 사용된다. 다음 코드에서:

```c
int x[4];
int y = x[2];
```

인덱스 "2"는 `int`의 크기만큼 스케일링되어 `x[]` 배열의 기준점에서 8바이트 오프셋이 된다.

따라서 함수 내부에서 다음과 같이 작성하면:

```c
char *str = (void *)0;
```

다음과 같은 AST 트리가 생성된다:

```
          A_ASSIGN
           /    \
       A_CAST  A_IDENT
         /      str
     A_INTLIT
         0
```

왼쪽 `tree`의 타입은 `void *`이고 `rtype`은 `char *`가 된다. 이 연산이 수행될 수 있도록 보장해야 한다.

포인터를 위해 `modify_type()`을 다음과 같이 변경했다:

```c
// 포인터의 경우
if (ptrtype(ltype) && ptrtype(rtype)) {
  // 비교할 수 있다
  if (op >= A_EQ && op <= A_GE)
    return(tree);

  // 비이항 연산에서 동일한 타입의 비교는 허용되며,
  // 왼쪽 트리가 `void *` 타입인 경우도 허용된다.
  if (op == 0 && (ltype == rtype || ltype == pointer_to(P_VOID)))
    return (tree);
}
```

이제 포인터 비교는 허용되지만, 다른 이항 연산(예: 덧셈)은 허용되지 않는다. "비이항 연산"은 할당과 같은 것을 의미한다. 동일한 타입 간의 할당은 확실히 가능하다. 이제 `void *` 포인터에서 다른 포인터로의 할당도 가능하다.


## NULL 추가하기

이제 `void *` 포인터를 다룰 수 있으므로, NULL을 헤더 파일에 추가할 수 있다. `stdio.h`와 `stddef.h` 두 파일에 다음과 같이 추가했다:

```c
#ifndef NULL
# define NULL (void *)0
#endif
```

하지만 마지막 문제가 하나 있었다. 전역 변수 선언을 다음과 같이 시도했을 때:

```c
#include <stdio.h>
char *str= NULL;
```

이런 결과가 나왔다:

```
str:
        .quad   L0
```

`char *` 포인터의 초기화 값은 모두 레이블 번호로 처리되기 때문이다. 따라서 NULL의 "0"이 "L0" 레이블로 변환되었다. 이 문제를 해결해야 한다. 이제 `cg.c` 파일의 `cgglobsym()` 함수를 다음과 같이 수정한다:

```c
      case 8:
        // 문자열 리터럴에 대한 포인터를 생성한다. 0 값을 실제 0으로 처리하고, L0 레이블로 처리하지 않는다.
        if (node->initlist != NULL && type== pointer_to(P_CHAR) && initvalue != 0)
          fprintf(Outfile, "\t.quad\tL%d\n", initvalue);
        else
          fprintf(Outfile, "\t.quad\t%d\n", initvalue);
```

보기에는 깔끔하지 않지만, 이렇게 하면 문제가 해결된다!


## 변경 사항 테스트

모든 테스트를 하나씩 살펴보지는 않겠지만, `tests/input101.c`부터 `tests/input108.c` 파일들은 위에서 설명한 기능과 컴파일러의 오류 검사 기능을 테스트한다.


## 마무리 및 다음 단계

캐스팅이 쉬울 거라고 생각했고, 실제로 그랬다. 하지만 `void *`와 관련된 문제들은 예상치 못했다. 이 부분은 대부분 다루었지만, 아직 발견하지 못한 `void *` 관련 예외 사항이 있을 수 있다.

컴파일러 작성 여정의 다음 단계에서는 몇 가지 빠진 연산자를 추가할 것이다. [다음 단계](../43_More_Operators/Readme.md)



# 45장: 전역 변수 선언 다시 살펴보기

두 장 전에, 나는 다음과 같은 코드를 컴파일하려고 시도했다:

```c
enum { TEXTLEN = 512 };         // 입력에서 식별자의 길이
extern char Text[TEXTLEN + 1];
```

그리고 우리의 선언 파싱 코드가 배열의 크기로 단일 정수 리터럴만 처리할 수 있다는 사실을 깨달았다. 하지만 위에 보이는 내 컴파일러 코드는 두 개의 정수 리터럴로 이루어진 표현식을 사용한다.

지난 장에서, 나는 컴파일러에 상수 폴딩을 추가해 정수 리터럴로 이루어진 표현식을 단일 정수 리터럴로 축소할 수 있게 했다.

이제 우리는 리터럴 값과 관련된 캐스팅을 직접 작성한 멋진 파싱 코드를 모두 버리고, 표현식 파서를 호출해 리터럴 값을 포함한 AST 트리를 얻어야 한다.


## `parse_literal()` 함수를 유지할지 폐기할지?

현재 컴파일러의 `decl.c` 파일에는 문자열과 정수 리터럴을 수동으로 파싱하는 `parse_literal()` 함수가 있다. 이 함수를 그대로 유지할지, 아니면 폐기하고 다른 곳에서 직접 `binexpr()`를 호출할지 고민했다.

결론적으로 이 함수를 유지하기로 결정했다. 기존 코드를 모두 제거하고, 이 함수의 목적을 약간 변경했다. 이제 이 함수는 여러 리터럴 값이 포함된 표현식 앞에 오는 캐스트도 파싱한다.

`decl.c` 파일의 함수 헤더는 다음과 같이 변경되었다:

```c
// 주어진 타입에 대해 리터럴 표현식을 파싱하고,
// 이 표현식의 타입이 주어진 타입과 일치하는지 확인한다.
// 표현식 앞에 오는 타입 캐스트를 파싱한다.
// 정수 리터럴인 경우 해당 값을 반환한다.
// 문자열 리터럴인 경우 문자열의 라벨 번호를 반환한다.
int parse_literal(int type);
```

이제 이 함수는 기존 `parse_literal()` 함수를 대체하며, 이전에 사용하던 캐스트 파싱 코드를 제거할 수 있다. 이제 새로운 `parse_literal()` 코드를 살펴보자.

```c
int parse_literal(int type) {
  struct ASTnode *tree;

  // 표현식을 파싱하고 결과 AST 트리를 최적화한다.
  tree= optimise(binexpr(0));
```

`binexpr()`를 호출해 입력 파일의 현재 위치에 있는 표현식을 파싱하고, `optimise()`를 호출해 모든 리터럴 표현식을 폴딩한다.

이 트리를 사용하려면 루트 노드가 A_INTLIT, A_STRLIT 또는 A_CAST(캐스트가 있는 경우) 중 하나여야 한다.

```c
  // 캐스트가 있는 경우 자식 노드를 가져오고,
  // 캐스트된 타입을 자식 노드에 표시한다.
  if (tree->op == A_CAST) {
    tree->left->type= tree->type;
    tree= tree->left;
  }
```

캐스트가 있었으므로 A_CAST 노드를 제거하지만, 자식 노드가 캐스트된 타입은 유지한다.

```c
  // 이제 트리는 정수 또는 문자열 리터럴을 가져야 한다.
  if (tree->op != A_INTLIT && tree->op != A_STRLIT)
    fatal("전역 변수를 일반 표현식으로 초기화할 수 없습니다.");
```

사용할 수 없는 표현식이 제공되었으므로 오류 메시지를 출력하고 종료한다.

```c
  // 타입이 char *이고,
  if (type == pointer_to(P_CHAR)) {
    // 문자열 리터럴인 경우 라벨 번호를 반환한다.
    if (tree->op == A_STRLIT)
      return(tree->a_intvalue);
    // 0 정수 리터럴인 경우 NULL로 처리한다.
    if (tree->op == A_INTLIT && tree->a_intvalue==0)
      return(0);
  }
```

다음 두 가지 입력을 모두 처리할 수 있어야 한다:

```c
              char *c= "Hello";
              char *c= (char *)0;
```

위의 두 내부 IF 문은 두 입력 라인과 일치한다. 문자열 리터럴이 아니라면...

```c
  // 여기서는 정수 리터럴만 처리한다. 입력 타입은
  // 정수 타입이어야 하며, 리터럴 값을 담을 수 있을 만큼 충분히 커야 한다.
  if (inttype(type) && typesize(type, NULL) >= typesize(tree->type, NULL))
    return(tree->a_intvalue);

  fatal("타입 불일치: 리터럴 vs. 변수");
  return(0);    // -Wall 경고를 방지하기 위해
}
```

이 부분을 이해하는 데 시간이 걸렸다. 다음과 같은 경우를 처리해야 한다:

```c
  long  x= 3;    // 허용, 여기서 3은 char 타입
  char  y= 4000; // 방지, 여기서 4000은 너무 큼
  char *z= 4000; // 방지, z는 정수 타입이 아님
```

따라서 IF 문은 입력 타입을 확인하고, 정수 리터럴을 수용할 수 있을 만큼 충분히 큰지 검사한다.


## `decl.c`의 기타 파싱 변경 사항

이제 캐스트가 앞에 올 수 있는 리터럴 표현식을 파싱할 수 있는 함수를 갖게 되었다. 이 함수를 활용해 기존의 캐스트 파싱 코드를 제거하고 새로운 코드로 대체한다. 주요 변경 사항은 다음과 같다.

```c
// 스칼라 변수 선언 파싱
static struct symtable *scalar_declaration(...) {
    ...
    // 전역 변수는 리터럴 값으로 초기화해야 함
    if (class == C_GLOBAL) {
      // 변수를 위한 초기값 생성 및 파싱
      sym->initlist= (int *)malloc(sizeof(int));
      sym->initlist[0]= parse_literal(type);
    }
    ...
}

// 배열 선언 파싱
static struct symtable *array_declaration(...) {

  ...
  // 배열 크기 확인
  if (Token.token != T_RBRACKET) {
    nelems= parse_literal(P_INT);
    if (nelems <= 0)
      fatald("Array size is illegal", nelems);
  }

  ...
  // 초기값 리스트 가져오기
  while (1) {
    ...
    initlist[i++]= parse_literal(type);
    ...
  }
  ...
}
```

이렇게 변경함으로써, 이전 `parse_literal()` 앞에 오던 모든 캐스트를 파싱하는 데 사용되던 약 20~30줄의 코드를 제거할 수 있었다. 다만, 이 30줄의 코드를 줄이기 위해 상수 폴딩(constant folding)을 구현하는 데 100줄의 코드를 추가해야 했다. 다행히 상수 폴딩은 일반 표현식에서도 활용되기 때문에 여전히 이득이다.


## `expr.c` 파일의 한 가지 변경사항

새로운 `parse_literal()` 함수를 지원하기 위해 컴파일러 코드에 한 가지 추가 변경이 필요하다. 일반적인 표현식을 파싱하는 함수인 `binexpr()`에서 이제는 표현식이 '}' 토큰으로 끝날 수 있음을 알려줘야 한다. 예를 들어 다음과 같은 경우다:

```c
  int fred[]= { 1, 2, 6 };
```

`binexpr()` 함수에 추가된 작은 변경사항은 다음과 같다:

```c
    // 종료 토큰을 만나면 왼쪽 노드만 반환
    tokentype = Token.token;
    if (tokentype == T_SEMI || tokentype == T_RPAREN ||
        tokentype == T_RBRACKET || tokentype == T_COMMA ||
        tokentype == T_COLON || tokentype == T_RBRACE) {    // T_RBRACE가 새로 추가됨
      left->rvalue = 1;
      return (left);
    }
```


## 코드를 통해 변경 사항 테스트하기

기존 테스트는 전역 변수를 초기화할 때 단일 리터럴 값을 사용하는 경우를 검증한다. `tests/input112.c` 파일의 이 코드는 스칼라 변수를 초기화하는 리터럴 표현식과 배열의 크기를 지정하는 리터럴 표현식 두 가지를 모두 테스트한다:

```c
#include <stdio.h>
char* y = NULL;
int x= 10 + 6;
int fred [ 2 + 3 ];

int main() {
  fred[2]= x;
  printf("%d\n", fred[2]);
  return(0);
}
```


## 결론과 다음 단계

컴파일러 작성 여정의 다음 단계에서는 컴파일러 소스 코드를 더 많이 스스로에게 입력해보고, 아직 구현해야 할 부분이 무엇인지 확인할 계획이다. [다음 단계](../46_Void_Functions/Readme.md)로 넘어가보자.



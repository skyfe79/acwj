# 57장: 마무리 작업, 3부

컴파일러 개발 과정의 이번 단계에서는 컴파일러의 몇 가지 작은 문제를 해결한다.


## -D 플래그 없이 작업하기

컴파일러에는 전처리기에 심볼을 정의하기 위한 런타임 `-D` 플래그가 없다. 이 기능을 추가하려면 상당히 복잡한 작업이 필요하다. 하지만 `Makefile`에서 헤더 파일이 위치한 디렉토리를 설정하기 위해 이 방식을 사용한다.

이 문제를 해결하기 위해 `Makefile`을 수정해 해당 위치를 새로운 헤더 파일에 기록하도록 했다:

```
# 헤더 파일 디렉토리 위치 정의
INCDIR=/tmp/include
...

incdir.h:
        echo "#define INCDIR \"$(INCDIR)\"" > incdir.h
```

그리고 `defs.h` 파일에는 다음과 같은 내용을 추가했다:

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include "incdir.h"
```

이렇게 하면 소스 코드에서 해당 디렉토리의 위치를 정확히 알 수 있다.


## 외부 변수 로딩

`include/stdio.h` 파일에 다음과 같은 세 개의 외부 변수를 추가했다:

```c
extern FILE *stdin;
extern FILE *stdout;
extern FILE *stderr;
```

그런데 이 변수들을 사용하려고 하니 로컬 변수로 처리되는 문제가 발생했다. 전역 변수를 선택하는 로직에 오류가 있었던 것이다. `gen.c` 파일의 `genAST()` 함수를 다음과 같이 수정했다:

```c
    case A_IDENT:
      // rvalue이거나 역참조 중인 경우 값을 로드
      if (n->rvalue || parentASTop == A_DEREF) {
        if (n->sym->class == C_GLOBAL || n->sym->class == C_STATIC
            || n->sym->class == C_EXTERN) {
          return (cgloadglob(n->sym, n->op));
        } else {
          return (cgloadlocal(n->sym, n->op));
        }
```

여기서 `C_EXTERN` 조건을 추가하여 외부 변수도 전역 변수로 처리하도록 했다.


## Pratt 파서의 문제점

이 여정의 3부에서, 각 토큰에 우선순위 값을 연관시킨 [Pratt 파서](https://en.wikipedia.org/wiki/Pratt_parser)를 소개했다. 그 이후로 이 파서를 사용해 왔고 잘 작동했다.

그러나 Pratt 파서로 파싱되지 않는 토큰들을 추가했다: 전위 연산자, 후위 연산자, 형변환, 배열 요소 접근 등이 그것이다. 그리고 그 과정에서 Pratt 파서가 이전 연산자 토큰의 우선순위를 알 수 있게 해주는 체인을 깨뜨렸다.

다음은 `expr.c` 파일의 `binexpr()` 함수에 있는 기본 Pratt 알고리즘이다:

```c
  // 왼쪽 트리를 가져온다.
  // 동시에 다음 토큰을 가져온다.
  left = prefix();
  tokentype = Token.token;

  // 이 토큰의 우선순위가 이전 토큰의 우선순위보다 높거나,
  // 오른쪽 결합성이고 이전 토큰의 우선순위와 같다면
  while ((op_precedence(tokentype) > ptp) ||
         (rightassoc(tokentype) && op_precedence(tokentype) == ptp)) {
    // 다음 정수 리터럴을 가져온다.
    scan(&Token);

    // 토큰의 우선순위를 사용해 binexpr()를 재귀적으로 호출하여 서브 트리를 만든다.
    right = binexpr(OpPrec[tokentype]);

    // 그 서브 트리를 현재 트리에 연결한다 (코드 생략).

    // 현재 토큰의 세부 정보를 업데이트한다.
    // 종료 토큰이면 루프를 빠져나간다 (코드 생략).
    tokentype = Token.token;
  }

  // 우선순위가 같거나 낮을 때 현재 트리를 반환한다.
  return (left);
```

`binexpr()`가 이전 토큰의 우선순위를 인자로 받도록 해야 한다. 이제 이 부분이 어떻게 깨졌는지 살펴보자.

다음은 세 개의 포인터가 유효한지 확인하는 표현식이다:

```c
  if (a == NULL || b == NULL || c == NULL)
```

`==` 연산자는 `||` 연산자보다 우선순위가 높으므로, Pratt 파서는 이를 다음과 같이 처리해야 한다:

```c
  if ((a == NULL) || (b == NULL) || (c == NULL))
```

NULL은 다음과 같이 정의되며, 여기에는 형변환이 포함된다:

```c
#define NULL (void *)0
```

이제 위의 IF 문의 호출 체인을 살펴보자:

 + `if_statement()`에서 `binexpr(0)`이 호출된다.
 + `binexpr(0)`은 `==` (우선순위 40)을 파싱하고 `binexpr(40)`을 호출한다.
 + `binexpr(40)`은 `prefix()`를 호출한다.
 + `prefix()`는 `postfix()`를 호출한다.
 + `postfix()`는 `primary()`를 호출한다.
 + `primary()`는 `(void *)0`의 시작 부분에 있는 왼쪽 괄호를 보고 `paren_expression()`을 호출한다.
 + `paren_expression()`은 `void` 토큰을 보고 `parse_cast()`를 호출한다. 형변환이 파싱된 후, `0`을 파싱하기 위해 `binexpr(0)`을 호출한다.

여기서 문제가 발생한다. NULL의 값인 `0`은 여전히 우선순위 40이어야 하지만, `paren_expression()`이 이를 다시 0으로 초기화했다.

이제 `NULL || b`를 파싱하여 AST 트리를 만들게 되고, `a == NULL`을 파싱하여 그 AST 트리를 만드는 대신 잘못된 파싱이 이루어진다.

해결책은 이전 토큰의 우선순위가 `binexpr()`부터 `paren_expression()`까지 호출 체인을 통해 전달되도록 하는 것이다. 즉:

 + `prefix()`, `postfix()`, `primary()`, `paren_expression()`

모두 `int ptp` 인자를 받고 이를 전달해야 한다.

`tests/input143.c` 프로그램은 이 변경이 `if (a==NULL || b==NULL || c==NULL)`에 대해 잘 작동하는지 확인한다.


## 포인터, `+=` 및 `-=` 연산자

얼마 전, 정수 값을 포인터에 더할 때 해당 포인터가 가리키는 타입의 크기만큼 정수를 스케일링해야 한다는 사실을 깨달았다. 예를 들어:

```c
int list[]= {3, 5, 7, 9, 11, 13, 15};
int *lptr;

int main() {
  lptr= list;
  printf("%d\n", *lptr);
  lptr= lptr + 1; printf("%d\n", *lptr);
}
```

이 코드는 `list`의 첫 번째 값인 3을 출력한다. `lptr`은 `int`의 크기인 4만큼 증가하므로, 이제 `list`의 다음 요소를 가리키게 된다.

이제 `+`와 `-` 연산자에 대해 이 작업을 수행하지만, `+=`와 `-=` 연산자에 대해 이를 구현하는 것을 잊었다. 다행히도 이 문제는 쉽게 해결할 수 있었다. `types.c` 파일의 `modify_type()` 함수 하단에 다음과 같은 코드를 추가했다:

```c
  // 덧셈과 뺄셈 연산에서만 스케일링 가능
  if (op == A_ADD || op == A_SUBTRACT ||
      op == A_ASPLUS || op == A_ASMINUS) {

    // 왼쪽이 정수 타입이고, 오른쪽이 포인터 타입이며, 원본 타입의 크기가 1보다 큰 경우: 왼쪽을 스케일링
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1)
        return (mkastunary(A_SCALE, rtype, rctype, tree, NULL, rsize));
      else
        return (tree);          // 크기가 1이면 스케일링 필요 없음
    }
  }
```

여기서 `A_ASPLUS`와 `A_ASMINUS`를 스케일링이 가능한 연산자 목록에 추가했다. 이제 `+=`와 `-=` 연산자도 정수 값을 적절히 스케일링할 수 있게 되었다.


## 결론과 다음 단계

지금까지는 여기까지 정리하는 것으로 충분하다. `+=`와 `-=` 문제를 해결하면서, 포인터에 적용된 `++`와 `--` 연산자(전위 및 후위)의 큰 문제점이 부각되었다.

컴파일러 작성 여정의 다음 단계에서는 이 문제를 해결할 것이다. [다음 단계](../58_Ptr_Increments/Readme.md)



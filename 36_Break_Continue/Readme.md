# 36장: `break`와 `continue`

예전에 타입이 없는 언어를 위한 [간단한 컴파일러](https://github.com/DoctorWkt/h-compiler)를 작성했을 때, 추상 구문 트리(Abstract Syntax Tree, AST)를 사용하지 않았다. 이로 인해 `break`와 `continue` 키워드를 언어에 추가하는 것이 어려웠다.

이번에는 각 함수마다 AST를 사용한다. 이렇게 하면 `break`와 `continue`를 구현하는 것이 훨씬 쉬워진다. 그 이유를 아래에서 간략히 설명한다.


## `break`와 `continue` 추가

예상대로, 새로운 토큰인 `T_BREAK`와 `T_CONTINUE`가 추가되었다. `scan.c` 파일 내 스캐너 코드는 `break`와 `continue` 키워드를 인식한다. 항상 그렇듯, 이 작업이 어떻게 수행되는지 코드를 살펴보면 도움이 된다.


## 새로운 AST 노드 타입

`defs.h` 파일에는 두 가지 새로운 AST 노드 타입이 추가되었다: A_BREAK와 A_CONTINUE.  
`break` 키워드를 파싱할 때 A_BREAK AST 리프 노드를 생성할 수 있고,  
`continue` 키워드에 대해서는 A_CONTINUE 리프 노드를 생성한다.

AST를 순회하며 어셈블리 코드를 생성할 때, A_BREAK 노드를 만나면  
현재 속한 루프의 끝을 가리키는 레이블로 점프하는 어셈블리 코드를 생성한다.  
A_CONTINUE 노드의 경우, 루프 조건이 평가되기 직전의 레이블로 점프하는 코드를 생성한다.

그렇다면 현재 어떤 루프에 속해 있는지 어떻게 알 수 있을까?


## 가장 최근 루프 추적하기

루프는 중첩될 수 있으며, 따라서 어떤 시점에도 여러 루프 라벨이 사용 중일 수 있다. 이 부분이 이전 컴파일러를 작성할 때 어려웠던 점이다. 이제는 재귀적으로 순회할 수 있는 AST(추상 구문 트리)가 있기 때문에, 가장 최근 루프의 라벨 정보를 AST 트리의 자식 노드로 전달할 수 있다.

이미 'if'나 'while' 문의 끝에 도달하기 위해 이런 방식을 사용하고 있다. 다음은 `gen.c`에서 'if' 문에 대한 어셈블리 코드를 생성하는 일부 코드이다:

```c
// IF 문과 선택적인 ELSE 절에 대한 코드 생성
static int genIF(struct ASTnode *n) {
  int Lfalse, Lend;

  // 두 개의 라벨 생성: 하나는 거짓 조건에 대한 복합문을 위한 것이고,
  // 다른 하나는 전체 IF 문의 끝을 위한 것임.
  Lfalse = genlabel();
  Lend = genlabel();

  // 조건 코드를 생성한 후 거짓 라벨로 점프하는 코드 생성
  genAST(n->left, Lfalse, n->op);
```

왼쪽 AST 자식 노드는 'if' 문의 조건을 평가하는 역할을 하므로, 방금 생성한 라벨에 접근할 필요가 있다. 따라서 `genAST()`를 통해 이 자식 노드의 어셈블리 출력을 생성할 때, 라벨 정보도 함께 전달한다.

루프의 경우, `genAST()`에 루프의 끝 라벨과 루프 조건을 평가하는 코드 바로 앞의 라벨을 전달해야 한다. 이를 위해 `genAST()`의 인터페이스를 다음과 같이 변경했다:

```c
int genAST(struct ASTnode *n, int iflabel, int looptoplabel,
           int loopendlabel, int parentASTop);
```

기존의 `iflabel`을 유지하면서 두 개의 루프 라벨을 추가했다. 이제 각 루프에 대해 생성된 라벨을 `genAST()`로 전달해야 한다. 따라서 'while' 루프 코드를 생성하는 부분은 다음과 같다:

```c
static int genWHILE(struct ASTnode *n) {
  int Lstart, Lend;

  // 시작과 끝 라벨 생성
  Lstart = genlabel();
  Lend = genlabel();

  // 조건 코드를 생성한 후 끝 라벨로 점프하는 코드 생성
  genAST(n->left, Lend, Lstart, Lend, n->op);

  // 루프 본문에 대한 복합문 생성
  genAST(n->right, NOLABEL, Lstart, Lend, n->op);
  ...
}
```


## `genAST()`는 재귀적으로 동작한다

이제 중첩된 루프를 살펴보자. 다음 코드를 고려해보자:

```
L1:
  while (x < 10) {
    if (x == 6) break;
L2:
    while (y < 10) {
      if (y == 6) break;
      y++;
    }
L3:
    x++;
  }
L4:
```

여기서 `if (y == 6) break`는 내부 루프를 벗어나 `x++` 코드(즉, L3)로 이동해야 한다. 그리고 `if (x == 6) break`는 외부 루프를 벗어나 L4로 이동해야 한다.

이 동작이 가능한 이유는 `genAST()`가 외부 루프를 위해 `genWHILE()`을 호출하기 때문이다. 이 함수는 `genAST(L1, L4)`를 호출하여 첫 번째 `break`가 이 루프 레이블을 인식하도록 한다. 그런 다음 두 번째 루프에 도달하면 다시 `genWHILE()`이 호출된다. 이 함수는 새로운 루프 레이블을 생성하고 `genAST(L2, L3)`를 호출하여 내부 루프 코드를 생성한다. 따라서 두 번째 `break`는 L1과 L4가 아닌 L2와 L3 레이블을 인식한다.

마지막으로, 내부 복합문이 생성되면 내부 `genAST()`가 반환되고, 이제는 L1과 L4를 루프 레이블로 인식하는 코드로 돌아간다.


## 위 내용의 함의

구현 측면에서 이 내용이 의미하는 바는, `genAST()`를 호출하는 모든 곳(자기 자신 포함)에서 루프에 있을 수 있으며, 이 경우 현재 루프 라벨을 관련된 자식 노드로 전파해야 한다는 것이다.

이미 `genWHILE()`을 변경해 새로운 루프 라벨을 `genAST()`에 전달하는 것을 살펴보았다. 이제 루프 라벨을 전파해야 하는 다른 부분을 살펴보자.

처음 `break`를 구현했을 때, 다음과 같은 테스트 프로그램을 작성했다.

```c
int main() {
  int x;
  x = 0;
  while (x < 100) {
    printf("%d\n", x);
    if (x == 14) { break; }
    x = x + 1;
  }
  printf("Done\n");
```

그리고 이 프로그램에 대한 어셈블리 코드를 생성했다. `break`는 L0 라벨로 점프하는 코드로 변환되었는데, 즉 루프의 종료 라벨이 `break`를 처리하는 코드에 전달되지 않았다는 것을 의미한다. 컴파일러의 스택 트레이스를 살펴보면서 다음 사실을 깨달았다.

  + 함수 호출을 위한 `genAST()`가 호출되었다.
  + 루프를 위한 `genWHILE()`이 라벨을 생성하고 이를 `genAST()`에 전달했다.
  + 루프 바디를 위한 `genAST()`가 호출되었고, 이는 `genIF()`를 호출했다.
  + `genIF()`는 라벨을 전달하지 않았고, 따라서 `break`는 라벨을 볼 수 없었다.

그래서 `genIF()`의 인자 목록도 수정해야 했다.

```c
static int genIF(struct ASTnode *n, int looptoplabel, int loopendlabel);
```

`gen.c` 파일의 모든 코드를 다루지는 않겠지만, 편집기나 텍스트 뷰어에서 파일을 열어 모든 `genAST()` 호출을 찾아보고 루프 라벨이 전파되는 위치를 확인할 수 있다.

마지막으로, 실제로 `break`와 `continue`에 대한 어셈블리 코드를 생성해야 한다. `gen.c`의 `genAST()`에서 이를 수행하는 코드는 다음과 같다.

```c
    case A_BREAK:
      cgjump(loopendlabel);
      return (NOREG);
    case A_CONTINUE:
      cgjump(looptoplabel);
      return (NOREG);
```


## `break`와 `continue` 구문 분석

이번에는 코드 생성 부분을 먼저 다루었지만, 이제는 이 새로운 키워드를 파싱할 차례다. 다행히도 `break ;` 또는 `continue ;`와 같은 간단한 구문이므로 파싱이 쉬워 보인다. 하지만 약간의 주의할 점이 있다.

개별 문장은 `stmt.c` 파일의 `single_statement()` 함수에서 파싱하므로 변경 사항은 크지 않다:

```c
    case T_BREAK:
      return (break_statement());
    case T_CONTINUE:
      return (continue_statement());
```

`compound_statement()` 함수에도 약간의 변경을 추가해 문장이 세미콜론으로 끝나는지 확인한다:

```c
compound_statement(void) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  ...
  while (1) {
    // 단일 문장 파싱
    tree = single_statement();

    // 특정 문장은 세미콜론으로 끝나야 함
    if (tree != NULL && (tree->op == A_ASSIGN || tree->op == A_RETURN
                         || tree->op == A_FUNCCALL || tree->op == A_BREAK
                         || tree->op == A_CONTINUE))
      semi();
    ...
}
```

여기서 주의할 점은 다음 프로그램이 유효하지 않다는 것이다:

```c
int main() {
  break;
}
```

이는 탈출할 루프가 없기 때문이다. 따라서 파싱 중인 루프의 깊이를 추적하고, 깊이가 0이 아닐 때만 `break` 또는 `continue` 문을 허용해야 한다. 이러한 키워드를 파싱하는 함수는 다음과 같다:

```c
// break 문을 파싱하고 AST를 반환
static struct ASTnode *break_statement(void) {

  if (Looplevel == 0)
    fatal("탈출할 루프가 없음");
  scan(&Token);
  return (mkastleaf(A_BREAK, 0, NULL, 0));
}

// continue_statement: 'continue' ;
//
// continue 문을 파싱하고 AST를 반환
static struct ASTnode *continue_statement(void) {

  if (Looplevel == 0)
    fatal("계속할 루프가 없음");
  scan(&Token);
  return (mkastleaf(A_CONTINUE, 0, NULL, 0));
}
```


## 루프 레벨

파싱 중인 루프의 깊이를 추적하기 위해 `Looplevel` 변수가 필요하다. 이 변수는 `data.h`에 정의되어 있다:

```c
extern_ int Looplevel;                  // 중첩된 루프의 깊이
```

필요에 따라 이 레벨을 설정해야 한다. 새로운 함수를 시작할 때마다 레벨을 0으로 초기화한다 (`decl.c`에서):

```c
// 함수 선언을 파싱한다.
struct ASTnode *function_declaration(int type) {
  ...
  // 복합문(compound statement)에 대한 AST 트리를 가져오고
  // 아직 루프를 파싱하지 않았음을 표시한다
  Looplevel= 0;
  tree = compound_statement();
  ...
}
```

이제 루프를 파싱할 때마다 루프의 본문에 대한 루프 레벨을 증가시킨다 (`stmt.c`에서):

```c
// WHILE 문을 파싱하고 해당 AST를 반환한다
static struct ASTnode *while_statement(void) {
  ...
  // 복합문에 대한 AST를 가져온다.
  // 이 과정에서 루프 깊이를 업데이트한다
  Looplevel++;
  bodyAST = compound_statement();
  Looplevel--;
  ...
}

// FOR 문을 파싱하고 해당 AST를 반환한다
static struct ASTnode *for_statement(void) {
  ...
  // 본문인 복합문을 가져온다
  // 이 과정에서 루프 깊이를 업데이트한다
  Looplevel++;
  bodyAST = compound_statement();
  Looplevel--;
  ...
}
```

이를 통해 현재 루프 내부에 있는지 여부를 판단할 수 있다.


## 테스트 코드

다음은 테스트 코드 `tests/input71.c`의 내용이다:

```c
#include <stdio.h>

int main() {
  int x;
  x = 0;
  while (x < 100) {
    if (x == 5) { x = x + 2; continue; }
    printf("%d\n", x);
    if (x == 14) { break; }
    x = x + 1;
  }
  printf("Done\n");
  return (0);
}
```

아직 'dangling else' 문제를 해결하지 못했기 때문에, `break` 문을 복합문으로 만들기 위해 '{' ... '}'로 감싸야 한다. 그 외에는 코드가 예상대로 동작한다:

```
0
1
2
3
4
7
8
9
10
11
12
13
14
Done
```


## 결론과 다음 단계

AST를 사용했기 때문에 `break`와 `continue`를 지원하는 기능을 추가하는 작업이 이전 컴파일러보다 더 쉬울 것이라고 생각했다. 하지만 구현 과정에서 여전히 몇 가지 사소한 문제와 까다로운 부분을 해결해야 했다.

이제 언어에 `break` 키워드가 추가되었으니, 다음 단계에서는 `switch` 문을 추가하려고 한다. 이 작업은 switch 점프 테이블을 추가해야 하기 때문에 복잡할 것으로 예상된다. 따라서 흥미로운 다음 단계를 준비하자. [다음 단계](../37_Switch/Readme.md)



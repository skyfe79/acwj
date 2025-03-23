# 43부: 버그 수정과 추가 연산자

컴파일러의 소스 코드 일부를 컴파일러 자체에 입력으로 전달하기 시작했다. 이 방법은 결국 컴파일러가 스스로를 컴파일할 수 있게 만드는 과정이다. 첫 번째 큰 장벽은 컴파일러가 자신의 소스 코드를 파싱하고 인식하도록 하는 것이다. 두 번째 큰 장벽은 컴파일러가 소스 코드로부터 정확하고 작동하는 코드를 생성하도록 하는 것이다.

이번이 컴파일러가 상당한 크기의 입력을 받아 처리하는 첫 번째 시도다. 이 과정에서 여러 버그, 잘못된 기능, 그리고 누락된 기능들이 드러날 것이다.


## 버그 수정

`cwj -S defs.h` 명령어로 시작했을 때, 여러 헤더 파일이 누락된 것을 발견했다. 현재 이 파일들은 존재하지만 비어 있다. 이 파일들을 추가한 후, 컴파일러가 세그멘테이션 오류(segfault)로 중단되는 문제가 발생했다. 몇몇 포인터를 NULL로 초기화해야 했고, NULL 포인터를 확인하지 않은 부분도 있었다.


## 누락된 기능

다음으로, `defs.h` 파일에서 `enum { NOREG = -1 ...` 부분을 만나면서 스캐너가 마이너스 기호로 시작하는 정수 리터럴을 처리하지 못한다는 사실을 깨달았다. 그래서 `scan.c` 파일의 `scan()` 함수에 다음과 같은 코드를 추가했다:

```c
    case '-':
      if ((c = next()) == '-') {
        t->token = T_DEC;
      } else if (c == '>') {
        t->token = T_ARROW;
      } else if (isdigit(c)) {          // 음의 정수 리터럴
        t->intvalue = -scanint(c);
        t->token = T_INTLIT;
      } else {
        putback(c);
        t->token = T_MINUS;
      }
```

'-' 뒤에 숫자가 오는 경우, 정수 리터럴을 스캔하고 그 값을 음수로 변환한다. 처음에는 `1 - 1`과 같은 표현식이 '1', '정수 리터럴 -1'로 처리될까 걱정했지만, `next()` 함수가 공백을 건너뛰지 않는다는 사실을 잊고 있었다. 따라서 '-'와 '1' 사이에 공백이 있으면 `1 - 1`이 올바르게 '1', '-', '1'로 파싱된다.

하지만 [Luke Gruber](https://github.com/luke-gru)가 지적한 것처럼, 이는 `1-1`과 같은 입력이 `1 -1`로 처리된다는 것을 의미한다. 즉, 스캐너가 너무 탐욕적이어서 `-1`이 항상 T_INTLIT로 처리되도록 강제하는데, 때로는 그렇게 해서는 안 된다. 현재는 이 문제를 그대로 두기로 했다. 소스 코드를 작성할 때 이 문제를 우회할 수 있기 때문이다. 물론, 상용 컴파일러에서는 이 문제를 반드시 해결해야 한다.


## 잘못된 설계

AST 노드와 심볼 테이블 노드 구조에서, 각 노드의 크기를 줄이기 위해 공용체(union)를 사용했다. 메모리 낭비를 우려하는 나의 오래된 습관 때문이다. 예를 들어, AST 노드 구조는 다음과 같다:

```c
struct ASTnode {
  int op;                       // 트리에서 수행할 "연산"
  ...
  union {                       // 심볼 테이블의 심볼
    int intvalue;               // A_INTLIT일 때, 정수 값
    int size;                   // A_SCALE일 때, 스케일링할 크기
  };
};
```

하지만 컴파일러는 구조체 내부의 공용체, 특히 이름 없는 공용체를 파싱하고 처리할 수 없다. 이 기능을 추가할 수도 있지만, 이렇게 사용한 두 구조체를 다시 작성하는 것이 더 쉬운 방법이다. 그래서 다음과 같이 변경했다:

```c
// 심볼 테이블 구조
struct symtable {
  char *name;                   // 심볼의 이름
  ...
#define st_endlabel st_posn     // 함수의 경우, 종료 레이블
  int st_posn;                  // 지역 변수의 경우, 스택 베이스 포인터로부터의 음수 오프셋
  ...
};

// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 트리에서 수행할 "연산"
  ...
#define a_intvalue a_size       // A_INTLIT일 때, 정수 값
  int a_size;                   // A_SCALE일 때, 스케일링할 크기
};
```

이제 두 구조체에서 각각 두 개의 필드가 같은 위치를 공유하지만, 컴파일러는 각 구조체에서 하나의 필드 이름만 보게 된다. 전역 네임스페이스 오염을 방지하기 위해 각 `#define`에 다른 접두사를 붙였다.

이 변경으로 인해 `endlabel`, `posn`, `intvalue`, `size` 필드를 여러 소스 파일에서 이름을 바꿔야 했다. 어쩔 수 없는 일이다.

이제 컴파일러가 `cwj -S misc.c`를 실행하면 다음과 같은 오류가 발생한다:

```
Expected:] on line 16 of data.h, where the line is
extern char Text[TEXTLEN + 1];
```

현재 컴파일러는 전역 변수 선언에서 표현식을 파싱할 수 없어 실패한다. 이 부분을 다시 생각해봐야 한다.

지금까지의 생각은 `binexpr()`를 사용해 표현식을 파싱하고, 결과로 나온 AST 트리에서 [상수 폴딩](https://en.wikipedia.org/wiki/Constant_folding)을 수행하는 최적화 코드를 추가하는 것이다. 이렇게 하면 단일 A_INTLIT 노드를 얻을 수 있고, 여기서 리터럴 값을 추출할 수 있다. 심지어 `binexpr()`가 캐스트도 파싱하도록 할 수 있다. 예를 들어:

```c
 char x= (char)('a' + 1024);
```

어쨌든 이건 나중에 할 일이다. 상수 폴딩을 언젠가는 할 생각이었지만, 더 먼 미래의 일이라고 생각했다.

이번 단계에서 할 일은 몇 가지 연산자를 추가하는 것이다. 구체적으로 '+=', '-=', '*=', '/=' 연산자를 추가할 예정이다. 현재 컴파일러 소스 코드에서 처음 두 연산자를 사용하고 있다.


## 새로운 토큰, 스캐닝 및 파싱

컴파일러에 새로운 키워드를 추가하는 것은 간단하다. 새로운 토큰을 정의하고 스캐너를 수정하면 된다. 하지만 새로운 연산자를 추가하는 것은 훨씬 복잡하다. 다음과 같은 작업이 필요하기 때문이다:

  + 토큰을 AST 연산과 일치시킨다.
  + 우선순위와 결합 방향을 처리한다.

우리는 네 가지 연산자를 추가한다: '+=', '-=', '*=', '/='. 이 연산자들은 각각 T_ASPLUS, T_ASMINUS, T_ASSTAR, T_ASSLASH라는 토큰과 매핑된다. 이 토큰들은 A_ASPLUS, A_ASMINUS, A_ASSTAR, A_ASSLASH라는 AST 연산에 대응한다. AST 연산은 토큰과 동일한 열거형 값을 가져야 한다. `expr.c` 파일에 있는 다음 함수 때문이다:

```c
// 이진 연산자 토큰을 이진 AST 연산으로 변환한다.
// 토큰과 AST 연산 간 1:1 매핑에 의존한다.
static int binastop(int tokentype) {
  if (tokentype > T_EOF && tokentype <= T_SLASH)
    return (tokentype);
  fatald("Syntax error, token", tokentype);
  return (0);                   // Keep -Wall happy
}
```

또한 새로운 연산자의 우선순위를 설정해야 한다. [C 연산자 우선순위 목록](https://en.cppreference.com/w/c/language/operator_precedence)에 따르면, 이 새로운 연산자들은 기존의 할당 연산자와 동일한 우선순위를 가진다. 따라서 `expr.c` 파일의 `OpPrec[]` 테이블을 다음과 같이 수정한다:

```c
// 각 토큰의 연산자 우선순위. defs.h 파일의 토큰 순서와 일치해야 한다.
static int OpPrec[] = {
  0, 10, 10,                    // T_EOF, T_ASSIGN, T_ASPLUS,
  10, 10, 10,                   // T_ASMINUS, T_ASSTAR, T_ASSLASH,
  20, 30,                       // T_LOGOR, T_LOGAND
  ...
};
```

그러나 C 연산자 목록은 할당 연산자들이 *오른쪽 결합*임을 명시한다. 예를 들어:

```c
   a += b + c;          // 다음과 같이 파싱되어야 한다.
   a += (b + c);        // 이렇게 파싱되면 안 된다.
   (a += b) + c;
```

따라서 `expr.c` 파일의 다음 함수도 수정해야 한다:

```c
// 토큰이 오른쪽 결합인지 여부를 반환한다.
static int rightassoc(int tokentype) {
  if (tokentype >= T_ASSIGN && tokentype <= T_ASSLASH)
    return (1);
  return (0);
}
```

다행히도, 스캐너와 표현식 파서에 필요한 변경 사항은 이게 전부다. 이진 표현식을 위한 Pratt 파서는 이제 새로운 연산자를 처리할 준비가 되었다.


## AST 트리 다루기

이제 새로운 네 가지 연산자를 사용해 표현식을 파싱할 수 있게 되었으니, 각 표현식에 대해 생성된 AST를 다루어야 한다. 먼저 AST 트리를 덤프하는 작업이 필요하다. `tree.c` 파일의 `dumpAST()` 함수에 다음 코드를 추가했다:

```c
    case A_ASPLUS:
      fprintf(stdout, "A_ASPLUS\n"); return;
    case A_ASMINUS:
      fprintf(stdout, "A_ASMINUS\n"); return;
    case A_ASSTAR:
      fprintf(stdout, "A_ASSTAR\n"); return;
    case A_ASSLASH:
      fprintf(stdout, "A_ASSLASH\n"); return;
```

이제 `cwj -T input.c` 명령어를 실행하고 `a += b + c` 표현식을 입력하면 다음과 같은 결과를 볼 수 있다:

```
  A_IDENT rval a
    A_IDENT rval b
    A_IDENT rval c
  A_ADD
A_ASPLUS
```

이를 다시 그려보면 다음과 같은 구조가 된다:

```
          A_ASPLUS
         /        \
     A_IDENT     A_ADD
     rval a     /     \
            A_IDENT  A_IDENT
             rval b  rval c
```


## 연산자에 대한 어셈블리 생성

`gen.c` 파일에서는 이미 AST 트리를 순회하며 A_ADD와 A_ASSIGN을 처리한다. 이 기존 코드를 활용해 새로운 A_ASPLUS 연산자를 구현하는 방법이 있을까? 있다!

위의 AST 트리를 다음과 같이 재구성할 수 있다:

```
                A_ASSIGN
               /       \
            A_ADD      lval a
         /        \
     A_IDENT     A_ADD
     rval a     /     \
            A_IDENT   A_IDENT
             rval b   rval c
```

이제 트리를 실제로 재구성할 필요 없이, 마치 트리가 이렇게 재구성된 것처럼 트리 순회를 수행하면 된다.

`genAST()` 함수에서는 다음과 같이 처리한다:

```c
int genAST(...) {
  ...
  // 왼쪽과 오른쪽 서브 트리 값을 가져온다. 이 코드는 이미 존재한다.
  if (n->left)
    leftreg = genAST(n->left, NOLABEL, NOLABEL, NOLABEL, n->op);
  if (n->right)
    rightreg = genAST(n->right, NOLABEL, NOLABEL, NOLABEL, n->op);
}
```

A_ASPLUS 노드의 작업을 수행하는 관점에서 보면, 왼쪽 자식(예: `a`의 값)과 오른쪽 자식(예: `b+c`)을 평가하고 두 레지스터에 값을 저장한 상태다. 만약 이 연산이 A_ADD라면, 이 시점에서 `cgadd(leftreg, rightreg)`를 호출할 것이다. 이 경우에는 이러한 자식들에 대해 A_ADD 연산을 수행한 후, 그 결과를 `a`에 다시 할당하는 과정이 필요하다.

따라서 `genAST()` 코드는 이제 다음과 같다:

```c
  switch (n->op) {
    ... 
    case A_ASPLUS:
    case A_ASMINUS:
    case A_ASSTAR:
    case A_ASSLASH:
    case A_ASSIGN:

      // '+=' 및 유사 연산자에 대해 적절한 코드를 생성하고
      // 결과가 저장된 레지스터를 얻는다. 그런 다음 왼쪽 자식을
      // 오른쪽 자식으로 만들어 할당 코드로 넘어갈 수 있게 한다.
      switch (n->op) {
        case A_ASPLUS:
          leftreg= cgadd(leftreg, rightreg);
          n->right= n->left;
          break;
        case A_ASMINUS:
          leftreg= cgsub(leftreg, rightreg);
          n->right= n->left;
          break;
        case A_ASSTAR:
          leftreg= cgmul(leftreg, rightreg);
          n->right= n->left;
          break;
        case A_ASSLASH:
          leftreg= cgdiv(leftreg, rightreg);
          n->right= n->left;
          break;
      }

      // A_ASSIGN을 수행하는 기존 코드는 여기에 있다.
     ...
  }
```

다시 말해, 각 새로운 연산자에 대해 자식 노드에 올바른 수학 연산을 수행한다. 하지만 A_ASSIGN으로 넘어가기 전에 왼쪽 자식 포인터를 오른쪽 자식으로 옮겨야 한다. 왜냐하면 A_ASSIGN 코드는 목적지가 오른쪽 자식이 될 것으로 기대하기 때문이다:

```c
      return (cgstorlocal(leftreg, n->right->sym));
```

이것으로 끝이다. 이 네 가지 새로운 연산자를 추가하기 위해 기존 코드를 활용할 수 있어 운이 좋았다. 아직 구현하지 않은 할당 연산자도 더 있다: '%=', '<=', '>>=', '&=', '^=', '|=' 등이다. 이들도 방금 추가한 네 연산자만큼 쉽게 추가할 수 있을 것이다.


## 예제 코드

`tests/input110.c` 프로그램은 테스트용 프로그램이다:

```c
#include <stdio.h>

int x;
int y;

int main() {
  x= 3; y= 15; y += x; printf("%d\n", y);
  x= 3; y= 15; y -= x; printf("%d\n", y);
  x= 3; y= 15; y *= x; printf("%d\n", y);
  x= 3; y= 15; y /= x; printf("%d\n", y);
  return(0);
}
```

이 프로그램은 다음과 같은 결과를 출력한다:

```
18
12
45
5
```


## 결론 및 다음 단계

몇 가지 새로운 연산자를 추가했고, 가장 어려웠던 부분은 모든 토큰, AST 연산자를 정렬하고 우선순위 레벨과 오른쪽 결합성을 설정하는 작업이었다. 이후에는 `genAST()` 함수에서 코드 생성 로직을 재사용하여 작업을 조금 더 쉽게 진행할 수 있었다.

컴파일러 개발 여정의 다음 단계에서는 상수 폴딩(constant folding)을 추가할 예정이다. [다음 단계](../44_Fold_Optimisation/Readme.md)



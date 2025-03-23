# 18장: Lvalue와 Rvalue 다시 보기

이 부분은 아직 진행 중인 작업으로, 설계 문서가 없어 가이드라인이 없다. 때로는 이미 작성한 코드를 제거하고 더 일반적으로 만들거나 문제점을 해결하기 위해 다시 작성해야 한다. 이번 장도 그런 경우에 해당한다.

15장에서 포인터에 대한 초기 지원을 추가했고, 이를 통해 다음과 같은 코드를 작성할 수 있었다:

```c
  int  x;
  int *y;
  int  z;
  x= 12; y= &x; z= *y;
```

이것은 괜찮지만, 결국에는 대입문의 왼쪽에서 포인터를 사용하는 것도 지원해야 한다는 것을 알고 있었다. 예를 들어:

```c
  *y = 14;
```

이를 위해 [Lvalue와 Rvalue](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue) 주제를 다시 살펴봐야 한다. 복습하자면, *Lvalue*는 특정 위치에 연결된 값을 의미하고, *Rvalue*는 그렇지 않은 값을 의미한다. Lvalue는 지속적이므로 이후 명령에서 그 값을 다시 가져올 수 있다. 반면 Rvalue는 일시적이며, 사용이 끝나면 버려진다.


### Rvalue와 Lvalue 예제

Rvalue의 예로는 정수 리터럴인 `23`을 들 수 있다. 이 값을 표현식에서 사용한 후 버릴 수 있다. Lvalue의 예로는 메모리 상의 위치를 들 수 있는데, 이 위치에 값을 *저장할 수 있다*. 예를 들어:

```
   a            스칼라 변수 a
   b[0]         배열 b의 0번째 요소
   *c           포인터 c가 가리키는 위치
   (*d)[0]      포인터 d가 가리키는 배열의 0번째 요소
```

앞서 언급했듯이, *lvalue*와 *rvalue*라는 이름은 대입문의 양쪽에서 유래했다. Lvalue는 왼쪽에, rvalue는 오른쪽에 위치한다.


## Lvalue 개념 확장

현재 컴파일러는 거의 모든 것을 rvalue로 처리한다. 변수의 경우, 변수 위치에서 값을 가져온다. lvalue 개념을 다루는 유일한 부분은 할당문 왼쪽에 있는 식별자를 A_LVIDENT로 표시하는 것이다. 이를 `genAST()` 함수에서 수동으로 처리한다:

```c
    case A_IDENT:
      return (cgloadglob(n->v.id));
    case A_LVIDENT:
      return (cgstorglob(reg, n->v.id));
    case A_ASSIGN:
      // 작업은 이미 완료되었으므로 결과를 반환
      return (rightreg);
```

이 코드는 `a = b;`와 같은 구문을 처리할 때 사용된다. 하지만 이제는 할당문 왼쪽에 있는 식별자뿐만 아니라 더 많은 요소를 lvalue로 표시해야 한다.

이 과정에서 어셈블리 코드를 쉽게 생성할 수 있도록 하는 것도 중요하다. 이 부분을 작성하면서 "A_LVALUE" AST 노드를 트리의 부모 노드로 추가하여 코드 생성기가 rvalue 버전 대신 lvalue 버전의 코드를 출력하도록 하는 아이디어를 시도해 보았다. 하지만 이 방법은 이미 너무 늦은 시점이었다. 서브 트리는 이미 평가되었고, rvalue 코드는 이미 생성된 상태였다.


### 또 다른 AST 노드 변경

AST 노드에 필드를 계속 추가하는 것을 꺼리지만, 결국 이렇게 하게 되었다. 이제 노드가 lvalue 코드를 생성할지 rvalue 코드를 생성할지 나타내는 필드를 추가했다:

```c
// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  int type;                     // 이 트리가 생성하는 표현식의 타입
  int rvalue;                   // 노드가 rvalue인지 여부
  ...
};
```

`rvalue` 필드는 단 한 비트의 정보만을 담는다. 나중에 다른 불리언 값을 저장해야 할 경우, 이 필드를 비트 필드로 사용할 수 있다.

왜 이 필드를 "rvalue" 여부를 나타내도록 만들었을까? 결국 대부분의 AST 노드는 rvalue를 담고 있지 lvalue를 담고 있지 않다. Nils Holm의 SubC 책을 읽다가 이런 문장을 보았다:

> 역참조는 나중에 되돌릴 수 없기 때문에, 파서는 각 부분 표현식을 lvalue로 가정한다.

파서가 `b = a + 2`라는 문장을 처리하는 상황을 생각해 보자. `b` 식별자를 파싱한 후, 이것이 lvalue인지 rvalue인지 아직 알 수 없다. `=` 토큰을 만나야 비로소 이것이 lvalue라는 결론을 내릴 수 있다.

또한 C 언어는 할당을 표현식으로 허용하기 때문에 `b = c = a + 2`와 같은 코드도 작성할 수 있다. 다시 말해, `a` 식별자를 파싱할 때는 이것이 lvalue인지 rvalue인지 다음 토큰을 파싱할 때까지 알 수 없다.

따라서 각 AST 노드를 기본적으로 lvalue로 가정하기로 결정했다. 노드가 명확하게 rvalue임을 알게 되면, 그때 `rvalue` 필드를 설정하여 이를 나타낸다.


## 할당 표현식

앞서 C 언어가 할당을 표현식으로 허용한다는 점을 언급했다. 이제 lvalue와 rvalue를 명확히 구분했으므로, 할당을 문장으로 파싱하던 방식을 바꿔 표현식 파서로 코드를 이동할 수 있다. 이 부분은 나중에 자세히 다룰 예정이다.

이제 이러한 변화를 구현하기 위해 컴파일러 코드 베이스에서 어떤 작업을 했는지 살펴보자. 언제나 그렇듯이, 토큰과 스캐너부터 시작한다.


## 토큰과 스캐닝 변경 사항

이번에는 새로운 토큰이나 키워드가 추가되지는 않았다. 하지만 토큰 코드에 영향을 미치는 변경 사항이 있다. `=` 연산자가 이제 양쪽에 표현식을 가지는 바이너리 연산자로 변경되었기 때문에, 이를 다른 바이너리 연산자들과 통합해야 한다.

[C 연산자 우선순위 목록](https://en.cppreference.com/w/c/language/operator_precedence)에 따르면, `=` 연산자는 `+`나 `-`보다 훨씬 낮은 우선순위를 가진다. 따라서 연산자 목록과 그 우선순위를 재정렬해야 한다. `defs.h` 파일에서는 다음과 같이 정의한다:

```c
// 토큰 타입
enum {
  T_EOF,
  // 연산자
  T_ASSIGN,
  T_PLUS, T_MINUS, ...
```

`expr.c` 파일에서는 바이너리 연산자의 우선순위를 담당하는 코드를 업데이트해야 한다:

```c
// 각 토큰에 대한 연산자 우선순위. 
// defs.h 파일의 토큰 순서와 일치해야 함
static int OpPrec[] = {
   0, 10,                       // T_EOF,  T_ASSIGN
  20, 20,                       // T_PLUS, T_MINUS
  30, 30,                       // T_STAR, T_SLASH
  40, 40,                       // T_EQ, T_NE
  50, 50, 50, 50                // T_LT, T_GT, T_LE, T_GE
};
```


## 파서 변경 사항

이제 할당문을 문장(statement)으로 파싱하는 기능을 제거하고 표현식(expression)으로 변경한다. 또한 `printint()` 함수를 호출할 수 있게 되었으므로 "print" 문을 언어에서 제거했다. 따라서 `stmt.c` 파일에서 `print_statement()`와 `assignment_statement()` 함수를 모두 삭제했다.

> T_PRINT 토큰과 'print' 키워드도 언어에서 제거했다. 또한 lvalue와 rvalue 개념이 달라졌기 때문에 A_LVIDENT AST 노드 타입도 제거했다.

현재 `stmt.c` 파일의 `single_statement()` 함수에서 첫 번째 토큰을 인식하지 못하면 다음에 오는 것을 표현식으로 간주한다:

```c
static struct ASTnode *single_statement(void) {
  int type;

  switch (Token.token) {
    ...
    default:
    // 일단 이게 표현식인지 확인한다.
    // 이렇게 하면 할당문도 잡아낸다.
    return (binexpr(0));
  }
}
```

이렇게 하면 `2+3;` 같은 코드도 현재는 합법적인 문장으로 처리된다. 이 문제는 나중에 수정할 예정이다. 그리고 `compound_statement()` 함수에서는 표현식 뒤에 세미콜론이 오는지 확인한다:

```c
    // 일부 문장은 세미콜론으로 끝나야 한다
    if (tree != NULL && (tree->op == A_ASSIGN ||
                         tree->op == A_RETURN || tree->op == A_FUNCCALL))
      semi();
```


## 표현식 파싱

`=`를 바이너리 표현식 연산자로 표시하고 우선순위를 설정했으니, 모든 작업이 끝났다고 생각할 수 있다. 하지만 그렇지 않다! 두 가지 문제를 해결해야 한다:

1. 왼쪽 lvalue에 대한 코드를 생성하기 전에 오른쪽 rvalue에 대한 어셈블리 코드를 먼저 생성해야 한다. 이전에는 문장 파서에서 이 작업을 수행했지만, 이제는 표현식 파서에서도 동일한 작업을 해야 한다.
2. 할당 표현식은 *오른쪽 결합성(right associative)*을 가진다: 연산자가 왼쪽보다 오른쪽의 표현식과 더 강하게 결합한다.

지금까지 오른쪽 결합성에 대해 다루지 않았다. 예제를 통해 살펴보자. `2 + 3 + 4`라는 표현식을 생각해 보자. 이 표현식은 왼쪽에서 오른쪽으로 파싱하여 AST 트리를 구성할 수 있다:

```
      +
     / \
    +   4
   / \
  2   3
```

하지만 `a= b= 3`이라는 표현식의 경우, 위와 같은 방식으로 파싱하면 다음과 같은 트리가 생성된다:

```
      =
     / \
    =   3
   / \
  a   b
```

이 경우, `a= b`를 먼저 수행한 후 3을 왼쪽 하위 트리에 할당하려고 시도하면 안 된다. 대신, 다음과 같은 트리를 생성해야 한다:

```
        =
       / \
      =   a
     / \
    3   b
```

리프 노드를 어셈블리 출력 순서에 맞게 뒤집었다. 먼저 3을 `b`에 저장한다. 그런 다음 이 할당의 결과인 3을 `a`에 저장한다.


### Pratt 파서 수정하기

우리는 이항 연산자의 우선순위를 올바르게 파싱하기 위해 Pratt 파서를 사용하고 있다. Pratt 파서에 오른쪽 결합성을 추가하는 방법을 찾기 위해 검색을 해보았고, [위키피디아](https://en.wikipedia.org/wiki/Operator-precedence_parser)에서 다음과 같은 정보를 발견했다:

```
   lookahead가 이항 연산자이고 그 우선순위가 op보다 크거나,
   오른쪽 결합성 연산자이고 그 우선순위가 op와 같을 때까지 반복
```

즉, 오른쪽 결합성 연산자의 경우, 다음 연산자의 우선순위가 현재 연산자의 우선순위와 같은지 확인한다. 이는 파서의 로직을 간단히 수정하면 된다. `expr.c` 파일에 오른쪽 결합성인지 확인하는 새로운 함수를 추가했다:

```c
// 토큰이 오른쪽 결합성인지 확인한다.
// 오른쪽 결합성이면 true, 아니면 false를 반환한다.
static int rightassoc(int tokentype) {
  if (tokentype == T_ASSIGN)
    return(1);
  return(0);
}
```

`binexpr()` 함수에서는 앞서 언급한대로 while 루프를 수정하고, A_ASSIGN에 특화된 코드를 추가해 자식 트리를 교체한다:

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  struct ASTnode *ltemp, *rtemp;
  int ASTop;
  int tokentype;

  // 왼쪽 트리를 가져온다.
  left = prefix();
  ...

  // 현재 토큰의 우선순위가 이전 토큰의 우선순위보다 크거나,
  // 오른쪽 결합성이고 이전 토큰의 우선순위와 같을 때까지 반복
  while ((op_precedence(tokentype) > ptp) ||
         (rightassoc(tokentype) && op_precedence(tokentype) == ptp)) {
    ...
    // 현재 토큰의 우선순위로 binexpr()를 재귀적으로 호출해 서브 트리를 만든다.
    right = binexpr(OpPrec[tokentype]);

    ASTop = binastop(tokentype);
    if (ASTop == A_ASSIGN) {
      // 대입 연산
      // 오른쪽 트리를 rvalue로 만든다.
      right->rvalue= 1;
      ...

      // 왼쪽과 오른쪽을 교체해 오른쪽 표현식의 코드가 왼쪽 표현식보다 먼저 생성되도록 한다.
      ltemp= left; left= right; right= ltemp;
    } else {
      // 대입 연산이 아니므로 양쪽 트리를 모두 rvalue로 만든다.
      left->rvalue= 1;
      right->rvalue= 1;
    }
    ...
  }
  ...
}
```

대입 표현식의 오른쪽을 명시적으로 rvalue로 표시하는 코드도 주목할 만하다. 그리고 대입 연산이 아닌 경우, 표현식의 양쪽을 모두 rvalue로 표시한다.

`binexpr()` 함수 곳곳에는 리프 노드에 도달했을 때 트리를 명시적으로 rvalue로 설정하는 코드가 더 있다. 예를 들어 `b= a;`에서 `a` 식별자는 rvalue로 표시되어야 하지만, while 루프 본문에 진입하지 않아도 이를 처리할 수 있다.


## 트리 출력하기

이제 파서의 변경 사항은 마무리했다. 현재 여러 노드가 rvalue로 표시되었고, 일부는 전혀 표시되지 않았다. 이 시점에서 생성된 AST 트리를 시각화하는 데 어려움을 느꼈다. 그래서 `tree.c` 파일에 `dumpAST()`라는 함수를 작성해 각 AST 트리를 표준 출력으로 출력하도록 했다. 이 함수는 복잡하지 않다. 이제 컴파일러는 `-T` 커맨드라인 인수를 통해 내부 플래그 `O_dumpAST`를 설정할 수 있다. 그리고 `decl.c` 파일의 `global_declarations()` 코드는 다음과 같이 동작한다:

```c
       // 함수 선언을 파싱하고
       // 해당 함수에 대한 어셈블리 코드를 생성한다
       tree = function_declaration(type);
       if (O_dumpAST) {
         dumpAST(tree, NOLABEL, 0);
         fprintf(stdout, "\n\n");
       }
       genAST(tree, NOLABEL, 0);
```

트리 덤퍼 코드는 각 노드를 트리 순회 순서대로 출력하므로, 결과물은 트리 모양이 아니다. 하지만 각 노드의 들여쓰기는 트리 내에서의 깊이를 나타낸다.

이제 할당 표현식에 대한 예제 AST 트리를 살펴보자. 먼저 `a= b= 34;`를 보자:

```
      A_INTLIT 34
    A_WIDEN
    A_IDENT b
  A_ASSIGN
  A_IDENT a
A_ASSIGN
```

34는 char 크기의 리터럴이지만, `b`의 타입에 맞게 확장된다. `A_IDENT b`는 "rvalue"라고 표시되지 않았으므로 lvalue다. 34의 값은 `b` lvalue에 저장된다. 이 값은 그 다음 `a` lvalue에 저장된다.

이번에는 `a= b + 34;`를 살펴보자:

```
    A_IDENT rval b
      A_INTLIT 34
    A_WIDEN
  A_ADD
  A_IDENT a
A_ASSIGN
```

이제 "rval `b`"가 보이므로, `b`의 값이 레지스터에 로드된다. 그리고 `b+34` 표현식의 결과는 `a` lvalue에 저장된다.

마지막으로 `*x= *y`를 살펴보자:

```
    A_IDENT y
  A_DEREF rval
    A_IDENT x
  A_DEREF
A_ASSIGN
```

식별자 `y`는 역참조되고, 이 rvalue가 로드된다. 이 값은 그 다음 `x`가 역참조된 lvalue에 저장된다.


## 코드로 변환하기

이제 lvalue와 rvalue 노드를 명확히 식별했으니, 이를 어셈블리 코드로 어떻게 변환하는지 살펴볼 차례다. 정수 리터럴, 덧셈 등과 같은 많은 노드가 명백히 rvalue에 속한다. `gen.c` 파일의 `genAST()` 함수에서 신경 써야 할 것은 lvalue가 될 가능성이 있는 AST 노드 타입뿐이다. 다음은 이러한 노드 타입에 대한 코드다:

```c
    case A_IDENT:
      // rvalue이거나 역참조 중이라면 값을 로드한다
      if (n->rvalue || parentASTop == A_DEREF)
        return (cgloadglob(n->v.id));
      else
        return (NOREG);

    case A_ASSIGN:
      // 식별자에 할당하거나 포인터를 통해 할당하는지 확인한다
      switch (n->right->op) {
        case A_IDENT: return (cgstorglob(leftreg, n->right->v.id));
        case A_DEREF: return (cgstorderef(leftreg, rightreg, n->right->type));
        default: fatald("Can't A_ASSIGN in genAST(), op", n->op);
      }

    case A_DEREF:
      // rvalue라면 포인터가 가리키는 값을 역참조한다
      // 그렇지 않다면 A_ASSIGN이 포인터를 통해 값을 저장할 수 있도록 남겨둔다
      if (n->rvalue)
        return (cgderef(leftreg, n->left->type));
      else
        return (leftreg);
```


### x86-64 코드 생성기의 변경 사항

`cg.c` 파일에서 변경된 부분은 포인터를 통해 값을 저장할 수 있게 해주는 함수뿐이다:

```c
// 역참조된 포인터를 통해 값을 저장
int cgstorderef(int r1, int r2, int type) {
  switch (type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovb\t%s, (%s)\n", breglist[r1], reglist[r2]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovq\t%s, (%s)\n", reglist[r1], reglist[r2]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, (%s)\n", reglist[r1], reglist[r2]);
      break;
    default:
      fatald("Can't cgstoderef on type:", type);
  }
  return (r1);
}
```

이 함수는 바로 앞에 위치한 `cgderef()` 함수와 거의 정반대의 기능을 수행한다.


## 결론과 다음 단계

이번 과정에서 나는 두세 가지 다른 설계 방향을 시도했다. 각각을 실험해 보았지만, 막다른 골목에 부딪히고 여기서 설명한 해결책에 도달하기 전에 되돌아와야 했다. SubC에서 Nils는 AST 트리의 노드가 처리되는 시점에 "lvalue" 상태를 담은 단일 구조체를 전달한다. 하지만 그의 트리는 단일 표현식만을 담고 있다. 반면 이 컴파일러의 AST 트리는 전체 함수에 해당하는 노드들을 포함한다. 다른 세 컴파일러를 살펴본다면 아마도 세 가지 다른 해결책을 발견할 수 있을 것이다.

다음 단계에서 우리가 다룰 수 있는 주제는 다양하다. 상대적으로 쉽게 추가할 수 있는 C 연산자들이 여럿 있다. A_SCALE이 있으므로 구조체를 시도해 볼 수 있다. 아직 지역 변수가 없으므로, 이를 추가해야 한다. 또한 함수가 여러 인자를 받고 이를 접근할 수 있도록 일반화해야 한다.

컴파일러 작성의 다음 단계에서는 배열을 다루고 싶다. 이는 역참조, lvalue와 rvalue, 그리고 배열 요소의 크기에 따른 인덱스 스케일링을 결합한 작업이 될 것이다. 우리는 이미 필요한 의미론적 요소들을 갖추고 있지만, 토큰 추가, 파싱, 그리고 실제 인덱싱 기능을 구현해야 한다. 이번 주제만큼 흥미로운 주제가 될 것이다. [다음 단계](../19_Arrays_pt1/Readme.md)



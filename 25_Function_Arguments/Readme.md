# 25부: 함수 호출과 인자 처리

컴파일러 작성 여정의 이번 부분에서는 임의의 수의 인자를 가진 함수를 호출하는 기능을 추가한다. 함수의 인자 값은 해당 함수의 매개변수로 복사되어 지역 변수처럼 사용된다.

이 작업을 아직 하지 않은 이유는 코딩을 시작하기 전에 약간의 설계 고민이 필요했기 때문이다. 다시 한 번, Eli Bendersky의 [x86-64 스택 프레임 레이아웃](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/) 글에서 나온 이미지를 살펴보자.

![](../22_Design_Locals/Figs/x64_frame_nonleaf.png)

함수에 전달되는 "값에 의한 호출(call by value)" 인자 중 최대 6개는 `%rdi`부터 `%r9`까지의 레지스터를 통해 전달된다. 6개를 초과하는 인자는 스택에 푸시된다.

스택에 있는 인자 값을 자세히 살펴보자. `h`가 마지막 인자임에도 불구하고, 스택(아래로 성장하는)에 먼저 푸시되고, `g` 인자는 `h` 인자 *이후*에 푸시된다.

C 언어의 *나쁜 점* 중 하나는 표현식 평가 순서가 정의되어 있지 않다는 것이다. [여기](https://en.cppreference.com/w/c/language/eval_order)에서 언급한 바와 같이:

> [C 언어에서] 모든 연산자의 피연산자 평가 순서, 함수 호출 표현식에서의 함수 인자 평가 순서 등은 명시되어 있지 않다... 컴파일러는 어떤 순서로든 평가할 수 있다...

이는 언어를 잠재적으로 이식 불가능하게 만든다. 한 플랫폼에서 한 컴파일러로 컴파일된 코드의 동작이 다른 플랫폼이나 다른 컴파일러에서 다르게 동작할 수 있다.

그러나 우리에게는 이 정의되지 않은 평가 순서가 *좋은 점*이다. 단지 컴파일러를 작성하기 쉬운 순서로 인자 값을 생성할 수 있기 때문이다. 여기서 나는 경솔하게 말하고 있다. 이는 실제로 그다지 좋은 점이 아니다.

x86-64 플랫폼은 마지막 인자 값이 스택에 먼저 푸시될 것을 기대하기 때문에, 인자를 마지막부터 첫 번째까지 처리하는 코드를 작성해야 한다. 코드가 다른 방향으로도 쉽게 변경될 수 있도록 해야 한다. 아마도 `genXXX()` 쿼리 함수를 작성하여 인자 처리 방향을 알려줄 수 있을 것이다. 이 부분은 나중에 작성하도록 남겨둔다.


### 표현식 AST 생성하기

이미 A_GLUE AST 노드 타입을 가지고 있으므로, 인자 표현식을 파싱하고 AST 트리를 구축하는 함수를 작성하는 것은 어렵지 않다. `function(expr1, expr2, expr3, expr4)`와 같은 함수 호출의 경우, 다음과 같은 트리를 구성하기로 결정했다:

```
                 A_FUNCCALL
                  /
              A_GLUE
               /   \
           A_GLUE  expr4
            /   \
        A_GLUE  expr3
         /   \
     A_GLUE  expr2
     /    \
   NULL  expr1
```

각 표현식은 오른쪽에 위치하고, 이전 표현식들은 왼쪽에 위치한다. `expr4`를 `expr3`보다 먼저 처리해야 할 경우를 대비해, 표현식 서브 트리를 오른쪽에서 왼쪽으로 순회해야 한다. 이렇게 하면 전자가 후자보다 먼저 x86-64 스택에 푸시될 수 있다.

이미 단일 인자를 가진 간단한 함수 호출을 파싱하는 `funccall()` 함수를 가지고 있다. 이 함수를 수정하여 `expression_list()` 함수를 호출하도록 해서 표현식 리스트를 파싱하고 A_GLUE 서브 트리를 구축할 것이다. 이 함수는 표현식의 개수를 최상위 A_GLUE AST 노드에 저장하여 반환한다. 그런 다음, `funccall()` 함수에서 모든 표현식의 타입을 전역 심볼 테이블에 저장된 함수 프로토타입과 비교할 수 있다.

이 정도로 설계 측면에서 충분히 논의한 것 같다. 이제 구현으로 넘어가자.


## 표현식 파싱 변경 사항

코드를 약 한 시간 만에 완성했고, 결과가 만족스럽다. 트위터에서 흔히 볼 수 있는 한 문장을 빌려오자면:

> 몇 주 동안 프로그래밍하는 것은 몇 시간 동안 계획하는 시간을 절약할 수 있다.

반대로, 설계에 시간을 조금 투자하면 코딩 효율성이 항상 향상된다. 이제 변경 사항을 살펴보자. 먼저 파싱부터 시작한다.

이제 쉼표로 구분된 표현식 목록을 파싱하고, 오른쪽에 자식 표현식을, 왼쪽에 이전 표현식 트리를 가진 A_GLUE AST 트리를 구축해야 한다. 다음은 `expr.c`의 코드다:

```c
// expression_list: <null>
//        | expression
//        | expression ',' expression_list
//        ;

// 쉼표로 구분된 0개 이상의 표현식 목록을 파싱하고,
// 왼쪽 자식으로 이전 표현식의 서브 트리(또는 NULL)를,
// 오른쪽 자식으로 다음 표현식을 가진 A_GLUE 노드로 구성된 AST를 반환한다.
// 각 A_GLUE 노드는 이 시점에서 트리에 있는 표현식의 수를 size 필드에 저장한다.
// 표현식이 파싱되지 않았다면 NULL을 반환한다.
static struct ASTnode *expression_list(void) {
  struct ASTnode *tree = NULL;
  struct ASTnode *child = NULL;
  int exprcount = 0;

  // 마지막 오른쪽 괄호까지 반복
  while (Token.token != T_RPAREN) {

    // 다음 표현식을 파싱하고 표현식 카운트를 증가
    child = binexpr(0);
    exprcount++;

    // 이전 트리를 왼쪽 자식으로, 새로운 표현식을 오른쪽 자식으로 하는
    // A_GLUE AST 노드를 구축한다. 표현식 카운트를 저장한다.
    tree = mkastnode(A_GLUE, P_NONE, tree, NULL, child, exprcount);

    // 이 시점에서 ',' 또는 ')'가 있어야 한다.
    switch (Token.token) {
      case T_COMMA:
        scan(&Token);
        break;
      case T_RPAREN:
        break;
      default:
        fatald("Unexpected token in expression list", Token.token);
    }
  }

  // 표현식 트리를 반환
  return (tree);
}
```

코딩이 예상보다 훨씬 쉬웠다. 이제 이를 기존 함수 호출 파서와 연결해야 한다:

```c
// 함수 호출을 파싱하고 그 AST를 반환
static struct ASTnode *funccall(void) {
  struct ASTnode *tree;
  int id;

  // 식별자가 함수로 정의되었는지 확인한 후, 리프 노드를 만든다.
  if ((id = findsymbol(Text)) == -1 || Symtable[id].stype != S_FUNCTION) {
    fatals("Undeclared function", Text);
  }
  // '('를 가져온다.
  lparen();

  // 인수 표현식 목록을 파싱
  tree = expression_list();

  // XXX 함수의 프로토타입과 각 인수의 타입을 비교하는 작업을 수행해야 함

  // 함수 호출 AST 노드를 구축한다. 함수의 반환 타입을 이 노드의 타입으로 저장한다.
  // 또한 함수의 심볼 ID를 기록한다.
  tree = mkastunary(A_FUNCCALL, Symtable[id].type, tree, id);

  // ')'를 가져온다.
  rparen();
  return (tree);
}
```

`XXX`는 아직 작업이 남아 있다는 나의 알림이다. 파서는 함수가 이전에 선언되었는지 확인하지만, 아직 인수 타입을 함수의 프로토타입과 비교하지는 않는다. 이 작업은 곧 수행할 예정이다.

이제 반환된 AST 트리는 이 글의 시작 부분에서 그린 형태와 같다. 이제 이를 순회하며 어셈블리 코드를 생성할 차례다.


## 일반 코드 생성기 변경 사항

컴파일러는 AST(Abstract Syntax Tree)를 순회하는 코드가 아키텍처에 중립적으로 작성되어 있다. 이 코드는 `gen.c`에 위치하며, 플랫폼에 종속적인 백엔드 코드는 `cg.c`에 있다. 따라서 `gen.c`의 변경 사항부터 살펴본다.

새로운 AST 구조를 순회하기 위해 상당한 양의 코드가 필요하다. 이제 함수 호출을 처리하기 위한 함수를 추가했다. `genAST()` 함수 내부에는 다음과 같은 코드가 있다:

```c
  // n은 처리 중인 AST 노드
  switch (n->op) {
    ...
    case A_FUNCCALL:
      return (gen_funccall(n));
  }
```

새로운 AST 구조를 순회하는 코드는 다음과 같다:

```c
// 함수 호출의 인자를 매개변수에 복사하고,
// 함수 자체를 호출하는 코드를 생성한다.
// 함수의 반환값을 담은 레지스터를 반환한다.
static int gen_funccall(struct ASTnode *n) {
  struct ASTnode *gluetree = n->left;
  int reg;
  int numargs=0;

  // 인자 리스트가 있다면, 마지막 인자(오른쪽 자식)부터
  // 첫 번째 인자까지 순회한다
  while (gluetree) {
    // 표현식의 값을 계산한다
    reg = genAST(gluetree->right, NOLABEL, gluetree->op);
    // 이 값을 n번째 함수 매개변수에 복사한다: 크기는 1, 2, 3, ...
    cgcopyarg(reg, gluetree->v.size);
    // 첫 번째(가장 높은) 인자 수를 유지한다
    if (numargs==0) numargs= gluetree->v.size;
    genfreeregs();
    gluetree = gluetree->left;
  }

  // 함수를 호출하고, 스택을 정리하고(인자 수 기반),
  // 결과를 반환한다
  return (cgcall(n->v.id, numargs));
}
```

여기서 주목할 점이 몇 가지 있다. 오른쪽 자식에 대해 `genAST()`를 호출해 표현식 코드를 생성한다. 또한 `numargs`를 첫 번째 `size` 값으로 설정하는데, 이 값은 인자의 수를 나타낸다(1부터 시작, 0부터 아님). 그런 다음 `cgcopyarg()`를 호출해 이 값을 함수의 n번째 매개변수에 복사한다. 복사가 완료되면 다음 표현식을 위해 모든 레지스터를 해제하고, 이전 표현식을 위해 왼쪽 자식으로 이동한다.

마지막으로 `cgcall()`을 실행해 함수를 실제로 호출한다. 스택에 인자 값을 푸시했을 수 있으므로, 총 인자 수를 제공해 스택에서 몇 개를 팝해야 하는지 계산할 수 있게 한다.

여기에는 하드웨어에 특화된 코드는 없지만, 앞서 언급했듯이 표현식 트리를 마지막 표현식부터 첫 번째 표현식까지 순회한다. 모든 아키텍처가 이를 원하는 것은 아니므로, 평가 순서 측면에서 코드를 더 유연하게 만들 여지가 있다.


## `cg.c`의 변경 사항

이제 실제 x86-64 어셈블리 코드를 생성하는 함수들을 살펴보자. 새로운 함수인 `cgcopyarg()`를 추가하고, 기존 함수인 `cgcall()`을 수정했다.

먼저, 사용할 레지스터 목록을 다시 확인해 보자:

```c
#define FIRSTPARAMREG 9         // 첫 번째 파라미터 레지스터 위치
static char *reglist[] =
  { "%r10", "%r11", "%r12", "%r13", "%r9", "%r8", "%rcx", "%rdx", "%rsi", "%rdi" };

static char *breglist[] =
  { "%r10b", "%r11b", "%r12b", "%r13b", "%r9b", "%r8b", "%cl", "%dl", "%sil", "%dil" };

static char *dreglist[] =
  { "%r10d", "%r11d", "%r12d", "%r13d", "%r9d", "%r8d", "%ecx", "%edx", "%esi", "%edi" };
```

`FIRSTPARAMREG`는 목록의 마지막 인덱스 위치로 설정되어 있다. 이 목록을 거꾸로 따라가며 작업할 것이다.

또한, 전달받는 인자 위치 번호는 1부터 시작한다(1, 2, 3, 4, ...). 하지만 배열은 0부터 시작하므로, 아래 코드에서 `+1` 또는 `-1` 조정이 몇 번 나타난다.

먼저 `cgcopyarg()` 함수를 보자:

```c
// 인자 값을 가진 레지스터가 주어지면,
// 이 인자를 argposn번째 파라미터로 복사하여
// 이후 함수 호출을 준비한다. argposn은 1, 2, 3, 4, ...이며 0이 아니다.
void cgcopyarg(int r, int argposn) {

  // 여섯 번째 인자를 초과하면, 레지스터 값을 스택에 푸시한다.
  // x86-64에서 연속된 인자를 올바른 순서로 호출할 수 있도록 의존한다.
  if (argposn > 6) {
    fprintf(Outfile, "\tpushq\t%s\n", reglist[r]);
  } else {
    // 그렇지 않으면, 파라미터 값을 저장하는 여섯 개의 레지스터 중 하나로 값을 복사한다.
    fprintf(Outfile, "\tmovq\t%s, %s\n", reglist[r],
            reglist[FIRSTPARAMREG - argposn + 1]);
  }
}
```

`+1` 조정을 제외하면 간단하다. 이제 `cgcall()` 함수를 보자:

```c
// 주어진 심볼 ID로 함수를 호출한다.
// 스택에 푸시된 인자들을 제거한다.
// 결과 값을 가진 레지스터를 반환한다.
int cgcall(int id, int numargs) {
  // 새로운 레지스터를 할당한다.
  int outr = alloc_register();
  // 함수를 호출한다.
  fprintf(Outfile, "\tcall\t%s\n", Symtable[id].name);
  // 스택에 푸시된 인자들을 제거한다.
  if (numargs>6) 
    fprintf(Outfile, "\taddq\t$%d, %%rsp\n", 8*(numargs-6));
  // 반환 값을 레지스터로 복사한다.
  fprintf(Outfile, "\tmovq\t%%rax, %s\n", reglist[outr]);
  return (outr);
}
```

이 함수도 마찬가지로 간단하다.


## 변경 사항 테스트

컴파일러 개발 과정의 마지막 단계에서는 두 개의 테스트 프로그램 `input27a.c`와 `input27b.c`를 사용했다. 이전에는 이 중 하나를 `gcc`로 컴파일해야 했다. 이제는 이 두 프로그램을 하나로 합쳐 우리가 만든 컴파일러로 모두 컴파일할 수 있다. 또한, 함수 호출을 더 다양한 예제로 테스트할 수 있는 두 번째 테스트 프로그램 `input28.c`가 있다. 이전과 마찬가지로:

```
$ make test
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c
    misc.c scan.c stmt.c sym.c tree.c types.c
(cd tests; chmod +x runtests; ./runtests)
  ...
input25.c: OK
input26.c: OK
input27.c: OK
input28.c: OK
```


## 결론과 다음 단계

현재 우리의 컴파일러는 단순한 '장난감' 수준에서 벗어나 거의 실용적인 수준에 도달했다. 이제는 여러 함수를 작성하고 함수 간 호출을 할 수 있게 되었다. 여기까지 오는 데 몇 단계가 필요했지만, 각 단계가 너무 거대하지는 않았다.

물론 아직 갈 길이 멀다. 구조체, 공용체, 외부 식별자, 그리고 전처리기를 추가해야 한다. 그런 다음 컴파일러를 더 견고하게 만들고, 더 나은 오류 감지 기능을 제공하며, 경고 메시지도 추가할 필요가 있다. 그래서 아마도 지금쯤이면 중간 지점에 온 것 같다.

컴파일러 작성의 다음 단계로, 함수 프로토타입을 작성하는 기능을 추가할 계획이다. 이를 통해 외부 함수와 연결할 수 있게 된다. `open()`, `read()`, `write()`, `strcpy()` 등과 같은 원래의 Unix 함수와 시스템 호출을 생각하고 있다. 이제 우리의 컴파일러로 유용한 프로그램들을 컴파일할 수 있게 될 것이다. [다음 단계](../26_Prototypes/Readme.md)



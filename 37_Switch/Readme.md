# 37장: Switch 문 구현

컴파일러 제작 과정의 이번 단계에서는 'switch' 문을 구현한다. 여러 가지 이유로 이 작업은 상당히 까다롭다. 이에 대해 자세히 살펴보기 전에, 먼저 예제를 통해 어떤 의미가 있는지 알아보자.


## switch 문 예제

```c
  switch(x) {
    case 1:  printf("One\n");  break;
    case 2:  printf("Two\n");  break;
    case 3:  printf("Three\n");
    default: printf("More than two\n");
  }
```

이 코드는 `x`의 값에 따라 실행할 분기를 선택하는 다중 선택 구조이다. `if` 문과 유사하지만, `break` 문을 사용해 다른 분기로 넘어가지 않도록 해야 한다. `break` 문을 생략하면 현재 분기가 끝나고 다음 분기가 계속 실행된다.

`switch` 문의 조건식은 정수 타입이어야 하며, 모든 `case` 옵션도 정수 리터럴이어야 한다. 예를 들어 `case 3*y+17`과 같은 표현은 사용할 수 없다.

`default` 케이스는 이전 `case`에서 다루지 않은 모든 값을 처리한다. `default`는 항상 마지막 케이스로 위치해야 한다. 또한 중복된 `case` 값을 사용할 수 없으므로 `case 2: ...; case 2`와 같은 코드는 허용되지 않는다.


## 위 내용을 어셈블리로 변환하기

'switch' 문을 어셈블리로 변환하는 한 가지 방법은 이를 다중 'if' 문으로 처리하는 것이다. 이 방법은 `x`를 정수 값과 하나씩 비교하며, 필요한 어셈블리 코드 섹션으로 이동하거나 건너뛰는 방식이다. 이 방법은 동작하지만, 특히 다음과 같은 예제를 고려할 때 어셈블리 코드가 비효율적이 된다:

```c
  switch (2 * x - (18 +y)/z) { ... }
```

현재 우리의 "[KISS](https://en.wikipedia.org/wiki/KISS_principle)" 컴파일러의 동작 방식을 고려하면, 리터럴 값과 비교할 때마다 이 표현식을 반복적으로 평가해야 한다.

'switch' 표현식을 한 번만 평가하는 것이 더 합리적이다. 그런 다음, 이 값을 케이스 리터럴 값 테이블과 비교한다. 일치하는 값을 찾으면, 해당 케이스 값과 연결된 코드 분기로 점프한다. 이를 [점프 테이블](https://en.wikipedia.org/wiki/Branch_table)이라고 한다.

이 방법은 각 케이스 옵션마다 해당 코드의 시작 부분에 레이블을 생성해야 한다는 것을 의미한다. 예를 들어, 위 첫 번째 예제의 점프 테이블은 다음과 같을 수 있다:

| 케이스 값 | 레이블 |
|:----------:|:-----:|
|     1      |  L18  |
|     2      |  L19  |
|     3      |  L22  |
|  기본값   |  L26  |

또한 'switch' 문 이후의 코드를 표시할 레이블도 필요하다. 코드 분기에서 `break;`를 실행하려면 이 'switch' 종료 레이블로 점프한다. 그렇지 않으면 코드 분기가 다음 코드 분기로 넘어가게 된다.


## 파싱 시 고려사항

위에서 설명한 내용은 모두 좋지만, 'switch' 문을 위에서 아래로 파싱해야 한다는 점을 고려해야 한다. 이는 모든 case를 파싱한 후에야 점프 테이블의 크기를 알 수 있다는 것을 의미한다. 또한, 특별한 트릭을 사용하지 않는 한 모든 case에 대한 어셈블리 코드를 생성한 후에야 점프 테이블을 생성할 수 있다.

여러분도 알다시피, 나는 "KISS 원칙(Keep It Simple, Stupid)"을 따르며 이 컴파일러를 작성하고 있다. 따라서 복잡한 트릭을 피하고 있는데, 이는 점프 테이블의 출력을 모든 case에 대한 어셈블리 코드를 생성한 후로 미루겠다는 것을 의미한다.

시각적으로, 우리가 코드를 배치할 방식은 다음과 같다:

![](Figs/switch_logic.png)

스위치 결정을 평가하는 코드는 가장 위에 위치한다. 우리는 이를 먼저 파싱한다. 첫 번째 case로 넘어가지 않기 위해, 나중에 출력할 레이블로 점프할 수 있다.

그런 다음 각 case 문을 파싱하고 해당 어셈블리 코드를 생성한다. 이미 "switch 종료" 레이블을 생성했으므로, 이를 통해 점프할 수 있다. 이 레이블 역시 나중에 출력할 것이다.

각 case를 생성할 때마다 해당 레이블을 얻고 이를 출력한다. 모든 case와 기본 case(존재한다면)를 출력한 후, 이제 점프 테이블을 생성할 수 있다.

하지만 이제 점프 테이블을 순회하며 스위치 결정을 각 case 값과 비교하고 적절히 점프할 코드가 필요하다. 이를 모든 'switch' 문에 대해 어셈블리 코드로 생성할 수도 있지만, 이 점프 처리 코드가 크다면 메모리를 낭비하게 될 것이다. 메모리에 점프 처리 코드를 한 번만 두는 것이 더 나은데, 이제는 이 코드로 점프해야 한다! 더 나쁜 점은, 이 코드가 스위치 결정 결과를 담고 있는 레지스터를 알지 못하므로, 이 레지스터를 알려진 레지스터로 복사하고 점프 테이블의 베이스를 알려진 레지스터로 복사해야 한다는 것이다.

여기서 우리는 파싱과 코드 생성의 복잡성을 어셈블리 코드의 스파게티와 교환했다. CPU는 이 점프 스파게티를 처리할 수 있으므로, 현재로서는 공정한 절충이다. 당연히, 실제 제품용 컴파일러는 이와 다르게 동작할 것이다.

다이어그램의 빨간 선은 스위치 결정에서 레지스터 로딩, 점프 테이블 처리, 그리고 특정 case 코드로의 실행 흐름을 보여준다. 초록색 선은 점프 테이블의 베이스 주소가 점프 테이블 처리 코드로 전달되는 것을 보여준다. 마지막으로, 파란색 선은 case가 `break;`로 끝나 스위치 어셈블리 코드의 끝으로 점프하는 것을 보여준다.

따라서, 어셈블리 출력은 보기 흉하지만 동작은 한다. 이제 'switch' 문을 어떻게 구현할지 살펴봤으니, 실제로 이를 구현해보자.


## 새로운 키워드와 토큰

새로운 `case`와 `default` 키워드와 함께 사용할 두 개의 새로운 토큰, T_CASE와 T_DEFAULT를 추가했다. 이와 관련된 구현 내용은 항상 그렇듯 코드를 직접 살펴보면 확인할 수 있다.


## 새로운 AST 노드 타입

'switch' 문을 표현하기 위해 AST 트리를 구축해야 한다. 'switch' 문의 구조는 우리가 다뤄왔던 표현식과 같은 이진 트리 형태가 아니다. 하지만 이는 우리만의 AST 트리이므로, 필요에 맞게 구조를 설계할 수 있다. 고민 끝에 다음과 같은 구조를 선택했다:

![](Figs/switch_ast.png)

'switch' 트리의 루트는 A_SWITCH 노드다. 왼쪽에는 switch 조건을 계산하는 표현식 서브트리가 위치한다. 오른쪽에는 각 케이스를 나타내는 A_CASE 노드들이 연결 리스트 형태로 배치된다. 마지막으로, 기본 케이스를 나타내는 A_DEFAULT 노드가 선택적으로 추가될 수 있다.

각 A_CASE 노드의 `intvalue` 필드는 표현식이 매칭해야 하는 케이스 값을 저장한다. 왼쪽 자식 서브트리는 해당 케이스의 본문을 구성하는 복합문의 세부 정보를 담는다. 현재 단계에서는 점프 레이블이나 점프 테이블은 존재하지 않으며, 이는 나중에 생성할 예정이다.


## switch 문 파싱하기

지금까지 배운 내용을 바탕으로 이제 `switch` 문을 파싱하는 방법을 살펴볼 준비가 되었다. 여기에는 상당히 많은 오류 검사 코드가 포함되어 있으므로, 작은 섹션으로 나누어 설명할 것이다. 이 코드는 `stmt.c` 파일에 있으며, `single_statement()` 함수에서 호출된다.

```c
    case T_SWITCH:
      return (switch_statement());
```

시작해 보자.

```c
// switch 문을 파싱하고 AST를 반환한다.
static struct ASTnode *switch_statement(void) {
  struct ASTnode *left, *n, *c, *casetree= NULL, *casetail;
  int inloop=1, casecount=0;
  int seendefault=0;
  int ASTop, casevalue;

  // 'switch'와 '('를 건너뛴다.
  scan(&Token);
  lparen();

  // switch 표현식, ')' 및 '{'를 가져온다.
  left= binexpr(0);
  rparen();
  lbrace();

  // 이 표현식이 정수 타입인지 확인한다.
  if (!inttype(left->type))
    fatal("Switch expression is not of integer type");
```

맨 위에 많은 지역 변수가 선언되어 있는데, 이는 이 함수에서 상태를 처리해야 함을 암시한다. 첫 번째 섹션은 간단하다: `switch (expression) {` 구문을 파싱하고, 표현식의 AST를 가져온 후, 이 표현식이 정수 타입인지 확인한다.

```c
  // 표현식을 자식으로 하는 A_SWITCH 서브트리를 만든다.
  n= mkastunary(A_SWITCH, 0, left, NULL, 0);

  // 이제 case를 파싱한다.
  Switchlevel++;
```

이제 switch 결정 트리를 얻었으므로, 반환할 A_SWITCH 노드를 만들 수 있다. `break;`는 최소한 하나의 루프 안에서만 발생할 수 있다는 것을 기억할 것이다. 이제는 `break;`가 최소한 하나의 `switch` 문 안에서도 발생할 수 있도록 해야 한다. 이를 위해 새로운 전역 변수 `Switchlevel`을 사용한다.

```c
  // 이제 case를 파싱한다.
  Switchlevel++;
  while (inloop) {
    switch(Token.token) {
      // '}'를 만나면 루프를 빠져나간다.
      case T_RBRACE: if (casecount==0)
                        fatal("No cases in switch");
                     inloop=0; break;
  ...
  }
```

루프는 `inloop` 변수에 의해 제어되며, 이 변수는 처음에 1로 설정된다. '}' 토큰을 만나면 이 변수를 0으로 설정하고 `switch` 문을 빠져나가 루프를 종료한다. 또한 최소한 하나의 case가 있는지 확인한다.

> `switch` 문을 파싱하기 위해 `switch` 문을 사용하는 것이 약간 이상하게 느껴질 수 있다.

이제 `case`와 `default` 파싱으로 넘어간다.

```c
      case T_CASE:
      case T_DEFAULT:
        // 이전에 'default'가 나온 후인지 확인한다.
        if (seendefault)
          fatal("case or default after existing default");
```

여기서는 많은 공통 코드를 실행해야 하므로, 두 토큰이 동일한 코드로 들어간다. 먼저, 이미 `default` case가 나왔는지 확인한다. `default` case는 반드시 마지막 case여야 한다.

```c
        // AST 연산을 설정한다. 필요한 경우 case 값을 스캔한다.
        if (Token.token==T_DEFAULT) {
          ASTop= A_DEFAULT; seendefault= 1; scan(&Token);
        } else ...
```

`default:`를 파싱하는 경우, 뒤에 오는 정수 값이 없다. 키워드를 건너뛰고 `default` case가 나왔음을 기록한다.

```c
        } else  {
          ASTop= A_CASE; scan(&Token);
          left= binexpr(0);
          // case 값이 정수 리터럴인지 확인한다.
          if (left->op != A_INTLIT)
            fatal("Expecting integer literal for case value");
          casevalue= left->intvalue;

          // 기존 case 값 목록을 순회하며 중복된 case 값이 없는지 확인한다.
          for (c= casetree; c != NULL; c= c -> right)
            if (casevalue == c->intvalue)
              fatal("Duplicate case value");
        }
```

이 코드는 `case <value>:`를 특별히 처리한다. `binexpr()`를 사용해 `case` 뒤의 값을 읽어온다. 여기서 `primary()`를 호출해 정수 리터럴을 바로 파싱할 수도 있었지만, `primary()`는 어차피 `binexpr()`를 호출할 수 있으므로 큰 차이는 없다. 결과 트리가 A_INTLIT 노드인지 확인하기 위해 오류 검사를 해야 한다.

그런 다음 이전에 만든 A_CASE 노드 목록(`casetree`가 이 목록의 헤드를 가리킴)을 순회하며 중복된 case 값이 없는지 확인한다.

이 과정에서 `ASTop` 변수를 정수 리터럴 값을 가진 case에 대해 A_CASE로, `default` case에 대해 A_DEFAULT로 설정한다. 이제 두 경우에 공통적으로 수행할 코드를 실행할 수 있다.

```c
        // ':'를 스캔하고 복합 표현식을 가져온다.
        match(T_COLON, ":");
        left= compound_statement(); casecount++;

        // 복합 문을 왼쪽 자식으로 하는 서브트리를 만들고, 
        // 성장 중인 A_CASE 트리에 연결한다.
        if (casetree==NULL) {
          casetree= casetail= mkastunary(ASTop, 0, left, NULL, casevalue);
        } else {
          casetail->right= mkastunary(ASTop, 0, left, NULL, casevalue);
          casetail= casetail->right;
        }
        break;
```

다음 토큰이 ':'인지 확인한다. 복합 문을 포함한 AST 서브트리를 가져온다. 이 서브트리를 왼쪽 자식으로 하는 A_CASE 또는 A_DEFAULT 노드를 만들고, 이 노드를 A_CASE/A_DEFAULT 노드의 연결 리스트에 연결한다: `casetree`는 이 목록의 헤드이고 `casetail`은 꼬리이다.

```c
      default:
        fatald("Unexpected token in switch", Token.token);
    }
  }
```

`switch` 본문에는 `case`와 `default` 키워드만 있어야 하므로, 이를 확인한다.

```c
  Switchlevel--;

  // case와 default를 포함한 서브트리가 있다. case 수를 A_SWITCH 노드에 넣고 case 트리를 연결한다.
  n->intvalue= casecount;
  n->right= casetree;
  rbrace();

  return(n);
```

이제 모든 case와 `default` case를 파싱했으며, 이들의 수와 `casetree`가 가리키는 목록을 얻었다. 이 값을 A_SWITCH 노드에 추가하고 최종 트리로 반환한다.

이렇게 해서 상당한 양의 파싱을 마쳤다. 이제 코드 생성에 주목할 차례이다.


## 스위치 코드 생성: 예제

이제 'switch' 문의 어셈블리 출력을 살펴보면서 코드가 어떻게 실행 흐름을 따르는지 확인해 보자. 다음은 예제 코드이다:

```c
#include <stdio.h>

int x; int y;

int main() {
  switch(x) {
    case 1:  { y= 5; break; }
    case 2:  { y= 7; break; }
    case 3:  { y= 9; }
    default: { y= 100; }
  }
  return(0);
}
```

먼저, 각 case 문의 본문을 '{' ... '}'로 감싸야 한다. 이는 "dangling else" 문제를 아직 해결하지 못했기 때문에 모든 복합 문을 '{' ... '}'로 감싸야 하기 때문이다.

점프 테이블 처리 코드는 잠시 생략하고, 이 예제의 어셈블리 출력을 살펴보자:

![](Figs/switch_logic2.png)

`x`를 레지스터에 로드하는 코드는 맨 위에 있으며, 점프 테이블을 지나 아래로 이동한다. 점프 테이블 처리 코드는 어떤 레지스터를 사용할지 모르기 때문에 항상 `%rax`에 값을 로드하고, 점프 테이블의 기본 주소를 `%rdx`에 로드한다.

점프 테이블의 구조는 다음과 같다:

 + 첫 번째는 정수 값을 가진 case의 개수이다.
 + 다음은 각 case에 대한 값/레이블 쌍이다.
 + 마지막은 기본 case의 레이블이다. 기본 case가 없으면 'switch' 문의 끝을 가리키는 레이블이어야 하며, 이는 일치하는 case가 없을 때 아무 코드도 실행하지 않도록 한다.

점프 테이블 처리 코드(곧 살펴볼 예정)는 점프 테이블을 해석하고 이 테이블의 레이블 중 하나로 점프한다. `case 2:`에 해당하는 `L11`로 점프했다고 가정해 보자. 이 case 옵션의 코드를 실행한다. 이 옵션에는 `break;` 문이 있으므로 'switch' 문의 끝을 가리키는 `L9`로 점프한다.


## 점프 테이블 처리 코드

x86-64 어셈블리 코드는 내 전문 분야가 아니다. 그래서 [SubC](http://www.t3x.org/subc/)에서 점프 테이블 처리 코드를 그대로 가져왔다. 이 코드를 `cg.c` 파일의 `cgpreamble()` 함수에 추가해서, 생성하는 모든 어셈블리 파일에 이 코드가 포함되도록 했다. 아래는 주석이 달린 코드다:

```
# switch(expr) 내부 루틴
# %rsi = 스위치 테이블, %rax = expr

switch:
        pushq   %rsi            # %rsi 저장
        movq    %rdx,%rsi       # 점프 테이블 베이스 -> %rsi
        movq    %rax,%rbx       # 스위치 값 -> %rbx
        cld                     # 방향 플래그 초기화
        lodsq                   # 케이스 개수를 %rcx에 로드,
        movq    %rax,%rcx       # 로드 과정에서 %rsi 증가
next:
        lodsq                   # 케이스 값을 %rdx에 로드
        movq    %rax,%rdx
        lodsq                   # 라벨 주소를 %rax에 로드
        cmpq    %rdx,%rbx       # 스위치 값과 케이스 값이 일치하는가?
        jnz     no              # 아니면, 이 코드를 건너뜀
        popq    %rsi            # %rsi 복원
        jmp     *%rax           # 선택된 케이스로 점프
no:
        loop    next            # 케이스 개수만큼 루프
        lodsq                   # 루프 종료 후, 기본 라벨 주소 로드
        popq    %rsi            # %rsi 복원
        jmp     *%rax           # 기본 케이스로 점프
```

이 코드를 작성한 Nils Holm에게 감사해야 한다. 내가 이 코드를 직접 작성할 수 있었을 리 없다!

이제 위 어셈블리 코드가 어떻게 생성되는지 살펴보자. 다행히 `cg.c` 파일에는 재사용할 수 있는 유용한 함수들이 이미 많이 있다.


## 어셈블리 코드 생성하기

`gen.c` 파일의 `genAST()` 함수 상단에서 A_SWITCH 노드를 식별하고, 이 노드와 그 아래의 트리를 처리하기 위해 함수를 호출한다.

```c
    case A_SWITCH:
      return (genSWITCH(n));
```

이제 이 새로운 함수를 단계별로 살펴보자.

```c
// SWITCH 문을 위한 코드 생성
static int genSWITCH(struct ASTnode *n) {
  int *caseval, *caselabel;
  int Ljumptop, Lend;
  int i, reg, defaultlabel = 0, casecount = 0;
  struct ASTnode *c;

  // case 값과 관련된 레이블을 위한 배열 생성.
  // 각 배열에 최소 하나의 위치가 있도록 보장한다.
  caseval = (int *) malloc((n->intvalue + 1) * sizeof(int));
  caselabel = (int *) malloc((n->intvalue + 1) * sizeof(int));
```

여기서 `+1`을 사용하는 이유는 기본 case가 있더라도 case 값이 없더라도 레이블이 필요하기 때문이다.

```c
  // 점프 테이블 상단과 switch 문의 끝을 위한 레이블 생성.
  // 기본적으로 switch의 끝을 위한 레이블을 설정한다.
  Ljumptop = genlabel();
  Lend = genlabel();
  defaultlabel = Lend;
```

이 레이블들은 생성되었지만 아직 어셈블리 코드로 출력되지는 않았다. 기본 레이블이 없을 경우 `Lend`로 설정한다.

```c
  // switch 조건을 계산하기 위한 코드 출력
  reg = genAST(n->left, NOLABEL, NOLABEL, NOLABEL, 0);
  cgjump(Ljumptop);
  genfreeregs();
```

점프 테이블 이후의 코드로 점프하기 위한 코드를 출력한다. 아직 출력되지 않았지만 이 시점에서 모든 레지스터를 해제할 수 있다.

```c
  // 오른쪽 자식 연결 리스트를 순회하며 각 case에 대한 코드 생성
  for (i = 0, c = n->right; c != NULL; i++, c = c->right) {

    // 이 case를 위한 레이블을 가져온다. 배열에 레이블과 case 값을 저장한다.
    // 기본 case인지 기록한다.
    caselabel[i] = genlabel();
    caseval[i] = c->intvalue;
    cglabel(caselabel[i]);
    if (c->op == A_DEFAULT)
      defaultlabel = caselabel[i];
    else
      casecount++;

    // case 코드 생성. break를 위한 끝 레이블을 전달한다.
    genAST(c->left, NOLABEL, NOLABEL, Lend, 0);
    genfreeregs();
  }
```

이 코드는 각 case에 대한 레이블을 생성하고, case 본문에 해당하는 어셈블리 코드를 출력한다. 두 배열에 case 값과 case 레이블을 저장한다. 그리고 이 case가 기본 case라면 `defaultlabel`을 올바른 레이블로 업데이트한다.

또한 `genAST()`에 `Lend`를 전달하는데, 이는 switch 코드 이후의 레이블이다. 이를 통해 case 본문의 `break;`가 다음으로 넘어갈 수 있게 한다.

```c
  // 마지막 case가 switch 테이블을 지나가도록 보장
  cgjump(Lend);

  // 이제 switch 테이블과 끝 레이블을 출력한다.
  cgswitch(reg, casecount, Ljumptop, caselabel, caseval, defaultlabel);
  cglabel(Lend);
  return (NOREG);
}
```

프로그래머가 마지막 case를 `break;`로 끝낼 것이라고 가정할 수 없으므로, 마지막 case가 switch 문의 끝으로 점프하도록 강제한다.

이 시점에서 우리는 다음을 가지고 있다:

 + switch 값을 가진 레지스터
 + case 값 배열
 + case 레이블 배열
 + case 개수
 + 유용한 레이블들

이 모든 것을 `cg.c`의 `cgswitch()`에 전달하며, (SubC의 코드를 제외하고) 이 부분에 필요한 새로운 어셈블리 코드는 이것뿐이다.


## `cgswitch()`

여기서는 점프 테이블을 구성하고 레지스터를 로드하여 `switch` 어셈블리 코드로 점프할 수 있도록 해야 한다. 점프 테이블 구조를 다시 한번 상기해 보자:

 + 첫 번째는 정수 값을 가진 케이스의 수다.
 + 다음은 각 케이스에 대한 값/레이블 쌍이다.
 + 마지막은 기본 케이스의 레이블이다. 기본 케이스가 없으면 'switch'의 끝을 가리키는 레이블이어야 한다. 이렇게 하면 일치하는 케이스가 없을 때 코드가 실행되지 않는다.

예제에서 점프 테이블은 다음과 같다:

```
L14:                                    # Switch jump table
        .quad   3                       # Three case values
        .quad   1, L10                  # case 1: jump to L10
        .quad   2, L11                  # case 2: jump to L11
        .quad   3, L12                  # case 3: jump to L12
        .quad   L13                     # default: jump to L13
```

이제 이 모든 것을 생성하는 방법을 살펴보자.

```c
// switch 점프 테이블과 레지스터를 로드하고 switch() 코드를 호출하는 코드를 생성한다.
void cgswitch(int reg, int casecount, int toplabel,
              int *caselabel, int *caseval, int defaultlabel) {
  int i, label;

  // switch 테이블을 위한 레이블을 가져온다.
  label = genlabel();
  cglabel(label);
```

이 부분이 위의 `L14:`에 해당한다.

```c
  // 휴리스틱. 케이스가 없으면 기본 케이스를 가리키는 하나의 케이스를 생성한다.
  if (casecount == 0) {
    caseval[0] = 0;
    caselabel[0] = defaultlabel;
    casecount = 1;
  }
```

점프 테이블에는 최소한 하나의 값/레이블 쌍이 있어야 한다. 이 코드는 기본 케이스를 가리키는 하나의 케이스를 만든다. 케이스 값은 중요하지 않다. 일치하면 좋고, 그렇지 않으면 어차피 기본 케이스로 점프한다.

```c
  // switch 점프 테이블을 생성한다.
  fprintf(Outfile, "\t.quad\t%d\n", casecount);
  for (i = 0; i < casecount; i++)
    fprintf(Outfile, "\t.quad\t%d, L%d\n", caseval[i], caselabel[i]);
  fprintf(Outfile, "\t.quad\tL%d\n", defaultlabel);
```

이 코드는 점프 테이블을 생성한다. 간단하고 명확하다.

```c
  // 특정 레지스터를 로드한다.
  cglabel(toplabel);
  fprintf(Outfile, "\tmovq\t%s, %%rax\n", reglist[reg]);
  fprintf(Outfile, "\tleaq\tL%d(%%rip), %%rdx\n", label);
  fprintf(Outfile, "\tjmp\tswitch\n");
}
```

마지막으로, `%rax` 레지스터에 switch 값을 로드하고, `%rdx`에 점프 테이블의 레이블을 로드한 후 `switch` 코드를 호출한다.


## 코드 테스트

'switch' 문의 모든 케이스를 테스트할 수 있도록 예제에 루프를 추가했다. 다음은 `tests/input74.c` 파일의 내용이다:

```c
#include <stdio.h>

int main() {
  int x;
  int y;
  y= 0;

  for (x=0; x < 5; x++) {
    switch(x) {
      case 1:  { y= 5; break; }
      case 2:  { y= 7; break; }
      case 3:  { y= 9; }
      default: { y= 100; }
    }
    printf("%d\n", y);
  }
  return(0);
}
```

이 프로그램의 출력 결과는 다음과 같다:

```
100
5
7
100
100
```

여기서 값 9가 출력되지 않는 이유는 case 3에서 default 케이스로 넘어가기 때문이다.


## 결론 및 다음 단계

우리는 방금 컴파일러에 'switch' 문이라는 첫 번째 큰 문법을 구현했다. 이전에 해본 적이 없어서 기본적으로 SubC 구현 방식을 따랐다. 'switch'를 구현하는 더 효율적인 방법이 많지만, 여기서는 "KISS 원칙"을 적용했다. 그럼에도 불구하고 꽤 복잡한 구현이었다.

여기까지 읽고 있다면, 여러분의 끈기를 칭찬한다!

이제는 모든 복합 문장에 필수로 '{' ... '}'를 사용해야 하는 점이 점점 불편해지고 있다. 그래서 컴파일러 작성 여정의 다음 단계에서는 "dangling else" 문제를 해결해 보려고 한다. [다음 단계](../38_Dangling_Else/Readme.md)



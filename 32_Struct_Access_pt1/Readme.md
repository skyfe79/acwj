# 32장: 구조체 멤버 접근 구현

컴파일러 작성 과정의 이번 단계는 상당히 간단했다. 우리의 언어에 '.'과 '->' 토큰을 추가했고, 전역 구조체 변수의 멤버에 한 단계 접근하는 기능을 구현했다.

구현한 언어 기능을 확인할 수 있도록 테스트 프로그램 `tests/input58.c`를 살펴보자:

```c
int printf(char *fmt);

struct fred {                   // 구조체 선언, 지난번에 구현
  int x;
  char y;
  long z;
};

struct fred var2;               // 변수 선언, 지난번에 구현
struct fred *varptr;            // 포인터 변수 선언, 지난번에 구현

int main() {
  long result;

  var2.x= 12;   printf("%d\n", var2.x);         // lvalue로 멤버 접근, 새로 구현
  var2.y= 'c';  printf("%d\n", var2.y);
  var2.z= 4005; printf("%d\n", var2.z);

  result= var2.x + var2.y + var2.z;             // rvalue로 멤버 접근, 새로 구현
  printf("%d\n", result);

  varptr= &var2;                                // 기존 동작
  result= varptr->x + varptr->y + varptr->z;    // 포인터를 통한 멤버 접근, 새로 구현
  printf("%d\n", result);
  return(0);
}
```

이 코드는 구조체 멤버에 접근하는 다양한 방법을 보여준다. 구조체 변수 `var2`의 멤버에 직접 접근하는 것뿐만 아니라, 포인터를 통해 간접적으로 접근하는 방법도 구현했다. 이제 구조체를 더 유연하게 다룰 수 있게 되었다.


## 새로운 토큰

입력에서 '.'과 '->' 엘리먼트를 매칭하기 위해 두 가지 새로운 토큰, T_DOT과 T_ARROW를 추가한다. 언제나 그렇듯, 이들을 식별하는 `scan.c`의 코드는 제공하지 않는다.


## 멤버 참조 파싱

이 작업은 기존의 배열 요소 접근 코드와 매우 유사하다. 유사점과 차이점을 살펴보자. 다음 코드를 보자:

```c
  int x[5];
  int y;
  ...
  y= x[3];
```

이 코드에서는 `x` 배열의 베이스 주소를 얻고, 3을 `int` 타입의 바이트 크기(예: 3*4 = 12)로 곱한 후, 베이스 주소에 더한다. 이 주소를 접근하려는 `int`의 주소로 취급한다. 그런 다음 이 주소를 역참조해 해당 배열 위치의 값을 가져온다.

구조체 멤버 접근도 비슷하다:

```c
  struct fred { int x; char y; long z; };
  struct fred var2;
  char y;
  ...
  y= var2.y;
```

여기서는 `var2`의 베이스 주소를 얻고, `fred` 구조체 내 `y` 멤버의 오프셋을 구해 베이스 주소에 더한다. 이 주소를 접근하려는 `char`의 주소로 취급한다. 그런 다음 이 주소를 역참조해 해당 위치의 값을 가져온다.


## 후위 연산자

T_DOT과 T_ARROW는 배열 참조의 '['와 마찬가지로 식별자 이름 뒤에 오는 후위 연산자다. 따라서 `expr.c` 파일의 기존 `postfix()` 함수에 이들의 파싱을 추가하는 것이 합리적이다:

```c
static struct ASTnode *postfix(void) {
  ...
    // 구조체나 공용체 접근
  if (Token.token == T_DOT)
    return (member_access(0));
  if (Token.token == T_ARROW)
    return (member_access(1));
  ...
}
```

`expr.c` 파일의 새로운 `member_access()` 함수에 전달되는 인자는 멤버에 직접 접근하는지, 아니면 포인터를 통해 접근하는지를 나타낸다. 이제 단계별로 새로운 `member_access()` 함수를 살펴보자.

```c
// 구조체(또는 공용체)의 멤버 참조를 파싱하고 이를 위한 AST 트리를 반환한다.
// withpointer가 참이면 멤버에 대한 포인터를 통해 접근한다.
static struct ASTnode *member_access(int withpointer) {
  struct ASTnode *left, *right;
  struct symtable *compvar;
  struct symtable *typeptr;
  struct symtable *m;

  // 식별자가 구조체(또는 공용체)로 선언되었는지, 또는 구조체/공용체 포인터인지 확인
  if ((compvar = findsymbol(Text)) == NULL)
    fatals("Undeclared variable", Text);
  if (withpointer && compvar->type != pointer_to(P_STRUCT))
    fatals("Undeclared variable", Text);
  if (!withpointer && compvar->type != P_STRUCT)
    fatals("Undeclared variable", Text);
```

먼저 몇 가지 오류 검사를 수행한다. 여기에 공용체에 대한 검사를 추가해야 한다는 것을 알고 있으므로, 아직 코드를 리팩토링하지는 않을 것이다.

```c
  // 구조체에 대한 포인터라면 포인터의 값을 가져온다.
  // 그렇지 않으면 기본을 가리키는 리프 노드를 만든다.
  // 어느 쪽이든 rvalue다
  if (withpointer) {
    left = mkastleaf(A_IDENT, pointer_to(P_STRUCT), compvar, 0);
  } else
    left = mkastleaf(A_ADDR, compvar->type, compvar, 0);
  left->rvalue = 1;
```

이 시점에서 복합 변수의 기본 주소를 가져와야 한다. 포인터가 주어지면 A_IDENT AST 노드를 만들어 포인터의 값을 로드한다. 그렇지 않으면 식별자 자체가 구조체나 공용체이므로 A_ADDR AST 노드를 사용해 주소를 가져온다.

이 노드는 lvalue가 될 수 없다. 즉, `var2. = 5`와 같은 표현은 불가능하다. 반드시 rvalue여야 한다.

```c
  // 복합 타입의 세부 정보를 가져온다
  typeptr = compvar->ctype;

  // '.' 또는 '->' 토큰을 건너뛰고 멤버의 이름을 가져온다
  scan(&Token);
  ident();
```

복합 타입에 대한 포인터를 가져와 타입의 멤버 목록을 탐색할 수 있게 한다. 그리고 '.' 또는 '->' 뒤에 오는 멤버의 이름을 가져온다(그리고 그것이 식별자인지 확인한다).

```c
  // 타입에서 일치하는 멤버의 이름을 찾는다
  // 찾지 못하면 오류 발생
  for (m = typeptr->member; m != NULL; m = m->next)
    if (!strcmp(m->name, Text))
      break;

  if (m == NULL)
    fatals("No member found in struct/union: ", Text);
```

멤버 목록을 탐색하여 일치하는 멤버 이름을 찾는다.

```c
  // 오프셋을 가진 A_INTLIT 노드를 만든다
  right = mkastleaf(A_INTLIT, P_INT, NULL, m->posn);

  // 멤버의 오프셋을 구조체의 기본 주소에 더하고 역참조한다.
  // 이 시점에서 여전히 lvalue다
  left = mkastnode(A_ADD, pointer_to(m->type), left, NULL, right, NULL, 0);
  left = mkastunary(A_DEREF, m->type, left, NULL, 0);
  return (left);
}
```

멤버의 바이트 단위 오프셋은 `m->posn`에 저장되어 있으므로, 이 값을 가진 A_INTLIT 노드를 만들고 `left`에 저장된 기본 주소에 A_ADD로 더한다. 이 시점에서 멤버의 주소를 가지게 되므로, 이를 역참조(A_DEREF)하여 멤버의 값에 접근한다. 이 시점에서 여전히 lvalue다. 이를 통해 `5 + var2.x`와 `var2.x = 6` 모두를 수행할 수 있다.


### 테스트 코드 실행


`tests/input58.c`의 출력 결과는 예상대로 다음과 같다:

```
12
99
4005
4116
4116
```

어셈블리 출력의 일부를 살펴보자:

```
                                        # var2.y= 'c';
        movq    $99, %r10               # 'c'를 %r10에 로드
        leaq    var2(%rip), %r11        # var2의 베이스 주소를 %r11에 가져옴
        movq    $4, %r12                
        addq    %r11, %r12              # 베이스 주소에 4를 더함
        movb    %r10b, (%r12)           # 새로운 주소에 'c'를 기록

                                        # printf("%d\n", var2.z);
        leaq    var2(%rip), %r10        # var2의 베이스 주소를 %r11에 가져옴
        movq    $4, %r11
        addq    %r10, %r11              # 베이스 주소에 4를 더함
        movzbq  (%r11), %r11            # 이 주소에서 바이트 값을 %r11에 로드
        movq    %r11, %rsi              # %rsi로 복사
        leaq    L4(%rip), %r10
        movq    %r10, %rdi
        call    printf@PLT              # printf() 호출
```


## 결론과 다음 단계

구조체를 이렇게 쉽게 구현할 수 있어서 매우 기쁘고 놀라운 경험이었다! 앞으로의 여정에서 더 많은 것을 배울 수 있을 것이라 확신한다. 현재 우리의 컴파일러는 아직 한계가 많다. 예를 들어, 다음과 같은 코드는 처리할 수 없다:

```c
struct foo {
  int x;
  struct foo *next;
};

struct foo *listhead;
struct foo *l;

int main() {
  ...
  l= listhead->next->next;
```

이 코드는 두 단계의 포인터를 따라가야 하는데, 현재 구현은 단일 포인터 레벨만 처리할 수 있다. 이 문제는 나중에 해결해야 한다.

이제는 컴파일러가 "제대로 동작하도록" 만드는 데 많은 시간을 투자해야 할 시점이다. 지금까지는 특정 기능을 동작시키기 위해 필요한 최소한의 기능만 추가했다. 언젠가 이 특정 기능들을 더 일반화해야 한다. 따라서 이 여정에는 "정리" 단계가 필요할 것이다.

이제 구조체가 대체로 동작하니, 다음 단계에서는 공용체(union)를 추가해 볼 것이다. [다음 단계](../33_Unions/Readme.md)



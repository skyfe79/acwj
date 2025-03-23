# 12부: 타입, 1부

컴파일러에 타입을 추가하는 작업을 막 시작했다. 하지만 이 작업은 나에게도 새로운 도전이다. [이전 컴파일러](https://github.com/DoctorWkt/h-compiler)에서는 `int` 타입만 다뤘기 때문이다. SubC 소스 코드를 참고하고 싶은 유혹을 참아냈다. 그래서 내 방식대로 진행하고 있으며, 타입과 관련된 더 큰 문제를 다루면서 일부 코드를 다시 작성해야 할 가능성이 크다.


## 현재 어떤 타입을 사용할까?

먼저 전역 변수로 `char`와 `int`를 사용한다. 이미 함수를 위해 `void` 키워드를 추가했다. 다음 단계에서는 함수의 반환 값을 추가할 예정이다. 따라서 지금은 `void`가 존재하지만, 아직 완전히 처리하지는 않는다.

명백히 `char`는 `int`보다 훨씬 제한된 값의 범위를 가진다. SubC와 마찬가지로 `char`의 범위는 0부터 255까지로 설정하고, `int`는 부호 있는 값의 범위를 사용한다.

이것은 `char` 값을 `int`로 확장할 수 있음을 의미한다. 하지만 `int` 값을 `char` 범위로 좁히려고 할 때는 개발자에게 경고를 표시해야 한다.


## 새로운 키워드와 토큰

새로 추가된 것은 'char' 키워드와 T_CHAR 토큰뿐이다. 특별히 흥미로운 내용은 없다.


## 표현식 타입

이제부터 모든 표현식은 타입을 가진다. 이는 다음을 포함한다:

+ 정수 리터럴, 예를 들어 56은 `int` 타입이다.
+ 수학 표현식, 예를 들어 45 - 12는 `int` 타입이다.
+ 변수, 예를 들어 `x`를 `char`로 선언했다면, 그 *rvalue*는 `char` 타입이다.

표현식을 평가할 때마다 각 표현식의 타입을 추적해야 한다. 이를 통해 필요에 따라 타입을 확장하거나, 필요한 경우 축소를 거부할 수 있다.

SubC 컴파일러에서 Nils는 단일 *lvalue* 구조체를 만들었다. 이 단일 구조체에 대한 포인터를 재귀 파서에 전달하여 파싱 중인 시점에서 모든 표현식의 타입을 추적했다.

나는 다른 방식을 선택했다. 추상 구문 트리(AST) 노드를 수정하여 해당 시점에서 트리의 타입을 보유하는 `type` 필드를 추가했다. `defs.h`에서 지금까지 만든 타입은 다음과 같다:

```c
// 기본 타입
enum {
  P_NONE, P_VOID, P_CHAR, P_INT
};
```

나는 이를 *기본* 타입이라고 불렀다. SubC에서 Nils가 그랬던 것처럼 더 나은 이름을 생각할 수 없었기 때문이다. 데이터 타입이라고 할 수도 있다. P_NONE 값은 AST 노드가 표현식을 나타내지 않으며 타입이 없음을 나타낸다. 예를 들어 문장을 결합하는 A_GLUE 노드 타입이 있다: 왼쪽 문장이 생성되면 더 이상 타입이 없다.

`tree.c`를 보면, AST 노드를 생성하는 함수들이 새로운 AST 노드 구조체(`defs.h`에 있음)의 `type` 필드에 값을 할당하도록 수정된 것을 확인할 수 있다:

```c
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  int type;                     // 이 트리가 생성하는 표현식의 타입
  ...
};
```


## 변수 선언과 타입

이제 전역 변수를 선언하는 데 최소 두 가지 방법이 있다:

```c
  int x; char y;
```

이 코드를 파싱해야 한다. 하지만 먼저 각 변수의 타입을 어떻게 기록할까? `symtable` 구조체를 수정해야 한다. 또한 앞으로 사용할 "구조적 타입"에 대한 세부 사항을 추가했다(`defs.h` 파일에 있음):

```c
// 구조적 타입
enum {
  S_VARIABLE, S_FUNCTION
};

// 심볼 테이블 구조체
struct symtable {
  char *name;                   // 심볼 이름
  int type;                     // 심볼의 기본 타입
  int stype;                    // 심볼의 구조적 타입
};
```

`sym.c` 파일의 `newglob()` 함수에 이 새로운 필드를 초기화하는 코드가 추가됐다:

```c
int addglob(char *name, int type, int stype) {
  ...
  Gsym[y].type = type;
  Gsym[y].stype = stype;
  return (y);
}
```


## 변수 선언 파싱

이제 타입 파싱과 변수 자체의 파싱을 분리할 때다. `decl.c` 파일에 다음과 같은 코드를 추가한다:

```c
// 현재 토큰을 파싱하고
// 기본 타입 enum 값을 반환한다
int parse_type(int t) {
  if (t == T_CHAR) return (P_CHAR);
  if (t == T_INT)  return (P_INT);
  if (t == T_VOID) return (P_VOID);
  fatald("Illegal type, token", t);
}

// 변수 선언을 파싱한다
void var_declaration(void) {
  int id, type;

  // 변수의 타입을 얻고, 다음으로 식별자를 얻는다
  type = parse_type(Token.token);
  scan(&Token);
  ident();
  id = addglob(Text, type, S_VARIABLE);
  genglobsym(id);
  semi();
}
```


## 표현식 타입 처리하기

지금까지는 비교적 쉬운 부분을 다뤘다. 현재까지 우리는 다음 작업을 완료했다:

  + 세 가지 타입(`char`, `int`, `void`)을 정의했다.
  + 변수 선언을 파싱하여 해당 변수의 타입을 확인했다.
  + 각 변수의 타입을 심볼 테이블에 저장했다.
  + 각 AST 노드에 표현식의 타입을 저장했다.

이제 실제로 AST 노드에 타입을 채워 넣어야 한다. 그런 다음 타입 확장이 필요한 경우와 타입 충돌을 거부해야 하는 경우를 결정해야 한다. 이제 본격적으로 작업을 진행해 보자!


## 기본 터미널 파싱

정수 리터럴 값과 변수 식별자의 파싱부터 시작한다. 여기서 주의할 점은 다음과 같은 코드를 처리할 수 있어야 한다는 것이다:

```c
  char j; j= 2;
```

만약 `2`를 P_INT로 표시하면, P_CHAR 타입의 `j` 변수에 저장할 때 값을 좁힐 수 없다. 현재는 작은 정수 리터럴 값을 P_CHAR로 유지하기 위해 일부 의미론적 코드를 추가했다:

```c
// 기본 요소를 파싱하고 이를 나타내는 AST 노드를 반환한다.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
    case T_INTLIT:
      // INTLIT 토큰인 경우, 이를 위한 리프 AST 노드를 생성한다.
      // P_CHAR 범위 내에 있다면 P_CHAR로 만든다.
      if ((Token.intvalue) >= 0 && (Token.intvalue < 256))
        n = mkastleaf(A_INTLIT, P_CHAR, Token.intvalue);
      else
        n = mkastleaf(A_INTLIT, P_INT, Token.intvalue);
      break;

    case T_IDENT:
      // 해당 식별자가 존재하는지 확인한다.
      id = findglob(Text);
      if (id == -1)
        fatals("Unknown variable", Text);

      // 이를 위한 리프 AST 노드를 생성한다.
      n = mkastleaf(A_IDENT, Gsym[id].type, id);
      break;

    default:
      fatald("Syntax error, token", Token.token);
  }

  // 다음 토큰을 스캔하고 리프 노드를 반환한다.
  scan(&Token);
  return (n);
}
```

또한 식별자의 경우, 전역 심볼 테이블에서 타입 정보를 쉽게 가져올 수 있다.


## 바이너리 표현식 구축: 타입 비교

바이너리 수학 연산자를 사용해 수학 표현식을 구축할 때, 왼쪽 자식과 오른쪽 자식의 타입을 고려해야 한다. 두 타입이 호환되지 않으면, 타입을 확장하거나 아무 작업도 하지 않거나, 표현식을 거부해야 한다.

현재 `types.c`라는 새 파일에 두 타입을 비교하는 함수를 작성했다. 코드는 다음과 같다:

```c
// 두 기본 타입이 주어졌을 때, 호환 가능하면 true를 반환하고, 그렇지 않으면 false를 반환한다.
// 또한 한 타입을 다른 타입에 맞추기 위해 확장해야 하는 경우 A_WIDEN 연산을 반환한다.
// onlyright가 true이면, 왼쪽을 오른쪽으로만 확장한다.
int type_compatible(int *left, int *right, int onlyright) {

  // void 타입은 어떤 타입과도 호환되지 않음
  if ((*left == P_VOID) || (*right == P_VOID)) return (0);

  // 같은 타입이면 호환 가능
  if (*left == *right) { *left = *right = 0; return (1);
  }

  // 필요에 따라 P_CHAR를 P_INT로 확장
  if ((*left == P_CHAR) && (*right == P_INT)) {
    *left = A_WIDEN; *right = 0; return (1);
  }
  if ((*left == P_INT) && (*right == P_CHAR)) {
    if (onlyright) return (0);
    *left = 0; *right = A_WIDEN; return (1);
  }
  // 나머지 타입은 모두 호환 가능
  *left = *right = 0;
  return (1);
}
```

여기서 몇 가지 중요한 점이 있다. 먼저, 두 타입이 동일하면 단순히 True를 반환한다. P_VOID 타입은 다른 타입과 혼합할 수 없다.

한쪽이 P_CHAR이고 다른 한쪽이 P_INT인 경우, 결과를 P_INT로 확장할 수 있다. 이를 위해 들어오는 타입 정보를 수정하고, 이를 0(아무 작업도 하지 않음) 또는 새로운 AST 노드 타입인 A_WIDEN으로 대체한다. 이는 더 좁은 자식의 값을 더 넓은 자식의 값으로 확장하라는 의미다. 이는 곧 실제 동작에서 확인할 수 있다.

`onlyright`라는 추가 인자가 있다. 이는 A_ASSIGN AST 노드에서 왼쪽 자식의 표현식을 오른쪽의 *lvalue* 변수에 할당할 때 사용한다. 이 값이 설정되면, P_INT 표현식을 P_CHAR 변수로 전달하지 않는다.

현재로서는 다른 모든 타입 쌍을 허용한다.

배열과 포인터를 도입하면 이 코드를 수정해야 할 것이라고 확신한다. 또한 코드를 더 간단하고 우아하게 만들 방법을 찾기를 바란다. 하지만 지금은 이 정도로 충분하다.


## `type_compatible()`를 표현식에서 사용하기


이번 컴파일러 버전에서 `type_compatible()`를 세 군데에서 사용했다. 먼저 이항 연산자를 사용한 표현식 병합부터 살펴본다. `expr.c` 파일의 `binexpr()` 함수를 다음과 같이 수정했다:

```c
    // 두 타입이 호환되는지 확인
    lefttype = left->type;
    righttype = right->type;
    if (!type_compatible(&lefttype, &righttype, 0))
      fatal("타입이 호환되지 않음");

    // 필요하다면 한쪽을 확장. 타입 변수는 이제 A_WIDEN 상태
    if (lefttype)
      left = mkastunary(lefttype, right->type, left, 0);
    if (righttype)
      right = mkastunary(righttype, left->type, right, 0);

    // 서브 트리를 병합. 토큰을 AST 연산으로 동시에 변환
    left = mkastnode(arithop(tokentype), left->type, left, NULL, right, 0);
```

호환되지 않는 타입은 거부한다. 하지만 `type_compatible()`가 0이 아닌 `lefttype` 또는 `righttype` 값을 반환하면, 이 값은 실제로 A_WIDEN이다. 이를 사용해 좁은 타입의 자식을 가진 단항 AST 노드를 생성할 수 있다. 코드 생성기에 도달하면 이 자식의 값을 확장해야 한다는 것을 알게 된다.

이제 표현식 값을 확장해야 하는 다른 부분은 어디인가?


## `type_compatible()`를 사용하여 표현식 출력하기

`print` 키워드를 사용할 때, 출력할 `int` 타입의 표현식이 필요하다. 따라서 `stmt.c` 파일의 `print_statement()` 함수를 수정해야 한다:

```c
static struct ASTnode *print_statement(void) {
  struct ASTnode *tree;
  int lefttype, righttype;
  int reg;

  ...
  // 다음 표현식을 파싱한다
  tree = binexpr(0);

  // 두 타입이 호환되는지 확인한다.
  lefttype = P_INT; righttype = tree->type;
  if (!type_compatible(&lefttype, &righttype, 0))
    fatal("타입이 호환되지 않음");

  // 필요하다면 트리를 확장한다.
  if (righttype) tree = mkastunary(righttype, P_INT, tree, 0);
```


## `type_compatible()`를 사용해 변수에 할당하기

이 부분은 타입을 확인해야 하는 마지막 부분이다. 변수에 값을 할당할 때, 오른쪽 표현식을 확장할 수 있는지 확인해야 한다. 넓은 타입을 좁은 변수에 저장하려는 시도를 거부해야 한다. 다음은 `stmt.c` 파일의 `assignment_statement()` 함수에 추가된 새로운 코드이다.

```c
static struct ASTnode *assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int lefttype, righttype;
  int id;

  ...
  // 변수를 위한 lvalue 노드 생성
  right = mkastleaf(A_LVIDENT, Gsym[id].type, id);

  // 다음 표현식을 파싱
  left = binexpr(0);

  // 두 타입이 호환되는지 확인
  lefttype = left->type;
  righttype = right->type;
  if (!type_compatible(&lefttype, &righttype, 1))  // 1을 주목
    fatal("Incompatible types");

  // 필요하다면 왼쪽을 확장
  if (lefttype)
    left = mkastunary(lefttype, right->type, left, 0);
```

`type_compatible()` 함수 호출 끝에 있는 1을 주목하자. 이 값은 넓은 값을 좁은 변수에 저장할 수 없다는 의미를 강제한다.

위의 모든 내용을 통해 이제 몇 가지 타입을 파싱하고 합리적인 언어 의미를 적용할 수 있다. 가능한 경우 값을 확장하고, 타입 축소를 방지하며, 적절하지 않은 타입 충돌을 막는다. 이제 코드 생성 부분으로 넘어갈 차례이다.


## x86-64 코드 생성 방식의 변화

어셈블리 출력은 레지스터 기반이며, 크기가 고정되어 있다. 우리가 영향을 줄 수 있는 부분은 다음과 같다:

+ 변수를 저장할 메모리 위치의 크기
+ 레지스터에 데이터를 저장할 때 사용할 부분의 크기 (예: 문자는 1바이트, 64비트 정수는 8바이트)

먼저 `cg.c` 파일의 x86-64 특정 코드를 살펴보고, `gen.c` 파일의 일반적인 코드 생성기에서 이를 어떻게 사용하는지 알아보자.

변수를 위한 저장 공간을 생성하는 것부터 시작해 보자.

```c
// 전역 심볼 생성
void cgglobsym(int id) {
  // P_INT 또는 P_CHAR 선택
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\t.comm\t%s,8,8\n", Gsym[id].name);
  else
    fprintf(Outfile, "\t.comm\t%s,1,1\n", Gsym[id].name);
}
```

심볼 테이블의 변수 슬롯에서 타입을 추출하고, 이 타입에 따라 1바이트 또는 8바이트를 할당한다. 이제 값을 레지스터로 로드해야 한다:

```c
// 변수에서 값을 레지스터로 로드
// 레지스터 번호 반환
int cgloadglob(int id) {
  // 새 레지스터 할당
  int r = alloc_register();

  // 초기화 코드 출력: P_CHAR 또는 P_INT
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
  else
    fprintf(Outfile, "\tmovzbq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
  return (r);
}
```

`movq` 명령어는 8바이트를 8바이트 레지스터로 이동시킨다. `movzbq` 명령어는 8바이트 레지스터를 0으로 초기화한 후 1바이트를 이동시킨다. 이는 1바이트 값을 8바이트로 암시적으로 확장한다. 저장 함수도 비슷하다:

```c
// 레지스터의 값을 변수에 저장
int cgstorglob(int r, int id) {
  // P_INT 또는 P_CHAR 선택
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], Gsym[id].name);
  else
    fprintf(Outfile, "\tmovb\t%s, %s(\%%rip)\n", breglist[r], Gsym[id].name);
  return (r);
}
```

이번에는 레지스터의 "바이트" 이름과 `movb` 명령어를 사용해 1바이트를 이동시킨다.

다행히 `cgloadglob()` 함수는 이미 P_CHAR 변수의 확장을 처리했다. 따라서 새로운 `cgwiden()` 함수의 코드는 다음과 같다:

```c
// 레지스터의 값을 이전 타입에서 새 타입으로 확장
// 새 값을 가진 레지스터 반환
int cgwiden(int r, int oldtype, int newtype) {
  // 할 일 없음
  return (r);
}
```


## 일반 코드 생성기의 변경 사항

위의 내용을 적용하면서 `gen.c`의 일반 코드 생성기에 몇 가지 변경이 발생했다:

  + `cgloadglob()`와 `cgstorglob()` 호출 시 이제 심볼의 이름 대신 슬롯 번호를 전달한다.
  + 마찬가지로, `genglobsym()`도 심볼의 슬롯 번호를 받아 이를 `cgglobsym()`에 전달한다.

가장 큰 변경 사항은 새로운 A_WIDEN AST 노드 타입을 처리하는 코드다. 현재 하드웨어 플랫폼에서는 이 노드가 필요하지 않지만(`cgwiden()`은 아무 작업도 수행하지 않음), 다른 플랫폼을 위해 남겨둔다:

```c
    case A_WIDEN:
      // 자식 타입을 부모 타입으로 확장
      return (cgwiden(leftreg, n->left->type, n->type));
```


## 새로운 타입 변경 사항 테스트

테스트 입력 파일 `tests/input10`은 다음과 같다:

```c
void main()
{
  int i; char j;

  j= 20; print j;
  i= 10; print i;

  for (i= 1;   i <= 5; i= i + 1) { print i; }
  for (j= 253; j != 2; j= j + 1) { print j; }
}
```

이 코드는 `char`와 `int` 타입에 값을 할당하고 출력할 수 있는지 확인한다. 또한 `char` 변수의 경우 값이 253, 254, 255, 0, 1, 2 순서로 오버플로우가 발생하는지 검증한다.

```
$ make test
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
   stmt.c sym.c tree.c types.c
./comp1 tests/input10
cc -o out out.s
./out
20
10
1
2
3
4
5
253
254
255
0
1
```

생성된 어셈블리 코드 일부를 살펴보자:

```
        .comm   i,8,8                   # 8바이트 i 저장 공간
        .comm   j,1,1                   # 1바이트 j 저장 공간
        ...
        movq    $20, %r8
        movb    %r8b, j(%rip)           # j= 20
        movzbq  j(%rip), %r8
        movq    %r8, %rdi               # j 출력
        call    printint

        movq    $253, %r8
        movb    %r8b, j(%rip)           # j= 253
L3:
        movzbq  j(%rip), %r8
        movq    $2, %r9
        cmpq    %r9, %r8                # j != 2인 동안
        je      L4
        movzbq  j(%rip), %r8
        movq    %r8, %rdi               # j 출력
        call    printint
        movzbq  j(%rip), %r8
        movq    $1, %r9                 # j= j + 1
        addq    %r8, %r9
        movb    %r9b, j(%rip)
        jmp     L3
```

이 코드는 가장 우아한 어셈블리 코드는 아니지만 정상적으로 동작한다. 또한 `$ make test` 명령어를 통해 이전의 모든 코드 예제가 여전히 작동하는지 확인했다.


## 결론과 다음 단계

컴파일러 개발 여정의 다음 단계에서는 인자를 하나 받는 함수 호출과 함수에서 값을 반환하는 기능을 추가할 예정이다. [다음 단계](../13_Functions_pt2/Readme.md)에서 이어서 살펴보자.



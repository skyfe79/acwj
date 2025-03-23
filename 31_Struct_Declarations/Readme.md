# 31부: 구조체 구현, 1부

이번 파트에서는 컴파일러 개발 과정 중 구조체를 언어에 구현하기 시작했다. 아직 완전히 동작하지는 않지만, 구조체를 선언하고 구조체 타입의 전역 변수를 정의할 수 있도록 코드에 상당한 변경을 가했다.


## 심볼 테이블 변경 사항

이전 부분에서 언급했듯이, 심볼이 복합 타입일 경우 복합 타입 노드를 가리키는 포인터를 심볼 테이블 구조에 추가해야 한다. 또한 연결 리스트를 지원하기 위해 `next` 포인터와 `member` 포인터를 추가했다. 함수 노드의 `member` 포인터는 함수의 매개변수 목록을 담는다. 구조체의 경우 `member` 노드를 사용해 구조체의 멤버 필드를 저장한다.

따라서 이제 심볼 테이블은 다음과 같은 구조를 가진다:

```c
struct symtable {
  char *name;                   // 심볼의 이름
  int type;                     // 심볼의 기본 타입
  struct symtable *ctype;       // 필요한 경우 복합 타입을 가리키는 포인터
  ...
  struct symtable *next;        // 리스트에서 다음 심볼을 가리키는 포인터
  struct symtable *member;      // 함수, 구조체, 공용체, 열거형의 첫 번째 멤버를 가리키는 포인터
};
```

또한 `data.h`에 두 개의 새로운 심볼 리스트가 추가되었다:

```c
// 심볼 테이블 리스트
struct symtable *Globhead, *Globtail;     // 전역 변수와 함수
struct symtable *Loclhead, *Locltail;     // 지역 변수
struct symtable *Parmhead, *Parmtail;     // 지역 매개변수
struct symtable *Membhead, *Membtail;     // 구조체/공용체 멤버의 임시 리스트
struct symtable *Structhead, *Structtail; // 구조체 타입 리스트
```


## `sym.c` 파일 변경 사항

`sym.c` 파일 및 코드 전체에서, 기존에는 `int type` 인자만 사용해 어떤 타입인지 판단했다. 하지만 이제는 복합 타입이 도입되면서 이 방식만으로는 충분하지 않다. P_STRUCT 정수 값은 어떤 것이 구조체인지는 알려주지만, 구체적으로 어떤 구조체인지는 알려주지 않는다.

따라서 많은 함수가 이제 `int type` 인자와 함께 `struct symtable *ctype` 인자를 받는다. `type`이 P_STRUCT일 때, `ctype`은 해당 구조체 타입을 정의하는 노드를 가리킨다.

`sym.c` 파일 내 모든 `addXX()` 함수는 이 추가 인자를 받도록 수정되었다. 또한 두 개의 새로운 리스트에 노드를 추가하기 위해 `addmemb()` 함수와 `addstruct()` 함수가 새로 추가되었다. 이 함수들은 기존 `addXX()` 함수와 동일하게 작동하지만, 다른 리스트를 대상으로 한다. 이 함수들에 대해서는 나중에 다시 설명할 것이다.


## 새로운 토큰의 등장

오랜만에 새로운 토큰인 P_STRUCT가 추가되었다. 이 토큰은 `struct` 키워드와 함께 사용된다. `scan.c` 파일의 변경 사항은 사소하므로 생략한다.


## 구조체 파싱하기

문법에서 `struct` 키워드를 파싱해야 하는 여러 경우가 있다:

  + 명명된 구조체 정의
  + 이름 없는 구조체 정의와 그 타입의 변수 선언
  + 다른 구조체나 유니온 내부의 구조체 정의
  + 이전에 정의된 구조체 타입의 변수 선언

처음에는 구조체 파싱을 어디에 넣어야 할지 고민했다. 새로운 구조체 정의를 파싱한다고 가정하고, 변수 식별자를 보면 빠져나올지, 아니면 변수 선언으로 가정할지 결정하기 어려웠다.

결국 `struct <식별자>`를 본 후에는, `int`가 `int` 타입을 명명하는 것처럼 단순히 타입을 명명하는 것으로 가정해야 한다는 것을 깨달았다. 다음 토큰을 파싱해서 추가 정보를 얻어야 했다.

따라서 `decl.c`의 `parse_type()` 함수를 수정해 스칼라 타입(예: `int`)과 복합 타입(예: `struct foo`)을 모두 파싱할 수 있게 했다. 이제 복합 타입을 반환할 수 있으므로, 이 복합 타입을 정의하는 노드에 대한 포인터를 반환하는 방법도 찾아야 했다:

```c
// 현재 토큰을 파싱하고
// 기본 타입 열거값과 복합 타입에 대한 포인터를 반환한다.
// 다음 토큰도 스캔한다.
int parse_type(struct symtable **ctype) {
  int type;
  switch (Token.token) {
    ...         // T_VOID, T_CHAR 등 기존 코드
    case T_STRUCT:
      type = P_STRUCT;
      *ctype = struct_declaration();
      break;
    ...
```

`struct_declaration()` 함수를 호출해 기존 구조체 타입을 찾거나 새로운 구조체 타입의 선언을 파싱한다.


## 변수 리스트 파싱 리팩토링

기존 코드에는 `param_declaration()`이라는 함수가 있었다. 이 함수는 쉼표로 구분된 파라미터 리스트를 파싱했다. 예를 들어:

```c
int fred(int x, char y, long z);
```

이런 식으로 함수 선언의 파라미터 리스트를 처리했다. 그런데 구조체와 공용체 선언에도 변수 리스트가 있다. 다만 이 경우 세미콜론으로 구분하고 중괄호로 둘러싸여 있다. 예를 들면:

```c
struct fred { int x; char y; long z; };
```

이제 이 함수를 리팩토링해 두 종류의 리스트를 모두 파싱할 수 있도록 했다. 함수는 두 개의 토큰을 받는다. 하나는 구분자 토큰(예: T_SEMI), 다른 하나는 종료 토큰(예: T_RBRACE)이다. 이렇게 하면 두 스타일의 리스트를 모두 파싱할 수 있다.

```c
// 변수 리스트를 파싱한다.
// 심볼 테이블 리스트 중 하나에 심볼로 추가하고, 변수의 개수를 반환한다.
// funcsym이 NULL이 아니면 기존 함수 프로토타입이 있으므로 각 변수의 타입을 이 프로토타입과 비교한다.
static int var_declaration_list(struct symtable *funcsym, int class,
                                int separate_token, int end_token) {
    ...
    // 타입과 식별자를 가져온다
    type = parse_type(&ctype);
    ...
    // 클래스에 따라 적절한 심볼 테이블 리스트에 새 파라미터를 추가한다
    var_declaration(type, ctype, class);
}
```

함수 파라미터 리스트를 파싱할 때는 다음과 같이 호출한다:

```c
    var_declaration_list(oldfuncsym, C_PARAM, T_COMMA, T_RPAREN);
```

구조체 멤버 리스트를 파싱할 때는 다음과 같이 호출한다:

```c
    var_declaration_list(NULL, C_MEMBER, T_SEMI, T_RBRACE);
```

또한 `var_declaration()` 호출 시 이제 변수의 타입, 복합 타입 포인터(구조체나 공용체인 경우), 그리고 변수의 클래스를 전달한다.

이제 구조체의 멤버 리스트를 파싱할 수 있다. 이제 전체 구조체를 어떻게 파싱하는지 살펴보자.


## `struct_declaration()` 함수

단계별로 살펴보자.

```c
static struct symtable *struct_declaration(void) {
  struct symtable *ctype = NULL;
  struct symtable *m;
  int offset;

  // struct 키워드를 건너뛴다
  scan(&Token);

  // 다음에 struct 이름이 있는지 확인
  if (Token.token == T_IDENT) {
    // 일치하는 복합 타입을 찾는다
    ctype = findstruct(Text);
    scan(&Token);
  }
```

이 시점에서 `struct` 키워드와 그 뒤에 오는 식별자를 확인했다.  
이미 존재하는 struct 타입이라면 `ctype`은 기존 타입 노드를 가리킨다. 그렇지 않으면 `ctype`은 NULL이다.

```c
  // 다음 토큰이 '{'가 아니라면, 이는 기존 struct 타입의 사용이다.
  // 타입에 대한 포인터를 반환한다.
  if (Token.token != T_LBRACE) {
    if (ctype == NULL)
      fatals("unknown struct type", Text);
    return (ctype);
  }
```

'{'를 보지 못했다면, 이는 기존 타입의 이름을 사용하는 경우다.  
`ctype`은 NULL이 될 수 없으므로, 먼저 이를 확인한 후 기존 struct 타입에 대한 포인터를 반환한다.  
이는 `parse_type()` 함수로 돌아가며, 다음과 같은 코드를 실행한다:

```c
      type = P_STRUCT; *ctype = struct_declaration();
```

하지만 반환하지 않았다면, '{'를 발견한 것이며, 이는 struct 타입의 정의를 의미한다. 계속 진행해보자.

```c
  // 이 struct 타입이 이전에 정의되지 않았는지 확인
  if (ctype)
    fatals("previously defined struct", Text);

  // struct 노드를 생성하고 '{'를 건너뛴다
  ctype = addstruct(Text, P_STRUCT, NULL, 0, 0);
  scan(&Token);
```

동일한 이름의 struct를 두 번 선언할 수 없으므로 이를 방지한다.  
그런 다음 심볼 테이블에 새로운 struct 타입의 노드를 생성한다.  
지금까지는 이름과 P_STRUCT 타입만 가지고 있다.

```c
  // 멤버 목록을 스캔하고 struct 타입의 노드에 연결
  var_declaration_list(NULL, C_MEMBER, T_SEMI, T_RBRACE);
  rbrace();
```

이 코드는 멤버 목록을 파싱한다. 각 멤버에 대해 새로운 심볼 노드가 `Membhead`와 `Membtail`이 가리키는 리스트에 추가된다.  
이 리스트는 임시적이며, 다음 코드에서 멤버 리스트를 새로운 struct 타입 노드로 이동시킨다:

```c
  ctype->member = Membhead;
  Membhead = Membtail = NULL;
```

이제 이름과 struct 내 멤버 목록을 가진 struct 타입 노드가 생겼다.  
남은 작업은 다음과 같다:

  + struct의 전체 크기를 결정하고,
  + struct의 시작점부터 각 멤버의 오프셋을 결정한다.

이 중 일부는 메모리에서 스칼라 값의 정렬과 관련해 하드웨어에 의존적이다.  
현재 코드를 보여주고, 나중에 함수 호출 구조를 설명하겠다.

```c
  // 첫 번째 멤버의 오프셋을 설정하고 그 뒤의 첫 번째 빈 바이트를 찾는다
  m = ctype->member;
  m->posn = 0;
  offset = typesize(m->type, m->ctype);
```

이제 `typesize()`라는 새로운 함수가 생겼다. 이 함수는 스칼라, 포인터, 복합 타입의 크기를 반환한다.  
첫 번째 멤버의 위치는 0으로 설정되고, 그 크기를 사용해 다음 멤버가 저장될 수 있는 첫 번째 바이트를 결정한다.  
하지만 이제 정렬을 고려해야 한다.

예를 들어, 4바이트 스칼라 값이 4바이트 경계에 정렬되어야 하는 32비트 아키텍처에서는:

```c
struct {
  char x;               // 오프셋 0
  int y;                // 오프셋 4, 1이 아님
};
```

따라서 각 연속 멤버의 오프셋을 계산하는 코드는 다음과 같다:

```c
  // struct 내 각 연속 멤버의 위치를 설정
  for (m = m->next; m != NULL; m = m->next) {
    // 이 멤버의 오프셋을 설정
    m->posn = genalign(m->type, offset, 1);

    // 이 멤버 다음의 빈 바이트 오프셋을 가져옴
    offset += typesize(m->type, m->ctype);
  }
```

`genalign()`이라는 새로운 함수가 있다. 이 함수는 현재 오프셋과 정렬이 필요한 타입을 받아, 이 타입의 정렬에 맞는 첫 번째 오프셋을 반환한다.  
예를 들어, `genalign(P_INT, 3, 1)`은 P_INT가 4바이트 정렬이 필요하다면 4를 반환할 수 있다. 마지막 인자 1에 대해서는 곧 설명하겠다.

따라서 `genalign()`은 이 멤버의 올바른 정렬을 계산하고, 그 다음 멤버를 위해 이 멤버의 크기를 더해 다음 빈(정렬되지 않은) 위치를 결정한다.

위 과정을 모든 멤버에 대해 수행하면 `offset`은 struct의 전체 크기(바이트 단위)가 된다. 따라서:

```c
  // struct의 전체 크기를 설정
  ctype->size = offset;
  return (ctype);
}
```


## `typesize()` 함수

새로운 함수들이 어떤 역할을 하고 어떻게 동작하는지 하나씩 살펴보자. 먼저 `types.c` 파일에 있는 `typesize()` 함수부터 시작한다.

```c
// 주어진 타입과 복합 타입 포인터를 기반으로
// 해당 타입의 크기를 바이트 단위로 반환한다
int typesize(int type, struct symtable *ctype) {
  if (type == P_STRUCT)
    return(ctype->size);
  return(genprimsize(type));
}
```

타입이 구조체(struct)인 경우, 구조체의 타입 노드에서 크기를 반환한다. 그렇지 않으면 스칼라 타입이나 포인터 타입이므로 `genprimsize()` 함수를 호출해 타입의 크기를 구한다. `genprimsize()`는 하드웨어에 특화된 `cgprimsize()`를 호출한다. 간단하고 명확하다.


## `genalign()`과 `cgalign()` 함수

이제 다소 복잡한 코드를 살펴볼 차례다. 특정 타입과 이미 존재하는 정렬되지 않은 오프셋이 주어졌을 때, 해당 타입의 값을 배치하기 위해 다음으로 정렬된 위치를 알아내야 한다.

또한 스택에서 이러한 작업을 수행해야 할 가능성도 고려했다. 스택은 위쪽이 아니라 아래쪽으로 증가하기 때문에 함수에 세 번째 인자를 추가했다. 이 인자는 다음 정렬된 위치를 찾아야 할 *방향*을 나타낸다.

정렬에 대한 지식은 하드웨어에 따라 다르기 때문에, 다음과 같이 코드를 작성했다:

```c
int genalign(int type, int offset, int direction) {
  return (cgalign(type, offset, direction));
}
```

그리고 `cg.c` 파일에 있는 `cgalign()` 함수를 살펴보자:

```c
// 스칼라 타입, 아직 할당되지 않은 메모리 오프셋,
// 그리고 방향(1은 위, -1은 아래)이 주어졌을 때,
// 이 스칼라 타입에 적합하게 정렬된 메모리 오프셋을 계산하고 반환한다.
// 이 오프셋은 원래 오프셋일 수도 있고, 원래 오프셋보다 위/아래일 수도 있다.
int cgalign(int type, int offset, int direction) {
  int alignment;

  // x86-64에서는 이 작업이 필요하지 않지만,
  // char는 어떤 오프셋에도 정렬하고, int/포인터는 4바이트 정렬을 적용한다.
  switch(type) {
    case P_CHAR: return (offset);
    case P_INT:
    case P_LONG: break;
    default:     fatald("Bad type in calc_aligned_offset:", type);
  }

  // 여기서는 int나 long을 다룬다. 4바이트 오프셋으로 정렬한다.
  // 이 코드를 일반화해 다른 곳에서도 재사용할 수 있도록 했다.
  alignment= 4;
  offset = (offset + direction * (alignment-1)) & ~(alignment-1);
  return (offset);
}
```

먼저, x86-64 아키텍처에서는 정렬을 걱정할 필요가 없다는 점을 알고 있다. 하지만 정렬을 처리하는 과정을 직접 다뤄보는 것이 좋을 것 같아서, 이 예제를 통해 다른 백엔드에서도 활용할 수 있도록 했다.

이 코드는 `char` 타입에 대해서는 주어진 오프셋을 그대로 반환한다. `char`는 어떤 정렬에도 저장될 수 있기 때문이다. 하지만 `int`와 `long`에 대해서는 4바이트 정렬을 강제한다.

이제 복잡한 오프셋 표현식을 자세히 살펴보자. 첫 번째 `alignment-1`은 오프셋 0을 3으로, 1을 4로, 2를 5로 바꾸는 식이다. 그리고 마지막에 이를 3의 역수, 즉 `...111111100`과 AND 연산하여 마지막 두 비트를 버리고 값을 다시 올바른 정렬 위치로 낮춘다.

따라서 다음과 같은 결과가 나온다:

| 오프셋 | 추가 값 | 새로운 오프셋 |
|:------:|:---------:|:----------:|
|   0    |    3      |    0       |
|   1    |    4      |    4       |
|   2    |    5      |    4       |
|   3    |    6      |    4       |
|   4    |    7      |    4       |
|   5    |    8      |    8       |
|   6    |    9      |    8       |
|   7    |   10      |    8       |

오프셋 0은 그대로 유지되지만, 1부터 3까지의 값은 4로 올라간다. 4는 정렬된 상태를 유지하고, 5부터 7까지의 값은 8로 올라간다.

이제 마법 같은 부분이다. `direction`이 1이면 지금까지 본 모든 동작이 수행된다. `direction`이 -1이면 오프셋을 반대 방향으로 보내서 값의 "상단"이 위에 있는 것과 충돌하지 않도록 한다:

| 오프셋 | 추가 값 | 새로운 오프셋 |
|:------:|:---------:|:----------:|
|   0    |   -3      |   -4       |
|  -1    |   -4      |   -4       |
|  -2    |   -5      |   -8       |
|  -3    |   -6      |   -8       |
|  -4    |   -7      |   -8       |
|  -5    |   -8      |   -8       |
|  -6    |   -9      |   -12      |
|  -7    |  -10      |   -12      |


## 전역 구조체 변수 생성하기

이제 구조체 타입을 파싱할 수 있고, 이 타입에 대한 전역 변수를 선언할 수 있다. 이제 전역 변수에 메모리 공간을 할당하도록 코드를 수정해 보자:

```c
// 전역 심볼을 생성하지만 함수는 생성하지 않음
void cgglobsym(struct symtable *node) {
  int size;

  if (node == NULL) return;
  if (node->stype == S_FUNCTION) return;

  // 타입의 크기를 가져옴
  size = typesize(node->type, node->ctype);

  // 전역 식별자와 레이블 생성
  cgdataseg();
  fprintf(Outfile, "\t.globl\t%s\n", node->name);
  fprintf(Outfile, "%s:", node->name);

  // 이 타입에 대한 공간 생성
  switch (size) {
    case 1: fprintf(Outfile, "\t.byte\t0\n"); break;
    case 4: fprintf(Outfile, "\t.long\t0\n"); break;
    case 8: fprintf(Outfile, "\t.quad\t0\n"); break;
    default:
      for (int i=0; i < size; i++)
        fprintf(Outfile, "\t.byte\t0\n");
  }
}
```


## 변경 사항 테스트

구조체를 파싱하고, 심볼 테이블에 새로운 노드를 저장하며, 전역 구조체 변수를 위한 저장 공간을 생성하는 기능 외에는 새로운 기능이 없다.

다음은 테스트 프로그램 `z.c`이다:

```c
struct fred { int x; char y; long z; };
struct foo { char y; long z; } var1;
struct { int x; };
struct fred var2;
```

이 코드는 두 개의 전역 변수 `var1`과 `var2`를 생성해야 한다. 두 개의 이름이 있는 구조체 타입 `fred`과 `foo`, 그리고 하나의 이름 없는 구조체를 생성한다. 세 번째 구조체는 구조체와 연결된 변수가 없기 때문에 오류(또는 적어도 경고)를 발생시켜야 한다. 이 경우 구조체 자체가 쓸모없다.

위 구조체들의 멤버 오프셋과 구조체 크기를 출력하기 위해 테스트 코드를 추가했고, 결과는 다음과 같다:

```
Offset for fred.x is 0
Offset for fred.y is 4
Offset for fred.z is 8
Size of struct fred is 13

Offset for foo.y is 0
Offset for foo.z is 4
Size of struct foo is 9

Offset for struct.x is 0
Size of struct struct is 4
```

마지막으로, `./cwj -S z.c`를 실행하면 다음과 같은 어셈블리 출력을 얻는다:

```
        .globl  var1
var1:   .byte   0       // 9바이트
        ...

        .globl  var2    // 13바이트
var2:   .byte   0
        ...
```


## 결론 및 다음 단계

이번 파트에서는 기존 코드를 상당 부분 수정해야 했다. 단순히 `int 타입`만 다루던 코드를 `int 타입; struct symtable *ctype` 쌍을 다루는 방식으로 변경했다. 앞으로도 이런 작업을 더 많이 해야 할 것이다.

우리는 구조체 정의를 파싱하는 기능과 구조체 변수를 선언하는 기능을 추가했고, 전역 구조체 변수를 위한 공간을 생성할 수 있게 됐다. 현재는 생성한 구조체 변수를 사용할 수는 없지만, 좋은 시작이다. 또한 지역 구조체 변수는 아직 다루지 않았다. 이는 스택과 관련이 있어서 분명 복잡할 것이다.

컴파일러 개발의 다음 단계에서는 '.' 토큰을 파싱하는 코드를 추가해 구조체 변수의 멤버에 접근할 수 있도록 할 계획이다. [다음 단계](../32_Struct_Access_pt1/Readme.md)



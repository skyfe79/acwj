# 40장: 전역 변수 초기화

컴파일러 개발 여정의 이전 파트에서, 우리는 언어에 변수 선언을 추가하기 위한 기반 작업을 시작했다. 이번 파트에서는 전역 스칼라 변수와 배열 변수를 구현할 수 있었다.

동시에, 기존의 심볼 테이블 구조가 변수의 크기와 배열 변수의 요소 수를 제대로 처리하지 못한다는 점을 깨달았다. 따라서 이번 파트의 절반은 심볼 테이블을 다루는 코드를 다시 작성하는 데 할애할 것이다.


## 전역 변수 할당 간단 요약

전역 변수 할당에 대한 간단한 요약을 해보자. 아래는 지원하려는 전역 변수 할당 예제들이다:

```c
int x= 2;
char y= 'a';
char *str= "Hello world";
int a[10];
char b[]= { 'q', 'w', 'e', 'r', 't', 'y' };
char c[10]= { 'q', 'w', 'e', 'r', 't', 'y' };   // Zero padded
char *d[]= { "apple", "banana", "peach", "pear" };
```

전역 구조체나 공용체 초기화는 다루지 않는다. 또한, 현재는 `char *` 변수에 NULL을 넣는 것도 다루지 않는다. 나중에 필요하다면 그때 다시 살펴볼 것이다.


## 우리가 가는 곳

여정의 마지막 부분에서, 나는 `decl.c` 파일에 다음과 같이 작성했다:

```c
static struct symtable *symbol_declaration(...) {
  ...
  // 배열 또는 스칼라 변수가 초기화되고 있음
  if (Token.token == T_ASSIGN) {
    ...
        // 배열 초기화
    if (stype == S_ARRAY)
      array_initialisation(sym, type, ctype, class);
    else {
      fatal("Scalar variable initialisation not done yet");
      // 변수 초기화
      // if (class== C_LOCAL)
      // 로컬 변수, 표현식을 파싱
      // expr= binexpr(0);
      // else 더 많은 코드를 작성하라!
    }
  }
  ...
}
```

즉, 나는 코드를 어디에 넣어야 하는지는 알았지만 어떤 코드를 작성해야 하는지는 몰랐다. 먼저, 몇 가지 리터럴 값을 파싱해야 한다...


## 스칼라 변수 초기화

전역 변수에 할당할 수 있는 것은 정수와 문자열 리터럴뿐이므로, 이들을 파싱해야 한다. 각 리터럴의 타입이 할당하려는 변수 타입과 호환되는지 확인해야 한다. 이를 위해 `decl.c`에 새로운 함수를 추가했다:

```c
// 주어진 타입에 대해, 최신 토큰이 해당 타입의 리터럴인지 확인한다.
// 정수 리터럴인 경우 해당 값을 반환한다.
// 문자열 리터럴인 경우 문자열의 레이블 번호를 반환한다.
// 다음 토큰을 스캔하지 않는다.
int parse_literal(int type) {

  // 문자열 리터럴인 경우, 메모리에 저장하고 레이블을 반환한다.
  if ((type == pointer_to(P_CHAR)) && (Token.token == T_STRLIT))
    return(genglobstr(Text));

  if (Token.token == T_INTLIT) {
    switch(type) {
      case P_CHAR: if (Token.intvalue < 0 || Token.intvalue > 255)
                     fatal("정수 리터럴 값이 char 타입에 비해 너무 큽니다");
      case P_INT:
      case P_LONG: break;
      default: fatal("타입 불일치: 정수 리터럴 vs. 변수");
    }
  } else
    fatal("정수 리터럴 값을 기대합니다");
  return(Token.intvalue);
}
```

첫 번째 IF 문은 다음과 같은 코드가 가능하도록 한다:

```c
char *str= "Hello world";
```

이 경우 문자열이 저장된 주소의 레이블 번호를 반환한다.

정수 리터럴의 경우, `char` 변수에 할당할 때 범위를 확인한다. 다른 토큰 타입이 오면 치명적인 오류를 발생시킨다.


## 심볼 테이블 구조 변경

위 함수는 파싱하는 리터럴 타입에 관계없이 항상 정수를 반환한다. 이제 각 변수의 심볼 엔트리에서 이 값을 저장할 위치가 필요하다. 그래서 `defs.h` 파일의 심볼 엔트리 구조에 다음 필드를 추가하거나 수정했다:

```c
// 심볼 테이블 구조
struct symtable {
  ...
  int size;              // 이 심볼의 총 바이트 크기
  int nelems;            // 함수: 매개변수 수. 배열: 요소 수
  ...
  int *initlist;         // 초기값 리스트
  ...
};
```

단일 초기값을 가진 스칼라 변수나 여러 초기값을 가진 배열의 경우, `nelems`에 요소 수를 저장하고 `initlist`에 정수 값 리스트를 연결한다. 이제 스칼라 변수에 값을 할당하는 경우를 살펴보자.


## 스칼라 변수 할당

`scalar_declaration()` 함수는 다음과 같이 수정된다:

```c
static struct symtable *scalar_declaration(...) {
  ...
    // 변수가 초기화되는 경우
  if (Token.token == T_ASSIGN) {
    // 전역 또는 지역 변수만 가능
    if (class != C_GLOBAL && class != C_LOCAL)
      fatals("변수를 초기화할 수 없음", varname);
    scan(&Token);

    // 전역 변수는 리터럴 값으로 할당해야 함
    if (class == C_GLOBAL) {
      // 변수를 위한 초기 값 생성 및 파싱
      sym->initlist= (int *)malloc(sizeof(int));
      sym->initlist[0]= parse_literal(type);
      scan(&Token);
    }                           // 아직 else 코드는 없음, 곧 추가
  }

  // 전역 공간 생성
  if (class == C_GLOBAL)
    genglobsym(sym);

  return (sym);
}
```

할당이 전역 또는 지역 컨텍스트에서만 발생할 수 있도록 보장하고, '=' 토큰을 건너뛴다. `initlist`를 정확히 하나로 설정하고, 이 변수의 타입과 함께 `parse_literal()`을 호출하여 리터럴 값(또는 문자열의 레이블 번호)을 얻는다. 그런 다음 리터럴 값을 건너뛰어 다음 토큰(',' 또는 ';')으로 이동한다.

이전에는 `addglob()`을 사용해 `sym` 심볼 테이블 엔트리를 생성하고, 요소의 개수를 하나로 설정했다. 이 변경 사항은 곧 다룰 예정이다.

이제 `genglobsym()` 호출을 (이전에는 `addglob()`에 있었음) 여기로 옮기고, `sym` 엔트리에 초기 값이 저장될 때까지 기다린다. 이렇게 하면 방금 파싱한 리터럴이 메모리 내 변수 저장소에 올바르게 배치된다.


### 스칼라 초기화 예제

간단한 예시를 살펴보자:

```c
int x= 5;
char *y= "Hello";
```

위 코드는 다음과 같은 어셈블리 코드를 생성한다:

```
        .globl  x
x:
        .long   5

L1:
        .byte   72
        .byte   101
        .byte   108
        .byte   108
        .byte   111
        .byte   0

        .globl  y
y:
        .quad   L1
```


## 심볼 테이블 코드 변경 사항

배열 초기화 파싱을 다루기 전에, 먼저 심볼 테이블 코드의 변경 사항을 살펴보자. 이전에 언급했듯이, 기존 코드는 변수의 크기나 배열의 요소 수를 제대로 처리하지 못했다. 이를 해결하기 위해 적용한 변경 사항을 자세히 알아보자.

먼저, 버그 수정이 필요했다. `types.c` 파일에서 다음과 같이 수정했다:

```c
// 타입이 int 타입인지 확인
// 모든 크기의 int 타입에 대해 true를 반환, 그 외에는 false
int inttype(int type) {
  return (((type & 0xf) == 0) && (type >= P_CHAR && type <= P_LONG));
}
```

이전에는 `P_CHAR`에 대한 검사가 없어 `void` 타입이 정수 타입으로 처리되는 문제가 있었다. 이제는 이를 수정했다.

`sym.c`에서는 각 변수가 다음 두 필드를 가지도록 변경했다:

```c
  int size;                     // 이 심볼의 총 바이트 크기
  int nelems;                   // 함수: 매개변수 수. 배열: 요소 수
```

나중에 `sizeof()` 연산자에서 `size` 필드를 사용할 것이다. 이제 전역 또는 로컬 심볼 테이블에 심볼을 추가할 때 두 필드를 모두 설정해야 한다.

`newsym()` 함수와 `sym.c`의 모든 `addXX()` 함수들은 이제 `size` 인자 대신 `nelems` 인자를 받는다. 스칼라 변수의 경우 이 값은 1로 설정된다. 배열의 경우, 리스트의 요소 수로 설정된다. 함수의 경우, 함수 매개변수의 수로 설정된다. 그 외의 심볼 테이블에서는 이 값이 사용되지 않는다.

이제 `newsym()` 함수에서 `size` 값을 계산한다:

```c
  // 포인터와 정수 타입의 경우 심볼의 크기를 설정
  // 구조체와 공용체 선언은 직접 이 값을 설정
  if (ptrtype(type) || inttype(type))
    node->size = nelems * typesize(type, ctype);
```

`typesize()` 함수는 `ctype` 포인터를 참조해 구조체나 공용체의 크기를 가져오거나, `genprimsize()` (내부적으로 `cgprimsize()`를 호출)를 호출해 포인터나 정수 타입의 크기를 가져온다.

구조체와 공용체에 대한 주석을 주목하자. 구조체의 크기를 알기 전에 `addstruct()` (내부적으로 `newsym()`을 호출)를 호출할 수 없다:

```c
struct foo {            // 여기서 addglob()을 호출
  int x;
  int y;                // 구조체의 크기를 알기 전에
  int z;
};
```

따라서 `decl.c`의 `composite_declaration()` 함수는 이제 다음과 같이 동작한다:

```c
static struct symtable *composite_declaration(...) {
  ...
  // 복합 타입 생성
  if (type == P_STRUCT)
    ctype = addstruct(Text);
  else
    ctype = addunion(Text);
  ...
  // 멤버 목록 스캔
  while (1) {
    ...
  }

  // 구조체 타입의 노드에 연결
  ctype->member = Membhead;
  ...

  // 복합 타입의 전체 크기 설정
  ctype->size = offset;
  return (ctype);
}
```

요약하면, 심볼 테이블 엔트리의 `size` 필드는 이제 변수의 모든 요소의 크기를 나타내고, `nelems`는 변수의 요소 수를 나타낸다: 배열의 경우 1, 배열이 아닌 경우 양의 정수 값이다.


## 배열 변수 초기화

이제 배열 초기화를 살펴본다. 세 가지 형태의 초기화를 허용한다:

```c
int a[10];                                      // 10개의 0으로 초기화된 요소
char b[]= { 'q', 'w', 'e', 'r', 't', 'y' };     // 6개의 요소
char c[10]= { 'q', 'w', 'e', 'r', 't', 'y' };   // 10개의 요소, 나머지는 0으로 채움
```

하지만 크기가 N으로 선언된 배열에 N개보다 많은 초기화 값을 지정하는 것은 방지한다. `array_declaration()` 함수의 변경 사항을 살펴본다. 이전에는 `array_initialisation()` 함수를 호출하려 했지만, 초기화 코드를 모두 `decl.c` 파일의 `array_declaration()` 함수로 옮기기로 결정했다. 단계별로 진행한다.

```c
// 타입, 이름, 클래스가 주어졌을 때, 배열의 크기(있으면)를 파싱한다.
// 그런 다음 초기화 값을 파싱하고 저장 공간을 할당한다.
// 변수의 심볼 테이블 항목을 반환한다.
static struct symtable *array_declaration(...) {
  int nelems= -1;       // 요소 수가 주어지지 않았다고 가정
  ...
  // '[' 토큰을 건너뛴다
  scan(&Token);

  // 배열 크기가 있는지 확인
  if (Token.token == T_INTLIT) {
    if (Token.intvalue <= 0)
      fatald("Array size is illegal", Token.intvalue);
    nelems= Token.intvalue;
    scan(&Token);
  }

  // ']' 토큰이 있는지 확인
  match(T_RBRACKET, "]");
```

'['와 ']' 토큰 사이에 숫자가 있으면 이를 파싱하고 `nelems`를 해당 값으로 설정한다. 숫자가 없으면 -1로 두어 이를 나타낸다. 또한 숫자가 양수이고 0이 아닌지 확인한다.

```c
    // 배열 초기화
  if (Token.token == T_ASSIGN) {
    if (class != C_GLOBAL)
      fatals("Variable can not be initialised", varname);
    scan(&Token);

    // 다음 왼쪽 중괄호를 얻는다
    match(T_LBRACE, "{");
```

현재는 전역 배열만 처리한다.

```c
#define TABLE_INCREMENT 10

    // 배열에 nelems가 이미 있으면 리스트에 해당 수만큼 요소를 할당한다.
    // 그렇지 않으면 TABLE_INCREMENT로 시작한다.
    if (nelems != -1)
      maxelems= nelems;
    else
      maxelems= TABLE_INCREMENT;
    initlist= (int *)malloc(maxelems *sizeof(int));
```

10개의 정수로 초기 리스트를 생성하거나, 배열이 고정 크기로 주어졌다면 `nelems`만큼 생성한다. 하지만 고정 크기가 없는 배열의 경우 초기화 리스트의 크기를 예측할 수 없다. 따라서 리스트를 확장할 준비가 되어 있어야 한다.

```c
    // 리스트에서 새로운 리터럴 값을 얻는 루프
    while (1) {

      // 다음 값을 추가할 수 있는지 확인한 후 파싱하고 추가한다
      if (nelems != -1 && i == maxelems)
        fatal("Too many values in initialisation list");
      initlist[i++]= parse_literal(type);
      scan(&Token);
```

다음 리터럴 값을 얻고, 배열 크기가 지정된 경우 초기화 값이 그 크기를 초과하지 않도록 한다.

```c
      // 초기 크기가 설정되지 않았고 현재 리스트의 끝에 도달했다면
      // 리스트 크기를 증가시킨다
      if (nelems == -1 && i == maxelems) {
        maxelems += TABLE_INCREMENT;
        initlist= (int *)realloc(initlist, maxelems *sizeof(int));
      }
```

여기서 필요한 경우 초기화 리스트의 크기를 증가시킨다.

```c
      // 오른쪽 중괄호를 만나면 루프를 종료한다
      if (Token.token == T_RBRACE) {
        scan(&Token);
        break;
      }

      // 다음 토큰은 콤마여야 한다
      comma();
    }
```

닫는 오른쪽 중괄호 또는 값을 구분하는 콤마를 파싱한다. 루프를 빠져나오면 값이 포함된 `initlist`가 준비된다.

```c
    // initlist에서 사용되지 않은 요소를 0으로 채운다.
    // 리스트를 심볼 테이블 항목에 연결한다
    for (j=i; j < sym->nelems; j++) initlist[j]=0;
    if (i > nelems) nelems = i;
    sym->initlist= initlist;
  }
```

초기화 리스트의 지정된 크기를 충족하기에 충분한 초기화 값이 제공되지 않았을 수 있으므로, 초기화되지 않은 모든 요소를 0으로 채운다. 여기서 초기화 리스트를 심볼 테이블 항목에 연결한다.

```c
  // 배열의 크기와 요소 수를 설정한다
  sym->nelems= nelems;
  sym->size= sym->nelems * typesize(type, ctype);
  // 전역 공간을 생성한다
  if (class == C_GLOBAL)
    genglobsym(sym);
  return (sym);
}
```

마지막으로 심볼 테이블 항목에서 `nelems`와 `size`를 업데이트한다. 이 작업이 완료되면 `genglobsym()`을 호출하여 배열의 메모리 공간을 생성한다.


## `cgglobsym()` 함수의 변경 사항

예제 배열 초기화의 어셈블리 출력을 살펴보기 전에, `nelems`와 `size`의 변경이 메모리 저장소를 생성하는 코드에 어떤 영향을 미쳤는지 확인해야 한다.

`genglobsym()`은 단순히 `cgglobsym()`을 호출하는 프론트엔드 함수다. 이제 `cg.c` 파일에 있는 이 함수를 살펴보자:

```c
// 전역 심볼을 생성하지만 함수는 생성하지 않음
void cgglobsym(struct symtable *node) {
  int size, type;
  int initvalue;
  int i;

  if (node == NULL)
    return;
  if (node->stype == S_FUNCTION)
    return;

  // 변수의 크기(또는 배열인 경우 요소의 크기)와 타입을 가져옴
  if (node->stype == S_ARRAY) {
    size= typesize(value_at(node->type), node->ctype);
    type= value_at(node->type);
  } else {
    size = node->size;
    type= node->type;
  }
```

현재 배열은 기본 요소 타입에 대한 포인터로 `type`이 설정되어 있다. 이로 인해 다음과 같은 작업이 가능하다:

```c
  char a[45];
  char *b;
  b= a;         // 둘 다 같은 타입이기 때문
```

저장소를 생성하는 측면에서, 요소의 크기를 알아야 하므로 `value_at()`을 호출한다. 스칼라 타입의 경우 `size`와 `type`은 심볼 테이블 항목에 그대로 저장된다.

```c
  // 전역 식별자와 라벨을 생성
  cgdataseg();
  fprintf(Outfile, "\t.globl\t%s\n", node->name);
  fprintf(Outfile, "%s:\n", node->name);
```

이전과 동일하다. 하지만 이제 코드가 달라졌다:

```c
  // 하나 이상의 요소를 위한 공간을 출력
  for (i=0; i < node->nelems; i++) {
  
    // 초기값을 가져옴
    initvalue= 0;
    if (node->initlist != NULL)
      initvalue= node->initlist[i];
  
    // 이 타입을 위한 공간을 생성
    switch (size) {
      case 1:
        fprintf(Outfile, "\t.byte\t%d\n", initvalue);
        break;
      case 4:
        fprintf(Outfile, "\t.long\t%d\n", initvalue);
        break;
      case 8:
        // 문자열 리터럴에 대한 포인터를 생성
        if (node->initlist != NULL && type== pointer_to(P_CHAR))
          fprintf(Outfile, "\t.quad\tL%d\n", initvalue);
        else
          fprintf(Outfile, "\t.quad\t%d\n", initvalue);
        break;
      default:
        for (int i = 0; i < size; i++)
        fprintf(Outfile, "\t.byte\t0\n");
    }
  }
}
```

각 요소에 대해 초기값을 `initlist`에서 가져오거나, 초기화 리스트가 없으면 0을 사용한다. 요소의 크기에 따라 바이트, 롱, 또는 쿼드를 출력한다.

`char *` 타입의 요소인 경우, 초기화 리스트에 문자열 리터럴의 베이스 라벨이 있으므로 정수 리터럴 값 대신 "L%d"(즉, 라벨)를 출력한다.


### 배열 초기화 예제

다음은 배열 초기화의 간단한 예제이다:

```c
int x[4]= { 1, 4, 17 };
```

이 코드는 다음과 같이 생성된다:

```
        .globl  x
x:
        .long   1
        .long   4
        .long   17
        .long   0
```


## 테스트 프로그램

여기서는 테스트 프로그램을 자세히 다루지 않는다. 하지만 `tests/input89.c`부터 `tests/input99.c`까지의 프로그램은 컴파일러가 적절한 초기화 코드를 생성하는지 확인하고, 치명적인 오류를 잡아내는지 검증한다.


## 결론 및 다음 단계

여기까지 정말 많은 작업을 해냈다! 세 걸음 앞으로 가고 한 걸음 뒤로 물러나는 느낌이었다. 하지만 기쁘다. 기존에 사용하던 방식보다 새로운 심볼 테이블 구조가 훨씬 더 직관적이고 합리적이기 때문이다.

컴파일러 개발 여정의 다음 단계에서는 **지역 변수 초기화** 기능을 추가해 볼 예정이다. [다음 단계](../41_Local_Var_Init/Readme.md)



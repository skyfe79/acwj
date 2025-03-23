# 23장: 지역 변수 구현

지금까지 설명한 컴파일러 설계 개념에 따라 스택에 지역 변수를 구현했다. 모든 과정이 순조롭게 진행되었으며, 실제 코드 변경 사항을 아래에 정리한다.


## 심볼 테이블 변경 사항

심볼 테이블의 변경 사항부터 살펴본다. 이는 전역(global)과 지역(local) 두 가지 변수 스코프를 구현하는 데 핵심적인 부분이다. 심볼 테이블 엔트리의 구조는 이제 다음과 같다(`defs.h` 참조):

```c
// 스토리지 클래스
enum {
        C_GLOBAL = 1,           // 전역적으로 보이는 심볼
        C_LOCAL                 // 지역적으로 보이는 심볼
};

// 심볼 테이블 구조체
struct symtable {
  char *name;                   // 심볼의 이름
  int type;                     // 심볼의 기본 타입
  int stype;                    // 심볼의 구조적 타입
  int class;                    // 심볼의 스토리지 클래스
  int endlabel;                 // 함수의 경우, 종료 레이블
  int size;                     // 심볼의 요소 수
  int posn;                     // 지역 변수의 경우, 스택 베이스 포인터로부터의 음수 오프셋
};
```

여기서 `class`와 `posn` 필드가 추가되었다. 이전 부분에서 설명했듯이, `posn`은 음수 값을 가지며 스택 베이스 포인터로부터의 오프셋을 나타낸다. 즉, 지역 변수는 스택에 저장된다. 이번 파트에서는 매개변수는 구현하지 않고, 지역 변수만 다룬다. 또한 심볼이 이제 C_GLOBAL 또는 C_LOCAL로 표시된다는 점에 유의한다.

심볼 테이블의 이름과 인덱스도 변경되었다(`data.h` 참조):

```c
extern_ struct symtable Symtable[NSYMBOLS];     // 전역 심볼 테이블
extern_ int Globs;                              // 다음 사용 가능한 전역 심볼 슬롯 위치
extern_ int Locls;                              // 다음 사용 가능한 지역 심볼 슬롯 위치
```

시각적으로, 전역 심볼은 심볼 테이블의 왼쪽에 저장되며, `Globs`는 다음 사용 가능한 전역 심볼 슬롯을 가리키고, `Locls`는 다음 사용 가능한 지역 심볼 슬롯을 가리킨다.

```
0xxxx......................................xxxxxxxxxxxxNSYMBOLS-1
     ^                                    ^
     |                                    |
   Globs                                Locls
```

`sym.c`에는 기존의 `findglob()`과 `newglob()` 함수 외에, 이제 `findlocl()`과 `newlocl()` 함수가 추가되었다. 이 함수들은 `Globs`와 `Locls` 사이의 충돌을 감지하는 코드를 포함한다:

```c
// 새로운 전역 심볼 슬롯의 위치를 가져오거나, 위치가 부족하면 오류를 발생시킴
static int newglob(void) {
  int p;

  if ((p = Globs++) >= Locls)
    fatal("Too many global symbols");
  return (p);
}

// 새로운 지역 심볼 슬롯의 위치를 가져오거나, 위치가 부족하면 오류를 발생시킴
static int newlocl(void) {
  int p;

  if ((p = Locls--) <= Globs)
    fatal("Too many local symbols");
  return (p);
}
```

이제 심볼 테이블 엔트리의 모든 필드를 설정하는 일반 함수 `updatesym()`이 있다. 이 함수는 각 필드를 하나씩 설정하는 간단한 코드이므로 여기서는 생략한다.

`updatesym()` 함수는 `addglobl()`과 `addlocl()`에서 호출된다. 이 함수들은 먼저 기존 심볼을 찾고, 찾지 못하면 새로운 심볼을 할당한 후, `updatesym()`을 호출해 해당 심볼의 값을 설정한다. 마지막으로, 새로운 함수 `findsymbol()`이 추가되었다. 이 함수는 심볼 테이블의 지역 및 전역 섹션에서 심볼을 검색한다:

```c
// 심볼 s가 심볼 테이블에 있는지 확인한다.
// 슬롯 위치를 반환하거나, 찾지 못하면 -1을 반환한다.
int findsymbol(char *s) {
  int slot;

  slot = findlocl(s);
  if (slot == -1)
    slot = findglob(s);
  return (slot);
}
```

코드의 나머지 부분에서는 기존의 `findglob()` 호출이 `findsymbol()` 호출로 대체되었다.


## 선언문 파싱 변경 사항

전역 변수와 지역 변수 선언을 모두 파싱할 수 있어야 한다. 현재는 두 경우를 파싱하는 코드가 동일하므로, 함수에 플래그를 추가했다:

```c
void var_declaration(int type, int islocal) {
    ...
      // 알려진 배열로 추가
      if (islocal) {
        addlocl(Text, pointer_to(type), S_ARRAY, 0, Token.intvalue);
      } else {
        addglob(Text, pointer_to(type), S_ARRAY, 0, Token.intvalue);
      }
    ...
    // 알려진 스칼라로 추가
    if (islocal) {
      addlocl(Text, type, S_VARIABLE, 0, 1);
    } else {
      addglob(Text, type, S_VARIABLE, 0, 1);
    }
    ...
}
```

현재 컴파일러에는 `var_declaration()`을 호출하는 두 곳이 있다. `decl.c` 파일의 `global_declarations()` 함수에서는 전역 변수 선언을 파싱한다:

```c
void global_declarations(void) {
      ...
      // 전역 변수 선언 파싱
      var_declaration(type, 0);
      ...
}
```

`stmt.c` 파일의 `single_statement()` 함수에서는 지역 변수 선언을 파싱한다:

```c
static struct ASTnode *single_statement(void) {
  int type;

  switch (Token.token) {
    case T_CHAR:
    case T_INT:
    case T_LONG:

      // 변수 선언의 시작
      // 타입을 파싱하고 식별자를 가져온 후
      // 나머지 선언을 파싱
      type = parse_type();
      ident();
      var_declaration(type, 1);
   ...
  }
  ...
}
```


## x86-64 코드 생성기의 변경 사항

언제나 그렇듯이, `cg.c` 파일에 있는 플랫폼별 코드의 많은 `cgXX()` 함수들은 `gen.c` 파일의 `genXX()` 함수로 컴파일러의 다른 부분에 노출된다. 여기서도 마찬가지다. 따라서 `cgXX()` 함수만 언급하더라도, 이에 대응하는 `genXX()` 함수가 있다는 것을 잊지 말아야 한다.

각 지역 변수에 대해 위치를 할당하고 이를 심볼 테이블의 `posn` 필드에 기록해야 한다. 이를 위해 `cg.c`에 새로운 정적 변수와 이를 조작하는 두 개의 함수를 추가한다:

```c
// 스택 베이스 포인터를 기준으로 한 다음 지역 변수의 위치.
// 스택 포인터 정렬을 쉽게 하기 위해 오프셋을 양수로 저장한다.
static int localOffset;
static int stackOffset;

// 새로운 함수를 파싱할 때 지역 변수의 위치를 초기화한다.
void cgresetlocals(void) {
  localOffset = 0;
}

// 다음 지역 변수의 위치를 얻는다.
// isparam 플래그를 사용해 매개변수를 할당한다(아직 XXX는 아님).
int cggetlocaloffset(int type, int isparam) {
  // 오프셋을 최소 4바이트 단위로 감소시키고 스택에 할당한다.
  localOffset += (cgprimsize(type) > 4) ? cgprimsize(type) : 4;
  return (-localOffset);
}
```

현재는 모든 지역 변수를 스택에 할당한다. 각 변수 사이에는 최소 4바이트의 정렬이 이루어진다. 64비트 정수와 포인터의 경우 각각 8바이트가 할당된다.

> 과거에는 멀티바이트 데이터 항목이 메모리에 올바르게 정렬되지 않으면 CPU가 오류를 일으켰다. 그러나 적어도 x86-64에서는 [데이터 항목을 정렬할 필요가 없다](https://lemire.me/blog/2012/05/31/data-alignment-for-speed-myth-or-reality/).

> 하지만 x86-64에서 스택 포인터는 함수 호출 전에 반드시 올바르게 정렬되어야 한다. Agner Fog의 "[Optimizing Subroutines in Assembly Language](https://www.agner.org/optimize/optimizing_assembly.pdf)" 30페이지에 따르면, "CALL 명령어 실행 전에 스택 포인터는 16바이트로 정렬되어야 하며, 함수 진입 시 RSP의 값은 8 modulo 16이어야 한다."

> 이는 함수 프리앰블의 일부로 `%rsp`를 올바르게 정렬된 값으로 설정해야 함을 의미한다.

`cgresetlocals()`는 `function_declaration()`에서 함수 이름을 심볼 테이블에 추가한 후, 지역 변수 선언을 파싱하기 전에 호출된다. 이는 `localOffset`을 다시 0으로 설정한다.

새로운 지역 스칼라 변수나 지역 배열이 파싱될 때 `addlocl()`이 호출된다는 것을 확인했다. `addlocl()`은 새로운 변수의 타입과 함께 `cggetlocaloffset()`을 호출한다. 이는 스택 베이스 포인터로부터 오프셋을 적절한 크기만큼 감소시키고, 이 오프셋을 심볼의 `posn` 필드에 저장한다.

이제 심볼의 스택 베이스 포인터로부터의 오프셋을 얻었으므로, 전역 변수가 아닌 지역 변수에 접근할 때 `%rbp`에 대한 오프셋을 출력하도록 코드 생성기를 수정해야 한다.

따라서 이제 `cgloadlocal()` 함수가 추가되었다. 이 함수는 `cgloadglob()`과 거의 동일하지만, `Symtable[id].name`을 출력하는 `%s(%%rip)` 형식 문자열 대신 `Symtable[id].posn`을 출력하는 `%d(%%rbp)` 형식 문자열로 대체되었다. 실제로 `cg.c`에서 `Symtable[id].posn`을 검색하면 이러한 새로운 지역 변수 참조를 모두 찾을 수 있다.


스택 포인터 업데이트

스택의 위치를 사용하고 있기 때문에, 스택 포인터를 로컬 변수가 저장된 영역 아래로 이동시켜야 한다. 따라서 함수 프리앰블과 포스트앰블에서 스택 포인터를 수정해야 한다:

```c
// 함수 프리앰블 출력
void cgfuncpreamble(int id) {
  char *name = Symtable[id].name;
  cgtextseg();

  // 스택 포인터를 16의 배수로 정렬
  // 이전 값보다 작은 값으로 설정
  stackOffset= (localOffset+15) & ~15;
  
  fprintf(Outfile,
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n"
          "\taddq\t$%d,%%rsp\n", name, name, name, -stackOffset);
}

// 함수 포스트앰블 출력
void cgfuncpostamble(int id) {
  cglabel(Symtable[id].endlabel);
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", stackOffset);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```

`localOffset`이 음수라는 점을 기억해야 한다. 따라서 함수 프리앰블에서는 음수 값을 더하고, 함수 포스트앰블에서는 음수의 음수 값을 더한다.


## 변경 사항 테스트

컴파일러에 지역 변수를 추가하는 주요 변경 사항을 적용했다. 테스트 프로그램 `tests/input25.c`는 스택에 지역 변수를 저장하는 방식을 보여준다:

```c
int a; int b; int c;

int main()
{
  char z; int y; int x;
  x= 10;  y= 20; z= 30;
  a= 5;   b= 15; c= 25;
}
```

주석이 추가된 어셈블리 출력은 다음과 같다:

```
        .data
        .globl  a
a:      .long   0                       # 전역 변수 3개
        .globl  b
b:      .long   0
        .globl  c
c:      .long   0

        .text
        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp
        addq    $-16,%rsp               # 스택 포인터를 16만큼 낮춤
        movq    $10, %r8
        movl    %r8d, -12(%rbp)         # z는 오프셋 -12에 위치
        movq    $20, %r8
        movl    %r8d, -8(%rbp)          # y는 오프셋 -8에 위치
        movq    $30, %r8
        movb    %r8b, -4(%rbp)          # x는 오프셋 -4에 위치
        movq    $5, %r8
        movl    %r8d, a(%rip)           # a는 전역 레이블을 가짐
        movq    $15, %r8
        movl    %r8d, b(%rip)           # b는 전역 레이블을 가짐
        movq    $25, %r8
        movl    %r8d, c(%rip)           # c는 전역 레이블을 가짐
        jmp     L1
L1:
        addq    $16,%rsp                # 스택 포인터를 16만큼 올림
        popq    %rbp
        ret
```

마지막으로 `$ make test`를 실행하면 컴파일러가 모든 이전 테스트를 통과함을 확인할 수 있다.


## 결론과 다음 단계

지역 변수를 구현하는 작업이 까다로울 거라고 생각했지만, 해결책을 설계하면서 생각해보니 예상보다 쉬웠다. 어쩌면 다음 단계가 더 어려울 수도 있겠다는 생각이 든다.

컴파일러 개발 여정의 다음 단계에서는 함수 인자와 매개변수를 컴파일러에 추가해볼 예정이다. 행운을 빈다! [다음 단계](../24_Function_Params/Readme.md)



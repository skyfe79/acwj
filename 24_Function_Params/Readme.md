# 24장: 함수 파라미터

함수 파라미터를 레지스터에서 함수의 스택으로 복사하는 기능을 구현했지만, 아직 인자를 전달하여 함수를 호출하는 기능은 구현하지 않았다.

이전 내용을 복습해 보자. Eli Bendersky의 글 [x86-64 스택 프레임 레이아웃](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)에서 가져온 이미지를 참고하자.

![](../22_Design_Locals/Figs/x64_frame_nonleaf.png)

함수에 전달되는 "값에 의한 호출(call by value)" 인자 중 최대 6개는 '%rdi'부터 '%r9'까지의 레지스터를 통해 전달된다. 6개를 초과하는 인자는 스택에 푸시된다.

함수가 호출되면, 이전 스택 베이스 포인터를 스택에 푸시하고, 스택 베이스 포인터를 스택 포인터와 같은 위치로 이동시킨다. 그런 다음 스택 포인터를 가장 낮은 로컬 변수 위치로 이동시킨다(최소한 이 위치까지는 이동한다).

왜 "최소한"일까? 다른 함수를 호출하기 전에 스택 베이스 포인터가 올바르게 정렬되도록 스택 포인터를 16의 배수로 낮춰야 하기 때문이다.

스택에 푸시된 인자들은 스택 베이스 포인터로부터 양의 오프셋을 가지고 그대로 유지된다. 레지스터로 전달된 인자들은 스택으로 복사되며, 로컬 변수를 위한 공간도 스택에 설정된다. 이들은 스택 베이스 포인터로부터 음의 오프셋을 가진다.

이것이 목표이지만, 먼저 몇 가지 작업을 완료해야 한다.


## 새로운 토큰과 스캐닝

ANSI C에서 함수 선언은 타입과 변수명을 쉼표로 구분한 목록으로 이루어진다. 예를 들면 다음과 같다.

```c
int function(int x, char y, long z) { ... }
```

따라서 새로운 토큰인 T_COMMA가 필요하며, 이를 읽기 위해 렉시컬 스캐너를 수정해야 한다. `scan.c` 파일에서 `scan()` 함수의 변경 사항은 여러분이 직접 확인하길 바란다.


## 새로운 스토리지 클래스

컴파일러 작성 과정의 마지막 부분에서, 전역 변수와 지역 변수를 모두 지원하기 위해 심볼 테이블을 변경한 내용을 설명했다. 전역 변수는 테이블의 한쪽 끝에, 지역 변수는 다른 쪽 끝에 저장한다. 이제 함수 매개변수를 추가할 차례다.

`defs.h` 파일에 새로운 스토리지 클래스 정의를 추가했다:

```c
// 스토리지 클래스
enum {
        C_GLOBAL = 1,           // 전역적으로 접근 가능한 심볼
        C_LOCAL,                // 지역적으로 접근 가능한 심볼
        C_PARAM                 // 지역적으로 접근 가능한 함수 매개변수
};
```

이 매개변수들은 심볼 테이블의 어디에 위치할까? 사실 동일한 매개변수가 전역과 지역 양쪽 끝에 모두 나타난다.

전역 심볼 리스트에서는 함수의 심볼을 먼저 C_GLOBAL, S_FUNCTION 항목으로 정의한다. 그런 다음, 연속된 항목으로 모든 매개변수를 C_PARAM으로 표시해 정의한다. 이는 함수의 *프로토타입*을 의미한다. 나중에 함수를 호출할 때, 인수 목록과 매개변수 목록을 비교해 일치하는지 확인할 수 있다.

동시에, 동일한 매개변수 목록이 지역 심볼 리스트에도 C_LOCAL 대신 C_PARAM으로 저장된다. 이를 통해 외부에서 전달된 변수와 직접 선언한 변수를 구분할 수 있다.


## 파서 수정

이번 단계에서는 함수 선언만 다룬다. 이를 위해 파서를 수정해야 한다. 함수의 타입, 이름, 그리고 여는 괄호 '('를 파싱한 후, 매개변수를 찾기 시작한다. 각 매개변수는 일반 변수 선언 문법을 따르지만, 세미콜론으로 끝나지 않고 콤마로 구분된다.

이전에 `decl.c` 파일의 `var_declaration()` 함수는 변수 선언의 끝에서 T_SEMI 토큰을 스캔했다. 이제 이 부분은 `var_declaration()`을 호출하는 상위 함수로 옮겼다.

새로운 함수 `param_declaration()`을 추가했다. 이 함수는 여는 괄호 뒤에 오는 (0개 이상의) 매개변수 목록을 읽는 역할을 한다:

```c
// param_declaration: <null>
//           | variable_declaration
//           | variable_declaration ',' param_declaration
//
// 함수 이름 뒤의 괄호 안에 있는 매개변수를 파싱한다.
// 심볼 테이블에 심볼로 추가하고 매개변수의 개수를 반환한다.
static int param_declaration(void) {
  int type;
  int paramcnt=0;

  // 닫는 괄호가 나올 때까지 반복
  while (Token.token != T_RPAREN) {
    // 타입과 식별자를 가져와
    // 심볼 테이블에 추가
    type = parse_type();
    ident();
    var_declaration(type, 1, 1);
    paramcnt++;

    // 이 시점에서 ',' 또는 ')'가 있어야 함
    switch (Token.token) {
      case T_COMMA: scan(&Token); break;
      case T_RPAREN: break;
      default:
        fatald("매개변수 목록에서 예상치 못한 토큰", Token.token);
    }
  }

  // 매개변수 개수 반환
  return(paramcnt);
}
```

`var_declaration()` 함수에 전달되는 두 개의 '1' 인자는 이 변수가 로컬 변수이면서 동시에 매개변수 선언임을 나타낸다. 그리고 `var_declaration()` 함수 내부에서는 이제 다음과 같이 처리한다:

```c
    // 알려진 스칼라로 추가하고
    // 어셈블리에서 공간을 생성
    if (islocal) {
      if (addlocl(Text, type, S_VARIABLE, isparam, 1)==-1)
       fatals("중복된 로컬 변수 선언", Text);
    } else {
      addglob(Text, type, S_VARIABLE, 0, 1);
    }
```

이전 코드는 중복된 로컬 변수 선언을 허용했지만, 이제는 스택이 필요 이상으로 커지는 문제를 방지하기 위해 중복 선언을 치명적인 오류로 처리한다.


## 심볼 테이블 변경 사항

이전에 매개변수가 심볼 테이블의 전역과 지역 양쪽에 배치된다고 언급했지만, 위 코드에서는 `addlocl()` 호출만 보인다. 그렇다면 무슨 일이 일어나고 있는 걸까?

`addlocal()`을 수정하여 매개변수를 전역 끝에도 추가하도록 변경했다:

```c
int addlocl(char *name, int type, int stype, int isparam, int size) {
  int localslot, globalslot;
  ...
  localslot = newlocl();
  if (isparam) {
    updatesym(localslot, name, type, stype, C_PARAM, 0, size, 0);
    globalslot = newglob();
    updatesym(globalslot, name, type, stype, C_PARAM, 0, size, 0);
  } else {
    updatesym(localslot, name, type, stype, C_LOCAL, 0, size, 0);
  }
```

매개변수에 대해 심볼 테이블의 지역 슬롯뿐만 아니라 전역 슬롯도 얻는다. 그리고 둘 다 C_LOCAL이 아니라 C_PARAM으로 표시된다.

전역 끝에 C_GLOBAL이 아닌 심볼이 포함되게 되었으므로, 전역 심볼을 검색하는 코드를 수정해야 한다:

```c
// 심볼 s가 전역 심볼 테이블에 있는지 확인한다.
// 슬롯 위치를 반환하거나, 찾지 못하면 -1을 반환한다.
// C_PARAM 항목은 건너뛴다
int findglob(char *s) {
  int i;

  for (i = 0; i < Globs; i++) {
    if (Symtable[i].class == C_PARAM) continue;
    if (*s == *Symtable[i].name && !strcmp(s, Symtable[i].name))
      return (i);
  }
  return (-1);
}
```


## x86-64 코드 생성기 변경 사항

함수 파라미터를 파싱하고 심볼 테이블에 기록하는 부분은 여기까지다. 이제는 스택에 위치한 인자들을 레지스터에서 복사하고, 새로운 스택 베이스 포인터와 스택 포인터를 설정하는 적절한 함수 프리앰블을 생성해야 한다.

이전 부분에서 `cgresetlocals()`를 작성한 후, `cgfuncpreamble()`을 호출할 때 스택 오프셋을 리셋할 수 있다는 사실을 깨달았다. 그래서 이 함수를 제거했다. 또한, 새로운 로컬 변수의 오프셋을 계산하는 코드는 `cg.c`에서만 보이면 되므로, 다음과 같이 이름을 변경했다:

```c
// 스택 베이스 포인터를 기준으로 한 다음 로컬 변수의 위치.
// 스택 포인터 정렬을 쉽게 하기 위해 오프셋을 양수로 저장한다.
static int localOffset;
static int stackOffset;

// 새로운 로컬 변수의 위치를 생성한다.
static int newlocaloffset(int type) {
  // 오프셋을 최소 4바이트씩 감소시키고 스택에 할당한다.
  localOffset += (cgprimsize(type) > 4) ? cgprimsize(type) : 4;
  return (-localOffset);
}
```

또한, 음수 오프셋을 계산하는 대신 양수 오프셋을 계산하도록 변경했다. 이렇게 하면 계산이 더 쉽다. 여전히 반환 값은 음수 오프셋이다.

인자 값을 저장할 여섯 개의 새로운 레지스터가 있으므로, 이들을 어딘가에 명명해야 한다. 레지스터 이름 목록을 다음과 같이 확장했다:

```c
#define NUMFREEREGS 4
#define FIRSTPARAMREG 9         // 첫 번째 파라미터 레지스터의 위치
static int freereg[NUMFREEREGS];
static char *reglist[] =
  { "%r10", "%r11", "%r12", "%r13", "%r9", "%r8", "%rcx", "%rdx", "%rsi",
"%rdi" };
static char *breglist[] =
  { "%r10b", "%r11b", "%r12b", "%r13b", "%r9b", "%r8b", "%cl", "%dl", "%sil",
"%dil" };
static char *dreglist[] =
  { "%r10d", "%r11d", "%r12d", "%r13d", "%r9d", "%r8d", "%ecx", "%edx",
"%esi", "%edi" };
```

FIRSTPARAMREG는 실제로 각 목록의 마지막 항목 위치다. 이 끝에서 시작하여 뒤로 작업할 것이다.

이제 모든 작업을 수행할 함수인 `cgfuncpreamble()`에 주목해 보자. 코드를 단계별로 살펴보자:

```c
// 함수 프리앰블을 출력한다.
void cgfuncpreamble(int id) {
  char *name = Symtable[id].name;
  int i;
  int paramOffset = 16;         // 푸시된 파라미터는 이 스택 오프셋에서 시작한다.
  int paramReg = FIRSTPARAMREG; // 위 레지스터 목록에서 첫 번째 파라미터 레지스터의 인덱스

  // 텍스트 세그먼트에 출력하고, 로컬 오프셋을 리셋한다.
  cgtextseg();
  localOffset= 0;

  // 함수 시작을 출력하고, %rsp와 %rsp를 저장한다.
  fprintf(Outfile,
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n", name, name, name);
```

먼저 함수를 선언하고, 이전 베이스 포인터를 저장한 후 현재 스택 포인터 위치로 이동한다. 또한, 스택에 있는 인자들은 새로운 베이스 포인터에서 16바이트 위에 위치하며, 첫 번째 파라미터가 있는 레지스터도 알고 있다.

```c
  // 레지스터에 있는 파라미터를 스택으로 복사한다.
  // 최대 여섯 개의 파라미터 레지스터 이후에는 중단한다.
  for (i = NSYMBOLS - 1; i > Locls; i--) {
    if (Symtable[i].class != C_PARAM)
      break;
    if (i < NSYMBOLS - 6)
      break;
    Symtable[i].posn = newlocaloffset(Symtable[i].type);
    cgstorlocal(paramReg--, i);
  }
```

이 루프는 최대 여섯 번 반복되지만, C_PARAM이 아닌 C_LOCAL을 만나면 루프를 종료한다. `newlocaloffset()`을 호출해 스택의 베이스 포인터에서 오프셋을 생성하고, 레지스터 인자를 스택의 이 위치로 복사한다.

```c
  // 나머지에 대해, 파라미터라면 이미 스택에 있다.
  // 로컬 변수라면 스택 위치를 생성한다.
  for (; i > Locls; i--) {
    if (Symtable[i].class == C_PARAM) {
      Symtable[i].posn = paramOffset;
      paramOffset += 8;
    } else {
      Symtable[i].posn = newlocaloffset(Symtable[i].type);
    }
  }
```

각 나머지 로컬 변수에 대해: C_PARAM이라면 이미 스택에 있으므로 심볼 테이블에 기존 위치를 기록한다. C_LOCAL이라면 스택에 새로운 위치를 생성하고 기록한다. 이제 필요한 모든 로컬 변수를 포함한 새로운 스택 프레임이 설정되었다. 남은 작업은 스택 포인터를 16의 배수로 정렬하는 것이다:

```c
  // 스택 포인터를 이전 값보다 작은 16의 배수로 정렬한다.
  stackOffset = (localOffset + 15) & ~15;
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", -stackOffset);
}
```

`stackOffset`은 `cg.c` 전체에서 보이는 정적 변수다. 이 값을 기억해야 하는데, 함수의 포스트앰블에서 스택 값을 우리가 낮춘 만큼 다시 증가시키고, 이전 스택 베이스 포인터를 복원해야 하기 때문이다:

```c
// 함수 포스트앰블을 출력한다.
void cgfuncpostamble(int id) {
  cglabel(Symtable[id].endlabel);
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", stackOffset);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```


## 변경 사항 테스트

컴파일러에 이러한 변경 사항을 적용하면 여러 매개변수를 가진 함수를 선언하고 필요한 지역 변수를 사용할 수 있다. 하지만 아직 컴파일러는 인자를 레지스터로 전달하는 코드를 생성하지 않는다.

이 변경 사항을 테스트하기 위해 매개변수가 있는 함수를 작성하고 컴파일러로 컴파일한다(`input27a.c`):

```c
int param8(int a, int b, int c, int d, int e, int f, int g, int h) {
  printint(a); printint(b); printint(c); printint(d);
  printint(e); printint(f); printint(g); printint(h);
  return(0);
}

int param5(int a, int b, int c, int d, int e) {
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}

int param2(int a, int b) {
  int c; int d; int e;
  c= 3; d= 4; e= 5;
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}

int param0() {
  int a; int b; int c; int d; int e;
  a= 1; b= 2; c= 3; d= 4; e= 5;
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}
```

그리고 별도의 파일 `input27b.c`를 작성하고 `gcc`로 컴파일한다:

```c
#include <stdio.h>
extern int param8(int a, int b, int c, int d, int e, int f, int g, int h);
extern int param5(int a, int b, int c, int d, int e);
extern int param2(int a, int b);
extern int param0();

int main() {
  param8(1,2,3,4,5,6,7,8); puts("--");
  param5(1,2,3,4,5); puts("--");
  param2(1,2); puts("--");
  param0();
  return(0);
}
```

그런 다음 이들을 함께 링크하고 실행 파일이 동작하는지 확인한다:

```
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
./comp1 input27a.c
cc -o out input27b.c out.s lib/printint.c 
./out
1
2
3
4
5
6
7
8
--
1
2
3
4
5
--
1
2
3
4
5
--
1
2
3
4
5
```

동작한다! 느낌이 마치 마법 같아서 느낌표를 붙였다. 이제 `param8()`의 어셈블리 코드를 살펴보자:

```
param8:
        pushq   %rbp                    # %rbp 저장, %rsp 이동
        movq    %rsp, %rbp
        movl    %edi, -4(%rbp)          # 6개 인자를 로컬 스택에 복사
        movl    %esi, -8(%rbp)
        movl    %edx, -12(%rbp)
        movl    %ecx, -16(%rbp)
        movl    %r8d, -20(%rbp)
        movl    %r9d, -24(%rbp)
        addq    $-32,%rsp               # 스택 포인터를 32만큼 낮춤
        movslq  -4(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -4(%rbp) 출력, 즉 a
        movq    %rax, %r11
        movslq  -8(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -8(%rbp) 출력, 즉 b
        movq    %rax, %r11
        movslq  -12(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -12(%rbp) 출력, 즉 c
        movq    %rax, %r11
        movslq  -16(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -16(%rbp) 출력, 즉 d
        movq    %rax, %r11
        movslq  -20(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -20(%rbp) 출력, 즉 e
        movq    %rax, %r11
        movslq  -24(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # -24(%rbp) 출력, 즉 f
        movq    %rax, %r11
        movslq  16(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # 16(%rbp) 출력, 즉 g
        movq    %rax, %r11
        movslq  24(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # 24(%rbp) 출력, 즉 h
        movq    %rax, %r11
        movq    $0, %r10
        movl    %r10d, %eax
        jmp     L1
L1:
        addq    $32,%rsp                # 스택 포인터를 32만큼 올림
        popq    %rbp                    # %rbp 복원 및 반환
        ret
```

`input27a.c`의 다른 함수들은 매개변수 변수와 지역 변수를 모두 가지고 있으므로 생성되는 프리앰블이 올바르다는 것을 알 수 있다(이 테스트를 통과하기에 충분히 잘 동작한다!).


## 결론과 다음 단계

이번 작업을 완료하기까지 몇 차례 시도가 필요했다. 처음에는 지역 심볼 리스트를 잘못된 방향으로 탐색했고, 매개변수 순서를 잘못 설정했다. 또한 Eli Bendersky의 글에서 이미지를 잘못 해석해 이전 베이스 포인터를 덮어쓰는 실수를 저질렀다. 이러한 실수는 오히려 긍정적인 결과를 가져왔는데, 수정된 코드가 원본보다 훨씬 깔끔해졌기 때문이다.

컴파일러 개발의 다음 단계에서는 함수 호출 시 임의의 개수의 인자를 처리할 수 있도록 컴파일러를 수정할 계획이다. 그렇게 되면 `input27a.c`와 `input27b.c`를 `tests/` 디렉토리로 옮길 수 있을 것이다. [다음 단계](../25_Function_Arguments/Readme.md)



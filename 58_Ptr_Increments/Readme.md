# 58장: 포인터 증감 연산자 수정하기

컴파일러 작성 과정의 마지막 부분에서, 포인터 증감 연산자에 문제가 있다고 언급했다. 이제 그 문제가 무엇인지, 그리고 어떻게 해결했는지 살펴보자.

AST 연산자 A_ADD, A_SUBTRACT, A_ASPLUS, A_ASMINUS를 다룰 때, 한 피연산자가 포인터이고 다른 피연산자가 정수 타입인 경우, 정수 값을 포인터가 가리키는 타입의 크기만큼 조정해야 한다는 것을 확인했다. `types.c` 파일의 `modify_type()` 함수에서 이를 처리한다:

```c
  // 덧셈과 뺄셈 연산에서만 크기 조정 가능
  if (op == A_ADD || op == A_SUBTRACT ||
      op == A_ASPLUS || op == A_ASMINUS) {

    // 왼쪽이 정수 타입이고 오른쪽이 포인터 타입이며,
    // 원본 타입의 크기가 1보다 큰 경우: 왼쪽 피연산자 크기 조정
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1)
        return (mkastunary(A_SCALE, rtype, rctype, tree, NULL, rsize));
      else
        return (tree);          // 크기가 1이면 조정 필요 없음
    }
  }
```

그러나 `++` 또는 `--`를 사용할 때, 즉 전위 증감 연산자나 후위 증감 연산자를 사용할 때는 이 크기 조정이 발생하지 않는다. 이 경우, 단순히 A_PREINC, A_PREDEC, A_POSTINC, A_POSTDEC AST 노드를 AST 트리에 추가하고, 코드 생성기가 이를 처리하도록 남겨둔다.

지금까지는 이 문제가 `cg.c` 파일의 `cgloadglob()` 또는 `cgloadlocal()` 함수를 호출해 전역 변수나 지역 변수의 값을 로드할 때 해결되었다. 예를 들어:

```c
int cgloadglob(struct symtable *sym, int op) {
  ...
  if (cgprimsize(sym->type) == 8) {
    if (op == A_PREINC)
      fprintf(Outfile, "\tincq\t%s(%%rip)\n", sym->name);
    ...
    fprintf(Outfile, "\tmovq\t%s(%%rip), %s\n", sym->name, reglist[r]);

    if (op == A_POSTINC)
      fprintf(Outfile, "\tincq\t%s(%%rip)\n", sym->name);
  }
  ...
}
```

여기서 `incq`는 값을 1씩 증가시킨다. 이는 변수가 정수 타입일 때는 문제가 없지만, 포인터 타입인 경우에는 제대로 처리되지 않는다.

또한, `cgloadglob()`와 `cgloadlocal()` 함수는 매우 유사하다. 이 두 함수는 변수에 접근하는 데 사용하는 명령어에서 차이가 있다: 변수가 고정된 위치에 있는지, 아니면 스택 프레임을 기준으로 한 위치에 있는지에 따라 달라진다.


## 문제 해결 과정

한동안 파서가 `modify_type()`과 유사한 AST 트리를 생성하도록 만들 수 있을 거라고 생각했지만, 결국 그만뒀다. 다행히도 `++`와 `--` 연산은 이미 `cgloadglob()`에서 처리되고 있었기 때문에, 이 문제를 여기서 해결하기로 결정했다.

작업을 진행하던 중, `cgloadglob()`과 `cgloadlocal()`을 하나의 함수로 합칠 수 있다는 사실을 깨달았다. 이제 단계별로 해결책을 살펴보자.

```c
// 변수에서 값을 레지스터로 로드한다.
// 레지스터 번호를 반환한다. 만약 연산이 전위 또는 후위 증가/감소라면,
// 해당 동작도 수행한다.
int cgloadvar(struct symtable *sym, int op) {
  int r, postreg, offset=1;

  // 새 레지스터를 할당한다.
  r = alloc_register();

  // 심볼이 포인터라면, 포인터가 가리키는 타입의 크기를
  // 증가 또는 감소 값으로 사용한다. 그렇지 않으면 1이다.
  if (ptrtype(sym->type))
    offset= typesize(value_at(sym->type), sym->ctype);
```

우선 증가 연산에서 +1을 사용한다고 가정한다. 그러나 포인터를 증가시킬 가능성이 있다는 것을 깨닫고, 이를 포인터가 가리키는 타입의 크기로 변경한다.

```c
  // 감소 연산이라면 offset을 음수로 만든다.
  if (op==A_PREDEC || op==A_POSTDEC)
    offset= -offset;
```

이제 감소 연산을 수행할 경우 `offset`이 음수가 된다.

```c
  // 전위 연산이라면
  if (op==A_PREINC || op==A_PREDEC) {
    // 심볼의 주소를 로드한다.
    if (sym->class == C_LOCAL || sym->class == C_PARAM)
      fprintf(Outfile, "\tleaq\t%d(%%rbp), %s\n", sym->st_posn, reglist[r]);
    else
      fprintf(Outfile, "\tleaq\t%s(%%rip), %s\n", sym->name, reglist[r]);
```

여기서 우리의 알고리즘이 이전 코드와 달라진다. 이전 코드는 `incq` 명령어를 사용했는데, 이는 변수 변경을 정확히 1로 제한했다. 이제 변수의 주소를 레지스터에 담았으므로...

```c
    // 해당 주소의 값을 변경한다.
    switch (sym->size) {
      case 1: fprintf(Outfile, "\taddb\t$%d,(%s)\n", offset, reglist[r]); break;
      case 4: fprintf(Outfile, "\taddl\t$%d,(%s)\n", offset, reglist[r]); break;
      case 8: fprintf(Outfile, "\taddq\t$%d,(%s)\n", offset, reglist[r]); break;
    }
  }
```

레지스터를 변수에 대한 포인터로 사용해 offset을 더할 수 있다. 변수의 크기에 따라 다른 명령어를 사용해야 한다.

전위 증가 또는 전위 감소 연산을 수행했다. 이제 변수의 값을 레지스터에 로드할 수 있다:

```c
  // 이제 출력 레지스터에 값을 로드한다.
  if (sym->class == C_LOCAL || sym->class == C_PARAM) {
    switch (sym->size) {
      case 1: fprintf(Outfile, "\tmovzbq\t%d(%%rbp), %s\n", sym->st_posn, reglist[r]); break;
      case 4: fprintf(Outfile, "\tmovslq\t%d(%%rbp), %s\n", sym->st_posn, reglist[r]); break;
      case 8: fprintf(Outfile, "\tmovq\t%d(%%rbp), %s\n", sym->st_posn, reglist[r]);
    }
  } else {
    switch (sym->size) {
      case 1: fprintf(Outfile, "\tmovzbq\t%s(%%rip), %s\n", sym->name, reglist[r]); break;
      case 4: fprintf(Outfile, "\tmovslq\t%s(%%rip), %s\n", sym->name, reglist[r]); break;
      case 8: fprintf(Outfile, "\tmovq\t%s(%%rip), %s\n", sym->name, reglist[r]);
    }
  }
```

심볼이 로컬인지 글로벌인지에 따라, 명명된 위치 또는 프레임 포인터를 기준으로 한 위치에서 값을 로드한다. 심볼의 크기에 따라 결과를 제로 패딩할 명령어를 선택한다.

값은 이제 레지스터 `r`에 안전하게 저장되었다. 하지만 이제 후위 증가 또는 후위 감소 연산을 수행해야 한다. 전위 연산 코드를 재사용할 수 있지만, 새로운 레지스터가 필요하다:

```c
  // 후위 연산이라면 새로운 레지스터를 할당한다.
  if (op==A_POSTINC || op==A_POSTDEC) {
    postreg = alloc_register();

    // 이전과 동일한 코드를 사용하지만, postreg을 사용한다.

    // 레지스터를 해제한다.
    free_register(postreg);
  }

  // 값을 담은 레지스터를 반환한다.
  return(r);
}
```

이제 `cgloadvar()` 코드는 이전 코드만큼 복잡하지만, 포인터 증가를 처리할 수 있다. `tests/input145.c` 테스트 프로그램은 이 새로운 코드가 작동하는지 확인한다:

```c
int list[]= {3, 5, 7, 9, 11, 13, 15};
int *lptr;

int main() {
  lptr= list;
  printf("%d\n", *lptr);
  lptr= lptr + 1; printf("%d\n", *lptr);
  lptr += 1; printf("%d\n", *lptr);
  lptr += 1; printf("%d\n", *lptr);
  lptr -= 1; printf("%d\n", *lptr);
  lptr++   ; printf("%d\n", *lptr);
  lptr--   ; printf("%d\n", *lptr);
  ++lptr   ; printf("%d\n", *lptr);
  --lptr   ; printf("%d\n", *lptr);
}
```


## 모듈로 연산자를 놓친 이유는 무엇일까?

이 문제를 해결한 후, 컴파일러 소스 코드를 다시 컴파일러 자체에 입력했더니 놀랍게도 모듈로 연산자 `%`와 `%=`가 빠져 있었다. 왜 이전에 이 연산자들을 추가하지 않았는지 전혀 기억이 나지 않았다.


### 새로운 토큰과 AST 연산자 추가

컴파일러에 새로운 연산자를 추가하는 작업은 여러 곳에서 변경 사항을 동기화해야 하기 때문에 까다롭다. 어디를 수정해야 하는지 살펴보자. 먼저 `defs.h` 파일에 새로운 토큰을 추가해야 한다:

```c
// 토큰 타입
enum {
  T_EOF,

  // 이항 연산자
  T_ASSIGN, T_ASPLUS, T_ASMINUS,
  T_ASSTAR, T_ASSLASH, T_ASMOD,
  T_QUESTION, T_LOGOR, T_LOGAND,
  T_OR, T_XOR, T_AMPER,
  T_EQ, T_NE,
  T_LT, T_GT, T_LE, T_GE,
  T_LSHIFT, T_RSHIFT,
  T_PLUS, T_MINUS, T_STAR, T_SLASH, T_MOD,
  ...
};
```

여기서 `T_ASMOD`와 `T_MOD`가 새로 추가된 토큰이다. 이제 이에 맞는 AST 연산자를 생성해야 한다:

```c
// AST 노드 타입. 처음 몇 개는 관련 토큰과 일치한다
enum {
  A_ASSIGN = 1, A_ASPLUS, A_ASMINUS, A_ASSTAR,                  //  1
  A_ASSLASH, A_ASMOD, A_TERNARY, A_LOGOR,                       //  5
  A_LOGAND, A_OR, A_XOR, A_AND, A_EQ, A_NE, A_LT,               //  9
  A_GT, A_LE, A_GE, A_LSHIFT, A_RSHIFT,                         // 16
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE, A_MOD,               // 21
  ...
};
```

다음으로, 스캐너를 수정하여 이러한 토큰을 인식할 수 있게 해야 한다. 코드는 보여주지 않겠지만, `scan.c` 파일의 토큰 문자열 테이블에 다음과 같은 변경 사항이 필요하다:

```c
// 디버깅을 위한 토큰 문자열 목록
char *Tstring[] = {
  "EOF", "=", "+=", "-=", "*=", "/=", "%=",
  "?", "||", "&&", "|", "^", "&",
  "==", "!=", ",", ">", "<=", ">=", "<<", ">>",
  "+", "-", "*", "/", "%",
  ...
};
```


### 연산자 우선순위

이제 `expr.c` 파일에서 연산자의 우선순위를 설정해야 한다. 이전에는 T_SLASH가 가장 높은 우선순위를 가졌지만, 이제는 T_MOD로 대체되었다:

```c
// 이진 연산자 토큰을 이진 AST 연산으로 변환한다.
// 토큰과 AST 연산 간 1:1 매핑에 의존한다.
static int binastop(int tokentype) {
  if (tokentype > T_EOF && tokentype <= T_MOD)
    return (tokentype);
  fatals("Syntax error, token", Tstring[tokentype]);
  return (0);                   // -Wall 경고를 피하기 위해
}

// 각 토큰의 연산자 우선순위. defs.h 파일의 토큰 순서와 일치해야 한다.
static int OpPrec[] = {
  0, 10, 10,                    // T_EOF, T_ASSIGN, T_ASPLUS,
  10, 10,                       // T_ASMINUS, T_ASSTAR,
  10, 10,                       // T_ASSLASH, T_ASMOD,
  15,                           // T_QUESTION,
  20, 30,                       // T_LOGOR, T_LOGAND
  40, 50, 60,                   // T_OR, T_XOR, T_AMPER 
  70, 70,                       // T_EQ, T_NE
  80, 80, 80, 80,               // T_LT, T_GT, T_LE, T_GE
  90, 90,                       // T_LSHIFT, T_RSHIFT
  100, 100,                     // T_PLUS, T_MINUS
  110, 110, 110                 // T_STAR, T_SLASH, T_MOD
};

// 이진 연산자인지 확인하고 해당 우선순위를 반환한다.
static int op_precedence(int tokentype) {
  int prec;
  if (tokentype > T_MOD)
    fatals("Token with no precedence in op_precedence:", Tstring[tokentype]);
  prec = OpPrec[tokentype];
  if (prec == 0)
    fatals("Syntax error, token", Tstring[tokentype]);
  return (prec);
}
```


### 코드 생성

이미 `cgdiv()` 함수가 x86-64 명령어를 생성해 나눗셈을 수행한다. `idiv` 명령어 매뉴얼을 보면 다음과 같다:

> idivq S: `%rdx:%rax`를 S로 부호 있는 나눗셈을 수행한다. 몫은 `%rax`에 저장되고, 나머지는 `%rdx`에 저장된다.

따라서 `cgdiv()` 함수를 수정해 AST 연산을 수행하도록 하고, 나눗셈과 나머지(모듈로) 연산을 모두 처리할 수 있게 한다. `cg.c`에 추가된 새로운 함수는 다음과 같다:

```c
// 첫 번째 레지스터를 두 번째 레지스터로 나누거나 모듈로 연산을 수행하고
// 결과가 저장된 레지스터 번호를 반환한다
int cgdivmod(int r1, int r2, int op) {
  fprintf(Outfile, "\tmovq\t%s,%%rax\n", reglist[r1]);
  fprintf(Outfile, "\tcqo\n");
  fprintf(Outfile, "\tidivq\t%s\n", reglist[r2]);
  if (op== A_DIVIDE)
    fprintf(Outfile, "\tmovq\t%%rax,%s\n", reglist[r1]);
  else
    fprintf(Outfile, "\tmovq\t%%rdx,%s\n", reglist[r1]);
  free_register(r2);
  return (r1);
}
```

`tests/input147.c` 파일은 위의 변경 사항이 제대로 동작하는지 확인한다:

```c
#include <stdio.h>

int a;

int main() {
  printf("%d\n", 24 % 9);
  printf("%d\n", 31 % 11);
  a= 24; a %= 9; printf("%d\n",a);
  a= 31; a %= 11; printf("%d\n",a);
  return(0);
}
```


## 왜 링크가 안 될까?

이제 우리 컴파일러는 모든 소스 코드 파일을 파싱할 수 있다. 하지만 링크를 시도하면 `L0` 레이블이 없다는 경고가 발생한다.

조금 더 조사해 보니, `gen.c` 파일의 `genIF()` 함수에서 루프와 스위치의 종료 레이블을 제대로 전달하지 않았던 것이 문제였다. 이 문제는 49번째 줄에서 수정할 수 있다:

```c
// IF 문과 선택적 ELSE 절을 위한 코드를 생성한다.
static int genIF(struct ASTnode *n, int looptoplabel, int loopendlabel) {
  ...
  // 선택적 ELSE 절: 거짓 조건의 복합문과 종료 레이블을 생성한다.
  if (n->right) {
    genAST(n->right, NOLABEL, NOLABEL, loopendlabel, n->op);
    genfreeregs(NOREG);
    cglabel(Lend);
  }
  ...
}
```

이제 `loopendlabel`이 제대로 전달되므로, 아래와 같은 쉘 스크립트(`memake`라고 부름)를 실행할 수 있다:

```
#!/bin/sh
make install

rm *.s *.o

for i in cg.c decl.c expr.c gen.c main.c misc.c \
        opt.c scan.c stmt.c sym.c tree.c types.c
do echo "./cwj -c $i"; ./cwj -c $i ; ./cwj -S $i
done

cc -o cwj0 cg.o decl.o expr.o gen.o main.o misc.o \
        opt.o scan.o stmt.o sym.o tree.o types.o
```

이 과정을 통해 `cwj0`라는 바이너리가 생성된다. 이 바이너리는 컴파일러가 스스로를 컴파일한 결과물이다.

```
$ size cwj0
   text    data     bss     dec     hex filename
 106540    3008      48  109596   1ac1c cwj0

$ file cwj0
cwj0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically
      linked, interpreter /lib64/l, for GNU/Linux 3.2.0, not stripped
```


## 결론과 다음 단계

포인터 증가 문제를 해결하기 위해 상당히 머리를 쥐어짰고, 여러 가지 대안을 고민했다. A_SCALE을 포함한 새로운 AST 트리를 만들려고 시도하기도 했지만, 결국 모두 버리고 `cgloadvar()` 함수를 변경하는 방식으로 해결했다. 이 방법이 훨씬 깔끔했다.

모듈로 연산자는 이론적으로는 간단하게 추가할 수 있었지만, 실제로 모든 부분을 동기화하는 데는 상당히 애를 먹었다. 여기서는 리팩토링을 통해 동기화를 더 쉽게 만드는 여지가 있을 것이다.

그리고 컴파일러가 자신의 소스 코드로부터 생성한 모든 오브젝트 파일을 링크하려고 시도하던 중, 루프와 스위치의 끝 레이블이 제대로 전파되지 않는 문제를 발견했다.

이제 우리의 컴파일러는 모든 소스 코드 파일을 파싱하고, 이를 위한 어셈블리 코드를 생성하며, 링크까지 할 수 있는 단계에 도달했다. 이제는 아마도 가장 고통스러운 단계인 **WDIW**(왜 작동하지 않는가?) 단계에 접어들었다.

여기서는 디버거가 없기 때문에, 많은 어셈블리 출력을 살펴봐야 한다. 어셈블리를 한 줄씩 실행하며 레지스터 값을 확인해야 할 것이다.

컴파일러 작성 여정의 다음 단계에서는 **WDIW** 단계를 시작할 것이다. 이 작업을 효과적으로 진행하기 위한 몇 가지 전략이 필요할 것이다. [다음 단계](../59_WDIW_pt1/Readme.md)



# 4부: 실제 컴파일러 구현

이제 실제 컴파일러를 작성하겠다는 약속을 지킬 때가 되었다. 이번 여정에서는 현재 프로그램의 인터프리터를 x86-64 어셈블리 코드를 생성하는 코드로 교체할 것이다.

## 인터프리터 코드 재검토

컴파일러 작성에 앞서 `interp.c` 파일의 인터프리터 코드를 다시 살펴보는 것이 유익할 것이다:

```c
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  if (n->left) leftval = interpretAST(n->left);
  if (n->right) rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:      return (leftval + rightval);   // 덧셈 연산
    case A_SUBTRACT: return (leftval - rightval);   // 뺄셈 연산
    case A_MULTIPLY: return (leftval * rightval);   // 곱셈 연산
    case A_DIVIDE:   return (leftval / rightval);   // 나눗셈 연산
    case A_INTLIT:   return (n->intvalue);          // 정수 리터럴 값 반환

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);  // 알 수 없는 AST 연산자 오류
      exit(1);
  }
}
```

`interpretAST()` 함수는 주어진 AST(추상 구문 트리)를 깊이 우선 방식으로 순회한다. 먼저 왼쪽 서브트리를 평가하고, 그 다음 오른쪽 서브트리를 평가한다. 마지막으로 현재 트리의 기본에 있는 `op` 값을 사용하여 이 자식 노드들에 대한 연산을 수행한다.

`op` 값이 4가지 수학 연산자 중 하나라면 해당하는 수학 연산을 수행한다. `op` 값이 단순한 정수 리터럴을 나타내는 노드임을 가리킨다면 해당 리터럴 값을 반환한다.

이 함수는 현재 트리에 대한 최종 값을 반환한다. 또한 재귀적으로 동작하기 때문에 전체 트리의 최종 값을 서브트리 단위로 한 번에 하나씩 계산한다.

## 어셈블리 코드 생성으로의 전환

이제 범용적인 어셈블리 코드 생성기를 작성할 것이다. 이 생성기는 CPU별로 특화된 코드 생성 함수들을 호출하는 방식으로 동작한다.

`gen.c` 파일에 구현된 범용 어셈블리 코드 생성기의 내용은 다음과 같다:

```c
// AST를 입력받아 어셈블리 코드를 
// 재귀적으로 생성한다
static int genAST(struct ASTnode *n) {
  int leftreg, rightreg;

  // 좌우 서브트리의 값을 얻는다
  if (n->left) leftreg = genAST(n->left);
  if (n->right) rightreg = genAST(n->right);

  switch (n->op) {
    case A_ADD:      return (cgadd(leftreg,rightreg));
    case A_SUBTRACT: return (cgsub(leftreg,rightreg));
    case A_MULTIPLY: return (cgmul(leftreg,rightreg));
    case A_DIVIDE:   return (cgdiv(leftreg,rightreg));
    case A_INTLIT:   return (cgload(n->intvalue));

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

익숙해 보이지 않는가! 이전과 동일한 깊이 우선 트리 순회 방식을 사용한다.
이번에는 다음과 같이 동작한다:

  + A_INTLIT: 리터럴 값을 레지스터에 로드한다
  + 기타 연산자: 좌측 자식과 우측 자식의 값을 담고 있는 두 레지스터에 대해 수학 연산을 수행한다

이전과 달리 값을 직접 전달하는 대신, `genAST()` 함수는 레지스터 식별자를 주고받는다. 예를 들어 `cgload()` 함수는 값을 레지스터에 로드하고 해당 레지스터의 식별자를 반환한다.

`genAST()` 함수 자체는 현 시점에서 트리의 최종 값을 담고 있는 레지스터의 식별자를 반환한다. 이것이 코드 상단에서 레지스터 식별자를 얻어오는 이유이다:

```c
  if (n->left) leftreg = genAST(n->left);
  if (n->right) rightreg = genAST(n->right);
```

## genAST() 함수 호출하기

`genAST()` 함수는 주어진 표현식의 값만 계산한다. 이 최종 계산 결과를 출력해야 한다. 또한 생성된 어셈블리 코드의 앞부분에 선행 코드(프리앰블)와 뒷부분에 후행 코드(포스트앰블)를 추가해야 한다. 이 작업은 `gen.c`에 있는 다음 함수가 수행한다:

```c
void generatecode(struct ASTnode *n) {
  int reg;

  cgpreamble();
  reg = genAST(n);
  cgprintint(reg);      // 결과가 담긴 레지스터를 정수로 출력한다
  cgpostamble();
}
```

## x86-64 코드 생성기

이제 일반적인 코드 생성기에 대한 설명은 마쳤다. 실제 어셈블리 코드 생성 방법을 살펴보자. 현재 리눅스 플랫폼에서 가장 널리 사용되는 x86-64 CPU를 대상으로 한다. `cg.c` 파일을 열고 내용을 살펴보자.

### 레지스터 할당

모든 CPU는 제한된 수의 레지스터를 갖는다. 정수 리터럴 값과 이에 대한 계산을 위해 레지스터를 할당해야 한다. 값을 사용한 후에는 해당 값을 폐기하고 그 레지스터를 해제할 수 있다. 이렇게 하면 다른 값을 위해 해당 레지스터를 재사용할 수 있다.

레지스터 할당을 처리하는 세 가지 함수가 있다:

 + `freeall_registers()`: 모든 레지스터를 사용 가능한 상태로 설정한다
 + `alloc_register()`: 사용 가능한 레지스터를 할당한다
 + `free_register()`: 할당된 레지스터를 해제한다

코드가 간단하면서도 오류 검사를 포함하고 있어 자세한 설명은 생략한다. 현재는 레지스터가 부족하면 프로그램이 중단된다. 나중에 사용 가능한 레지스터가 없는 상황을 처리할 것이다.

이 코드는 r0, r1, r2, r3과 같은 일반 레지스터를 사용한다. 실제 레지스터 이름은 문자열 배열에 저장되어 있다:

```c
static char *reglist[4]= { "%r8", "%r9", "%r10", "%r11" };
```

이렇게 하면 CPU 아키텍처와 독립적으로 함수를 구현할 수 있다.

### 레지스터 로딩

`cgload()` 함수에서 이루어진다. 레지스터를 할당한 다음 `movq` 명령어로 리터럴 값을 할당된 레지스터에 로드한다.

```c
// 정수 리터럴 값을 레지스터에 로드한다.
// 레지스터 번호를 반환한다
int cgload(int value) {

  // 새 레지스터 획득
  int r= alloc_register();

  // 초기화 코드 출력
  fprintf(Outfile, "\tmovq\t$%d, %s\n", value, reglist[r]);
  return(r);
}
```

### 두 레지스터 더하기

`cgadd()` 함수는 두 개의 레지스터 번호를 받아 이들을 더하는 코드를 생성한다. 결과는 두 레지스터 중 하나에 저장되고, 다른 하나는 이후 사용을 위해 해제된다:

```c
// 두 레지스터를 더하고 결과가 저장된
// 레지스터 번호를 반환한다
int cgadd(int r1, int r2) {
  fprintf(Outfile, "\taddq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

덧셈은 *교환법칙이 성립*하므로 r2를 r1에 더하는 대신 r1을 r2에 더할 수도 있다. 최종 값이 저장된 레지스터의 번호가 반환된다.

### 두 레지스터 곱하기

덧셈과 매우 유사하며, 마찬가지로 *교환법칙이 성립*하므로 어떤 레지스터든 반환할 수 있다:

```c
// 두 레지스터를 곱하고 결과가 저장된
// 레지스터 번호를 반환한다
int cgmul(int r1, int r2) {
  fprintf(Outfile, "\timulq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

### 두 레지스터 빼기

뺄셈은 *교환법칙이 성립하지 않는다*. 순서가 중요하다. 두 번째 레지스터를 첫 번째 레지스터에서 빼므로, 첫 번째 레지스터를 반환하고 두 번째 레지스터를 해제한다:

```c
// 첫 번째 레지스터에서 두 번째 레지스터를 빼고
// 결과가 저장된 레지스터 번호를 반환한다
int cgsub(int r1, int r2) {
  fprintf(Outfile, "\tsubq\t%s, %s\n", reglist[r2], reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### 두 레지스터 나누기

나눗셈 역시 교환법칙이 성립하지 않아 앞서 언급한 주의사항이 적용된다. x86-64에서는 더욱 복잡하다. *피제수*를 r1에서 `%rax`로 로드해야 하며, 이를 `cqo`로 8바이트로 확장해야 한다. 그런 다음 `idivq`가 r2의 제수로 `%rax`를 나누어 *몫*을 `%rax`에 남긴다. 이를 r1이나 r2로 복사한 후 다른 레지스터를 해제해야 한다.

```c
// 첫 번째 레지스터를 두 번째 레지스터로 나누고
// 결과가 저장된 레지스터 번호를 반환한다
int cgdiv(int r1, int r2) {
  fprintf(Outfile, "\tmovq\t%s,%%rax\n", reglist[r1]);
  fprintf(Outfile, "\tcqo\n");
  fprintf(Outfile, "\tidivq\t%s\n", reglist[r2]);
  fprintf(Outfile, "\tmovq\t%%rax,%s\n", reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### 레지스터 출력

x86-64에는 레지스터의 값을 10진수로 출력하는 명령어가 없다. 이 문제를 해결하기 위해 어셈블리 전문에 `printint()` 함수를 포함했다. 이 함수는 레지스터 인자를 받아 `printf()`를 호출하여 10진수로 출력한다.

여기서는 `cgpreamble()` 코드를 보여주지 않는다. 이 코드는 `main()` 함수의 시작 코드도 포함하고 있어 출력 파일을 완전한 프로그램으로 어셈블할 수 있다. 마찬가지로 여기서는 보여주지 않는 `cgpostamble()` 코드는 단순히 `exit(0)`을 호출하여 프로그램을 종료한다.

하지만 `cgprintint()`의 코드는 다음과 같다:

```c
void cgprintint(int r) {
  fprintf(Outfile, "\tmovq\t%s, %%rdi\n", reglist[r]);
  fprintf(Outfile, "\tcall\tprintint\n");
  free_register(r);
}
```

리눅스 x86-64에서는 함수의 첫 번째 인자가 `%rdi` 레지스터에 있어야 하므로, `printint`를 호출하기 전에 레지스터의 값을 `%rdi`로 이동한다.

## 첫 번째 컴파일 수행하기

x86-64 코드 생성기의 핵심 구현이 완성되었다. `main()` 함수에는 출력 파일인 `out.s`를 생성하는 추가 코드가 포함되어 있다. 또한 입력 표현식에 대해 어셈블리 코드가 인터프리터와 동일한 결과를 계산하는지 확인하기 위해 인터프리터 코드도 그대로 유지했다.

이제 컴파일러를 빌드하고 `input01` 파일에 대해 실행해보자:

```shell
$ make
cc -o comp1 -g cg.c expr.c gen.c interp.c main.c scan.c tree.c

$ make test
./comp1 input01
15
cc -o out out.s
./out
15
```

성공이다! 첫 번째로 출력된 15는 인터프리터의 실행 결과이고, 두 번째 15는 어셈블리 코드의 실행 결과이다. 두 결과가 일치하므로 우리가 작성한 코드 생성기가 정상적으로 작동한다는 것을 확인할 수 있다.

## 어셈블리 출력 결과 분석

어셈블리 출력은 정확히 어떤 내용을 담고 있는지 살펴보자. 먼저 입력 파일의 내용은 다음과 같다:

```
2 + 3 * 5 - 8 / 3
```

이 입력에 대한 `out.s` 파일의 내용을 주석과 함께 살펴보면 다음과 같다:

```asm
        .text                           # 프리앰블 코드 시작
.LC0:
        .string "%d\n"                  # printf() 함수용 "%d\n" 문자열
printint:
        pushq   %rbp
        movq    %rsp, %rbp              # 프레임 포인터 설정
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movl    -4(%rbp), %eax          # printint() 함수의 인자값 가져오기
        movl    %eax, %esi
        leaq    .LC0(%rip), %rdi        # "%d\n" 문자열의 포인터 가져오기
        movl    $0, %eax
        call    printf@PLT              # printf() 함수 호출
        nop
        leave                           # 함수 반환
        ret

        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp              # 프레임 포인터 설정
                                        # 프리앰블 코드 끝

        movq    $2, %r8                 # %r8 = 2
        movq    $3, %r9                 # %r9 = 3
        movq    $5, %r10                # %r10 = 5
        imulq   %r9, %r10               # %r10 = 3 * 5 = 15
        addq    %r8, %r10               # %r10 = 2 + 15 = 17
                                        # %r8와 %r9 레지스터 재사용 가능
        movq    $8, %r8                 # %r8 = 8
        movq    $3, %r9                 # %r9 = 3
        movq    %r8,%rax
        cqo                             # 나눗셈을 위해 %rax에 8 로드
        idivq   %r9                     # 3으로 나누기
        movq    %rax,%r8                # 몫(2)을 %r8에 저장
        subq    %r8, %r10               # %r10 = 17 - 2 = 15
        movq    %r10, %rdi              # printint() 함수 호출 준비를 위해 15를 %rdi에 복사
        call    printint                # printint() 함수 호출

        movl    $0, %eax                # 포스트앰블: exit(0) 호출
        popq    %rbp
        ret
```

이것으로 우리는 진정한 컴파일러를 만들었다. 한 언어로 작성된 입력을 받아 다른 언어로 번역하는 프로그램이 완성된 것이다.

아직은 출력된 코드를 기계어로 변환하고 지원 라이브러리와 연결하는 과정을 수동으로 수행해야 한다. 하지만 앞으로는 이 과정을 자동으로 처리하는 코드를 작성할 것이다.

## 결론 및 다음 단계

인터프리터에서 범용 코드 생성기로 전환하는 과정은 간단했지만, 실제 어셈블리 출력을 생성하기 위한 코드를 작성해야 했다. 이를 위해 레지스터 할당 방법을 고민했고, 현재는 단순한 해결책을 적용했다. 또한 `idivq` 명령어와 같은 x86-64 아키텍처의 특이한 부분도 처리해야 했다.

아직 언급하지 않은 중요한 점이 있다. 표현식에 대한 AST를 생성하는 이유가 무엇일까? 프랫(Pratt) 파서에서 '+' 토큰을 만났을 때 `cgadd()` 함수를 직접 호출하고, 다른 연산자도 마찬가지로 처리하면 되지 않을까? 이 질문에 대한 답은 독자 여러분의 생각에 맡기겠다. 앞으로 한두 단계 더 진행하면서 이 문제를 다시 살펴볼 것이다.

컴파일러 개발 여정의 다음 단계에서는 언어에 여러 가지 문장을 추가하여 실제 프로그래밍 언어의 모습에 가깝게 만들 것이다. 

[다음 단계](../05_Statements/Readme.md)

# 60장: 트리플 테스트 통과하기

컴파일러 개발 여정의 이번 단계에서는 컴파일러가 트리플 테스트를 통과하도록 만들어 본다. 어떻게 알았냐고? 방금 컴파일러의 몇 줄을 수정해서 트리플 테스트를 통과시켰기 때문이다. 하지만 원래 코드가 왜 작동하지 않았는지는 아직 모른다.

이번 장은 문제를 추적하고 해결하는 과정을 다룬다. 단서를 모으고, 문제를 추론하고, 수정한 뒤 마침내 컴파일러가 트리플 테스트를 제대로 통과하도록 만든다.

그렇게 되길 바란다!


## 첫 번째 증거

현재 세 가지 컴파일러 바이너리가 있다:

1. Gnu C 컴파일러로 빌드한 `cwj`
2. `cwj` 컴파일러로 빌드한 `cwj0`
3. `cwj0` 컴파일러로 빌드한 `cwj1`

마지막 두 컴파일러는 동일해야 하지만 실제로는 그렇지 않다. 따라서 `cwj0`이 올바른 어셈블리 출력을 생성하지 못하고 있는데, 이는 컴파일러 소스 코드의 결함 때문이다.

이 문제를 어떻게 좁혀나갈 수 있을까? `tests/` 디렉토리에 여러 테스트 프로그램이 있다. `cwj`와 `cwj0`을 모든 테스트에 대해 실행해 보면서 차이가 있는지 확인해 보자.

`tests/input002.c`에서 차이가 발견된다:

```
$ ./cwj -o z tests/input002.c ; ./z
17
$ ./cwj0 -o z tests/input002.c ; ./z
24
```


## 문제가 무엇인가?

`cwj0`가 잘못된 어셈블리 출력을 생성하고 있다. 테스트 소스 코드부터 살펴보자:

```c
void main()
{
  int fred;
  int jim;
  fred= 5;
  jim= 12;
  printf("%d\n", fred + jim);
}
```

여기에는 두 개의 지역 변수 `fred`와 `jim`이 있다. 두 컴파일러가 생성한 어셈블리 코드에는 다음과 같은 차이가 있다:

```
42c42
<       movl    %r10d, -4(%rbp)
---
>       movl    %r10d, -8(%rbp)
51c51
<       movslq  -4(%rbp), %r10
---
>       movslq  -8(%rbp), %r10
```

두 번째 컴파일러가 `fred`의 오프셋을 잘못 계산하고 있다. 첫 번째 컴파일러는 프레임 포인터 아래 `-4`로 정확히 계산하지만, 두 번째 컴파일러는 `-8`로 계산한다.


## 문제의 원인은 무엇인가?

이 오프셋들은 `cg.c` 파일에 있는 `newlocaloffset()` 함수에 의해 계산된다:

```c
// 새로운 지역 변수의 위치를 생성한다.
static int localOffset;
static int newlocaloffset(int size) {
  // 오프셋을 최소 4바이트만큼 감소시키고
  // 스택에 할당한다
  localOffset += (size > 4) ? size : 4;
  return (-localOffset);
}
```

각 함수가 시작될 때 `localOffset`은 0으로 설정된다. 지역 변수를 생성할 때마다 각 변수의 크기를 구하고, 이를 `newlocaloffset()`에 전달한 뒤 오프셋을 반환받는다.

`fred`와 `jim` 지역 변수는 모두 `int` 타입이므로 크기는 4바이트다. 따라서 이들의 오프셋은 각각 `-4`와 `-8`이 되어야 한다.


## 더 많은 증거가 필요하다

`newlocaloffset()` 함수를 별도의 소스 파일인 `z.c`(임시 파일 이름)로 추상화한 후 컴파일해 보자. 소스 파일은 다음과 같다:

```c
static int localOffset=0;
static int newlocaloffset(int size) {
  localOffset += (size > 4) ? size : 4;
  return (-localOffset);
}
```

그리고 여기 주석이 달린 어셈블리 출력 결과가 있다:

```
        .data
localOffset:
        .long   0
        
        .text
newlocaloffset:
        pushq   %rbp                     
        movq    %rsp, %rbp               # 스택과 프레임 포인터 설정
        movl    %edi, -4(%rbp)           # 
        addq    $-16,%rsp               
        movslq  localOffset(%rip), %r10  # localOffset을 %r10에 로드
                                         # += 연산을 준비
        movslq  -4(%rbp), %r11           # size를 %r11에 로드
        movq    $4, %r12                 # 4를 %r12에 로드
        cmpl    %r12d, %r11d             # 비교
        jle     L2                       # size < 4이면 L2로 점프
        movslq  -4(%rbp), %r11
        movq    %r11, %r10               # size를 %r10에 로드
        jmp     L3                       # L3로 점프
L2:
        movq    $4, %r11                 # 아니면 4를
        movq    %r11, %r10               # %r10에 로드
L3:
        addq    %r10, %r10               # += 연산을 수행하며
                                         # localOffset의 캐시된 복사본을 사용
        movl    %r10d, localOffset(%rip) # %r10을 localOffset에 저장
        movslq  localOffset(%rip), %r10
        negq    %r10                     # localOffset을 부호 반전
        movl    %r10d, %eax              # 반환 값 설정
        jmp     L1                      
L1:
        addq    $16,%rsp                 # 스택과 프레임 포인터 복원
        popq    %rbp                     # 
        ret                              # 반환
```

흠, 이 코드는 `localOffset += expression`을 수행하려고 한다. 그리고 `localOffset`의 복사본을 `%r10`에 캐시해 두었다. 하지만 이 표현식 자체도 `%r10`을 사용하기 때문에, `localOffset`의 캐시된 버전이 파괴된다.

특히 `addq %r10, %r10`은 잘못된 명령이다. 이 명령은 두 개의 다른 레지스터를 더해야 한다.


## 트리플 테스트를 속여 통과하기

`newlocaloffset()` 함수의 소스 코드를 다음과 같이 수정하면 트리플 테스트를 통과할 수 있다:

```c
static int newlocaloffset(int size) {
  if (size > 4)
    localOffset= localOffset + size;
  else
    localOffset= localOffset + 4;
  return (-localOffset);
}
```

이제 다음 명령을 실행하면:

```
$ make triple
cc -Wall -o cwj  cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
./cwj    -o cwj0 cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
./cwj0   -o cwj1 cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
size cwj[01]
   text    data     bss     dec     hex filename
 109652    3028      48  112728   1b858 cwj0
 109652    3028      48  112728   1b858 cwj1
```

마지막 두 컴파일러 바이너리가 100% 동일하다. 하지만 이는 원래 `newlocaloffset()` 소스 코드가 작동해야 하지만 실제로는 작동하지 않는다는 사실을 숨긴다.

이미 할당된 `%r10`을 왜 다시 할당하는지 의문이 든다. 이는 불필요한 작업이며, 원래 코드가 의도한 동작과 일치하지 않을 수 있다.


## 문제의 원인

`cg.c` 파일에 `printf()` 문을 추가해 레지스터가 할당되고 해제되는 시점을 확인했다. 다음 어셈블리 코드 라인 이후에 모든 레지스터가 해제되는 것을 발견했다:

```
        movslq  -4(%rbp), %r11           # 크기를 %r11에 로드
        movq    $4, %r12                 # 4를 %r12에 로드
        cmpl    %r12d, %r11d             # 두 값을 비교
        jle     L2                       # 크기가 4보다 작으면 점프
```

이때 `%r10`에는 `localOffset`의 캐시된 복사본이 저장되어 있음에도 불구하고 모든 레지스터가 해제되었다. 이러한 코드를 생성하고 레지스터를 해제하는 함수는 무엇일까? 정답은 다음과 같다:

```c
// 두 레지스터를 비교하고 조건이 거짓이면 점프한다.
int cgcompare_and_jump(int ASTop, int r1, int r2, int label, int type) {
  int size = cgprimsize(type);

  // AST 연산의 범위를 확인
  if (ASTop < A_EQ || ASTop > A_GE)
    fatal("Bad ASTop in cgcompare_and_set()");

  switch (size) {
  case 1:
    fprintf(Outfile, "\tcmpb\t%s, %s\n", breglist[r2], breglist[r1]);
    break;
  case 4:
    fprintf(Outfile, "\tcmpl\t%s, %s\n", dreglist[r2], dreglist[r1]);
    break;
  default:
    fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  }
  fprintf(Outfile, "\t%s\tL%d\n", invcmplist[ASTop - A_EQ], label);
  freeall_registers(NOREG);
  return (NOREG);
}
```

코드를 살펴보면 `r1`과 `r2`는 확실히 해제할 수 있다. 따라서 모든 레지스터를 해제하는 대신 이 두 레지스터만 해제해 보자.

이렇게 수정하면 문제가 해결되고, 모든 회귀 테스트도 통과한다. 하지만 다른 함수에서도 모든 레지스터를 해제하고 있다. 이제 `gdb`를 사용해 실행 과정을 따라가며 문제를 해결할 차례다.


## 문제의 근본 원인

문제의 진짜 원인은 많은 연산이 표현식의 일부가 될 수 있다는 점을 잊었기 때문이다. 표현식의 결과가 사용되거나 버려질 때까지 모든 레지스터를 해제할 수 없다.

`gdb`로 실행 과정을 살펴보니, 삼항 연산자를 처리하는 코드가 레지스터를 해제하고 있었다. 이는 이미 할당된 레지스터가 있는 더 큰 표현식의 일부일 수 있다(`gen.c` 파일 참조):

```c
static int gen_ternary(struct ASTnode *n) {
  ...
  // 조건 코드 생성
  genAST(n->left, Lfalse, NOLABEL, NOLABEL, n->op);
  genfreeregs(NOREG);           // 여기서 문제 발생

  // 두 표현식의 결과를 담을 레지스터 할당
  reg = alloc_register();

  // 참인 경우의 표현식과 거짓 레이블 생성
  // 표현식 결과를 알려진 레지스터로 이동
  // 하지만 결과를 담은 레지스터는 해제하지 않음!
  expreg = genAST(n->mid, NOLABEL, NOLABEL, NOLABEL, n->op);
  cgmove(expreg, reg);
  genfreeregs(reg);             // 여기서 문제 발생

  // 거짓인 경우의 표현식과 종료 레이블 생성
  // 표현식 결과를 알려진 레지스터로 이동
  // 하지만 결과를 담은 레지스터는 해제하지 않음!
  expreg = genAST(n->right, NOLABEL, NOLABEL, NOLABEL, n->op);
  cgmove(expreg, reg);
  genfreeregs(reg);             // 여기서 문제 발생
  ...
}
```

`cg.c` 파일을 살펴보면, 더 이상 사용되지 않는 레지스터를 해제하는 함수들이 있다. 따라서 조건 코드 생성 직후의 `genfreeregs()` 호출을 제거해도 된다.

다음으로, 참인 경우의 표현식 값을 삼항 연산 결과를 담을 레지스터로 이동한 후에는 `expreg`를 해제할 수 있다. 거짓인 경우의 표현식 값도 마찬가지다.

이를 위해 `cg.c` 파일에 있던 이전의 정적 함수를 전역 함수로 변경하고 이름을 바꿨다:

```c
// 사용 가능한 레지스터 목록에 레지스터를 반환한다.
// 이미 목록에 있는지 확인한다.
void cgfreereg(int reg) { ... }
```

이제 `gen.c` 파일의 삼항 연산 처리 코드를 다음과 같이 수정할 수 있다:

```c
static int gen_ternary(struct ASTnode *n) {
  ...
    // 조건 코드 생성 후
  // 거짓 레이블로 점프
  genAST(n->left, Lfalse, NOLABEL, NOLABEL, n->op);

  // 두 표현식의 결과를 담을 레지스터 할당
  reg = alloc_register();

  // 참인 경우의 표현식과 거짓 레이블 생성
  // 표현식 결과를 알려진 레지스터로 이동
  expreg = genAST(n->mid, NOLABEL, NOLABEL, NOLABEL, n->op);
  cgmove(expreg, reg);
  cgfreereg(expreg);
  ...
  // 거짓인 경우의 표현식과 종료 레이블 생성
  // 표현식 결과를 알려진 레지스터로 이동
  expreg = genAST(n->right, NOLABEL, NOLABEL, NOLABEL, n->op);
  cgmove(expreg, reg);
  cgfreereg(expreg);
  ...
}
```

이 변경 사항을 적용한 후, 컴파일러는 여러 테스트를 통과한다:

  + 삼중 테스트: `$ make triple`
  + 컴파일러를 한 번 더 컴파일하는 사중 테스트:

```
$ make quad
...
./cwj  -o cwj0 cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
./cwj0 -o cwj1 cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
./cwj1 -o cwj2 cg.c decl.c expr.c gen.c main.c misc.c opt.c scan.c stmt.c sym.c tree.c types.c
size cwj[012]
   text    data     bss     dec     hex filename
 109636    3028      48  112712   1b848 cwj0
 109636    3028      48  112712   1b848 cwj1
 109636    3028      48  112712   1b848 cwj2
```

  + Gnu C로 컴파일한 컴파일러로 회귀 테스트: `$ make test`
  + 자체 컴파일러로 컴파일한 컴파일러로 회귀 테스트: `$ make test0`
  
이 결과는 매우 만족스럽다.


## 결론과 다음 단계

이 여정의 원래 목표인 자기 컴파일 컴파일러를 작성하는 데 성공했다. 60개의 파트, 5,700줄의 코드, 149개의 회귀 테스트, 그리고 *Readme* 파일에 108,000단어를 기록하며 이 작업을 마무리했다.

하지만 이 여정이 여기서 끝나야 하는 것은 아니다. 이 컴파일러를 더 생산 환경에 적합하게 만들기 위해 추가로 해야 할 작업이 많다. 그동안 약 두 달 동안 간헐적으로 작업해왔기 때문에, 이제는 잠시 휴식을 취해도 될 것 같다.

다음 컴파일러 작성 여정에서는 이 컴파일러로 더 할 수 있는 일들을 정리할 예정이다. 내가 이 중 몇 가지를 진행할 수도 있고, 여러분이 도전해볼 수도 있다. [다음 단계](../61_What_Next/Readme.md)



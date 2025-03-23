# 파트 14: ARM 어셈블리 코드 생성하기

이번 파트에서는 컴파일러를 [Raspberry Pi 4](https://en.wikipedia.org/wiki/Raspberry_Pi)의 ARM CPU로 포팅했다.

이 섹션을 시작하기 전에 몇 가지 말씀드리자면, 나는 MIPS 어셈블리 언어에 대해 꽤 잘 알고 있지만, 이 여정을 시작했을 때는 x86-32 어셈블리 언어에 대해 조금만 알고 있었고, x86-64나 ARM 어셈블리 언어에 대해서는 전혀 알지 못했다.

이 과정에서 나는 다양한 C 컴파일러를 사용해 예제 C 프로그램을 어셈블리어로 컴파일하여 어떤 종류의 어셈블리 코드가 생성되는지 확인해왔다. 이번에도 같은 방법으로 이 컴파일러의 ARM 출력 코드를 작성했다.


## 주요 차이점

첫째, ARM은 RISC CPU이고 x86-64는 CISC CPU다. ARM은 x86-64에 비해 주소 지정 모드가 적다. ARM 어셈블리 코드를 생성할 때 발생하는 다른 흥미로운 제약도 있다. 주요 차이점부터 시작하고, 공통점은 뒤에서 다룰 것이다.


### ARM 레지스터

ARM은 x86-64에 비해 훨씬 더 많은 레지스터를 가지고 있다. 여기서는 `r4`, `r5`, `r6`, `r7` 네 개의 레지스터를 할당하는 데 집중한다. 나중에 `r0`과 `r3`가 다른 용도로 사용되는 것을 확인할 수 있다.


### 전역 변수 처리

x86-64에서는 다음과 같은 방식으로 전역 변수를 선언한다:

```
        .comm   i,4,4        # int 변수
        .comm   j,1,1        # char 변수
```

이후에는 이 변수들에 쉽게 값을 저장하거나 불러올 수 있다:

```
        movb    %r8b, j(%rip)    # j에 저장
        movl    %r8d, i(%rip)    # i에 저장
        movzbl  i(%rip), %r8     # i에서 불러오기
        movzbq  j(%rip), %r8     # j에서 불러오기
```

ARM에서는 프로그램의 끝부분에서 모든 전역 변수를 위한 공간을 수동으로 할당해야 한다:

```
        .comm   i,4,4
        .comm   j,1,1
...
.L2:
        .word i
        .word j
```

이 변수들에 접근하려면 각 변수의 주소를 레지스터에 로드한 후, 그 주소에서 값을 불러와야 한다:

```
        ldr     r3, .L2+0
        ldr     r4, [r3]        # i 불러오기
        ldr     r3, .L2+4
        ldr     r4, [r3]        # j 불러오기
```

변수에 값을 저장하는 것도 비슷하다:

```
        mov     r4, #20
        ldr     r3, .L2+4
        strb    r4, [r3]        # i = 20
        mov     r4, #10
        ldr     r3, .L2+0
        str     r4, [r3]        # j = 10
```

이를 위해 `cgpostamble()` 함수에 .word 테이블을 생성하는 코드가 추가되었다:

```c
  // 전역 변수 출력
  fprintf(Outfile, ".L2:\n");
  for (int i = 0; i < Globs; i++) {
    if (Gsym[i].stype == S_VARIABLE)
      fprintf(Outfile, "\t.word %s\n", Gsym[i].name);
  }
```

이제 각 전역 변수의 `.L2`로부터의 오프셋을 결정해야 한다. KISS 원칙을 따라, 변수의 주소를 `r3`에 로드할 때마다 오프셋을 수동으로 계산한다. 물론 각 오프셋을 한 번 계산해 저장하는 것이 더 효율적이지만, 이는 나중에 개선할 예정이다:

```c
// 변수의 .L2 레이블로부터의 오프셋을 결정한다.
// 이 코드는 비효율적이다.
static void set_var_offset(int id) {
  int offset = 0;
  // 심볼 테이블을 id까지 순회한다.
  // S_VARIABLE을 찾을 때마다 4를 더해
  // 해당 변수에 도달할 때까지 반복한다.

  for (int i = 0; i < id; i++) {
    if (Gsym[i].stype == S_VARIABLE)
      offset += 4;
  }
  // r3에 이 오프셋을 로드한다.
  fprintf(Outfile, "\tldr\tr3, .L2+%d\n", offset);
}
```


### 정수 리터럴 로딩

로드 명령어에서 정수 리터럴의 크기는 11비트로 제한되며, 이는 부호 있는 값으로 보인다. 따라서 큰 정수 리터럴을 단일 명령어에 넣을 수 없다. 이 문제를 해결하기 위해 리터럴 값을 변수처럼 메모리에 저장한다. 이전에 사용한 리터럴 값들을 리스트로 유지한다. 그리고 포스트앰블에서 `.L3` 라벨 뒤에 이 값들을 출력한다. 변수와 마찬가지로 이 리스트를 순회하여 리터럴의 `.L3` 라벨로부터의 오프셋을 결정한다:

```c
// 큰 정수 리터럴 값을 메모리에 저장해야 한다.
// 포스트앰블에서 출력할 리스트를 유지한다
#define MAXINTS 1024
int Intlist[MAXINTS];
static int Intslot = 0;

// 큰 정수 리터럴의 .L3 라벨로부터의 오프셋을 결정한다.
// 정수가 리스트에 없으면 추가한다.
static void set_int_offset(int val) {
  int offset = -1;

  // 이미 리스트에 있는지 확인
  for (int i = 0; i < Intslot; i++) {
    if (Intlist[i] == val) {
      offset = 4 * i;
      break;
    }
  }

  // 리스트에 없으면 추가
  if (offset == -1) {
    offset = 4 * Intslot;
    if (Intslot == MAXINTS)
      fatal("set_int_offset()에서 정수 슬롯이 부족합니다");
    Intlist[Intslot++] = val;
  }
  // r3에 이 오프셋을 로드
  fprintf(Outfile, "\tldr\tr3, .L3+%d\n", offset);
}
```


### 함수 프리앰블과 포스트앰블

`int main(int x)` 함수의 프리앰블과 포스트앰블을 살펴보자. 각 명령어가 어떤 역할을 하는지 정리했다.

#### 프리앰블

```
  .text
  .globl        main
  .type         main, %function
  main:         push  {fp, lr}          # 프레임 포인터와 링크 레지스터를 스택에 저장
                add   fp, sp, #4        # 스택 포인터에 4를 더해 프레임 포인터 설정
                sub   sp, sp, #8        # 스택 포인터를 8만큼 낮춰 지역 변수 공간 확보
                str   r0, [fp, #-8]     # 인자 r0를 지역 변수로 저장
```

#### 포스트앰블

```
                sub   sp, fp, #4        # 스택 포인터를 프레임 포인터에서 4만큼 낮춰 복원
                pop   {fp, pc}          # 프레임 포인터와 링크 레지스터를 스택에서 복원
```

이 코드는 함수 호출 시 스택 프레임을 설정하고, 함수 종료 시 스택을 정리하는 과정을 보여준다. 프리앰블은 함수 시작 시 필요한 초기화 작업을 수행하고, 포스트앰블은 함수 종료 시 스택을 정리하며 반환 주소로 이동한다.


### Comparisons Returning 0 or 1

With the x86-64 there's an instruction to set a register to 0 or 1
based on the comparison being true, e.g. `sete`, but then we have to
zero-fill the rest of the register with `movzbq`. With the ARM, we run
two separate instructions which set a register to a value if the condition
we want is true or false e.g.
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
The first instruction is a command to set a register to 1 if the condition
is true, but then we have to zero-fill the rest of the register with `movzbq`. With the ARM, we run
two separate instructions which set a register to a value if the condition
we want is true or false e.g.
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
                movne r4, #1            Set r4 to 1 if values were equal
                movne r4, #0            Set r4 to 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0            Set r4 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1            Set r4 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                mov r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were 0
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1极速运动 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #极速运动 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were 0
```
                movne r4, #1 1 if values were equal
                movne r4, #0 0 if values were not equal
```
                movne r4, #1 1 if values were equal
                mov极速运动


### Data and information

## This is a comparison of the `cgXXX()` operation, any specific type for that
operation, and an example x86-64 and ARM instruction sequence to
perform it.

| Operation(type) | x86-64 Version | ARM Version |
|-----------------|----------------|-------------|
cgloadint() | movq $12, %r8 | mov r4, #13 |
cgloadglob(char) | movzbq foo(%rip), %r | ldr r3, .L2+#4 |
| | ldr r4, [r3] |
cgloadglob(int) | movzbl foo(%rip), %r8 | ldr r3, .L2+#4 |
| | ldr r4, [r] |
cgloadglob(long) | movq foo(%rip), %r8 | ldr r3, .L2+#4 |
| | ldr r4, [r] |
int cgadd() | addq %r8, %r9 | add r4, r4, r5 |
int cgsub() | subq %r8, %r | sub r4, r4, r5 |
int cgmul() | imulq %r8, %r9 | mul r4, r4, r5 |
int cgdiv() | movq %r8, %rax | mov r0, r4 |
| | cqo | mov r1, r5 |
| | idivq %r | bl __aeabi_idiv |
| | movq %rax, %r8 | mov r4, r0 |
cgprintint() | movq %r8, %rdi | mov r0, r4 |
| | call printint | bl printint |
| | nop |
cgcall() | movq %r8, %rdi | mov r0, r4 |
| | call foo | bl foo |
| | movq %rax, %r8 | mov r4, r0 |
cgstorglob(char) | movb %r8, foo(%rip) | ldr r3, .L2+#4 |
| | strb r4, [r3] |
cgstorglob(int) | movl %r8, foo(%rip) | ldr r3, .L2+#4 |
| | str r4, [r] |
cgstorglob(long) | movq %r, %r8 | ldr r3, .L2+#4 |
| | str r4, [r] |
cgcompare_and_set() | cmpq %r8, %r9 | cmp r4, r5 |
| | sete %r8 | moveq r4, #1 |
| | movzbq %r8, %r8 | movne r4, #1 |
cgcompare_and_jump() | cmpq %r8, %r9 | cmp r, r5 |
| | je L2 | beq L2 |
cgreturn(char) | movzbl %r8, %eax | mov r0, r4 |
| | jmp L2 | b L2 |
cgreturn(int) | movl %r8, %eax | mov r0, r4 |
| | jmp L2 | b L2 |
cgreturn(long) | movq %r8, %rax | mov r0, r4 |
| | jmp L2 | b L2 |

### Data and information



## ARM 코드 생성기 테스트

이 단계의 컴파일러를 라즈베리 파이 3 또는 4로 복사하면 다음과 같은 명령어를 실행할 수 있다:

```
$ make armtest
cc -o comp1arm -g -Wall cg_arm.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
cp comp1arm comp1
(cd tests; chmod +x runtests; ./runtests)
input01: OK
input02: OK
input03: OK
input04: OK
input05: OK
input06: OK
input07: OK
input08: OK
input09: OK
input10: OK
input11: OK
input12: OK
input13: OK
input14: OK

$ make armtest14
./comp1 tests/input14
cc -o out out.s lib/printint.c
./out
10
20
30
```


## 결론과 다음 단계

ARM 버전의 코드 생성기 `cg_arm.c`가 모든 테스트 입력을 올바르게 컴파일하도록 만드는 데 약간의 고민이 필요했다. 대체로 직관적이었지만, 아키텍처와 명령어 세트에 익숙하지 않아서 어려움을 겪었다.

이제 이 컴파일러를 3~4개의 레지스터, 2개 정도의 데이터 크기, 그리고 스택(및 스택 프레임)을 가진 플랫폼으로 포팅하는 것은 비교적 쉬울 것이다. 앞으로 `cg.c`와 `cg_arm.c`가 기능적으로 동기화되도록 유지할 계획이다.

다음 단계에서는 언어에 `char` 포인터와 단항 연산자 '*'와 '&'를 추가할 것이다. [다음 단계](../15_Pointers_pt1/Readme.md)



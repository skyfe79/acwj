# 55장: 지연 평가(Lazy Evaluation)

이전 컴파일러 작성 과정에서 다뤘던 `&&`와 `||` 연산자 수정에 대한 내용을 이 장으로 옮겼다. 이전 장은 이미 충분히 길어졌기 때문이다.

그렇다면 원래 구현된 `&&`와 `||` 연산자에는 어떤 문제가 있었을까? C 프로그래머들은 이 연산자들이 **지연 평가**를 수행할 것이라고 기대한다. 즉, `&&`와 `||`의 오른쪽 피연산자는 왼쪽 피연산자의 값만으로 결과를 결정할 수 없을 때만 평가된다.

지연 평가는 포인터가 특정 값을 가리키는지 확인할 때 유용하게 사용된다. 단, 포인터가 실제로 무언가를 가리키고 있을 때만 평가가 이루어져야 한다. `test/input138.c` 파일에 이에 대한 예제가 있다:

```c
  int *aptr;
  ...
  if (aptr && *aptr == 1)
    printf("aptr points at 1\n");
  else
    printf("aptr is NULL or doesn't point at 1\n");
```

`&&` 연산자의 두 피연산자를 모두 평가하고 싶지는 않다. 만약 `aptr`이 NULL이라면, `*aptr == 1` 표현식은 NULL 역참조를 유발해 프로그램을 중단시킬 수 있기 때문이다.


## 문제점

현재 `&&`와 `||` 연산자의 구현은 두 피연산자를 모두 평가한다. `gen.c` 파일의 `genAST()` 함수를 보면 다음과 같다:

```c
  // 왼쪽과 오른쪽 서브 트리의 값을 가져옴
  leftreg = genAST(n->left, NOLABEL, NOLABEL, NOLABEL, n->op);
  rightreg = genAST(n->right, NOLABEL, NOLABEL, NOLABEL, n->op);

  switch (n->op) {
    ...
    case A_LOGOR:
      return (cglogor(leftreg, rightreg));
    case A_LOGAND:
      return (cglogand(leftreg, rightreg));
    ...
  }
```

이 코드를 수정해 두 피연산자를 모두 평가하지 않도록 해야 한다. 대신, 왼쪽 피연산자를 먼저 평가한다. 만약 결과를 결정할 수 있다면, 결과를 설정하는 코드로 점프한다. 그렇지 않다면 오른쪽 피연산자를 평가한다. 다시 결과를 설정하는 코드로 점프한다. 점프하지 않았다면 반대 결과를 가진다.

이 코드는 IF 문의 코드 생성기와 매우 유사하지만, 충분히 다르기 때문에 `gen.c`에 새로운 코드 생성기를 작성했다. 이 코드는 왼쪽과 오른쪽 피연산자를 `genAST()`로 평가하기 전에 호출된다. 코드는 다음과 같다:

```c
// A_LOGAND 또는 A_LOGOR 연산을 위한 코드 생성
static int gen_logandor(struct ASTnode *n) {
  // 두 개의 레이블 생성
  int Lfalse = genlabel();
  int Lend = genlabel();
  int reg;

  // 왼쪽 표현식의 코드를 생성하고 거짓 레이블로 점프
  reg= genAST(n->left, NOLABEL, NOLABEL, NOLABEL, 0);
  cgboolean(reg, n->op, Lfalse);
  genfreeregs(NOREG);
```

왼쪽 피연산자를 평가한다. `&&` 연산을 수행한다고 가정해 보자. 이 결과가 0이라면 `Lfalse`로 점프해 결과를 0(거짓)으로 설정한다. 또한 표현식을 평가한 후 모든 레지스터를 해제한다. 이는 레지스터 할당의 부담을 줄이는 데도 도움이 된다.

```c
  // 오른쪽 표현식의 코드를 생성하고 거짓 레이블로 점프
  reg= genAST(n->right, NOLABEL, NOLABEL, NOLABEL, 0);
  cgboolean(reg, n->op, Lfalse);
  genfreeregs(reg);
```

오른쪽 피연산자에 대해 동일한 작업을 수행한다. 만약 거짓이라면 `Lfalse` 레이블로 점프한다. 점프하지 않았다면 `&&` 결과는 참이어야 한다. `&&`의 경우 이제 다음을 수행한다:

```c
  cgloadboolean(reg, 1);
  cgjump(Lend);
  cglabel(Lfalse);
  cgloadboolean(reg, 0);
  cglabel(Lend);
  return(reg);
}
```

`cgloadboolean()`은 레지스터를 참(인자가 1인 경우) 또는 거짓(인자가 0인 경우)으로 설정한다. x86-64에서는 1과 0이지만, 다른 아키텍처에서 참과 거짓에 대해 다른 레지스터 값을 가질 수 있으므로 이렇게 코딩했다. 위 코드는 `(aptr && *aptr == 1)` 표현식에 대해 다음과 같은 출력을 생성한다:

```
        movq    aptr(%rip), %r10
        test    %r10, %r10              # aptr가 NULL인지 테스트
        je      L38                     # NULL이면 L38로 점프
        movq    aptr(%rip), %r10
        movslq  (%r10), %r10            # *aptr를 %r10에 가져옴
        movq    $1, %r11
        cmpq    %r11, %r10              # *aptr == 1인지 확인
        sete    %r11b
        movzbq  %r11b, %r11
        test    %r11, %r11
        je      L38                     # 아니면 L38로 점프
        movq    $1, %r11                # 둘 다 참이면 결과는 참
        jmp     L39                     # 거짓 코드를 건너뜀
L38:
        movq    $0, %r11                # 하나 또는 둘 다 거짓이면 결과는 거짓
L39:                                    # 나머지 코드 계속 실행
```

`||` 연산을 평가하는 C 코드는 제공하지 않았다. 기본적으로 왼쪽 또는 오른쪽이 참이면 점프해 결과를 참으로 설정한다. 점프하지 않았다면 결과를 거짓으로 설정하고 참 설정 코드를 건너뛴다.


## 변경 사항 테스트

`test/input138.c` 파일에는 AND와 OR 진리표를 출력하는 코드가 포함되어 있다:

```c
  // 일반적인 AND 연산이 동작하는지 확인
  for (x=0; x <= 1; x++)
    for (y=0; y <= 1; y++) {
      z= x && y;
      printf("%d %d | %d\n", x, y, z);
    }

  // 일반적인 OR 연산이 동작하는지 확인
  for (x=0; x <= 1; x++)
    for (y=0; y <= 1; y++) {
      z= x || y;
      printf("%d %d | %d\n", x, y, z);
    }
```

이 코드는 다음과 같은 출력을 생성한다 (공백 추가):

```
0 0 | 0
0 1 | 0
1 0 | 0
1 1 | 1

0 0 | 0
0 1 | 1
1 0 | 1
1 1 | 1
```


## 결론 및 다음 단계

이제 우리는 컴파일러에 `&&`와 `||`에 대한 **지연 평가(lazy evaluation)**를 구현했다. 이 기능은 컴파일러가 자기 자신을 컴파일하기 위해 반드시 필요하다. 사실, 이 시점에서 컴파일러가 자신의 소스 코드를 파싱할 때 유일하게 처리하지 못하는 부분은 **로컬 배열**의 선언과 사용이다. 그래서 다음에는...

컴파일러 작성 여정의 다음 단계에서, **로컬 배열**을 어떻게 선언하고 사용할지에 대해 다룰 예정이다. [다음 단계](../56_Local_Arrays/Readme.md)



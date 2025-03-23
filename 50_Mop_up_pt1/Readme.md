# 50부: 마무리 작업, 1편

이제 정말로 '마무리 작업' 단계에 접어들었다. 이번 파트에서는 컴파일러 개발 과정에서 새로운 주요 기능을 추가하지 않는다. 대신 몇 가지 문제를 수정하고 사소한 기능을 조금 추가할 것이다.


## 연속된 case 문 처리

현재 컴파일러는 다음과 같은 코드를 파싱할 수 없다.

```c
  switch(x) {
    case 1:
    case 2: printf("Hello\n");
  }
```

파서가 ':' 토큰 뒤에 복합문(compound statement)을 기대하기 때문이다. `stmt.c` 파일의 `switch_statement()` 함수에서 이 부분을 처리한다.

```c
        // ':' 토큰을 스캔하고 복합문을 가져옴
        match(T_COLON, ":");
        left= compound_statement(1); casecount++;
        ...
        // 복합문을 왼쪽 자식으로 하는 서브 트리 생성
        casetail->right= mkastunary(ASTop, 0, left, NULL, casevalue);
```

여기서 우리가 원하는 것은 빈 복합문을 허용하여, 복합문이 없는 case가 다음에 존재하는 복합문으로 흘러들어가도록 하는 것이다.

`switch_statement()` 함수에서 다음과 같이 변경한다.

```c
        // ':' 토큰을 스캔하고 casecount를 증가시킴
        match(T_COLON, ":");
        casecount++;

        // 다음 토큰이 T_CASE인 경우, 현재 case는 다음 case로 흘러들어감. 
        // 그렇지 않으면 case 본문을 파싱함.
        if (Token.token == T_CASE) 
          body= NULL;
        else
          body= compound_statement(1);
```

하지만 이는 문제의 절반만 해결한 것이다. 이제 코드 생성 부분에서 NULL 복합문을 처리해야 한다. `gen.c` 파일의 `genSWITCH()` 함수에서 이를 처리한다.

```c
  // 오른쪽 자식 연결 리스트를 순회하며 각 case에 대한 코드 생성
  for (i = 0, c = n->right; c != NULL; i++, c = c->right) {
    ...
    // case 코드 생성. break를 위한 종료 라벨을 전달함.
    // case에 본문이 없으면 다음 본문으로 흘러들어감.
    if (c->left) genAST(c->left, NOLABEL, NOLABEL, Lend, 0);
    genfreeregs(NOREG);
  }
```

이렇게 간단하게 문제를 해결했다. `tests/input123.c` 파일은 이 변경 사항이 제대로 동작하는지 확인하기 위한 테스트 프로그램이다.


## 심볼 테이블 덤프하기

전역 변수 `Text`가 컴파일러에서 보이지 않는 이유를 파악하려고 할 때, `sym.c` 파일에 각 소스 코드 파일의 끝에서 심볼 테이블을 덤프하는 코드를 추가했다. 이 기능을 활성화하려면 `-M` 커맨드라인 인자를 사용한다. 여기서 코드를 자세히 설명하지는 않겠지만, 출력 결과의 예시를 보여준다:

```
Symbols for misc.c
Global
--------
void exit(): global, 1 params
    int status: param, size 4
void _Exit(): global, 1 params
    int status: param, size 4
void *malloc(): global, 1 params
    int size: param, size 4
...
int Line: extern, size 4
int Putback: extern, size 4
struct symtable *Functionid: extern, size 8
char **Infile: extern, size 8
char **Outfile: extern, size 8
char *Text[]: extern, 513 elems, size 513
struct symtable *Globhead: extern, size 8
struct symtable *Globtail: extern, size 8
...
struct mkastleaf *mkastleaf(): global, 4 params
    int op: param, size 4
    int type: param, size 4
    struct symtable *sym: param, size 8
    int intvalue: param, size 4
...
Enums
--------
int (null): enumtype, size 0
int TEXTLEN: enumval, value 512
int (null): enumtype, size 0
int T_EOF: enumval, value 0
int T_ASSIGN: enumval, value 1
int T_ASPLUS: enumval, value 2
int T_ASMINUS: enumval, value 3
int T_ASSTAR: enumval, value 4
int T_ASSLASH: enumval, value 5
...
Typedefs
--------
long size_t: typedef, size 0
char *FILE: typedef, size 0
```


## 배열을 인자로 전달하기

다음과 같은 변경을 했지만, 지금 생각해보니 배열을 처리하는 방식을 완전히 재고해야 할 필요가 있다. 어쨌든... `decl.c`를 컴파일하면 다음과 같은 오류가 발생한다:

```
Unknown variable:Text on line 87 of decl.c
```

이 오류는 심볼 덤프 코드를 작성하게 만든 원인이 되었다. `Text`는 전역 심볼 테이블에 존재하는데, 왜 파서가 이를 찾지 못한다는 오류를 발생시키는 걸까?

그 이유는 `expr.c`의 `postfix()` 함수가 식별자를 찾은 후, 다음 토큰을 확인하기 때문이다. 만약 다음 토큰이 '['라면, 해당 식별자는 배열로 간주한다. '['가 없다면, 식별자는 변수로 간주한다:

```c
  // 변수일 경우. 변수가 존재하는지 확인한다.
  if ((varptr = findsymbol(Text)) == NULL || varptr->stype != S_VARIABLE)
    fatals("Unknown variable", Text);
```

이 로직 때문에 함수에 배열 참조를 인자로 전달할 수 없게 된다. 오류 메시지를 발생시키는 "문제의" 라인은 `decl.c`에 있다:

```c
      type = type_of_typedef(Text, ctype);
```

여기서 `Text`의 베이스 주소를 인자로 전달한다. 하지만 '['가 뒤따르지 않기 때문에, 컴파일러는 이를 스칼라 변수로 간주하고 `Text`라는 스칼라 변수가 없다고 오류를 발생시킨다.

이 문제를 해결하기 위해 S_VARIABLE뿐만 아니라 S_ARRAY도 허용하도록 변경했지만, 이는 더 큰 문제의 시작일 뿐이다. 우리 컴파일러에서 배열과 포인터는 서로 교환 가능해야 하지만, 현재는 그렇지 않다. 이 문제는 다음 부분에서 다룰 예정이다.


## 누락된 연산자 구현

컴파일러 개발 과정에서 21번째 파트부터 이미 토큰과 AST 연산자를 정의해 두었다:

 + <code>&#124;&#124;</code>, T_LOGOR, A_LOGOR
 + `&&`, T_LOGAND, A_LOGAND

그런데 이 연산자들을 실제로 구현하지 않았었다! 이제 이를 구현할 차례이다.

A_LOGAND의 경우 두 표현식을 평가한다. 둘 다 참이면 레지스터에 1을 설정하고, 그렇지 않으면 0을 설정한다. A_LOGOR의 경우 둘 중 하나라도 참이면 레지스터에 1을 설정하고, 그렇지 않으면 0을 설정한다.

`expr.c`의 `binexpr()` 코드는 이미 토큰을 파싱하고 A_LOGOR와 A_LOGAND AST 노드를 생성한다. 따라서 코드 생성기를 수정해야 한다.

`gen.c`의 `genAST()`에는 이제 다음 코드가 추가되었다:

```c
  case A_LOGOR:
    return (cglogor(leftreg, rightreg));
  case A_LOGAND:
    return (cglogand(leftreg, rightreg));
```

이에 대응하는 두 함수가 `cg.c`에 추가되었다. `cg.c` 함수를 살펴보기 전에, 간단한 C 표현식과 생성될 어셈블리 코드를 먼저 확인해 보자.

```c
int x, y, z;
  ...
  z= x || y;
```

이 코드를 컴파일하면 다음과 같은 어셈블리 코드가 생성된다:

```
        movslq  x(%rip), %r10           # x의 값을 로드
        movslq  y(%rip), %r11           # y의 값을 로드
        test    %r10, %r10              # x의 불리언 값을 테스트
        jne     L13                     # 참이면 L13으로 점프
        test    %r11, %r11              # y의 불리언 값을 테스트
        jne     L13                     # 참이면 L13으로 점프
        movq    $0, %r10                # 둘 다 거짓이면 %r10에 0 설정
        jmp     L14                     # L14로 점프
L13:
        movq    $1, %r10                # %r10에 1 설정
L14:
        movl    %r10d, z(%rip)          # 불리언 결과를 z에 저장
```

각 표현식을 테스트하고, 불리언 결과에 따라 점프한 후 출력 레지스터에 0 또는 1을 저장한다. A_LOGAND의 어셈블리 코드도 비슷하지만, 조건부 점프는 `je`(0이면 점프)이고, `movq $0`과 `movq $1`이 서로 바뀌어 있다.

이제 추가 설명 없이 새로운 `cg.c` 함수를 살펴보자:

```c
// 두 레지스터를 논리 OR 연산하고 결과(1 또는 0)를 가진 레지스터를 반환
int cglogor(int r1, int r2) {
  // 두 개의 레이블 생성
  int Ltrue = genlabel();
  int Lend = genlabel();

  // r1을 테스트하고 참이면 true 레이블로 점프
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r1], reglist[r1]);
  fprintf(Outfile, "\tjne\tL%d\n", Ltrue);

  // r2를 테스트하고 참이면 true 레이블로 점프
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r2], reglist[r2]);
  fprintf(Outfile, "\tjne\tL%d\n", Ltrue);

  // 점프하지 않았으므로 결과는 거짓
  fprintf(Outfile, "\tmovq\t$0, %s\n", reglist[r1]);
  fprintf(Outfile, "\tjmp\tL%d\n", Lend);

  // 누군가 true 레이블로 점프했으므로 결과는 참
  cglabel(Ltrue);
  fprintf(Outfile, "\tmovq\t$1, %s\n", reglist[r1]);
  cglabel(Lend);
  free_register(r2);
  return(r1);
}
```

```c
// 두 레지스터를 논리 AND 연산하고 결과(1 또는 0)를 가진 레지스터를 반환
int cglogand(int r1, int r2) {
  // 두 개의 레이블 생성
  int Lfalse = genlabel();
  int Lend = genlabel();

  // r1을 테스트하고 참이 아니면 false 레이블로 점프
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r1], reglist[r1]);
  fprintf(Outfile, "\tje\tL%d\n", Lfalse);

  // r2를 테스트하고 참이 아니면 false 레이블로 점프
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r2], reglist[r2]);
  fprintf(Outfile, "\tje\tL%d\n", Lfalse);

  // 점프하지 않았으므로 결과는 참
  fprintf(Outfile, "\tmovq\t$1, %s\n", reglist[r1]);
  fprintf(Outfile, "\tjmp\tL%d\n", Lend);

  // 누군가 false 레이블로 점프했으므로 결과는 거짓
  cglabel(Lfalse);
  fprintf(Outfile, "\tmovq\t$0, %s\n", reglist[r1]);
  cglabel(Lend);
  free_register(r2);
  return(r1);
}
```

`tests/input122.c` 프로그램은 이 새로운 기능이 작동하는지 확인하는 테스트이다.


## 결론과 다음 단계

여기까지 우리가 여정의 이 부분에서 고친 몇 가지 사항들을 살펴봤다. 이제는 한 발짝 물러서서 배열과 포인터 설계를 다시 생각해보고, 컴파일러 작성 여정의 다음 단계에서 이를 개선해보려 한다. [다음 단계](../51_Arrays_pt2/Readme.md)



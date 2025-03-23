# 9장: While 반복문

이번 장에서는 우리의 언어에 WHILE 반복문을 추가한다. 어떤 면에서 WHILE 반복문은 'else' 절이 없는 IF 문과 매우 유사하다. 단, 항상 반복문의 시작 부분으로 돌아간다는 점이 다르다.

다음 코드를 보자:

```
  while (condition is true) {
    statements;
  }
```

이 코드는 다음과 같이 변환된다:

```
Lstart: evaluate condition
	jump to Lend if condition false
	statements
	jump to Lstart
Lend:
```

이는 IF 문에서 사용한 스캐닝, 파싱, 코드 생성 구조를 그대로 활용할 수 있다는 것을 의미한다. WHILE 문을 처리하기 위해 약간의 수정만 가하면 된다.

이제 이 과정을 어떻게 구현하는지 살펴보자.


## 새로운 토큰

새로운 'while' 키워드를 위해 새로운 토큰인 T_WHILE가 필요하다. `defs.h`와 `scan.c` 파일에 대한 변경 사항은 명확하므로 여기서는 생략한다.


## WHILE 문법 분석

WHILE 루프의 BNF 문법은 다음과 같다:

```
// while_statement: 'while' '(' true_false_expression ')' compound_statement  ;
```

이를 파싱하기 위해 `stmt.c`에 함수를 추가한다. IF 문 파싱에 비해 상대적으로 간단한 구조를 확인할 수 있다:

```c
// WHILE 문을 파싱하고 AST를 반환한다
struct ASTnode *while_statement(void) {
  struct ASTnode *condAST, *bodyAST;

  // 'while' '('가 있는지 확인한다
  match(T_WHILE, "while");
  lparen();

  // 다음 표현식을 파싱하고 ')'를 확인한다.
  // 트리의 연산이 비교 연산자인지 확인한다.
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("잘못된 비교 연산자");
  rparen();

  // 복합문에 대한 AST를 가져온다
  bodyAST = compound_statement();

  // 이 문에 대한 AST를 생성하고 반환한다
  return (mkastnode(A_WHILE, condAST, NULL, bodyAST, 0));
}
```

`defs.h`에 새로운 AST 노드 타입인 A_WHILE을 추가한다. 이 노드는 조건을 평가하기 위한 왼쪽 자식 서브트리와 WHILE 루프의 본문인 복합문을 위한 오른쪽 자식 서브트리를 가진다.


## 일반적인 코드 생성

루프의 시작과 끝을 나타내는 레이블을 생성하고, 조건을 평가한 후 루프를 빠져나가거나 루프의 시작으로 돌아가기 위한 점프 명령어를 삽입해야 한다. 이 과정은 IF 문을 생성하는 코드보다 훨씬 간단하다. `gen.c` 파일에서 다음과 같이 구현한다:

```c
// WHILE 문과 선택적인 ELSE 절을 위한 코드 생성
static int genWHILE(struct ASTnode *n) {
  int Lstart, Lend;

  // 시작과 끝 레이블 생성
  // 시작 레이블 출력
  Lstart = label();
  Lend = label();
  cglabel(Lstart);

  // 조건 코드 생성 후 끝 레이블로 점프
  // Lfalse 레이블을 레지스터로 전달하여 간단히 처리
  genAST(n->left, Lend, n->op);
  genfreeregs();

  // 루프 본문을 위한 복합문 생성
  genAST(n->right, NOREG, n->op);
  genfreeregs();

  // 조건으로 돌아가는 점프와 끝 레이블 출력
  cgjump(Lstart);
  cglabel(Lend);
  return (NOREG);
}
```

여기서 주의할 점은 비교 연산자의 부모 AST 노드가 A_WHILE일 수도 있다는 점이다. 따라서 `genAST()` 함수에서 비교 연산자를 처리하는 코드는 다음과 같다:

```c
    case A_EQ:
    case A_NE:
    case A_LT:
    case A_GT:
    case A_LE:
    case A_GE:
      // 부모 AST 노드가 A_IF 또는 A_WHILE인 경우
      // 비교 후 점프를 생성한다. 그렇지 않으면 레지스터를 비교하고
      // 결과에 따라 하나의 레지스터를 1 또는 0으로 설정한다.
      if (parentASTop == A_IF || parentASTop == A_WHILE)
        return (cgcompare_and_jump(n->op, leftreg, rightreg, reg));
      else
        return (cgcompare_and_set(n->op, leftreg, rightreg));
```

이렇게 하면 WHILE 루프를 구현하는 데 필요한 모든 작업이 완료된다!


## 새로운 언어 추가 기능 테스트

모든 입력 파일을 `test/` 디렉토리로 이동했다. 이제 `make test` 명령을 실행하면 이 디렉토리로 이동해 각 입력 파일을 컴파일하고, 알려진 정상 출력과 비교한다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c stmt.c
      sym.c tree.c
(cd tests; chmod +x runtests; ./runtests)
input01: OK
input02: OK
input03: OK
input04: OK
input05: OK
input06: OK
```

`make test6` 명령도 사용할 수 있다. 이 명령은 `tests/input06` 파일을 컴파일한다:

```c
{ int i;
  i=1;
  while (i <= 10) {
    print i;
    i= i + 1;
  }
}
```

이 코드는 1부터 10까지의 숫자를 출력한다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
      stmt.c sym.c tree.c
./comp1 tests/input06
cc -o out out.s
./out
1
2
3
4
5
6
7
8
9
10
```

컴파일 결과로 생성된 어셈블리 출력은 다음과 같다:

```
	.comm	i,8,8
	movq	$1, %r8
	movq	%r8, i(%rip)		# i= 1
L1:
	movq	i(%rip), %r8
	movq	$10, %r9
	cmpq	%r9, %r8		# Is i <= 10?
	jg	L2			# Greater than, jump to L2
	movq	i(%rip), %r8
	movq	%r8, %rdi		# Print out i
	call	printint
	movq	i(%rip), %r8
	movq	$1, %r9
	addq	%r8, %r9		# Add 1 to i
	movq	%r9, i(%rip)
	jmp	L1			# and loop back
L2:
```


## 결론과 다음 단계

WHILE 루프는 이미 IF문을 구현했기 때문에 쉽게 추가할 수 있었다. 두 구문은 많은 부분에서 유사하기 때문이다.

이제 우리는 [튜링 완전](https://en.wikipedia.org/wiki/Turing_completeness)한 언어를 갖추게 되었다:

  + 무한한 저장 공간, 즉 무한한 수의 변수
  + 저장된 값을 기반으로 결정을 내릴 수 있는 능력, 즉 IF문
  + 방향을 변경할 수 있는 능력, 즉 WHILE 루프

그렇다면 이제 작업을 멈춰도 될까? 물론 아니다. 우리는 여전히 컴파일러가 스스로를 컴파일할 수 있도록 하는 목표를 향해 나아가고 있다.

컴파일러 작성 여정의 다음 단계에서는 FOR 루프를 언어에 추가할 것이다. [다음 단계](../10_For_Loops/Readme.md)



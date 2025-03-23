# 10부: FOR 루프 구현

컴파일러 개발 여정의 이번 단계에서는 FOR 루프를 추가한다. 구현 과정에서 해결해야 할 복잡한 문제가 하나 있는데, 이 문제에 대해 먼저 설명하고 나서 어떻게 해결했는지 논의할 것이다.


## FOR 반복문 구문

FOR 반복문의 구문에 익숙할 것이라 가정한다. 한 가지 예를 들면 다음과 같다.

```c
  for (i=0; i < MAX; i++)
    printf("%d\n", i);
```

이제 이 언어를 위해 BNF 구문을 사용할 것이다.

```
 for_statement: 'for' '(' preop_statement ';'
                          true_false_expression ';'
                          postop_statement ')' compound_statement  ;

 preop_statement:  statement  ;        (현재는 이렇게 정의)
 postop_statement: statement  ;        (현재는 이렇게 정의)
```

`preop_statement`는 반복문이 시작되기 전에 실행된다. 나중에 여기서 수행할 수 있는 동작의 종류를 제한해야 한다(예: IF 문은 허용하지 않음). 그 다음 `true_false_expression`이 평가된다. 이 표현식이 참이면 `compound_statement`가 실행된다. 이 작업이 완료되면 `postop_statement`가 수행되고, 코드는 다시 `true_false_expression`을 평가하기 위해 반복된다.


## 문제점

`postop_statement`는 `compound_statement`보다 먼저 파싱되지만, `postop_statement`의 코드는 `compound_statement`의 코드 *이후에* 생성해야 한다. 이 문제를 해결하는 방법은 여러 가지가 있다. 이전에 컴파일러를 작성할 때는 `compound_statement`의 어셈블리 코드를 임시 버퍼에 저장하고, `postop_statement`의 코드를 생성한 후에 그 버퍼를 "재생"하는 방식을 선택했다. SubC 컴파일러에서는 Nils가 라벨과 라벨로의 점프를 교묘하게 사용해 코드 실행을 "스레드"처럼 연결하여 올바른 순서를 강제한다.

하지만 여기서는 AST 트리를 구축한다. 이를 활용해 생성된 어셈블리 코드가 올바른 순서로 나오도록 해보자.


## 어떤 종류의 AST 트리일까?

FOR 루프는 네 가지 구조적 요소를 가진다:

1. `preop_statement`
2. `true_false_expression`
3. `postop_statement`
4. `compound_statement`

AST 노드 구조를 다시 변경하여 네 개의 자식을 갖게 하고 싶지는 않다. 하지만 FOR 루프를 확장된 WHILE 루프로 시각화할 수 있다:

```
   preop_statement;
   while ( true_false_expression ) {
     compound_statement;
     postop_statement;
   }
```

기존의 노드 타입을 사용하여 이 구조를 반영한 AST 트리를 만들 수 있을까? 가능하다:

```
         A_GLUE
        /     \
   preop     A_WHILE
             /    \
        decision  A_GLUE
                  /    \
            compound  postop
```

이 트리를 위에서 아래로, 왼쪽에서 오른쪽으로 수동으로 탐색해 보면, 올바른 순서로 어셈블리 코드를 생성할 것임을 확인할 수 있다. WHILE 루프가 종료될 때 `compound_statement`와 `postop_statement`를 모두 건너뛰도록 하기 위해 이 두 문장을 함께 붙여야 한다.

이는 또한 새로운 T_FOR 토큰이 필요하지만 새로운 AST 노드 타입은 필요하지 않음을 의미한다. 따라서 컴파일러에서 변경해야 할 부분은 스캐닝과 파싱뿐이다.


## 토큰과 스캐닝

새로운 키워드 'for'와 관련된 토큰 T_FOR가 추가되었다. 여기서는 큰 변화가 없다.


## 구문 분석 로직 개선

파서의 구조를 조정해야 한다. FOR 문법에서 `preop_statement`와 `postop_statement`는 단일 문장만 허용해야 한다. 현재 `compound_statement()` 함수는 닫는 중괄호 '}'를 만날 때까지 반복하며 문장을 처리한다. 이를 분리해 `compound_statement()`가 `single_statement()`를 호출해 하나의 문장을 처리하도록 변경해야 한다.

여기서 또 다른 문제가 발생한다. `assignment_statement()`에서 할당문을 파싱할 때, 문장 끝에 세미콜론을 찾는다. 이는 복합문(compound statement)에서는 적합하지만 FOR 루프에서는 동작하지 않는다. 예를 들어 다음과 같이 작성해야 한다:

```c
for (i=1 ; i < 10 ; i= i + 1; )
```

각 할당문이 반드시 세미콜론으로 끝나기 때문이다.

단일 문장 파서는 세미콜론을 스캔하지 않고, 이를 복합문 파서에 맡겨야 한다. 그리고 일부 문장(예: 할당문 사이)에서는 세미콜론을 스캔하지만, 다른 문장(예: 연속된 IF 문)에서는 스캔하지 않아야 한다.

이를 반영한 새로운 단일 및 복합문 파싱 코드는 다음과 같다:

```c
// 단일 문장을 파싱하고 AST를 반환
static struct ASTnode *single_statement(void) {
  switch (Token.token) {
    case T_PRINT:
      return (print_statement());
    case T_INT:
      var_declaration();
      return (NULL);		// 여기서는 AST 생성하지 않음
    case T_IDENT:
      return (assignment_statement());
    case T_IF:
      return (if_statement());
    case T_WHILE:
      return (while_statement());
    case T_FOR:
      return (for_statement());
    default:
      fatald("Syntax error, token", Token.token);
  }
}

// 복합문을 파싱하고 AST를 반환
struct ASTnode *compound_statement(void) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  // 여는 중괄호를 요구
  lbrace();

  while (1) {
    // 단일 문장 파싱
    tree = single_statement();

    // 일부 문장은 세미콜론으로 끝나야 함
    if (tree != NULL &&
	(tree->op == A_PRINT || tree->op == A_ASSIGN))
      semi();

    // 각 새로운 트리를 left에 저장
    // left가 비어 있으면 새 트리를 저장하고,
    // 그렇지 않으면 left와 새 트리를 결합
    if (tree != NULL) {
      if (left == NULL)
	left = tree;
      else
	left = mkastnode(A_GLUE, left, NULL, tree, 0);
    }
    // 닫는 중괄호를 만나면
    // 이를 건너뛰고 AST를 반환
    if (Token.token == T_RBRACE) {
      rbrace();
      return (left);
    }
  }
}
```

또한 `print_statement()`와 `assignment_statement()`에서 `semi()` 호출을 제거했다.


## FOR 루프 파싱

위에서 제시한 FOR 루프의 BNF 문법을 바탕으로, 이를 파싱하는 것은 간단하다. 또한 원하는 AST 트리의 구조를 고려하면, 이 트리를 생성하는 코드도 직관적이다. 다음은 해당 코드이다:

```c
// FOR 문을 파싱하고
// 해당 AST를 반환한다
static struct ASTnode *for_statement(void) {
  struct ASTnode *condAST, *bodyAST;
  struct ASTnode *preopAST, *postopAST;
  struct ASTnode *tree;

  // 'for' '('가 있는지 확인한다
  match(T_FOR, "for");
  lparen();

  // pre_op 문과 ';'를 가져온다
  preopAST= single_statement();
  semi();

  // 조건문과 ';'를 가져온다
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("잘못된 비교 연산자");
  semi();

  // post_op 문과 ')'를 가져온다
  postopAST= single_statement();
  rparen();

  // 본문에 해당하는 복합문을 가져온다
  bodyAST = compound_statement();

  // 현재는 네 개의 서브 트리가 모두 NULL이 아니어야 한다.
  // 나중에 일부가 누락된 경우의 의미를 변경할 예정이다

  // 복합문과 postop 트리를 결합한다
  tree= mkastnode(A_GLUE, bodyAST, NULL, postopAST, 0);

  // 조건문과 새로운 본문으로 WHILE 루프를 만든다
  tree= mkastnode(A_WHILE, condAST, NULL, tree, 0);

  // preop 트리를 A_WHILE 트리에 결합한다
  return(mkastnode(A_GLUE, preopAST, NULL, tree, 0));
}
```


## 어셈블리 코드 생성

지금까지 우리는 WHILE 루프와 여러 하위 트리를 결합한 트리를 합성했다. 따라서 컴파일러의 코드 생성 부분에는 아무런 변경이 필요하지 않다.


## 직접 해보기

`tests/input07` 파일에는 다음과 같은 프로그램이 들어 있다:

```c
{
  int i;
  for (i= 1; i <= 10; i= i + 1) {
    print i;
  }
}
```

`make test7`를 실행하면 다음과 같은 결과가 출력된다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c
./comp1 tests/input07
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

그리고 여기 관련된 어셈블리 출력이 있다:

```
	.comm	i,8,8
	movq	$1, %r8
	movq	%r8, i(%rip)		# i = 1
L1:
	movq	i(%rip), %r8
	movq	$10, %r9
	cmpq	%r9, %r8		# i < 10인가?
	jg	L2			# i >= 10이면 L2로 점프
	movq	i(%rip), %r8
	movq	%r8, %rdi
	call	printint		# i 출력
	movq	i(%rip), %r8
	movq	$1, %r9
	addq	%r8, %r9		# i = i + 1
	movq	%r9, i(%rip)
	jmp	L1			# 루프의 시작으로 점프
L2:
```


## 결론 및 다음 단계

현재 우리의 언어에는 상당수의 제어 구조가 구현되어 있다:
IF 문, WHILE 루프, FOR 루프 등이 그렇다. 이제 다음 단계로 무엇을 다뤄야 할까? 고려할 만한 주제는 다양하다:

 + 타입
 + 지역 변수와 전역 변수
 + 함수
 + 배열과 포인터
 + 구조체와 공용체
 + auto, static 등의 키워드

여기서는 함수를 다루기로 결정했다. 따라서 컴파일러 개발의 다음 단계에서는, 우리의 언어에 함수를 추가하기 위한 첫 번째 작업을 시작할 것이다. [다음 단계](../11_Functions_pt1/Readme.md)



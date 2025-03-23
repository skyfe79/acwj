# 11장: 함수 구현 (1부)

이제 우리의 언어에 함수를 구현하려고 한다. 하지만 이 작업은 많은 단계를 거쳐야 한다는 걸 알고 있다. 이 과정에서 다뤄야 할 주요 사항은 다음과 같다:

+ 데이터 타입: `char`, `int`, `long` 등
+ 각 함수의 반환 타입
+ 함수의 인자 개수
+ 함수 내부의 지역 변수와 전역 변수

이 모든 것을 한 번에 처리하기에는 너무 많은 작업이다. 그래서 이번 단계에서는 다양한 함수를 **선언**할 수 있는 수준까지 진행할 예정이다. 생성된 실행 파일에서는 `main()` 함수만 실행되지만, 여러 함수에 대한 코드를 생성할 수 있는 기능을 추가할 것이다.

이 작업을 통해 우리의 컴파일러가 인식하는 언어가 '실제' C 컴파일러에서도 인식할 수 있는 C 언어의 부분집합(subset)에 가까워지길 바란다. 하지만 아직 그 단계는 아니다.


## 간단한 함수 구문

이 부분은 일단 자리만 잡아두는 용도다. 함수처럼 보이는 것을 파싱할 수 있게 하기 위함이다. 이 작업이 끝나면 타입, 반환 타입, 인자 등 다른 중요한 요소들을 추가할 수 있다.

현재는 BNF로 다음과 같은 함수 문법을 추가한다:

```
 function_declaration: 'void' identifier '(' ')' compound_statement   ;
```

모든 함수는 `void`로 선언되며 인자를 받지 않는다. 또한 함수를 호출하는 기능은 도입하지 않기 때문에 `main()` 함수만 실행된다.

새로운 키워드 `void`와 새로운 토큰 T_VOID를 추가해야 한다. 둘 다 쉽게 추가할 수 있다.


## 단순한 함수 구문 파싱하기

새로운 함수 구문은 매우 단순해서 이를 파싱하는 깔끔하고 작은 함수를 작성할 수 있다(`decl.c` 파일에 위치):

```c
// 단순한 함수 선언 파싱
struct ASTnode *function_declaration(void) {
  struct ASTnode *tree;
  int nameslot;

  // 'void', 식별자, 그리고 '(' ')'를 찾는다.
  // 현재는 이들에 대해 아무런 작업도 하지 않는다
  match(T_VOID, "void");
  ident();
  nameslot= addglob(Text);
  lparen();
  rparen();

  // 복합문에 대한 AST 트리를 얻는다
  tree= compound_statement();

  // 함수의 nameslot과 복합문 서브 트리를 가진 A_FUNCTION 노드를 반환한다
  return(mkastunary(A_FUNCTION, tree, nameslot));
}
```

이 함수는 구문 검사와 AST 구성을 수행하지만, 의미론적 오류 검사는 거의 없다. 예를 들어 함수가 재선언되면 어떻게 될까? 아직은 이를 감지하지 못한다.


## `main()` 함수 수정

위 함수를 활용하면 `main()` 함수의 일부 코드를 수정해 여러 함수를 연속적으로 파싱할 수 있다.

```c
  scan(&Token);                 // 입력에서 첫 번째 토큰을 가져옴
  genpreamble();                // 프리앰블 출력
  while (1) {                   // 함수를 파싱하고
    tree = function_declaration();
    genAST(tree, NOREG, 0);     // 해당 함수에 대한 어셈블리 코드 생성
    if (Token.token == T_EOF)   // EOF에 도달하면 종료
      break;
  }
```

`genpostamble()` 함수 호출을 제거했다. 이 함수는 `main()` 함수에 대해 생성된 어셈블리 코드의 포스트앰블을 출력했기 때문이다. 이제 함수의 시작과 끝을 생성하는 코드 생성 함수가 필요하다.


## 함수를 위한 일반적인 코드 생성

이제 A_FUNCTION AST 노드가 생겼으니, `gen.c`에 있는 일반 코드 생성기에 이 노드를 처리할 코드를 추가해야 한다. 위에서 보았듯이, 이 노드는 단일 자식을 가진 *단항* AST 노드다:

```c
  // 함수의 이름 슬롯과 복합문 서브트리를 가진 A_FUNCTION 노드를 반환
  return(mkastunary(A_FUNCTION, tree, nameslot));
```

자식 노드는 함수의 본문인 복합문을 담고 있는 서브트리를 가지고 있다. 복합문에 대한 코드를 생성하기 *전에* 함수의 시작 부분을 생성해야 한다. `genAST()`에서 이를 처리하는 코드는 다음과 같다:

```c
    case A_FUNCTION:
      // 코드 생성 전에 함수의 프리앰블 생성
      cgfuncpreamble(Gsym[n->v.id].name);
      genAST(n->left, NOREG, n->op);
      cgfuncpostamble();
      return (NOREG);
```


## x86-64 코드 생성

이제 각 함수의 스택 포인터와 프레임 포인터를 설정하고, 함수가 끝날 때 이를 해제한 후 호출자에게 제어를 반환하는 코드를 생성해야 한다.

이미 `cgpreamble()`과 `cgpostamble()`에 이러한 코드가 있지만, `cgpreamble()`에는 `printint()` 함수의 어셈블리 코드도 포함되어 있다. 따라서 이 어셈블리 코드 조각들을 `cg.c` 파일의 새로운 함수로 분리해야 한다.

```c
// 어셈블리 프리앰블 출력
void cgpreamble() {
  freeall_registers();
  // printint() 함수의 코드만 출력
}

// 함수 프리앰블 출력
void cgfuncpreamble(char *name) {
  fprintf(Outfile,
          "\t.text\n"
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n", name, name, name);
}

// 함수 포스트앰블 출력
void cgfuncpostamble() {
  fputs("\tmovl $0, %eax\n" "\tpopq     %rbp\n" "\tret\n", Outfile);
}
```


## 함수 생성 기능 테스트

새로운 테스트 프로그램인 `tests/input08`을 살펴보자. 이 프로그램은 `print` 구문을 제외하면 C 프로그램과 유사한 형태를 띤다:

```c
void main()
{
  int i;
  for (i= 1; i <= 10; i= i + 1) {
    print i;
  }
}
```

이 프로그램을 테스트하기 위해 `make test8` 명령어를 실행한다. 이 명령어는 다음 작업을 수행한다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c
./comp1 tests/input08
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

어셈블리 출력은 이전에 FOR 루프 테스트에서 생성된 코드와 동일하므로 따로 살펴보지 않는다.

하지만 이전 테스트 입력 파일들에 `void main()`을 추가했다. 이는 언어가 복합문 코드 앞에 함수 선언을 요구하기 때문이다.

테스트 프로그램 `tests/input09`에는 두 개의 함수가 선언되어 있다. 컴파일러는 각 함수에 대해 동작하는 어셈블리 코드를 성공적으로 생성하지만, 현재는 두 번째 함수의 코드를 실행할 수 없다.


## 결론과 다음 단계

우리는 언어에 함수를 추가하는 첫걸음을 성공적으로 마쳤다. 현재는 아주 기본적인 함수 선언만 구현한 상태다.

컴파일러 개발의 다음 단계에서는 타입 시스템을 추가하는 작업을 시작할 것이다. [다음 단계](../12_Types_pt1/Readme.md)



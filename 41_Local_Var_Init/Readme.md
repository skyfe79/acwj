# 41장: 지역 변수 초기화

이전 장에서 많은 변경 사항을 다룬 후, 지역 변수 초기화는 상대적으로 쉬운 작업이었다.

함수 내부에서 다음과 같은 작업을 가능하게 하고 싶다:

```c
  int x= 2, y= x+3, z= 5 * x - y;
  char *foo= "Hello world";
```

함수 내부에 있기 때문에, 표현식에 대한 AST(추상 구문 트리)를 만들고, 변수에 대한 A_IDENT 노드를 생성한 후, 이를 A_ASSIGN 부모 노드와 연결할 수 있다. 또한, 여러 선언과 할당이 있을 수 있으므로, 모든 할당 트리를 담을 수 있는 A_GLUE 트리를 만들어야 할 수도 있다.

유일한 문제는 지역 선언을 파싱하는 코드가 문장 파싱을 처리하는 코드와 상당히 멀리 떨어져 있다는 점이다. 구체적으로:

 + `stmt.c`의 `single_statement()` 함수는 타입 식별자를 보고,
 + `decl.c`의 `declaration_list()` 함수를 호출해 여러 선언을 파싱하며,
 + `symbol_declaration()` 함수를 호출해 하나의 선언을 파싱하고,
 + `scalar_declaration()` 함수를 호출해 스칼라 변수 선언과 할당을 파싱한다.

주요 문제는 이 함수들이 이미 값을 반환하고 있기 때문에, `scalar_declaration()`에서 AST 트리를 만들어 `single_statement()`로 반환할 수 없다는 점이다.

또한, `declaration_list()`는 여러 선언을 파싱하므로, 이를 모두 담을 A_GLUE 트리를 만들어야 한다.

해결책은 `single_statement()`에서 `declaration_list()`로 "포인터의 포인터"를 전달해 A_GLUE 트리에 대한 포인터를 반환할 수 있게 하는 것이다. 마찬가지로, `declaration_list()`에서 `scalar_declaration()`로 "포인터의 포인터"를 전달해, 생성된 할당 트리에 대한 포인터를 반환할 수 있게 한다.


## `scalar_declaration()`의 변경 사항

로컬 컨텍스트에서 스칼라 변수 선언 시 '='을 만나면 다음과 같이 처리한다.

```c
  struct ASTnode *varnode, *exprnode;
  struct ASTnode **tree;                 // 전달받은 ptr ptr 인자

  // 변수가 초기화되는 경우
  if (Token.token == T_ASSIGN) {
    ...
    if (class == C_LOCAL) {
      // 변수를 나타내는 A_IDENT AST 노드 생성
      varnode = mkastleaf(A_IDENT, sym->type, sym, 0);

      // 할당을 위한 표현식을 가져와 rvalue로 변환
      exprnode = binexpr(0);
      exprnode->rvalue = 1;

      // 표현식의 타입이 변수의 타입과 일치하는지 확인
      exprnode = modify_type(exprnode, varnode->type, 0);
      if (exprnode == NULL)
        fatal("Incompatible expression in assignment");

      // 할당 AST 트리 생성
      *tree = mkastnode(A_ASSIGN, exprnode->type, exprnode,
                                        NULL, varnode, NULL, 0);
    }
  }
```

이 과정은 일반적으로 `expr.c`에서 할당 표현식을 처리할 때 발생하는 AST 트리 구축을 시뮬레이션한다. 작업이 완료되면 할당 트리를 반환한다. 이 트리는 `declaration_list()`로 전달된다. 이제 `declaration_list()`는 다음과 같이 동작한다.

```c
  struct ASTnode **gluetree;            // 전달받은 ptr ptr 인자
  struct ASTnode *tree;
  *gluetree= NULL;
  ...
  // 이제 심볼 리스트를 파싱
  while (1) {
    ...
    // 심볼 파싱
    sym = symbol_declaration(type, *ctype, class, &tree);
    ...
    // 로컬 선언에서 생성된 AST 트리를 연결하여
    // 할당 작업 시퀀스 구축
    if (*gluetree== NULL)
      *gluetree= tree;
    else
      *gluetree = mkastnode(A_GLUE, P_NONE, *gluetree, NULL, tree, NULL, 0);
    ...
  }
```

따라서 `gluetree`는 여러 A_GLUE 노드로 구성된 AST 트리로 설정된다. 각 A_GLUE 노드는 A_ASSIGN 자식 노드를 가지며, A_ASSIGN 노드는 A_IDENT 자식 노드와 표현식 자식 노드를 포함한다.

`stmt.c`의 `single_statement()`에서는 다음과 같이 처리한다.

```c
    ...
    case T_IDENT:
      // 식별자가 typedef와 일치하는지 확인
      // 일치하지 않으면 표현식으로 처리
      // 일치하면 parse_type() 호출로 넘어감
      if (findtypedef(Text) == NULL) {
        stmt= binexpr(0); semi(); return(stmt);
      }
    case T_CHAR:
    case T_INT:
    case T_LONG:
    case T_STRUCT:
    case T_UNION:
    case T_ENUM:
    case T_TYPEDEF:
      // 변수 선언 리스트의 시작
      declaration_list(&ctype, C_LOCAL, T_SEMI, T_EOF, &stmt);
      semi();
      return (stmt);            // 선언에서 발생한 할당 작업
    ...
```


## 새로운 코드 테스트하기

위에서 변경한 내용은 매우 짧고 간단해서 컴파일도 잘 되고 처음 실행부터 정상적으로 동작했다. 하지만 이런 경우는 흔하지 않다! 우리의 테스트 프로그램인 `tests/input100.c`는 다음과 같다:

```c
#include <stdio.h>
int main() {
  int x= 3, y=14;
  int z= 2 * x + y;
  char *str= "Hello world";
  printf("%s %d %d\n", str, x+y, z);
  return(0);
}
```

이 프로그램은 `Hello world 17 20`이라는 정확한 출력을 생성한다.


## 결론과 다음 단계

이 여정 중에 간단한 부분을 다룰 수 있어서 좋다. 이제 나는 스스로와 내기를 시작한다:

+ 이 여정의 전체 파트 수가 얼마나 될지
+ 올해 말까지 모든 것을 완료할 수 있을지

현재로는 약 60개의 파트가 있을 것으로 예상하고, 올해 말까지 완료할 확률은 75% 정도로 보고 있다. 하지만 아직 컴파일러에 추가해야 할 작지만 어려울 수 있는 기능들이 많이 남아 있다.

컴파일러 작성 여정의 다음 단계에서는 캐스트 파싱을 추가할 것이다. [다음 단계](../42_Casting/Readme.md)



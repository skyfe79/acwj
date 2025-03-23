# 52부: 포인터, 2부

이번 컴파일러 개발 과정에서는 포인터 관련 문제를 해결하기 시작했다. 결국 `expr.c` 파일의 절반을 재구성하고, 컴파일러 내 함수의 4분의 1에 해당하는 API를 변경하게 되었다. 코드 라인 수로 보면 큰 변화이지만, 실제로는 버그 수정이나 기능 개선 측면에서 큰 진전은 아니다.


## 문제의 시작

이 모든 문제의 근원부터 살펴보자. 컴파일러의 소스 코드를 컴파일러 자체로 실행할 때, 포인터 체인을 파싱할 수 없다는 사실을 깨달았다. 예를 들어 다음과 같은 표현식을 처리할 수 없었다:

```c
  ptr->next->next->next
```

이 문제의 원인은 `primary()` 함수가 호출되면서 표현식의 시작 부분에 있는 식별자의 값을 가져오기 때문이다. `primary()` 함수는 뒤에 붙는 후위 연산자를 발견하면 `postfix()` 함수를 호출해 처리한다. `postfix()` 함수는 예를 들어 하나의 `->` 연산자를 처리하고 반환한다. 여기서 끝이다. `->` 연산자 체인을 따라가기 위한 반복문이 존재하지 않는다.

더 심각한 문제는 `primary()` 함수가 단일 식별자만을 찾는다는 점이다. 이 때문에 다음과 같은 표현식도 파싱할 수 없다:

```c
  ptrarray[4]->next     OR
  unionvar.member->next
```

왜냐하면 `->` 연산자 앞에 단일 식별자가 아닌 다른 형태의 표현식이 오기 때문이다.


## 어떻게 이런 일이 발생했을까?

이런 일이 발생한 이유는 우리 개발 방식의 빠른 프로토타이핑 특성 때문이다. 기능을 한 번에 조금씩 추가하며, 미래의 필요까지 크게 고려하지 않는다. 그래서 가끔은 더 일반적이고 유연하게 만들기 위해 작성한 코드를 다시 수정해야 하는 상황이 발생한다.


## 문제 해결 방법

[C 언어의 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)을 살펴보면 다음과 같은 내용을 확인할 수 있다:

```
primary_expression
        : IDENTIFIER
        | CONSTANT
        | STRING_LITERAL
        | '(' expression ')'
        ;

postfix_expression
        : primary_expression
        | postfix_expression '[' expression ']'
        | postfix_expression '(' ')'
        | postfix_expression '(' argument_expression_list ')'
        | postfix_expression '.' IDENTIFIER
        | postfix_expression '->' IDENTIFIER
        | postfix_expression '++'
        | postfix_expression '--'
        ;
```

다시 말해, 현재 코드는 순서가 뒤바뀌어 있다. `postfix`는 `primary()`를 호출해 식별자를 나타내는 AST 노드를 가져와야 한다. 그런 다음, 후위 연산자 토큰이 있는지 반복해서 확인하고, 이를 파싱한 후 `primary()`에서 반환된 식별자 노드에 새로운 AST 부모 노드를 추가해야 한다.

이 모든 과정은 간단해 보이지만 한 가지 문제가 있다. 현재 `primary()`는 AST 노드를 생성하지 않으며, 단순히 식별자를 파싱해 `Text`에 남겨둔다. 식별자와 후위 연산자를 포함한 AST 노드나 AST 트리를 생성하는 작업은 `postfix()`의 역할이다.

동시에, `defs.h`에 정의된 AST 노드 구조는 기본 타입만 알고 있다:

```c
// 추상 구문 트리 구조
struct ASTnode {
  int op;           // 이 트리에서 수행할 "연산"
  int type;         // 이 트리가 생성하는 표현식의 타입
  int rvalue;       // 참고: ctype 없음
  ...
};
```

이러한 구조는 최근에 구조체와 공용체를 추가했기 때문이다. 또한 `postfix()`가 대부분의 파싱 작업을 담당했기 때문에 구조체나 공용체 심볼에 대한 포인터를 저장할 필요가 없었다.

따라서 문제를 해결하려면 다음 단계를 따라야 한다:

1. `struct ASTnode`에 `ctype` 포인터를 추가해 각 AST 노드에 전체 타입을 저장한다.
2. AST 노드를 생성하는 모든 함수와 이러한 함수를 호출하는 모든 부분을 찾아 수정해 노드의 `ctype`이 저장되도록 한다.
3. `primary()`를 `expr.c`의 상단으로 이동시켜 AST 노드를 생성하도록 한다.
4. `postfix()`가 `primary()`를 호출해 식별자(A_IDENT)에 대한 기본 AST 노드를 가져오도록 한다.
5. `postfix()`가 후위 연산자를 처리하는 동안 반복하도록 한다.

이 작업은 상당히 많으며, AST 노드 호출이 모든 곳에 흩어져 있기 때문에 컴파일러의 모든 소스 파일을 수정해야 한다.


## AST 노드 함수 변경 사항

모든 세부 사항을 다루지는 않겠지만, `defs.h` 파일의 AST 노드 구조 변경과 `tree.c` 파일에서 AST 노드를 생성하는 주요 함수부터 살펴보자.

```c
// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  int type;                     // 이 트리가 생성하는 표현식의 타입
  struct symtable *ctype;       // 구조체/공용체일 경우, 해당 타입에 대한 포인터
  ...
};

// 일반적인 AST 노드를 생성하고 반환
struct ASTnode *mkastnode(int op, int type,
                          struct symtable *ctype, ...) {
  ...
  // 필드 값을 복사하고 반환
  n->op = op;
  n->type = type;
  n->ctype = ctype;
  ...
}
```

`mkastleaf()`와 `mkastunary()` 함수에도 변경 사항이 있다. 이제 이 함수들은 `ctype`을 받아 `mkastnode()`를 호출할 때 이 인자를 전달한다.

컴파일러에는 이 세 함수를 호출하는 약 40개의 코드가 있다. 각각의 호출을 모두 다루지는 않겠다. 대부분의 경우, 기본적인 `type`과 `ctype` 포인터가 사용 가능하다. 일부 호출은 AST 노드 타입을 P_INT로 설정하고, 이 경우 `ctype`은 NULL이다. 또 다른 호출들은 AST 노드 타입을 P_NONE으로 설정하며, 이 경우에도 `ctype`은 NULL이다.


## `modify_type()` 함수 변경 사항

`modify_type()` 함수는 AST 노드의 타입이 다른 타입과 호환되는지 확인하고, 필요한 경우 노드를 확장하여 다른 타입과 일치시키는 역할을 한다. 이 함수는 `mkastunary()`를 호출하므로, `ctype` 인자를 제공해야 한다. 이 변경 사항을 반영하면서, `modify_type()`을 호출하는 여섯 군데의 코드도 수정하여 비교 대상 타입의 `ctype`을 전달하도록 했다.


## `expr.c`의 변경 사항

이제 주요 변경 사항인 `primary()`와 `postfix()`의 구조 재조정에 대해 알아보자. 앞서 이미 해야 할 작업을 간략히 설명했다. 지금까지 진행한 많은 작업과 마찬가지로, 이 과정에서도 몇 가지 주의해야 할 점이 있다.


## `postfix()` 함수의 변경 사항

`postfix()` 함수는 이제 훨씬 깔끔해졌다:

```c
// 후위 표현식을 파싱하고 이를 나타내는 AST 노드를 반환한다. 
// 식별자는 이미 Text에 있다.
static struct ASTnode *postfix(void) {
  struct ASTnode *n;

  // 기본 표현식을 가져온다
  n = primary();

  // 더 이상 후위 연산자가 없을 때까지 반복한다
  while (1) {
    switch (Token.token) {
    ...
    default:
      return (n);
    }
  }
```

이제 `primary()`를 호출해 식별자나 상수를 가져온다. 그런 다음 `primary()`에서 받은 AST 노드에 후위 연산자를 적용하기 위해 반복문을 사용한다. `[..]`, `.`, `->` 연산자에 대해서는 `array_access()`나 `member_access()` 같은 헬퍼 함수를 호출한다.

여기서 후위 증가 및 후위 감소 연산자를 처리한다. 반복문이 있기 때문에, 이러한 연산자를 두 번 이상 수행하지 않도록 확인해야 한다. 또한 `primary()`에서 받은 AST가 lvalue인지 rvalue인지 확인한다. 왜냐하면 증가나 감소 연산을 수행하려면 메모리 주소가 필요하기 때문이다.


## 새로운 함수 `paren_expression()`

`primary()` 함수가 조금 커져서, 코드 일부를 새로운 함수인 `paren_expression()`로 분리했다. 이 함수는 `(..)`로 둘러싸인 표현식을 파싱한다. 여기에는 캐스트 표현식과 일반 괄호 표현식이 포함된다. 코드는 이전과 거의 동일하므로 여기서 자세히 설명하지 않는다. 이 함수는 캐스트 표현식이나 괄호 표현식을 나타내는 AST 노드를 반환한다.


## `primary()` 함수의 변경 사항

여기서 가장 큰 변화가 발생했다. 우선, 이 함수가 찾는 토큰들을 살펴보자:

 + 'static', 'extern': 로컬 컨텍스트에서만 표현식을 파싱할 수 있기 때문에, 이 토큰들은 에러를 발생시킨다.
 + 'sizeof()'
 + 정수와 문자열 리터럴
 + 식별자: 이는 알려진 타입(예: 'int'), 열거형 이름, typedef 이름, 함수 이름, 배열 이름, 그리고/또는 스칼라 변수 이름일 수 있다. 이 부분이 `primary()`에서 가장 큰 부분을 차지하며, 생각해보면 별도의 함수로 분리하는 것이 더 나을 수도 있다.
 + `(..)`: 이 경우 `paren_expression()` 함수가 호출된다.

코드를 살펴보면, `primary()`는 이제 위의 각 항목에 대해 AST 노드를 생성하여 `postfix()`로 반환한다. 이전에는 이 작업을 `postfix()`에서 수행했지만, 이제는 `primary()`에서 처리한다.


## `member_access()` 함수의 변경 사항

이전 `member_access()` 함수에서는 전역 `Text` 변수가 여전히 식별자를 담고 있었고, `member_access()`가 구조체/공용체 식별자를 나타내는 AST 노드를 직접 생성했다.

현재 `member_access()` 함수에서는 구조체/공용체 식별자에 대한 AST 노드를 인자로 받는다. 이 노드는 배열 요소일 수도 있고, 다른 구조체/공용체의 멤버일 수도 있다.

따라서 이제는 원래 식별자에 대한 리프 AST 노드를 직접 생성하지 않는다. 대신, 기본 주소에서 오프셋을 추가하고 멤버에 대한 포인터를 역참조하기 위한 AST 노드는 여전히 생성한다.

또 다른 차이점은 다음 코드에 있다:

```c
  // 왼쪽 AST 트리가 구조체 또는 공용체인지 확인한다.
  // 맞다면 A_IDENT에서 A_ADDR로 변경하여
  // 해당 주소의 값이 아니라 기본 주소를 얻는다.
  if (!withpointer) {
    if (left->type == P_STRUCT || left->type == P_UNION)
      left->op = A_ADDR;
    else
      fatal("Expression is not a struct/union");
  }
```

`foo.bar`라는 표현식을 생각해 보자. `foo`는 구조체의 이름이고, `bar`는 그 구조체의 멤버다.

`primary()` 함수에서는 `foo`에 대해 A_IDENT AST 노드를 생성한다. 왜냐하면 이 변수가 스칼라 변수(예: `int foo`)인지 구조체(예: `struct fred foo`)인지 구분할 수 없기 때문이다. 이제 이 변수가 구조체나 공용체임을 알았으므로, 기본 주소의 값이 아니라 기본 주소 자체가 필요하다. 따라서 이 코드는 A_IDENT AST 노드의 연산을 식별자에 대한 A_ADDR 연산으로 변환한다.


## 코드 테스트

100개가 넘는 회귀 테스트를 실행하며 놓친 부분을 찾고 수정하는 데 약 두 시간이 걸렸다. 모든 테스트를 다시 통과하니 기분이 좋았다.

`tests/input128.c`는 포인터 체인을 따라갈 수 있는지 확인하는 테스트다. 이번 작업의 핵심 목표였다.

```c
struct foo {
  int val;
  struct foo *next;
};

struct foo head, mid, tail;

int main() {
  struct foo *ptr;
  tail.val= 20; tail.next= NULL;
  mid.val= 15; mid.next= &tail;
  head.val= 10; head.next= &mid;

  ptr= &head;
  printf("%d %d\n", head.val, ptr->val);
  printf("%d %d\n", mid.val, ptr->next->val);
  printf("%d %d\n", tail.val, ptr->next->next->val);
  return(0);
}
```

`tests/input129.c`는 연속으로 두 번 post-increment를 할 수 없는지 확인하는 테스트다.


## 추가 변경 사항: `Linestart`

컴파일러가 스스로 컴파일할 수 있도록 만들기 위해 한 가지 더 변경한 사항이 있다.

스캐너가 '#' 토큰을 찾도록 설정했다. 이 토큰을 발견하면 C 전처리기 라인으로 간주하고 해당 라인을 파싱했다. 하지만 스캐너가 각 라인의 첫 번째 컬럼에서만 이를 찾도록 제한하지 않았다. 그래서 컴파일러가 아래와 같은 소스 코드 라인을 만났을 때:

```c
  while (c == '#') {
```

')'와 '{'가 C 전처리기 라인이 아니라는 이유로 문제가 발생했다.

이제 `Linestart` 변수를 도입해 스캐너가 새로운 라인의 첫 번째 컬럼에 있는지 여부를 표시한다. 주로 변경된 함수는 `scan.c` 파일의 `next()` 함수다. 이 변경 사항이 약간 지저분해 보이기는 하지만 동작은 한다. 나중에 시간을 내어 이 부분을 조금 더 깔끔하게 정리할 필요가 있다. 어쨌든, 첫 번째 컬럼에서 '#'을 발견했을 때만 C 전처리기 라인으로 간주한다.


## 결론 및 다음 단계

컴파일러 작성 여정의 다음 단계에서는 컴파일러 소스 코드를 다시 스스로에게 입력해 보며 발생하는 오류를 살펴보고, 그중 하나 이상을 해결할 예정이다. [다음 단계](../53_Mop_up_pt2/Readme.md)



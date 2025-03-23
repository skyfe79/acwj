# 34부: 열거형과 타입 정의

이번 파트에서는 열거형(enum)과 타입 정의(typedef)를 함께 구현하기로 했다. 두 기능 모두 규모가 작아서 한 번에 다루기 적합했다.

이미 30부에서 열거형의 설계 측면을 다뤘다. 간단히 복습하자면, 열거형은 이름이 붙은 정수 리터럴이다. 여기서 해결해야 할 두 가지 문제가 있다:

1. 열거형 타입 이름을 재정의할 수 없다.
2. 열거형 값의 이름을 재정의할 수 없다.

이를 예제로 살펴보자:

```c
enum fred { x, y, z };
enum fred { a, b };             // fred가 재정의됨
enum jane { x, y };             // x와 y가 재정의됨
```

위 예제에서 볼 수 있듯이, 열거형 값의 목록은 식별자 이름만 있고 타입은 없다. 이는 기존 변수 선언 파싱 코드를 재사용할 수 없음을 의미한다. 따라서 여기서는 별도의 파싱 코드를 작성해야 한다.


## 새로운 키워드와 토큰

문법에 'enum'과 'typedef'라는 두 가지 새로운 키워드를 추가했다. 이와 함께 두 개의 토큰인 T_ENUM과 T_TYPEDEF도 도입했다. 자세한 내용은 `scan.c` 파일의 코드를 참고한다.


## 열거형과 타입 정의를 위한 심볼 테이블 목록

선언된 열거형과 타입 정의의 세부 정보를 기록하기 위해 `data.h` 파일에 두 개의 새로운 심볼 테이블 목록을 추가한다:

```c
extern_ struct symtable *Enumhead,  *Enumtail;    // 열거형 타입과 값의 목록
extern_ struct symtable *Typehead,  *Typetail;    // 타입 정의의 목록
```

또한 `sym.c` 파일에는 각 목록에 항목을 추가하거나 특정 이름을 검색하는 관련 함수가 포함되어 있다. 이 목록의 노드들은 `defs.h` 파일에서 정의된 다음 클래스 중 하나로 표시된다:

```c
  C_ENUMTYPE,                   // 명명된 열거형 타입
  C_ENUMVAL,                    // 명명된 열거형 값
  C_TYPEDEF                     // 명명된 타입 정의
```

두 개의 목록이 있지만 세 개의 노드 클래스가 존재한다. 왜 그럴까? 열거형 값(예: 상단 예제의 `x`와 `y`)은 특정 열거형 타입에 속하지 않는다. 또한, 열거형 타입 이름(예: 상단 예제의 `fred`와 `jane`)은 실제로 아무런 기능을 하지 않지만, 이들의 재정의를 방지해야 한다.

따라서 하나의 열거형 심볼 테이블 목록을 사용해 `C_ENUMTYPE`과 `C_ENUMVAL` 노드를 동일한 목록에 저장한다. 상단의 예제를 사용하면 다음과 같은 구조가 된다:

```
   fred           x            y            z
C_ENUMTYPE -> C_ENUMVAL -> C_ENUMVAL -> C_ENUMVAL
                  0            1            2
```

이 구조는 열거형 심볼 테이블 목록을 검색할 때 `C_ENUMTYPE` 또는 `C_ENUMVAL`을 검색할 수 있는 기능이 필요함을 의미한다.


## Enum 선언 파싱

이 코드를 설명하기 전에, 파싱해야 할 몇 가지 예제를 먼저 살펴보자:

```c
enum fred { a, b, c };                  // a는 0, b는 1, c는 2
enum foo  { d=2, e=6, f };              // d는 2, e는 6, f는 7
enum bar  { g=2, h=6, i } var1;         // var1은 실제로 int 타입
enum      { j, k, l }     var2;         // var2는 실제로 int 타입
```

먼저, enum 파싱은 기존 파싱 코드의 어디에 연결되는가? 구조체(struct)와 공용체(union)와 마찬가지로, 타입을 파싱하는 코드(`decl.c` 파일 내)에 연결된다:

```c
// 현재 토큰을 파싱하고,
// 기본 타입 enum 값과 복합 타입에 대한 포인터를 반환한다.
// 또한 다음 토큰을 스캔한다.
int parse_type(struct symtable **ctype) {
  int type;
  switch (Token.token) {

      // 다음 경우, 파싱 후 ';'가 있으면 타입이 없으므로 -1을 반환
      ...
    case T_ENUM:
      type = P_INT;             // enum은 실제로 int 타입
      enum_declaration();
      if (Token.token == T_SEMI)
        type = -1;
      break;
  }
  ...
}
```

`parse_type()`의 반환 값을 변경하여, 구조체, 공용체, enum 또는 typedef 선언이 실제 타입(식별자 앞에 오는)이 아닌 경우를 식별할 수 있도록 했다.

이제 `enum_declaration()` 코드를 단계별로 살펴보자.

```c
// enum 선언을 파싱한다.
static void enum_declaration(void) {
  struct symtable *etype = NULL;
  char *name;
  int intval = 0;

  // enum 키워드를 건너뛴다.
  scan(&Token);

  // 이후 enum 타입 이름이 있으면,
  // 기존 enum 타입 노드에 대한 포인터를 가져온다.
  if (Token.token == T_IDENT) {
    etype = findenumtype(Text);
    name = strdup(Text);        // 곧 덮어씌워지므로 복사
    scan(&Token);
  }
```

전역 변수 `Text`는 스캔된 단어를 보관하는 유일한 변수이며, `enum foo var1`과 같은 구문을 파싱할 수 있어야 한다. `foo` 이후의 토큰을 스캔하면 `foo` 문자열을 잃게 된다. 따라서 이를 `strdup()`으로 복사해야 한다.

```c
  // 다음 토큰이 LBRACE가 아니면,
  // enum 타입 이름이 있는지 확인한 후 반환한다.
  if (Token.token != T_LBRACE) {
    if (etype == NULL)
      fatals("undeclared enum type:", name);
    return;
  }
```

`enum foo var1`과 같은 선언을 만났고, `enum foo { ...`와 같은 선언이 아니다. 따라서 `foo`는 이미 알려진 enum 타입으로 존재해야 한다. 반환 값은 없어도 되는데, 모든 enum의 타입은 P_INT로 설정되며, 이는 `enum_declaration()`을 호출하는 코드에서 설정된다.

```c
  // LBRACE가 있다. 건너뛴다.
  scan(&Token);

  // enum 타입 이름이 있으면,
  // 이전에 선언되지 않았는지 확인한다.
  if (etype != NULL)
    fatals("enum type redeclared:", etype->name);
  else
    // 이 식별자에 대해 enum 타입 노드를 생성한다.
    etype = addenum(name, C_ENUMTYPE, 0);
```

이제 `enum foo { ...`와 같은 구문을 파싱하고 있으므로, `foo`가 이미 enum 타입으로 선언되지 않았는지 확인해야 한다.

```c
  // 모든 enum 값을 가져오기 위해 반복한다.
  while (1) {
    // 식별자가 있는지 확인한다.
    // 정수 리터럴이 올 경우를 대비해 복사한다.
    ident();
    name = strdup(Text);

    // 이 enum 값이 이전에 선언되지 않았는지 확인한다.
    etype = findenumval(name);
    if (etype != NULL)
      fatals("enum value redeclared:", Text);
```

다시 한번, enum 값 식별자를 `strdup()`으로 복사한다. 또한 이 enum 값 식별자가 이미 정의되지 않았는지 확인한다.

```c
    // 다음 토큰이 '='이면, 건너뛰고
    // 이후의 정수 리터럴을 가져온다.
    if (Token.token == T_ASSIGN) {
      scan(&Token);
      if (Token.token != T_INTLIT)
        fatal("Expected int literal after '='");
      intval = Token.intvalue;
      scan(&Token);
    }
```

이것이 `strdup()`을 사용한 이유다. 정수 리터럴을 스캔하면 전역 변수 `Text`를 덮어쓸 수 있다. 여기서 '='와 정수 리터럴 토큰을 스캔하고, `intval` 변수를 정수 리터럴 값으로 설정한다.

```c
    // 이 식별자에 대해 enum 값 노드를 생성한다.
    // 다음 enum 식별자를 위해 값을 증가시킨다.
    etype = addenum(name, C_ENUMVAL, intval++);

    // 오른쪽 중괄호가 나오면 종료, 아니면 쉼표를 가져온다.
    if (Token.token == T_RBRACE)
      break;
    comma();
  }
  scan(&Token);                 // 오른쪽 중괄호를 건너뛴다.
}
```

이제 enum 값의 이름과 값을 `intval`로 가지고 있다. 이를 `addenum()`을 사용해 enum 심볼 테이블 목록에 추가할 수 있다. 또한 다음 enum 값 식별자를 위해 `intval`을 증가시킨다.


## 열거형 이름 접근하기

현재 열거형 값 이름 목록을 파싱하고, 그 정수 리터럴 값을 심볼 테이블에 저장하는 코드를 가지고 있다. 이 값을 언제 어떻게 검색하고 사용할까?

이 작업은 표현식에서 변수 이름을 사용할 수 있는 지점에서 수행해야 한다. 열거형 이름을 찾으면, 이를 특정 값을 가진 A_INTLIT AST 노드로 변환한다. 이 작업을 수행할 위치는 `expr.c` 파일의 `postfix()` 함수다.

```c
// 포스트픽스 표현식을 파싱하고 이를 나타내는 AST 노드를 반환한다.
// 식별자는 이미 Text에 있다.
static struct ASTnode *postfix(void) {
  struct symtable *enumptr;

  // 식별자가 열거형 값과 일치하면 A_INTLIT 노드를 반환한다.
  if ((enumptr = findenumval(Text)) != NULL) {
    scan(&Token);
    return (mkastleaf(A_INTLIT, P_INT, NULL, enumptr->posn));
  }
  ...
}
```


## 기능 테스트

모든 작업이 완료되었다. 재정의된 enum 타입과 이름을 잘 식별하는지 확인하는 여러 테스트 프로그램이 있다. 그중 `test/input63.c` 코드는 enum이 제대로 동작하는지 보여준다.

```c
int printf(char *fmt);

enum fred { apple=1, banana, carrot, pear=10, peach, mango, papaya };
enum jane { aple=1, bnana, crrot, par=10, pech, mago, paaya };

enum fred var1;
enum jane var2;
enum fred var3;

int main() {
  var1= carrot + pear + mango;
  printf("%d\n", var1);
  return(0);
```

이 코드는 `carrot + pear + mango`(즉, 3+10+12)를 계산해 25를 출력한다.


## Typedefs

이제 enum을 마쳤으니 typedef를 살펴보자. typedef 선언의 기본 문법은 다음과 같다:

```
typedef_declaration: 'typedef' identifier existing_type
                   | 'typedef' identifier existing_type variable_name
                   ;
```

따라서 `typedef` 키워드를 파싱한 후, 뒤따르는 타입을 파싱하고 이름을 가진 C_TYPEDEF 심볼 노드를 생성할 수 있다. 실제 타입의 `type`과 `ctype`은 이 심볼 노드에 저장한다.

파싱 코드는 간단하고 명확하다. `decl.c` 파일의 `parse_type()` 함수에 연결한다:

```c
    case T_TYPEDEF:
      type = typedef_declaration(ctype);
      if (Token.token == T_SEMI)
        type = -1;
      break;
```

다음은 `typedef_declaration()` 함수의 코드다. 이 함수는 선언 뒤에 변수 이름이 오는 경우를 대비해 실제 `type`과 `ctype`을 반환한다.

```c
// typedef 선언을 파싱하고 해당 타입과 ctype을 반환한다.
int typedef_declaration(struct symtable **ctype) {
  int type;

  // typedef 키워드를 건너뛴다.
  scan(&Token);

  // 키워드 뒤에 오는 실제 타입을 가져온다.
  type = parse_type(ctype);

  // typedef 식별자가 이미 존재하는지 확인한다.
  if (findtypedef(Text) != NULL)
    fatals("typedef 재정의", Text);

  // 존재하지 않으면 typedef 리스트에 추가한다.
  addtypedef(Text, type, *ctype, 0, 0);
  scan(&Token);
  return (type);
}
```

코드는 직관적이지만, `parse_type()`을 재귀적으로 호출하는 부분에 주목하자. typedef 이름 뒤에 오는 타입 정의를 파싱하기 위한 코드는 이미 준비되어 있다.


## 타입 정의 검색 및 사용

이제 심볼 테이블 리스트에 typedef 정의 목록이 있다. 이 정의들을 어떻게 사용할까? 우리는 문법에 새로운 타입 키워드를 추가한 것과 같다. 예를 들어:

```c
FILE    *zin;
int32_t cost;
```

이는 타입을 파싱할 때 인식할 수 없는 키워드를 만나면, typedef 리스트에서 해당 키워드를 찾아볼 수 있다는 의미이다. 따라서 `parse_type()` 함수를 다시 수정한다:

```c
    case T_IDENT:
      type = type_of_typedef(Text, ctype);
      break;
```

여기서 `type`과 `ctype`은 `type_of_typedef()` 함수에서 반환된다:

```c
// typedef 이름이 주어지면, 그 이름이 나타내는 타입을 반환한다
int type_of_typedef(char *name, struct symtable **ctype) {
  struct symtable *t;

  // typedef 리스트에서 이름을 찾는다
  t = findtypedef(name);
  if (t == NULL)
    fatals("unknown type", name);
  scan(&Token);
  *ctype = t->ctype;
  return (t->type);
}
```

현재 코드는 "재귀적"으로 동작하지 않는다는 점에 유의한다. 예를 들어, 아래 예제는 파싱하지 못한다:

```c
typedef int FOO;
typedef FOO BAR;
BAR x;                  // x는 BAR 타입 -> FOO 타입 -> int 타입
```

하지만 `tests/input68.c` 파일은 컴파일한다:

```c
int printf(char *fmt);

typedef int FOO;
FOO var1;

struct bar { int x; int y} ;
typedef struct bar BAR;
BAR var2;

int main() {
  var1= 5; printf("%d\n", var1);
  var2.x= 7; var2.y= 10; printf("%d\n", var2.x + var2.y);
  return(0);
}
```

이 코드는 `int` 타입을 `FOO`로 재정의하고, 구조체를 `BAR` 타입으로 재정의한다.


## 결론과 다음 단계

컴파일러 개발 여정의 이번 파트에서는 enum과 typedef를 모두 지원하도록 추가했다. 두 기능 모두 상대적으로 쉽게 구현할 수 있었지만, enum을 위해 다소 많은 파싱 코드를 작성해야 했다. 변수 목록, 구조체 멤버 목록, 공용체 멤버 목록을 위해 동일한 파싱 코드를 재사용할 수 있어서 편했던 것 같다.

typedef를 추가하는 코드는 정말 깔끔하고 간단했다. 하지만 typedef의 typedef를 처리하는 코드를 추가해야 한다. 이 부분도 간단하게 구현할 수 있을 것이다.

컴파일러 개발의 다음 단계에서는 C 전처리기를 도입할 차례다. 이제 구조체, 공용체, enum, typedef를 모두 갖추었으므로, 일반적인 Unix/Linux 라이브러리 함수의 정의를 담은 여러 헤더 파일을 작성할 수 있다. 그리고 이 헤더 파일을 소스 파일에 포함시켜 실제로 유용한 프로그램을 작성할 수 있을 것이다. [다음 단계](../35_Preprocessor/Readme.md)



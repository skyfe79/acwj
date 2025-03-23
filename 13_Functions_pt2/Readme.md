# 13부: 함수, 2부

컴파일러 작성 과정의 이번 파트에서는 함수를 호출하고 값을 반환하는 기능을 추가하려고 한다. 구체적으로 다음과 같은 작업을 수행한다:

+ 함수를 정의하는 기능(이미 구현되어 있음),
+ 단일 값을 인자로 받는 함수를 호출하는 기능(현재는 이 값을 사용할 수 없음),
+ 함수에서 값을 반환하는 기능,
+ 함수 호출을 문장과 표현식으로 모두 사용하는 기능,
+ void 함수는 값을 반환하지 않고, non-void 함수는 반드시 값을 반환하도록 보장하는 기능.

이 기능을 방금 구현했다. 대부분의 시간을 타입 처리에 할애했다. 이제 이에 대해 자세히 설명하겠다.


## 새로운 키워드와 토큰

지금까지 컴파일러에서 8바이트(64비트) `int`를 사용해 왔지만, Gcc는 `int`를 4바이트(32비트)로 처리한다는 사실을 깨달았다. 따라서 `long` 타입을 도입하기로 결정했다. 이제 각 타입의 크기는 다음과 같다:

  + `char`은 1바이트 크기
  + `int`는 4바이트(32비트) 크기
  + `long`은 8바이트(64비트) 크기

또한 'return' 기능도 필요하므로, 새로운 키워드 'long'과 'return'을 추가하고, 이에 해당하는 토큰 T_LONG과 T_RETURN을 도입했다.


## 함수 호출 파싱

현재 함수 호출을 위해 사용하는 BNF 문법은 다음과 같다:

```
  function_call: identifier '(' expression ')'   ;
```

함수는 이름과 괄호 쌍으로 구성된다. 괄호 안에는 정확히 하나의 인자가 있어야 한다. 이 문법은 표현식으로도 사용할 수 있고, 독립적인 문장으로도 사용할 수 있다.

함수 호출 파서인 `funccall()`은 `expr.c` 파일에 있다. 이 함수가 호출될 때, 식별자는 이미 스캔되어 있고 함수 이름은 전역 변수 `Text`에 저장되어 있다:

```c
// 단일 표현식 인자를 가진 함수 호출을 파싱하고 
// 그에 해당하는 AST를 반환한다
struct ASTnode *funccall(void) {
  struct ASTnode *tree;
  int id;

  // 식별자가 정의되었는지 확인한 후, 
  // 리프 노드를 생성한다. XXX 구조적 타입 테스트 추가 필요
  if ((id = findglob(Text)) == -1) {
    fatals("Undeclared function", Text);
  }
  // '(' 토큰을 확인한다
  lparen();

  // 다음 표현식을 파싱한다
  tree = binexpr(0);

  // 함수 호출 AST 노드를 생성한다. 
  // 함수의 반환 타입을 이 노드의 타입으로 저장한다.
  // 또한 함수의 심볼 ID를 기록한다
  tree = mkastunary(A_FUNCCALL, Gsym[id].type, tree, id);

  // ')' 토큰을 확인한다
  rparen();
  return (tree);
}
```

여기서 주석으로 *Add structural type test*라고 남겨두었다. 함수나 변수가 선언될 때, 심볼 테이블은 각각 S_FUNCTION과 S_VARIABLE로 구조적 타입이 표시된다. 이 코드에 식별자가 실제로 S_FUNCTION인지 확인하는 코드를 추가해야 한다.

새로운 단항 AST 노드인 A_FUNCCALL을 생성한다. 자식 노드는 인자로 전달할 단일 표현식이다. 노드에 함수의 심볼 ID를 저장하고, 함수의 반환 타입도 기록한다.


## 더 이상 그 토큰이 필요 없을 때

파싱 과정에서 다음과 같은 문제가 발생한다. 아래 두 경우를 구분해야 한다:

```
   x= fred + jim;
   x= fred(5) + jim;
```

다음 토큰이 '('인지 확인하려면 한 토큰을 미리 살펴봐야 한다. '('가 있으면 함수 호출로 판단한다. 하지만 이렇게 하면 기존 토큰을 잃게 된다. 이 문제를 해결하기 위해 스캐너를 수정했다. 이제 원하지 않는 토큰을 되돌릴 수 있다. 이 토큰은 다음 토큰을 가져올 때 새 토큰 대신 반환된다. `scan.c`의 새로운 코드는 다음과 같다:

```c
// 거부된 토큰을 가리키는 포인터
static struct token *Rejtoken = NULL;

// 방금 스캔한 토큰을 거부한다
void reject_token(struct token *t) {
  if (Rejtoken != NULL)
    fatal("토큰을 두 번 거부할 수 없음");
  Rejtoken = t;
}

// 입력에서 다음 토큰을 스캔하고 반환한다.
// 토큰이 유효하면 1을, 토큰이 더 이상 없으면 0을 반환한다.
int scan(struct token *t) {
  int c, tokentype;

  // 거부된 토큰이 있으면 그것을 반환한다
  if (Rejtoken != NULL) {
    t = Rejtoken;
    Rejtoken = NULL;
    return (1);
  }

  // 정상적인 스캐닝을 계속한다
  ...
}
```


## 함수를 표현식으로 호출하기

이제 `expr.c`에서 변수 이름과 함수 호출을 구분해야 하는 부분을 살펴볼 차례다. 이 로직은 `primary()` 함수에 구현되어 있다. 새로운 코드는 다음과 같다:

```c
// 기본 요소를 파싱하고 이를 나타내는 AST 노드를 반환한다.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
    ...
    case T_IDENT:
      // 이 부분은 변수일 수도 있고 함수 호출일 수도 있다.
      // 다음 토큰을 스캔해서 확인한다
      scan(&Token);

      // '('가 나오면 함수 호출이다
      if (Token.token == T_LPAREN)
        return (funccall());

      // 함수 호출이 아니면 새 토큰을 거부한다
      reject_token(&Token);

      // 일반적인 변수 파싱을 계속한다
      ...
}
```

이 코드는 식별자(identifier)를 만났을 때, 그것이 변수인지 함수 호출인지 판단한다. 만약 다음 토큰이 여는 괄호(`(`)라면 함수 호출로 간주하고 `funccall()` 함수를 호출한다. 그렇지 않으면 일반 변수 파싱 로직을 이어간다. 이 방식으로 변수와 함수 호출을 명확히 구분할 수 있다.


## 함수를 문장으로 호출하기

함수를 문장으로 호출하려고 할 때도 본질적으로 동일한 문제가 발생한다. 이 경우 다음 두 가지를 구분해야 한다:

```
  fred = 2;
  fred(18);
```

따라서 `stmt.c`의 새로운 문장 코드는 위와 유사하다:

```c
// 할당 문장을 파싱하고 AST를 반환
static struct ASTnode *assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int lefttype, righttype;
  int id;

  // 식별자가 있는지 확인
  ident();

  // 이 부분은 변수일 수도 있고 함수 호출일 수도 있다.
  // 다음 토큰이 '('라면 함수 호출이다
  if (Token.token == T_LPAREN)
    return (funccall());

  // 함수 호출이 아니면 할당 문장으로 진행
  ...
}
```

여기서 "원치 않는" 토큰을 거부하지 않아도 된다. 왜냐하면 다음 토큰은 반드시 '=' 또는 '('가 되어야 하기 때문이다. 이 사실을 알고 파서 코드를 작성할 수 있다.


## 반환문 파싱하기

BNF에서 반환문은 다음과 같이 정의된다:

```
  return_statement: 'return' '(' expression ')'  ;
```

파싱 과정은 간단하다: 'return', '(', `binexpr()` 호출, ')', 끝! 더 어려운 부분은 타입을 검사하고, 반환문을 사용할 수 있는지 여부를 확인하는 것이다.

반환문에 도달했을 때 현재 어떤 함수 안에 있는지 알아야 한다. 이를 위해 `data.h`에 전역 변수를 추가했다:

```c
extern_ int Functionid;         // 현재 함수의 심볼 ID
```

이 변수는 `decl.c`의 `function_declaration()`에서 설정된다:

```c
struct ASTnode *function_declaration(void) {
  ...
  // 함수를 심볼 테이블에 추가하고
  // Functionid 전역 변수 설정
  nameslot = addglob(Text, type, S_FUNCTION, endlabel);
  Functionid = nameslot;
  ...
}
```

함수 선언에 진입할 때마다 `Functionid`가 설정되므로, 반환문의 파싱과 의미 체크로 돌아갈 수 있다. 새로운 코드는 `stmt.c`의 `return_statement()`이다:

```c
// 반환문을 파싱하고 AST를 반환
static struct ASTnode *return_statement(void) {
  struct ASTnode *tree;
  int returntype, functype;

  // 함수 반환 타입이 P_VOID라면 값을 반환할 수 없음
  if (Gsym[Functionid].type == P_VOID)
    fatal("void 함수에서는 반환할 수 없음");

  // 'return' '(' 확인
  match(T_RETURN, "return");
  lparen();

  // 다음 표현식 파싱
  tree = binexpr(0);

  // 함수의 타입과 호환되는지 확인
  returntype = tree->type;
  functype = Gsym[Functionid].type;
  if (!type_compatible(&returntype, &functype, 1))
    fatal("타입이 호환되지 않음");

  // 필요하다면 왼쪽을 확장
  if (returntype)
    tree = mkastunary(returntype, functype, tree, 0);

  // A_RETURN 노드 추가
  tree = mkastunary(A_RETURN, P_NONE, tree, 0);

  // ')' 확인
  rparen();
  return (tree);
}
```

새로운 A_RETURN AST 노드는 자식 트리의 표현식을 반환한다. `type_compatible()`을 사용해 표현식이 반환 타입과 일치하는지 확인하고, 필요하다면 확장한다.

마지막으로 함수가 `void`로 선언되었는지 확인한다. `void`로 선언되었다면 이 함수에서는 반환문을 사용할 수 없다.


## 타입 재검토

이전 단계에서 `type_compatible()` 함수를 소개하며 리팩토링이 필요하다고 언급했다. 이제 `long` 타입을 추가하면서 리팩토링이 더욱 필요해졌다. 아래는 `types.c` 파일에 있는 새로운 버전의 코드다. 이전 단계에서 이 함수에 대한 설명을 다시 읽어보는 것도 도움이 될 것이다.

```c
// 두 개의 기본 타입이 주어졌을 때,
// 서로 호환 가능하면 true를 반환하고,
// 그렇지 않으면 false를 반환한다. 또한
// 한 타입을 다른 타입으로 확장해야 하는 경우
// A_WIDEN 연산을 반환한다.
// onlyright가 true인 경우, 왼쪽 타입만 오른쪽 타입으로 확장한다.
int type_compatible(int *left, int *right, int onlyright) {
  int leftsize, rightsize;

  // 타입이 동일하면 호환 가능
  if (*left == *right) { *left = *right = 0; return (1); }
  // 각 타입의 크기를 가져옴
  leftsize = genprimsize(*left);
  rightsize = genprimsize(*right);

  // 크기가 0인 타입은 어떤 타입과도 호환되지 않음
  if ((leftsize == 0) || (rightsize == 0)) return (0);

  // 필요에 따라 타입 확장
  if (leftsize < rightsize) { *left = A_WIDEN; *right = 0; return (1);
  }
  if (rightsize < leftsize) {
    if (onlyright) return (0);
    *left = 0; *right = A_WIDEN; return (1);
  }
  // 남은 타입은 크기가 동일하므로 호환 가능
  *left = *right = 0;
  return (1);
}
```

이제 일반 코드 생성기에서 `genprimsize()`를 호출하며, 이 함수는 `cg.c` 파일에 있는 `cgprimsize()`를 호출해 다양한 타입의 크기를 가져온다:

```c
// P_XXX 순서대로 타입 크기를 나타내는 배열.
// 0은 크기가 없음을 의미. P_NONE, P_VOID, P_CHAR, P_INT, P_LONG
static int psize[] = { 0,       0,      1,     4,     8 };

// P_XXX 타입 값이 주어졌을 때,
// 해당 기본 타입의 크기를 바이트 단위로 반환.
int cgprimsize(int type) {
  // 타입이 유효한지 확인
  if (type < P_NONE || type > P_LONG)
    fatal("Bad type in cgprimsize()");
  return (psize[type]);
}
```

이렇게 하면 타입 크기가 플랫폼에 따라 달라질 수 있다. 다른 플랫폼에서는 다른 타입 크기를 선택할 수 있다. 아마도 P_INTLIT를 `int`가 아닌 `char`로 표시하는 코드도 리팩토링이 필요할 것이다:

```c
  if ((Token.intvalue) >= 0 && (Token.intvalue < 256))
```


## Non-Void 함수가 값을 반환하도록 보장하기

void 함수가 값을 반환하지 못하도록 보장했다면, 이제 non-void 함수가 항상 값을 반환하도록 어떻게 보장할 수 있을까? 이를 위해 함수의 마지막 문장이 return 문이 되도록 해야 한다.

`decl.c` 파일의 `function_declaration()` 함수 하단에는 다음과 같은 코드가 있다:

```c
  struct ASTnode *tree, *finalstmt;
  ...
  // 함수 타입이 P_VOID가 아닌 경우, 
  // 복합 문장의 마지막 AST 연산이 return 문인지 확인
  if (type != P_VOID) {
    finalstmt = (tree->op == A_GLUE) ? tree->right : tree;
    if (finalstmt == NULL || finalstmt->op != A_RETURN)
      fatal("No return for function with non-void type");
  }
```

여기서 주의할 점은 함수에 정확히 하나의 문장만 있는 경우, A_GLUE AST 노드가 없고 트리에 왼쪽 자식만 존재한다는 것이다. 이 경우 해당 자식이 복합 문장이 된다.

이 시점에서 우리는 다음 작업을 수행할 수 있다:

  + 함수를 선언하고, 그 타입을 저장하며, 해당 함수 내에 있음을 기록한다.
  + 단일 인자를 사용해 함수 호출을 수행한다(표현식 또는 문장으로).
  + non-void 함수에서만 반환하며, non-void 함수의 마지막 문장이 return 문이 되도록 강제한다.
  + 반환되는 표현식을 검사하고 함수의 타입 정의와 일치하도록 확장한다.

이제 AST 트리에는 return 문과 함수 호출을 처리하기 위한 A_RETURN 및 A_FUNCCAL 노드가 포함되어 있다. 이들이 어떻게 어셈블리 출력을 생성하는지 살펴보자.


## 단일 인자를 사용하는 이유

여기서 여러분은 의문이 들 수 있다. 왜 함수에 단일 인자를 사용하려는 걸까? 특히 그 인자가 함수 내에서 사용되지도 않는데 말이다.

이유는 간단하다. 우리 언어의 `print x;` 구문을 실제 함수 호출인 `printint(x);`로 대체하고 싶기 때문이다. 이를 위해 실제 C 함수인 `printint()`를 컴파일하고, 컴파일러의 출력 결과와 연결할 수 있다.


## 새로운 AST 노드

`gen.c` 파일의 `genAST()` 함수에는 새로운 코드가 많지 않다:

```c
    case A_RETURN:
      cgreturn(leftreg, Functionid);
      return (NOREG);
    case A_FUNCCALL:
      return (cgcall(leftreg, n->v.id));
```

A_RETURN은 표현식이 아니기 때문에 값을 반환하지 않는다. A_FUNCCALL은 당연히 표현식이다.


## x86-64 출력의 변화

새로운 코드 생성 작업은 플랫폼별 코드 생성기인 `cg.c`에서 이루어진다. 이 부분을 자세히 살펴보자.


### 새로운 타입

먼저, `char`, `int`, `long` 타입이 추가되었고, x86-64 아키텍처에서는 각 타입에 맞는 레지스터 이름을 사용해야 한다:

```c
// 사용 가능한 레지스터 목록과 그 이름
static int freereg[4];
static char *reglist[4] = { "%r8", "%r9", "%r10", "%r11" };
static char *breglist[4] = { "%r8b", "%r9b", "%r10b", "%r11b" };
static char *dreglist[4] = { "%r8d", "%r9d", "%r10d", "%r11d" }
```


### 변수 정의, 로드 및 저장

변수는 이제 세 가지 타입을 가질 수 있다. 생성하는 코드는 이를 반영해야 한다. 다음은 변경된 함수들이다:

```c
// 전역 심볼 생성
void cgglobsym(int id) {
  int typesize;
  // 타입 크기 가져오기
  typesize = cgprimsize(Gsym[id].type);

  fprintf(Outfile, "\t.comm\t%s,%d,%d\n", Gsym[id].name, typesize, typesize);
}

// 변수에서 값을 레지스터로 로드.
// 레지스터 번호 반환
int cgloadglob(int id) {
  // 새 레지스터 할당
  int r = alloc_register();

  // 초기화 코드 출력
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovzbq\t%s(\%%rip), %s\n", Gsym[id].name,
              reglist[r]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovzbl\t%s(\%%rip), %s\n", Gsym[id].name,
              reglist[r]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}

// 레지스터 값을 변수에 저장
int cgstorglob(int r, int id) {
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovb\t%s, %s(\%%rip)\n", breglist[r],
              Gsym[id].name);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovl\t%s, %s(\%%rip)\n", dreglist[r],
              Gsym[id].name);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], Gsym[id].name);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}
```


### 함수 호출

인자가 하나인 함수를 호출하려면, 인자 값을 가진 레지스터를 `%rdi`로 복사해야 한다. 함수가 반환되면, 반환 값을 `%rax`에서 새로운 값을 가질 레지스터로 복사해야 한다:

```c
// 주어진 레지스터에서 인자 하나를 사용해 함수 호출
// 결과를 가진 레지스터 반환
int cgcall(int r, int id) {
  // 새로운 레지스터 할당
  int outr = alloc_register();
  fprintf(Outfile, "\tmovq\t%s, %%rdi\n", reglist[r]);
  fprintf(Outfile, "\tcall\t%s\n", Gsym[id].name);
  fprintf(Outfile, "\tmovq\t%%rax, %s\n", reglist[outr]);
  free_register(r);
  return (outr);
}
```


### 함수 반환 처리

함수 내 어느 지점에서든 반환하기 위해, 함수의 끝에 위치한 레이블로 점프해야 한다. `function_declaration()` 함수에 레이블을 생성하고 이를 심볼 테이블에 저장하는 코드를 추가했다. 반환 값은 `%rax` 레지스터를 통해 전달되므로, 끝 레이블로 점프하기 전에 이 레지스터에 값을 복사해야 한다.

```c
// 함수에서 값을 반환하는 코드 생성
void cgreturn(int reg, int id) {
  // 함수의 타입에 따라 코드 생성
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovzbl\t%s, %%eax\n", breglist[reg]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovl\t%s, %%eax\n", dreglist[reg]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, %%rax\n", reglist[reg]);
      break;
    default:
      fatald("Bad function type in cgreturn:", Gsym[id].type);
  }
  cgjump(Gsym[id].endlabel);
}
```


### 함수 프리앰블과 포스트앰블 변경 사항

프리앰블에는 변경 사항이 없지만, 이전에는 반환 시 `%rax`를 0으로 설정하는 코드가 있었다. 이제 이 코드를 제거해야 한다:

```c
// 함수 포스트앰블을 출력
void cgfuncpostamble(int id) {
  cglabel(Gsym[id].endlabel);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```


### 초기 프리앰블 변경 내용

지금까지 어셈블리 출력의 시작 부분에 `printint()`의 어셈블리 버전을 수동으로 삽입해 왔다. 이제는 실제 C 함수 `printint()`을 컴파일하고 컴파일러의 출력과 링크할 수 있기 때문에 더 이상 이 작업이 필요하지 않다.


## 변경 사항 테스트

새로운 테스트 프로그램 `tests/input14`를 살펴보자:

```c
int fred() {
  return(20);
}

void main() {
  int result;
  printint(10);
  result= fred(15);
  printint(result);
  printint(fred(15)+10);
  return(0);
}
```

먼저 10을 출력하고, `fred()`를 호출해 20을 반환받아 출력한다. 마지막으로 `fred()`를 다시 호출하고, 반환값에 10을 더해 30을 출력한다. 이 예제는 단일 값을 가진 함수 호출과 반환을 보여준다.

테스트 결과는 다음과 같다:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c types.c
./comp1 tests/input14
cc -o out out.s lib/printint.c
./out; true
10
20
30
```

어셈블리 출력을 `lib/printint.c`와 링크한다:

```c
#include <stdio.h>
void printint(long x) {
  printf("%ld\n", x);
}
```


## 거의 C 언어 수준에 도달했다

이번 변경으로 다음과 같은 작업이 가능해졌다:

```
$ cat lib/printint.c tests/input14 > input14.c
$ cc -o out input14.c
$ ./out 
10
20
30
```

다시 말해, 우리가 만든 언어는 C 언어의 부분집합(subset) 수준에 도달했기 때문에 다른 C 함수와 함께 컴파일해 실행 파일을 만들 수 있다. 훌륭한 성과다!


## 결론 및 다음 단계

우리는 함수 호출과 반환 기능을 단순하게 구현했고, 새로운 데이터 타입도 추가했다. 예상대로 쉽지 않은 작업이었지만, 대부분의 변경 사항은 합리적이라고 생각한다.

컴파일러 개발 여정의 다음 단계에서는 라즈베리 파이의 ARM CPU라는 새로운 하드웨어 플랫폼으로 컴파일러를 이식할 것이다. [다음 단계](../14_ARM_Platform/Readme.md)



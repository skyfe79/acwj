# 6부: 변수

방금 컴파일러에 전역 변수를 추가하는 작업을 마쳤다. 예상대로 많은 노력이 필요했다. 또한 이 과정에서 컴파일러의 거의 모든 파일이 수정되었다. 따라서 이번 여정은 길어질 것이다.


## 변수에서 원하는 것

변수를 통해 다음과 같은 기능을 구현하려 한다:

+ 변수를 선언한다
+ 저장된 값을 변수로 가져온다
+ 변수에 값을 할당한다

아래는 테스트 프로그램으로 사용할 `input02`이다:

```
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

가장 눈에 띄는 변화는 문법에 변수 선언, 할당문, 그리고 표현식 내의 변수명이 추가된 점이다. 하지만 그 전에, 변수를 어떻게 구현하는지 먼저 살펴보자.


## 심볼 테이블

모든 컴파일러는 [심볼 테이블](https://en.wikipedia.org/wiki/Symbol_table)이 필요하다. 나중에는 전역 변수 외에도 더 많은 정보를 저장할 것이다. 하지만 지금은 테이블의 엔트리 구조를 먼저 살펴보자 (`defs.h` 파일에 정의됨):

```c
// 심볼 테이블 구조체
struct symtable {
  char *name;                   // 심볼의 이름
};
```

`data.h` 파일에는 심볼 배열이 정의되어 있다:

```c
#define NSYMBOLS        1024            // 심볼 테이블 엔트리의 수
extern_ struct symtable Gsym[NSYMBOLS]; // 전역 심볼 테이블
static int Globs = 0;                   // 다음으로 사용 가능한 전역 심볼 슬롯의 위치
```

`Globs`는 실제로 심볼 테이블을 관리하는 `sym.c` 파일에 있다. 이 파일에는 다음과 같은 관리 함수가 있다:

  + `int findglob(char *s)`: 심볼 s가 전역 심볼 테이블에 있는지 확인한다. 존재하면 해당 슬롯의 위치를 반환하고, 없으면 -1을 반환한다.
  + `static int newglob(void)`: 새로운 전역 심볼 슬롯의 위치를 가져온다. 슬롯이 모두 사용되었으면 오류를 발생시킨다.
  + `int addglob(char *name)`: 전역 심볼을 심볼 테이블에 추가한다. 심볼 테이블에서의 슬롯 번호를 반환한다.

코드는 매우 직관적이므로 여기서 자세히 설명하지 않는다. 이 함수들을 사용하면 심볼을 찾고, 새로운 심볼을 심볼 테이블에 추가할 수 있다.


## 스캐닝과 새로운 토큰

예제 입력 파일을 보면 몇 가지 새로운 토큰이 필요하다:

  + 'int', T_INT로 알려짐
  + '=', T_EQUALS로 알려짐
  + 식별자 이름, T_IDENT로 알려짐

'='의 스캐닝은 `scan()`에 쉽게 추가할 수 있다:

```c
  case '=':
    t->token = T_EQUALS; break;
```

'int' 키워드는 `keyword()`에 추가할 수 있다:

```c
  case 'i':
    if (!strcmp(s, "int"))
      return (T_INT);
    break;
```

식별자의 경우, 이미 `scanident()`를 사용해 단어를 `Text` 변수에 저장하고 있다. 단어가 키워드가 아닐 때 종료하는 대신 T_IDENT 토큰을 반환할 수 있다:

```c
   if (isalpha(c) || '_' == c) {
      // 키워드나 식별자를 읽어옴
      scanident(c, Text, TEXTLEN);

      // 인식된 키워드라면 해당 토큰을 반환
      if (tokentype = keyword(Text)) {
        t->token = tokentype;
        break;
      }
      // 인식된 키워드가 아니므로 식별자임
      t->token = T_IDENT;
      break;
    }
```


## 새로운 문법

이제 입력 언어의 문법 변화를 살펴볼 준비가 되었다. 이전과 마찬가지로 BNF 표기법을 사용해 정의한다:

```
 statements: statement
      |      statement statements
      ;

 statement: 'print' expression ';'
      |     'int'   identifier ';'
      |     identifier '=' expression ';'
      ;

 identifier: T_IDENT
      ;
```

식별자는 T_IDENT 토큰으로 반환되며, 이미 print 문을 파싱하는 코드가 있다. 하지만 이제 세 가지 타입의 문이 있으므로, 각각을 처리하는 함수를 작성하는 것이 합리적이다. `stmt.c` 파일의 최상위 `statements()` 함수는 이제 다음과 같다:

```c
// 하나 이상의 문을 파싱
void statements(void) {

  while (1) {
    switch (Token.token) {
    case T_PRINT:
      print_statement();
      break;
    case T_INT:
      var_declaration();
      break;
    case T_IDENT:
      assignment_statement();
      break;
    case T_EOF:
      return;
    default:
      fatald("Syntax error, token", Token.token);
    }
  }
}
```

기존의 print 문 코드를 `print_statement()` 함수로 옮겼으며, 이는 직접 확인할 수 있다.


## 변수 선언

변수 선언에 대해 살펴보자. 앞으로 다양한 타입의 선언을 다룰 예정이므로, 새로운 파일 `decl.c`에서 작업을 진행한다.

```c
// 변수 선언을 파싱하는 함수
void var_declaration(void) {

  // 'int' 토큰 다음에 식별자와 세미콜론이 오는지 확인한다.
  // 이 시점에서 Text 버퍼에는 식별자 이름이 저장된다.
  // 이를 알려진 식별자로 추가한다.
  match(T_INT, "int");
  ident();
  addglob(Text);
  genglobsym(Text);
  semi();
}
```

`ident()`와 `semi()` 함수는 `match()` 함수를 감싼 래퍼 함수다:

```c
void semi(void)  { match(T_SEMI, ";"); }
void ident(void) { match(T_IDENT, "identifier"); }
```

다시 `var_declaration()` 함수로 돌아가서, `Text` 버퍼에 식별자를 스캔한 후에는 `addglob(Text)`를 통해 전역 심볼 테이블에 이를 추가할 수 있다. 현재 코드에서는 변수를 여러 번 선언할 수 있도록 허용한다.


## 할당문

`stmt.c` 파일에 있는 `assignment_statement()` 함수의 코드는 다음과 같다:

```c
void assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int id;

  // 식별자가 있는지 확인
  ident();

  // 정의된 변수인지 확인한 후 리프 노드를 생성
  if ((id = findglob(Text)) == -1) {
    fatals("Undeclared variable", Text);
  }
  right = mkastleaf(A_LVIDENT, id);

  // 등호(=)가 있는지 확인
  match(T_EQUALS, "=");

  // 다음 표현식을 파싱
  left = binexpr(0);

  // 할당 AST 트리 생성
  tree = mkastnode(A_ASSIGN, left, right, 0);

  // 할당을 위한 어셈블리 코드 생성
  genAST(tree, -1);
  genfreeregs();

  // 다음 세미콜론(;)을 확인
  semi();
}
```

여기서 두 가지 새로운 AST 노드 타입이 등장한다. `A_ASSIGN`은 왼쪽 자식에 있는 표현식을 오른쪽 자식에 할당한다. 그리고 오른쪽 자식은 `A_LVIDENT` 노드가 된다.

왜 이 노드를 `A_LVIDENT`라고 명명했을까? 이 노드는 *lvalue* 식별자를 나타내기 때문이다. 그렇다면 [lvalue](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue)란 무엇인가?

lvalue는 특정 위치에 연결된 값을 의미한다. 여기서는 변수의 값을 저장하는 메모리 주소를 가리킨다. 다음과 같은 코드를 작성할 때:

```
   area = width * height;
```

우리는 오른쪽 표현식의 결과(즉, *rvalue*)를 왼쪽 변수(즉, *lvalue*)에 할당한다. *rvalue*는 특정 위치에 고정되지 않는다. 이 경우, 표현식의 결과는 임의의 레지스터에 저장될 것이다.

또한, 할당문의 문법이 다음과 같음에도 불구하고:

```
  identifier '=' expression ';'
```

우리는 표현식을 `A_ASSIGN` 노드의 왼쪽 서브 트리로 만들고, `A_LVIDENT`의 세부 정보를 오른쪽 서브 트리에 저장한다. 왜냐하면 변수에 값을 저장하기 전에 표현식을 먼저 평가해야 하기 때문이다.


## AST 구조 변경 사항

이제 A_INTLIT AST 노드에는 정수 리터럴 값을 저장하거나, A_IDENT AST 노드에는 심볼의 세부 정보를 저장해야 한다. 이를 위해 AST 구조에 *union*을 추가했다 (`defs.h` 파일에 있음):

```c
// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  struct ASTnode *left;         // 왼쪽 및 오른쪽 자식 트리
  struct ASTnode *right;
  union {
    int intvalue;               // A_INTLIT의 경우, 정수 값
    int id;                     // A_IDENT의 경우, 심볼 슬롯 번호
  } v;
};
```


## Assignment Code 생성

이제 `gen.c` 파일의 `genAST()` 함수의 변경 사항을 살펴보자.

```c
int genAST(struct ASTnode *n, int reg) {
  int leftreg, rightreg;

  // 왼쪽과 오른쪽 서브 트리의 값을 가져옴
  if (n->left)
    leftreg = genAST(n->left, -1);
  if (n->right)
    rightreg = genAST(n->right, leftreg);

  switch (n->op) {
  ...
    case A_INTLIT:
    return (cgloadint(n->v.intvalue));
  case A_IDENT:
    return (cgloadglob(Gsym[n->v.id].name));
  case A_LVIDENT:
    return (cgstorglob(reg, Gsym[n->v.id].name));
  case A_ASSIGN:
    // 작업이 이미 완료되었으므로 결과를 반환
    return (rightreg);
  default:
    fatald("Unknown AST operator", n->op);
  }
```

왼쪽 AST 자식을 먼저 평가하고, 왼쪽 서브 트리의 값을 보유한 레지스터 번호를 반환한다. 이 레지스터 번호를 오른쪽 서브 트리에 전달한다. 이 작업은 A_LVIDENT 노드에서 필요하며, `cg.c` 파일의 `cgstorglob()` 함수가 할당 표현식의 rvalue 결과를 보유한 레지스터를 알 수 있도록 한다.

다음 AST 트리를 고려해보자.

```
           A_ASSIGN
          /        \
     A_INTLIT   A_LVIDENT
        (3)        (5)
```

A_INTLIT 연산을 평가하기 위해 `leftreg = genAST(n->left, -1);`를 호출한다. 이는 `return (cgloadint(n->v.intvalue));`를 반환하며, 값 3을 레지스터에 로드하고 레지스터 ID를 반환한다.

그런 다음, A_LVIDENT 연산을 평가하기 위해 `rightreg = genAST(n->right, leftreg);`를 호출한다. 이는 `return (cgstorglob(reg, Gsym[n->v.id].name));`를 반환하며, 레지스터를 `Gsym[5]`에 있는 이름의 변수에 저장한다.

그 후 A_ASSIGN 케이스로 전환한다. 모든 작업이 이미 완료되었으므로, rvalue는 여전히 레지스터에 남아 있으므로 그대로 두고 반환한다. 나중에 다음과 같은 표현식을 처리할 수 있다.

```
  a= b= c = 0;
```

여기서 할당은 단순한 문장이 아니라 표현식이기도 하다.


## x86-64 코드 생성

이전 `cgload()` 함수의 이름을 `cgloadint()`로 변경한 것을 눈치챘을 것이다. 이는 더 구체적인 이름이다. 이제 전역 변수에서 값을 로드하는 함수를 추가한다(`cg.c`에 위치):

```c
int cgloadglob(char *identifier) {
  // 새로운 레지스터를 할당
  int r = alloc_register();

  // 레지스터를 초기화하는 코드를 출력
  fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", identifier, reglist[r]);
  return (r);
}
```

마찬가지로, 레지스터의 값을 변수에 저장하는 함수도 필요하다:

```c
// 레지스터의 값을 변수에 저장
int cgstorglob(int r, char *identifier) {
  fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], identifier);
  return (r);
}
```

또한, 새로운 전역 정수 변수를 생성하는 함수도 필요하다:

```c
// 전역 심볼 생성
void cgglobsym(char *sym) {
  fprintf(Outfile, "\t.comm\t%s,8,8\n", sym);
}
```

물론, 파서가 이 코드에 직접 접근하는 것은 허용할 수 없다. 대신, `gen.c`에 있는 일반 코드 생성기 함수를 인터페이스로 사용한다:

```c
void genglobsym(char *s) { cgglobsym(s); }
```


## 표현식에서 변수 사용하기

이제 변수에 값을 할당할 수 있다. 그렇다면 표현식 안에서 변수의 값을 어떻게 가져올 수 있을까? 이미 정수 리터럴을 가져오는 `primary()` 함수가 있다. 이 함수를 수정해 변수의 값도 가져올 수 있게 해보자.

```c
// 기본 요소를 파싱하고 이를 나타내는 AST 노드를 반환한다.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
  case T_INTLIT:
    // INTLIT 토큰의 경우, 이를 위한 리프 AST 노드를 생성한다.
    n = mkastleaf(A_INTLIT, Token.intvalue);
    break;

  case T_IDENT:
    // 해당 식별자가 존재하는지 확인한다.
    id = findglob(Text);
    if (id == -1)
      fatals("Unknown variable", Text);

    // 이를 위한 리프 AST 노드를 생성한다.
    n = mkastleaf(A_IDENT, id);
    break;

  default:
    fatald("Syntax error, token", Token.token);
  }

  // 다음 토큰을 스캔하고 리프 노드를 반환한다.
  scan(&Token);
  return (n);
}
```

T_IDENT 케이스에서 변수가 선언되었는지 확인하는 구문 검사를 주목하자. 이는 변수를 사용하기 전에 선언되었는지 확인하는 중요한 단계다.

또한 변수의 값을 가져오는 AST 리프 노드는 A_IDENT 노드다. 반면 변수에 값을 저장하는 리프 노드는 A_LVIDENT 노드다. 이것이 *rvalues*와 *lvalues*의 차이점이다.


## 직접 해보기

변수 선언에 대한 내용은 이 정도로 마무리하고, `input02` 파일로 직접 테스트해 보자.

```
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

`make test` 명령어를 실행하면 다음과 같은 결과를 얻을 수 있다.

```
$ make test
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
               stmt.c sym.c tree.c
...
./comp1 input02
cc -o out out.s
./out
17
```

결과에서 볼 수 있듯이, `fred + jim`을 계산했고, 이는 5 + 12로 17이 된다. `out.s` 파일에 새로 추가된 어셈블리 코드는 다음과 같다.

```
        .comm   fred,8,8                # fred 선언
        .comm   jim,8,8                 # jim 선언
        ...
        movq    $5, %r8
        movq    %r8, fred(%rip)         # fred = 5
        movq    $12, %r8
        movq    %r8, jim(%rip)          # jim = 12
        movq    fred(%rip), %r8
        movq    jim(%rip), %r9
        addq    %r8, %r9                # fred + jim
```


## 기타 변경 사항

몇 가지 다른 변경 사항도 있었다. 기억나는 주요 변경 사항 중 하나는 `misc.c` 파일에 몇 가지 헬퍼 함수를 추가한 것이다. 이 함수들은 치명적인 오류를 더 쉽게 보고할 수 있도록 도와준다:

```c
// 치명적인 오류 메시지 출력
void fatal(char *s) {
  fprintf(stderr, "%s on line %d\n", s, Line); exit(1);
}

void fatals(char *s1, char *s2) {
  fprintf(stderr, "%s:%s on line %d\n", s1, s2, Line); exit(1);
}

void fatald(char *s, int d) {
  fprintf(stderr, "%s:%d on line %d\n", s, d, Line); exit(1);
}

void fatalc(char *s, int c) {
  fprintf(stderr, "%s:%c on line %d\n", s, c, Line); exit(1);
}
```


## 결론 및 다음 단계

상당히 많은 작업을 마쳤다. 심볼 테이블 관리를 위한 초기 코드를 작성했다. 두 가지 새로운 구문 타입을 처리했다. 새로운 토큰과 AST 노드 타입을 추가했다. 마지막으로 올바른 x86-64 어셈블리 코드를 생성하기 위한 코드를 작성했다.

몇 가지 예제 입력 파일을 작성해 컴파일러가 제대로 동작하는지 확인해 보자. 특히 구문 오류와 변수를 선언하지 않고 사용하는 경우 같은 의미 오류를 잘 감지하는지 테스트해 보자.

컴파일러 개발의 다음 단계에서는 여섯 가지 비교 연산자를 언어에 추가할 예정이다. 이를 통해 그다음 단계에서 제어 구조를 구현할 수 있게 된다. [다음 단계](../07_Comparisons/Readme.md)



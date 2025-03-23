# 20장: 문자와 문자열 리터럴

컴파일러로 "Hello world"를 출력하고 싶은 마음이 오래전부터 있었다. 이제 포인터와 배열을 다뤘으니, 이번 장에서는 문자와 문자열 리터럴을 추가할 차례다.

문자와 문자열 리터럴은 리터럴 값(즉, 바로 보이는 값)이다. 문자 리터럴은 작은따옴표로 둘러싸인 단일 문자를 의미한다. 문자열 리터럴은 큰따옴표로 둘러싸인 일련의 문자를 의미한다.

C 언어에서 문자와 문자열 리터럴은 정말 복잡하다. 여기서는 가장 기본적인 백슬래시 이스케이프 문자만 구현할 것이다. 또한, SubC의 문자와 문자열 리터럴 스캐닝 코드를 빌려와 작업을 더 쉽게 할 예정이다.

이번 장은 짧지만, 결국 "Hello world"를 출력하는 것으로 마무리될 것이다.


## 새로운 토큰 추가

우리 언어에 단 하나의 새로운 토큰 T_STRLIT를 추가해야 한다. 이 토큰은 T_IDENT와 매우 유사하다. 토큰과 관련된 텍스트는 토큰 구조 자체가 아니라 전역 변수 `Text`에 저장된다.


## 문자 리터럴 스캔하기

문자 리터럴은 작은따옴표로 시작하고, 단일 문자 정의가 이어지며, 다시 작은따옴표로 끝난다. 이 단일 문자를 해석하는 코드는 복잡하므로 `scan.c` 파일의 `scan()` 함수를 수정하여 호출하도록 한다:

```c
      case '\'':
      // 작은따옴표가 나오면
      // 문자 리터럴 값을 스캔하고
      // 뒤따르는 작은따옴표를 확인한다
      t->intvalue = scanch();
      t->token = T_INTLIT;
      if (next() != '\'')
        fatal("Expected '\\'' at end of char literal");
      break;
```

문자 리터럴을 `char` 타입의 정수 리터럴로 처리할 수 있다. 여기서는 ASCII만 다루고 유니코드는 고려하지 않는 전제로 접근한다. 이 방식이 현재 구현의 기본이다.


### `scanch()` 함수 코드

`scanch()` 함수의 코드는 SubC에서 가져왔으며 몇 가지 단순화 작업을 거쳤다:

```c
// 문자나 문자열 리터럴에서 다음 문자를 반환한다
static int scanch(void) {
  int c;

  // 다음 입력 문자를 가져오고 백슬래시로 시작하는 메타 문자를 해석한다
  c = next();
  if (c == '\\') {
    switch (c = next()) {
      case 'a':  return '\a';
      case 'b':  return '\b';
      case 'f':  return '\f';
      case 'n':  return '\n';
      case 'r':  return '\r';
      case 't':  return '\t';
      case 'v':  return '\v';
      case '\\': return '\\';
      case '"':  return '"' ;
      case '\'': return '\'';
      default:
        fatalc("unknown escape sequence", c);
    }
  }
  return (c);                   // 일반적인 문자를 반환한다!
}
```

이 코드는 대부분의 이스케이프 문자 시퀀스를 인식하지만, 8진수 문자 코드나 복잡한 시퀀스는 처리하지 않는다.


## 문자열 리터럴 스캔하기

문자열 리터럴은 큰따옴표로 시작하고, 그 뒤에 0개 이상의 문자가 오며, 마지막에 다시 큰따옴표로 끝난다. 문자 리터럴과 마찬가지로 `scan()` 함수 내에서 별도의 함수를 호출해야 한다:

```c
    case '"':
      // 문자열 리터럴을 스캔
      scanstr(Text);
      t->token= T_STRLIT;
      break;
```

새로운 T_STRLIT 토큰을 생성하고, 문자열을 `Text` 버퍼에 스캔한다. 다음은 `scanstr()` 함수의 코드다:

```c
// 입력 파일에서 문자열 리터럴을 스캔하고,
// buf[]에 저장한다. 문자열의 길이를 반환한다.
static int scanstr(char *buf) {
  int i, c;

  // 버퍼 공간이 충분한 동안 반복
  for (i=0; i<TEXTLEN-1; i++) {
    // 다음 문자를 가져와 buf에 추가
    // 끝나는 큰따옴표를 만나면 반환
    if ((c = scanch()) == '"') {
      buf[i] = 0;
      return(i);
    }
    buf[i] = c;
  }
  // buf[] 공간이 부족
  fatal("String literal too long");
  return(0);
}
```

이 코드는 직관적으로 이해할 수 있다. 스캔된 문자열을 NUL로 종료하고, `Text` 버퍼가 오버플로우되지 않도록 보장한다. 개별 문자를 스캔하기 위해 `scanch()` 함수를 사용한다는 점에 주목한다.


## 문자열 리터럴 파싱

앞서 언급했듯이, 문자 리터럴은 이미 다룬 정수 리터럴과 동일하게 처리된다. 그렇다면 문자열 리터럴은 어디에 있을까? 1985년 Jeff Lee가 작성한 [C 언어 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)을 보면 다음과 같은 내용을 확인할 수 있다:

```
primary_expression
        : IDENTIFIER
        | CONSTANT
        | STRING_LITERAL
        | '(' expression ')'
        ;
```

이를 통해 `expr.c` 파일의 `primary()` 함수를 수정해야 한다는 것을 알 수 있다:

```c
// primary factor를 파싱하고 이를 나타내는 AST 노드를 반환한다.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;


  switch (Token.token) {
  case T_STRLIT:
    // STRLIT 토큰의 경우, 어셈블리 코드를 생성한다.
    // 그리고 이를 위한 리프 AST 노드를 만든다. id는 문자열의 레이블이다.
    id= genglobstr(Text);
    n= mkastleaf(A_STRLIT, P_CHARPTR, id);
    break;
```

현재는 익명의 전역 문자열을 만들려고 한다. 이 문자열은 메모리에 모든 문자가 저장되어 있어야 하며, 이를 참조할 방법도 필요하다. 이 새로운 문자열로 심볼 테이블을 오염시키고 싶지 않기 때문에, 문자열에 대한 레이블을 할당하고 이 레이블 번호를 해당 문자열 리터럴의 AST 노드에 저장하기로 했다. 또한 새로운 AST 노드 타입인 A_STRLIT가 필요하다. 이 레이블은 문자열 내 문자 배열의 베이스 역할을 하므로, P_CHARPTR 타입이어야 한다.

`genglobstr()` 함수에 의해 생성되는 어셈블리 출력에 대해서는 곧 다시 다룰 예정이다.


### AST 트리 예제

현재 문자열 리터럴은 익명 포인터로 처리된다. 다음은 해당 문장의 AST 트리다:

```c
  char *s;
  s= "Hello world";

  A_STRLIT rval label L2
  A_IDENT s
A_ASSIGN
```

두 타입이 동일하기 때문에 스케일링이나 확장이 필요하지 않다.


## 어셈블리 출력 생성하기

일반적인 코드 생성기에서는 거의 변화가 없다. 새로운 문자열을 저장하기 위한 함수가 필요하다. 문자열을 위한 레이블을 할당하고, 그 내용을 출력한다(`gen.c` 파일에서):

```c
int genglobstr(char *strvalue) {
  int l= genlabel();
  cgglobstr(l, strvalue);
  return(l);
}
```

또한 A_STRLIT AST 노드 타입을 인식하고, 이를 위한 어셈블리 코드를 생성해야 한다. `genAST()` 함수에서:

```c
    case A_STRLIT:
        return (cgloadglobstr(n->v.id));
```


## x86-64 어셈블리 출력 생성

이제 실제로 새로운 어셈블리 출력 함수를 살펴본다. 두 가지 함수가 있다: 하나는 문자열 저장소를 생성하는 함수이고, 다른 하나는 문자열의 베이스 주소를 로드하는 함수이다.

```c
// 전역 문자열과 시작 레이블 생성
void cgglobstr(int l, char *strvalue) {
  char *cptr;
  cglabel(l);
  for (cptr= strvalue; *cptr; cptr++) {
    fprintf(Outfile, "\t.byte\t%d\n", *cptr);
  }
  fprintf(Outfile, "\t.byte\t0\n");
}

// 전역 문자열의 레이블 번호를 받아
// 새로운 레지스터에 해당 주소를 로드
int cgloadglobstr(int id) {
  // 새로운 레지스터 할당
  int r = alloc_register();
  fprintf(Outfile, "\tleaq\tL%d(\%%rip), %s\n", id, reglist[r]);
  return (r);
}
```

다음 예제로 돌아가 보자:
```c
  char *s;
  s= "Hello world";
```

이에 대한 어셈블리 출력은 다음과 같다:

```
L2:     .byte   72              # 익명 문자열
        .byte   101
        .byte   108
        .byte   108
        .byte   111
        .byte   32
        .byte   119
        .byte   111
        .byte   114
        .byte   108
        .byte   100
        .byte   0
        ...
        leaq    L2(%rip), %r8   # L2의 주소 로드
        movq    %r8, s(%rip)    # 그리고 s에 저장
```


## 기타 변경 사항

이번 작업의 테스트 프로그램을 작성하던 중 기존 코드에서 또 다른 버그를 발견했다. 정수 값을 포인터가 가리키는 타입의 크기에 맞게 스케일링할 때, 스케일이 1인 경우 아무 작업도 하지 않아야 하는데 이를 잊고 있었다. `types.c` 파일의 `modify_type()` 함수 코드는 이제 다음과 같다:

```c
    // 왼쪽이 정수 타입이고, 오른쪽이 포인터 타입이며
    // 원본 타입의 크기가 1보다 큰 경우: 왼쪽을 스케일링
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1)
        return (mkastunary(A_SCALE, rtype, tree, rsize));
      else
        return (tree);          // 크기가 1이면 스케일링 필요 없음
    }
```

이전에는 `return (tree)`를 빼먹어서 `char *` 포인터를 스케일링하려고 할 때 NULL 트리를 반환했다.


## 결론과 다음 단계

이제 텍스트를 출력할 수 있게 되어 정말 기쁘다:

```
$ make test
./comp1 tests/input21.c
cc -o out out.s lib/printint.c
./out
10
Hello world
```

이번 작업의 대부분은 문자와 문자열 리터럴 구분자와 그 안에 있는 이스케이프 문자를 처리하기 위해 어휘 분석기를 확장하는 데 집중했다. 하지만 코드 생성기에도 약간의 작업을 진행했다.

컴파일러 작성 여정의 다음 단계에서는 컴파일러가 인식하는 언어에 몇 가지 이진 연산자를 추가할 것이다. [다음 단계](../21_More_Operators/Readme.md)



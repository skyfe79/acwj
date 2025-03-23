# 35부: C 전처리기

컴파일러 개발 과정의 이번 단계에서는 외부 C 전처리기를 지원하도록 추가했고, `extern` 키워드도 언어에 포함시켰다.

이제 [헤더 파일](https://www.tutorialspoint.com/cprogramming/c_header_files.htm)을 작성할 수 있는 단계에 이르렀으며, 헤더 파일에 주석을 추가할 수도 있다. 이렇게 진전이 이루어지니 기분이 좋다.


## C 전처리기

C 전처리기 자체에 대해 깊이 다루지는 않겠다. 비록 C 환경에서 매우 중요한 부분이지만, 대신 다음 두 글을 읽어보길 권한다:

+ [C Preprocessor](https://en.wikipedia.org/wiki/C_preprocessor) - *Wikipedia*
+ [C Preprocessor and Macros](https://www.programiz.com/c-programming/c-preprocessor-macros) - *www.programiz.com*

이 글들은 C 전처리기의 기본 개념부터 매크로 사용법까지 상세히 설명한다. C 언어를 더 깊이 이해하는 데 큰 도움이 될 것이다.


## C 전처리기 통합하기

[SubC](http://www.t3x.org/subc/)와 같은 다른 컴파일러에서는 전처리기가 언어에 내장되어 있다. 여기서는 외부 시스템 C 전처리기를 사용하기로 결정했으며, 일반적으로 [GNU C 전처리기](https://gcc.gnu.org/onlinedocs/cpp/)를 활용한다.

이를 어떻게 구현했는지 설명하기 전에, 먼저 전처리기가 동작하면서 삽입하는 라인들을 살펴보자.

다음은 짧은 프로그램 예제이다(라인 번호 포함):

```c
1 #include <stdio.h>
2 
3 int main() {
4   printf("Hello world\n");
5   return(0);
6 }
```

이 파일을 전처리기가 처리한 후 컴파일러가 받을 수 있는 출력은 다음과 같다:

```c
# 1 "z.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "z.c"
# 1 "include/stdio.h" 1
# 1 "include/stddef.h" 1

typedef long size_t;
# 5 "include/stdio.h" 2

typedef char * FILE;

FILE *fopen(char *pathname, char *mode);
...
# 2 "z.c" 2

int main() {
  printf("Hello world\n");
  return(0);
}
```

각 전처리기 라인은 '#'으로 시작하고, 그 뒤에 다음 라인 번호와 해당 라인이 어느 파일에서 왔는지 파일 이름이 온다. 일부 라인 끝에 있는 숫자들은 정확히 무엇을 의미하는지 알 수 없다. 한 파일이 다른 파일을 포함할 때, 포함을 한 파일의 라인 번호를 나타내는 것 같다.

이제 전처리기를 컴파일러와 통합하는 방법을 설명하겠다. `popen()`을 사용해 전처리기 프로세스로부터 파이프를 열고, 전처리기에 입력 파일을 처리하도록 지시할 것이다. 그런 다음 렉시컬 스캐너를 수정해 전처리기 라인을 식별하고, 현재 라인 번호와 처리 중인 파일 이름을 설정할 것이다.


## `main.c` 파일 수정

`data.h` 파일에 새로운 전역 변수 `char *Infilename`을 정의했다. 이제 `main.c` 파일의 `do_compile()` 함수에서 다음과 같이 처리한다:

```c
// 입력 파일 이름을 받아서 해당 파일을 어셈블리 코드로 컴파일한다. 새 파일 이름을 반환한다.
static char *do_compile(char *filename) {
  char cmd[TEXTLEN];
  ...
  // 전처리기 명령어 생성
  snprintf(cmd, TEXTLEN, "%s %s %s", CPPCMD, INCDIR, filename);

  // 전처리기 파이프 열기
  if ((Infile = popen(cmd, "r")) == NULL) {
    fprintf(stderr, "Unable to open %s: %s\n", filename, strerror(errno));
    exit(1);
  }
  Infilename = filename;
```

이 코드는 직관적으로 이해할 수 있지만, `CPPCMD`와 `INCDIR`이 어디서 오는지 설명하지 않았다.

`CPPCMD`는 `defs.h` 파일에서 전처리기 명령어 이름으로 정의된다:

```c
#define CPPCMD "cpp -nostdinc -isystem "
```

이 명령어는 Gnu 전처리기에게 표준 포함 디렉토리인 `/usr/include`를 사용하지 말고, 대신 `-isystem` 다음에 오는 `INCDIR`을 사용하라고 지시한다.

`INCDIR`은 실제로 `Makefile`에서 정의된다. 이는 설정 시간에 변경할 수 있는 값을 넣는 일반적인 장소다:

```make
# 포함 디렉토리 위치와 컴파일러 바이너리 설치 위치 정의
INCDIR=/tmp/include
BINDIR=/tmp
```

이제 컴파일러 바이너리는 다음과 같은 `Makefile` 규칙으로 컴파일된다:

```make
cwj: $(SRCS) $(HSRCS)
        cc -o cwj -g -Wall -DINCDIR=\"$(INCDIR)\" $(SRCS)
```

이 규칙은 `/tmp/include` 값을 `INCDIR`로 컴파일에 전달한다. 이제 `/tmp/include` 디렉토리는 언제 생성되고, 무엇이 그곳에 들어가는지 궁금할 것이다.


## 첫 번째 헤더 파일 세트

이 영역의 `include/` 디렉토리에서 컴파일러가 처리할 수 있는 간단한 헤더 파일을 만들었다. 실제 시스템 헤더 파일은 다음과 같은 라인을 포함하고 있어 사용할 수 없다:

```c
extern int _IO_feof (_IO_FILE *__fp) __attribute__ ((__nothrow__ , __leaf__));
extern int _IO_ferror (_IO_FILE *__fp) __attribute__ ((__nothrow__ , __leaf__));
```

이런 코드는 컴파일러에 문제를 일으킬 수 있다. 이제 `Makefile`에 우리만의 헤더 파일을 `INCDIR` 디렉토리로 복사하는 규칙을 추가했다:

```make
install: cwj
        mkdir -p $(INCDIR)
        rsync -a include/. $(INCDIR)
        cp cwj $(BINDIR)
        chmod +x $(BINDIR)/cwj
```


## 전처리기 입력 스캔하기

이제 우리는 입력 파일에서 직접 읽는 대신, 전처리기가 처리한 출력을 읽고 있다. 전처리기 라인을 인식하고, 다음 라인 번호와 해당 라인이 속한 파일 이름을 설정해야 한다.

이미 라인 번호를 증가시키는 기능을 담당하는 스캐너를 수정했다. `scan.c` 파일에서 `scan()` 함수를 다음과 같이 변경했다:

```c
// 입력 파일에서 다음 문자를 가져온다.
static int next(void) {
  int c, l;

  if (Putback) {                        // 되돌려진 문자가 있다면
    c = Putback;                        // 그것을 사용한다
    Putback = 0;
    return (c);
  }

  c = fgetc(Infile);                    // 입력 파일에서 읽는다

  while (c == '#') {                    // 전처리기 문장을 만났다
    scan(&Token);                       // 라인 번호를 l에 저장
    if (Token.token != T_INTLIT)
      fatals("Expecting pre-processor line number, got:", Text);
    l = Token.intvalue;

    scan(&Token);                       // 파일 이름을 Text에 저장
    if (Token.token != T_STRLIT)
      fatals("Expecting pre-processor file name, got:", Text);

    if (Text[0] != '<') {               // 실제 파일 이름이라면
      if (strcmp(Text, Infilename))     // 현재 파일 이름과 다를 때
        Infilename = strdup(Text);      // 저장하고 라인 번호를 업데이트
      Line = l;
    }

    while ((c = fgetc(Infile)) != '\n'); // 라인 끝까지 건너뛴다
    c = fgetc(Infile);                  // 다음 문자를 가져온다
  }

  if ('\n' == c)
    Line++;                             // 라인 카운트 증가
  return (c);
}
```

연속된 전처리기 라인이 있을 수 있으므로 'while' 루프를 사용한다. 다행히 `scan()`을 재귀적으로 호출해 라인 번호를 T_INTLIT로, 파일 이름을 T_STRLIT로 스캔할 수 있다.

코드는 '<' ... '>'로 둘러싸인 파일 이름은 무시한다. 이는 실제 파일 이름을 나타내지 않기 때문이다. 전역 변수 `Text`에 있는 파일 이름을 `strdup()`로 복제해야 한다. 그러나 `Text`의 이름이 이미 `Infilename`과 동일하다면 복제할 필요가 없다.

라인 번호와 파일 이름을 얻은 후, 라인 끝까지 읽고 한 문자를 더 읽은 다음, 원래의 문자 스캔 코드로 돌아간다.

이렇게 하면 C 전처리기를 컴파일러와 통합하는 데 필요한 모든 작업이 완료된다. 처음에는 이 작업이 복잡할 것이라고 걱정했지만, 그렇지 않았다.


## 원치 않는 함수/변수 재선언 방지

헤더 파일은 종종 다른 헤더 파일을 포함하기 때문에, 하나의 헤더 파일이 여러 번 포함될 가능성이 크다. 이로 인해 동일한 함수나 전역 변수가 재선언될 수 있다.

이를 방지하기 위해, 헤더 파일이 처음 포함될 때 헤더 고유의 매크로를 정의하는 일반적인 헤더 메커니즘을 사용한다. 이렇게 하면 헤더 파일의 내용이 두 번째로 포함되는 것을 막을 수 있다.

예를 들어, 현재 `include/stdio.h` 파일에는 다음과 같은 내용이 있다:

```c
#ifndef _STDIO_H_
# define _STDIO_H_

#include <stddef.h>

// FILE 정의는 일단 이렇게 해두자
typedef char * FILE;

FILE *fopen(char *pathname, char *mode);
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(void *ptr, size_t size, size_t nmemb, FILE *stream);
int fclose(FILE *stream);
int printf(char *format);
int fprintf(FILE *stream, char *format);

#endif  // _STDIO_H_
```

`_STDIO_H_`가 정의되면, 이 파일의 내용이 두 번째로 포함되는 것을 방지한다.


## `extern` 키워드

이제 우리는 동작하는 전처리기를 가지고 있으므로, 언어에 `extern` 키워드를 추가할 때가 되었다고 생각했다. 이 키워드를 사용하면 전역 변수를 정의할 수 있지만, 해당 변수에 대한 저장 공간을 생성하지 않는다. 이는 해당 변수가 다른 소스 파일에서 전역으로 선언되었다고 가정하는 것이다.

`extern` 키워드를 추가하는 것은 실제로 여러 파일에 걸쳐 영향을 미친다. 큰 영향은 아니지만, 광범위한 영향을 준다. 이제 이를 살펴보자.


새로운 키워드 `extern`과 새로운 토큰 T_EXTERN이 `scan.c`에 추가되었다.  
언제나 그렇듯, 코드는 여러분이 직접 읽어볼 수 있게 준비되어 있다.


### 새로운 클래스 추가

`defs.h` 파일에 새로운 스토리지 클래스를 추가했다:

```c
// 스토리지 클래스
enum {
  C_GLOBAL = 1,                 // 전역적으로 접근 가능한 심볼
  ...
  C_EXTERN,                     // 외부에서 접근 가능한 전역 심볼
  ...
};
```

이를 추가한 이유는 `sym.c` 파일에 이미 전역 심볼을 다루는 코드가 있기 때문이다:

```c
// 심볼 테이블 리스트에 추가할 심볼 노드를 생성한다.
struct symtable *newsym(char *name, int type, struct symtable *ctype,
                        int stype, int class, int size, int posn) {
  // 새로운 노드를 할당
  struct symtable *node = (struct symtable *) malloc(sizeof(struct symtable));
  // 값들을 채움
  ...
    // 전역 공간 생성
  if (class == C_GLOBAL)
    genglobsym(node);
```

`extern` 심볼은 전역 리스트에 추가해야 하지만, 이 심볼을 위한 스토리지를 생성하기 위해 `genglobsym()`을 호출하고 싶지는 않다. 따라서 `newsym()`을 호출할 때 `C_GLOBAL`이 아닌 클래스를 사용해야 한다.


### `sym.c` 파일 변경 사항

이를 위해 `addglob()` 함수를 수정하여 `class` 인자를 받도록 만들었다. 이 인자는 `newsym()` 함수로 전달된다:

```c
// 전역 심볼 리스트에 심볼 추가
struct symtable *addglob(char *name, int type, struct symtable *ctype,
                         int stype, int class, int size) {
  struct symtable *sym = newsym(name, type, ctype, stype, class, size, 0);
  appendsym(&Globhead, &Globtail, sym);
  return (sym);
}
```

이 변경으로 인해, 컴파일러에서 `addglob()`을 호출할 때마다 이제 `class` 값을 전달해야 한다. 이전에는 `addglob()`이 `newsym()`에 C_GLOBAL을 명시적으로 전달했지만, 이제는 원하는 `class` 값을 `addglob()`에 전달해야 한다.


### `extern` 키워드와 문법

우리 언어의 문법 측면에서 `extern` 키워드는 타입 설명에서 다른 단어들보다 먼저 와야 한다는 규칙을 강제할 것이다. 나중에 `static`도 이 목록에 추가할 것이다. 이전에 살펴본 [C 언어의 BNF 문법](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)에는 다음과 같은 생성 규칙이 있다:

```
storage_class_specifier
        : TYPEDEF
        | EXTERN
        | STATIC
        | AUTO
        | REGISTER
        ;

type_specifier
        : VOID
        | CHAR
        | SHORT
        | INT
        | LONG
        | FLOAT
        | DOUBLE
        | SIGNED
        | UNSIGNED
        | struct_or_union_specifier
        | enum_specifier
        | TYPE_NAME
        ;

declaration_specifiers
        : storage_class_specifier
        | storage_class_specifier declaration_specifiers
        | type_specifier
        | type_specifier declaration_specifiers
        | type_qualifier
        | type_qualifier declaration_specifiers
        ;
```

이 문법은 `extern`이 타입 명세 어디에든 올 수 있도록 허용하는 것으로 보인다. 하지만 여기서는 C 언어의 부분 집합을 만들고 있으므로 이 규칙을 따르지 않을 것이다!


### `extern` 키워드 파싱

이번에는 지난 다섯 여섯 번의 과정과 마찬가지로 `decl.c` 파일의 `parse_type()` 함수를 다시 수정했다:

```c
int parse_type(struct symtable **ctype, int *class) {
  int type, exstatic=1;

  // 클래스가 extern(나중에 static)으로 변경되었는지 확인
  while (exstatic) {
    switch (Token.token) {
      case T_EXTERN: *class= C_EXTERN; scan(&Token); break;
      default: exstatic= 0;
    }
  }
  ...
}
```

이제 `parse_type()` 함수는 두 번째 매개변수로 `int *class`를 받는다. 이를 통해 호출자는 타입의 초기 스토리지 클래스(아마도 C_GLOBAL, G_LOCAL 또는 C_PARAM)를 전달할 수 있다. 만약 `parse_type()` 함수에서 `extern` 키워드를 발견하면, 이를 T_EXTERN으로 변경할 수 있다. 그리고 'while' 루프를 제어하는 불리언 플래그에 적절한 이름을 짓지 못한 점에 대해 사과한다.


### `parse_type()`과 `addglob()` 호출처

`parse_type()`과 `addglob()`의 인자를 수정했다. 이제 컴파일러 내에서 이 두 함수가 호출되는 모든 곳을 찾아 적절한 `class` 값을 전달해야 한다.

`decl.c` 파일의 `var_declaration_list()` 함수에서는 변수나 매개변수 목록을 파싱할 때 이미 이 변수들의 스토리지 클래스를 가져온다:

```c
static int var_declaration_list(struct symtable *funcsym, int class,
                                int separate_token, int end_token);
```

따라서 `class`를 `parse_type()`에 전달할 수 있으며, 이 함수는 `class`를 변경할 수 있다. 그 후 실제 클래스를 사용해 `var_declaration()`을 호출한다:

```c
    ...
    // 타입과 식별자를 가져옴
    type = parse_type(&ctype, &class);
    ident();
    ...
    // 클래스를 기반으로 올바른 심볼 테이블 리스트에 새로운 매개변수를 추가
    var_declaration(type, ctype, class);
```

`var_declaration()` 함수 내부에서는 다음과 같이 처리한다:

```c
      switch (class) {
        case C_EXTERN:
        case C_GLOBAL:
          sym = addglob(Text, type, ctype, S_VARIABLE, class, 1);
        ...
      }
```

로컬 변수의 경우 `stmt.c` 파일의 `single_statement()` 함수를 살펴봐야 한다. 여기서는 이전에 구조체, 공용체, 열거형, typedef에 대한 케이스를 추가하는 것을 잊었다는 점을 언급해야 한다.

```c
// 단일 문장을 파싱하고 AST를 반환
static struct ASTnode *single_statement(void) {
  int type, class= C_LOCAL;
  struct symtable *ctype;

  switch (Token.token) {
    case T_IDENT:
      // 식별자가 typedef와 일치하는지 확인해야 함.
      // 일치하지 않으면 이 switch 문의 기본 코드를 실행.
      // 그렇지 않으면 parse_type() 호출로 넘어감.
      if (findtypedef(Text) == NULL)
        return (binexpr(0));
    case T_CHAR:
    case T_INT:
    case T_LONG:
    case T_STRUCT:
    case T_UNION:
    case T_ENUM:
    case T_TYPEDEF:
      // 변수 선언의 시작.
      // 타입을 파싱하고 식별자를 가져옴.
      // 그 후 선언의 나머지 부분을 파싱하고 세미콜론을 건너뜀.
      type = parse_type(&ctype, &class);
      ident();
      var_declaration(type, ctype, class);
      semi();
      return (NULL);            // 여기서는 AST가 생성되지 않음
      ...
   }
   ...
}
```

여기서는 `class= C_LOCAL`로 시작하지만, `parse_type()`에 의해 수정될 수 있다. 그 후 `var_declaration()`에 전달된다. 이를 통해 다음과 같은 코드를 작성할 수 있다:

```c
int main() {
  extern int foo;
  ...
}
```


## 코드 테스트

새로 추가한 헤더 파일이 제대로 동작하는지 확인하기 위해 `test/input70.c`라는 테스트 프로그램을 작성했다:

```c
#include <stdio.h>

typedef int FOO;

int main() {
  FOO x;
  x= 56;
  printf("%d\n", x);
  return(0);
}
```

원래는 `errno`가 일반 정수형 변수일 줄 알고 `include/errno.h` 파일에 `extern int errno;`라고 선언하려고 했다. 하지만 `errno`는 이제 전역 정수 변수가 아니라 함수로 구현된 것으로 보인다. 이 사실은 내가 a) 얼마나 오래된 개발자인지, b) C 코드를 얼마나 오랜만에 작성하는지를 잘 보여준다.


## 결론과 다음 단계

우리는 또 하나의 중요한 이정표를 달성했다. 이제 외부 변수와 헤더 파일을 사용할 수 있게 되었다. 이는 결국 소스 파일에 주석을 추가할 수 있다는 의미이기도 하다. 이 점이 나를 정말 기쁘게 한다.

현재 코드는 총 4,100줄 이상이며, 그중 약 2,800줄은 주석과 공백을 제외한 실제 코드다. 컴파일러가 스스로 컴파일할 수 있도록 하려면 얼마나 더 많은 코드가 필요한지 정확히 알 수 없지만, 대략 7,000줄에서 9,000줄 사이일 것이라 추측한다. 결과는 두고 볼 일이다!

컴파일러 작성 여정의 다음 단계에서는 `break`와 `continue` 키워드를 루프 구조에 추가할 것이다. [다음 단계](../36_Break_Continue/Readme.md)



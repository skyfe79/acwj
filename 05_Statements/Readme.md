# 5부: 구문

이제 우리가 만드는 프로그래밍 언어에 "진정한" 구문을 추가할 차례다. 
다음과 같은 코드를 작성할 수 있게 만들 것이다:

```python
   print 2 + 3 * 5;
   print 18 - 6/3 + 4*2;
```

프로그래밍 언어는 공백을 무시하므로, 하나의 구문을 구성하는 모든 토큰이 반드시 같은 줄에 있을 필요는 없다. 각 구문은 `print` 키워드로 시작하고 세미콜론으로 끝난다. 따라서 이들은 우리 언어에서 새로운 토큰이 될 것이다.

## 문법의 BNF 표기법 설명

앞서 표현식에 대한 BNF 표기법을 살펴보았다. 이제 위와 같은 유형의 구문에 대한 BNF 문법을 정의해보자:

```bnf
statements: statement
     | statement statements
     ;

statement: 'print' expression ';'
     ;
```

입력 파일은 여러 개의 구문으로 구성된다. 구문은 단일 구문이거나, 하나의 구문 뒤에 여러 구문이 이어지는 형태다. 각 구문은 `print` 키워드로 시작하고, 그 뒤에 하나의 표현식이 오며, 세미콜론으로 끝난다.

## 어휘 스캐너 변경하기

위 문법을 파싱하는 코드를 작성하기에 앞서, 기존 코드에 몇 가지 요소를 추가해야 한다. 먼저 어휘 스캐너부터 시작하자.

세미콜론을 위한 토큰 추가는 간단하다. 다음으로 `print` 키워드를 처리해야 한다. 앞으로 프로그래밍 언어에는 많은 키워드와 변수를 위한 식별자가 필요하므로, 이를 처리하는 코드를 추가해야 한다.

`scan.c` 파일에 SubC 컴파일러에서 가져온 코드를 추가했다. 이 코드는 영숫자가 아닌 문자를 만날 때까지 영숫자 문자를 버퍼에 읽어들인다.

```c
// 입력 파일에서 식별자를 스캔하여 buf[]에 저장하고
// 식별자의 길이를 반환한다
static int scanident(int c, char *buf, int lim) {
  int i = 0;

  // 숫자, 알파벳, 언더스코어를 허용한다
  while (isalpha(c) || isdigit(c) || '_' == c) {
    // 식별자 길이 제한에 도달하면 에러 처리
    // 그렇지 않으면 buf[]에 추가하고 다음 문자를 가져온다
    if (lim - 1 == i) {
      printf("%d번 줄에서 식별자가 너무 깁니다\n", Line);
      exit(1);
    } else if (i < lim - 1) {
      buf[i++] = c;
    }
    c = next();
  }
  // 유효하지 않은 문자를 만나면 되돌리고
  // buf[]를 널 종료하고 길이를 반환한다
  putback(c);
  buf[i] = '\0';
  return (i);
}
```

또한 언어의 키워드를 인식하는 함수도 필요하다. 한 가지 방법은 키워드 목록을 만들고, 목록을 순회하면서 `scanident()`의 버퍼와 각 키워드를 `strcmp()`로 비교하는 것이다. SubC의 코드는 최적화를 위해 `strcmp()`를 하기 전에 첫 글자를 먼저 비교한다. 이는 수십 개의 키워드를 비교할 때 속도를 높여준다. 지금은 이 최적화가 필요하지 않지만, 나중을 위해 추가했다:

```c
// 입력된 단어가 키워드인지 확인하여
// 일치하는 키워드 토큰 번호를 반환하거나
// 키워드가 아닌 경우 0을 반환한다
// 첫 글자로 구분하여 모든 키워드와 strcmp()를
// 수행하는 낭비를 줄인다
static int keyword(char *s) {
  switch (*s) {
    case 'p':
      if (!strcmp(s, "print"))
        return (T_PRINT);
      break;
  }
  return (0);
}
```

이제 `scan()`의 switch 문 마지막에 세미콜론과 키워드를 인식하는 코드를 추가한다:

```c
    case ';':
      t->token = T_SEMI;
      break;
    default:

      // 숫자인 경우 정수 리터럴 값을 스캔한다
      if (isdigit(c)) {
        t->intvalue = scanint(c);
        t->token = T_INTLIT;
        break;
      } else if (isalpha(c) || '_' == c) {
        // 키워드나 식별자를 읽는다
        scanident(c, Text, TEXTLEN);

        // 인식된 키워드인 경우 해당 토큰을 반환한다
        if (tokentype = keyword(Text)) {
          t->token = tokentype;
          break;
        }
        // 인식되지 않은 키워드는 현재는 에러 처리한다
        printf("%d번 줄에서 인식할 수 없는 기호 %s\n", Line, Text);
        exit(1);
      }
      // 인식할 수 있는 토큰이 아닌 문자는 에러 처리한다
      printf("%d번 줄에서 인식할 수 없는 문자 %c\n", Line, c);
      exit(1);
```

키워드와 식별자를 저장할 전역 `Text` 버퍼도 추가했다:

```c
#define TEXTLEN         512             // 입력 기호의 최대 길이
extern_ char Text[TEXTLEN + 1];         // 마지막으로 스캔한 식별자
```

## 표현식 파서 변경하기

지금까지 입력 파일에는 단일 표현식만 포함되어 있었다. 따라서 `expr.c` 파일의 Pratt 파서 코드인 `binexpr()` 함수에서 파서를 종료하기 위해 다음과 같은 코드를 사용했다:

```c
  // 토큰이 더 이상 없으면 왼쪽 노드만 반환한다
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);
```

새로운 문법에서는 각 표현식이 세미콜론으로 끝난다. 따라서 표현식 파서에서 `T_SEMI` 토큰을 감지하고 표현식 파싱을 종료하도록 코드를 변경해야 한다:

```c
// 이진 연산자를 루트로 하는 AST 트리를 반환한다.
// ptp 매개변수는 이전 토큰의 우선순위다.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // 왼쪽의 정수 리터럴을 가져온다.
  // 동시에 다음 토큰을 가져온다.
  left = primary();

  // 세미콜론을 만나면 왼쪽 노드만 반환한다
  tokentype = Token.token;
  if (tokentype == T_SEMI)
    return (left);

    while (op_precedence(tokentype) > ptp) {
      ...

    // 현재 토큰의 세부 정보를 업데이트한다.
    // 세미콜론을 만나면 왼쪽 노드만 반환한다
    tokentype = Token.token;
    if (tokentype == T_SEMI)
      return (left);
    }
}
```

## 코드 생성기 변경하기

`gen.c` 파일에 있는 일반 코드 생성기와 `cg.c` 파일의 CPU 특화 코드를 분리하여 관리한다. 이는 컴파일러의 다른 부분에서 오직 `gen.c`의 함수만 호출하고, `gen.c`만이 `cg.c`의 코드를 호출할 수 있음을 의미한다.

이를 위해 `gen.c`에 다음과 같은 새로운 "프론트엔드" 함수들을 정의했다:

```c
void genpreamble()        { cgpreamble(); }
void genpostamble()       { cgpostamble(); }
void genfreeregs()       { freeall_registers(); }
void genprintint(int reg) { cgprintint(reg); }
```
## 구문 파서 추가하기

새로운 파일 `stmt.c`는 프로그래밍 언어의 모든 주요 구문을 분석하는 코드를 포함한다. 현재 위에서 제시한 BNF 문법에 따라 구문을 분석해야 한다. 다음 함수가 이 작업을 수행하며, 재귀적 정의를 반복문으로 변환했다:

```c
// 하나 이상의 구문을 분석한다
void statements(void) {
  struct ASTnode *tree;
  int reg;

  while (1) {
    // 첫 번째 토큰으로 'print'를 찾는다
    match(T_PRINT, "print");

    // 표현식을 분석하고
    // 어셈블리 코드를 생성한다
    tree = binexpr(0);
    reg = genAST(tree);
    genprintint(reg);
    genfreeregs();

    // 세미콜론을 찾고
    // EOF에 도달하면 중단한다
    semi();
    if (Token.token == T_EOF)
      return;
  }
}
```

반복문은 매번 T_PRINT 토큰을 찾는다. 그 다음 `binexpr()` 함수를 호출하여 표현식을 분석한다. 마지막으로 T_SEMI 토큰을 찾는다. T_EOF 토큰이 나타나면 반복문을 종료한다.

표현식 트리를 생성한 후, `gen.c`의 코드가 이 트리를 어셈블리 코드로 변환하고 어셈블리 함수 `printint()`를 호출하여 최종 값을 출력한다.

## 헬퍼 함수들

위 코드에는 몇 가지 새로운 헬퍼 함수가 있다. 이 함수들은 `misc.c` 파일에 다음과 같이 작성했다:

```c
// 현재 토큰이 t인지 확인하고
// 다음 토큰을 가져온다. 그렇지 않으면
// 에러를 발생시킨다
void match(int t, char *what) {
  if (Token.token == t) {
    scan(&Token);
  } else {
    printf("%s expected on line %d\n", what, Line);
    exit(1);
  }
}

// 세미콜론을 매칭하고 다음 토큰을 가져온다
void semi(void) {
  match(T_SEMI, ";");
}
```

이 함수들은 파서의 구문 검사를 담당한다. 앞으로 구문 검사를 더 쉽게 만들기 위해 `match()` 함수를 호출하는 짧은 함수들을 더 추가할 예정이다.

## `main()` 함수 변경하기

`main()` 함수는 이전에 단일 표현식을 파싱하기 위해 `binexpr()`를 직접 호출했다. 이제는 다음과 같이 동작한다:

```c
  scan(&Token);                 // 입력에서 첫 번째 토큰을 가져온다
  genpreamble();                // 프리앰블을 출력한다
  statements();                 // 입력의 문장들을 파싱한다
  genpostamble();               // 포스트앰블을 출력한다
  fclose(Outfile);              // 출력 파일을 닫고 종료한다
  exit(0);
```

## 실행 테스트

새로 작성하고 변경한 코드는 여기까지다. 이제 새 코드를 실행해보자. 다음은 새로운 입력 파일 `input01`의 내용이다:

```
print 12 * 3;
print 
   18 - 2
      * 4; print
1 + 2 +
  9 - 5/2 + 3*5;
```

토큰들이 여러 줄에 걸쳐 있어도 처리할 수 있는지 확인헤보자. 입력 파일을 컴파일하고 실행하려면 `make test` 명령을 실행한다:

```make
$ make test
cc -o comp1 -g cg.c expr.c gen.c main.c misc.c scan.c stmt.c tree.c
./comp1 input01
cc -o out out.s
./out
36
10
25
```

모든 것이 정상적으로 작동한다!

## 결론 및 다음 단계

우리는 처음으로 "실제" 문장 문법을 언어에 추가했다. BNF 표기법으로 정의했지만, 재귀적으로 구현하지 않고 반복문으로 구현하는 것이 더 쉬웠다. 곧 재귀적 파싱으로 돌아갈 예정이니 걱정하지 않아도 된다.

이 과정에서 스캐너를 수정하고, 키워드와 식별자 지원을 추가했으며, 일반 코드 생성기와 CPU별 생성기를 더 깔끔하게 분리했다.

컴파일러 제작 여정의 다음 단계에서는 언어에 변수를 추가할 것이다. 이는 상당한 작업량이 필요할 것이다. 

[다음 단계](../06_Variables/Readme.md)

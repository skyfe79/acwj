# 1부: 어휘 스캐닝 소개

컴파일러 제작의 첫 단계로 간단한 어휘 스캐너를 만들어본다. 이전 장에서 언급했듯이, 스캐너의 주요 역할은 입력 언어에서 어휘 요소(토큰)를 식별하는 것이다.

우리가 다룰 언어는 다음 다섯 가지 어휘 요소만을 포함한다:

 + 사칙연산 연산자: `*`, `/`, `+`, `-`
 + 십진수 정수: 하나 이상의 숫자로 구성된 `0` .. `9`

스캔한 각 토큰은 다음과 같은 구조체에 저장된다(`defs.h`에서 정의):

```c
// 토큰 구조체
struct token {
  int token;
  int intvalue;
};
```

여기서 `token` 필드는 다음 값들 중 하나가 될 수 있다(`defs.h`에서 정의):

```c
// 토큰 종류
enum {
  T_PLUS, T_MINUS, T_STAR, T_SLASH, T_INTLIT
};
```

토큰이 `T_INTLIT`(정수 리터럴)인 경우, `intvalue` 필드에 스캔한 정수의 실제 값을 저장한다.

## scan.c 파일의 함수들

`scan.c` 파일은 어휘 스캐너의 함수들을 담고 있다. 이 스캐너는 입력 파일에서 한 번에 한 문자씩 읽어들인다. 그러나 입력 스트림에서 너무 앞서 읽었을 때는 문자를 "되돌려놓아야" 하는 경우가 있다. 또한 디버그 메시지에서 줄 번호를 출력할 수 있도록 현재 처리 중인 줄을 추적해야 한다. 이 모든 기능은 `next()` 함수에서 구현한다:

```c
// 입력 파일에서 다음 문자를 가져온다.
static int next(void) {
  int c;

  if (Putback) {                // 되돌려 놓은 문자가 있다면
    c = Putback;                // 그 문자를 사용한다
    Putback = 0;
    return c;
  }

  c = fgetc(Infile);            // 입력 파일에서 읽는다
  if ('\n' == c)
    Line++;                     // 줄 수를 증가시킨다
  return c;
}
```

`Putback`과 `Line` 변수는 입력 파일 포인터와 함께 `data.h`에 정의되어 있다:

```c
extern_ int     Line;
extern_ int     Putback;
extern_ FILE    *Infile;
```

모든 C 파일은 이 헤더를 포함하며, 여기서 `extern_`은 `extern`으로 대체된다. 하지만 `main.c`에서는 `extern_`을 제거한다. 따라서 이 변수들은 `main.c`에 "속하게" 된다.

마지막으로, 문자를 입력 스트림으로 되돌리는 방법은 다음과 같다:

```c
// 원하지 않는 문자를 되돌린다
static void putback(int c) {
  Putback = c;
}
```

## 공백 문자 무시하기

프로그램에서 입력 처리 시 의미 있는 문자를 만날 때까지 공백 문자를 건너뛰어야 할 때가 있다. 이를 위해 공백 문자를 읽고 조용히 건너뛴 다음, 공백이 아닌 첫 번째 문자를 반환하는 함수가 필요하다.

다음은 이러한 기능을 구현한 코드이다:

```c
// 처리할 필요가 없는 입력을 건너뛴다.
// 즉, 공백, 줄바꿈 등을 무시하고
// 실제로 처리해야 할 첫 번째 문자를 반환한다.
static int skip(void) {
  int c;

  c = next();
  while (' ' == c || '\t' == c || '\n' == c || '\r' == c || '\f' == c) {
    c = next();
  }
  return (c);
}
```

이 `skip()` 함수는 다음과 같은 특징을 가진다:

1. 공백 문자들을 연속해서 읽어 들인다
2. 스페이스(' '), 탭('\t'), 개행('\n'), 캐리지 리턴('\r'), 폼 피드('\f') 등 모든 종류의 공백 문자를 처리한다
3. 공백이 아닌 문자를 만나면 해당 문자를 반환한다
4. 입력의 다음 문자를 가져오는 `next()` 함수를 사용한다

이러한 함수는 텍스트 파서나 컴파일러에서 토큰을 분리할 때 매우 유용하게 사용된다. 의미 있는 토큰 사이의 공백을 효과적으로 제거하여 실제 처리해야 할 내용만 남기기 때문이다.

## 토큰 스캔하기: `scan()` 함수 구현

이제 공백 문자를 건너뛰며 문자를 읽을 수 있고, 필요한 경우 한 문자를 앞서 읽었을 때 되돌릴 수도 있다. 이를 바탕으로 첫 번째 어휘 분석기를 구현해보자:

```c
// 입력에서 다음 토큰을 스캔하고 반환한다.
// 토큰이 유효하면 1을, 더 이상 토큰이 없으면 0을 반환한다.
int scan(struct token *t) {
  int c;

  // 공백 문자 건너뛰기
  c = skip();

  // 입력된 문자를 기반으로
  // 토큰 결정하기
  switch (c) {
  case EOF:
    return (0);
  case '+':
    t->token = T_PLUS;
    break;
  case '-':
    t->token = T_MINUS;
    break;
  case '*':
    t->token = T_STAR;
    break;
  case '/':
    t->token = T_SLASH;
    break;
  default:
    // 곧 더 많은 내용이 추가될 예정
  }

  // 토큰을 찾았음
  return (1);
}
```

한 문자로 이루어진 간단한 토큰에 대한 처리는 이것으로 끝이다. 인식된 각 문자를 해당하는 토큰으로 변환하면 된다. "왜 인식된 문자를 그냥 `struct token`에 넣지 않는가?"라는 의문이 들 수 있다. 그 이유는 나중에 `==`와 같은 여러 문자로 이루어진 토큰이나 `if`, `while` 같은 키워드를 인식해야 하기 때문이다. 따라서 토큰 값을 열거형 목록으로 관리하면 앞으로의 작업이 더 수월해질 것이다.

# 정수 리터럴 값 처리하기

실제로 `3827`이나 `87731`과 같은 정수 리터럴 값을 인식해야 하는 상황에 직면했다. 다음은 `switch` 문에서 빠진 `default` 코드이다:

```c
  default:
    // 숫자인 경우 정수 리터럴 값을 
    // 스캔한다
    if (isdigit(c)) {
      t->intvalue = scanint(c);
      t->token = T_INTLIT;
      break;
    }

    printf("인식할 수 없는 문자 %c (라인 %d)\n", c, Line);
    exit(1);
```

십진수 문자를 만나면 첫 문자를 인자로 `scanint()` 헬퍼 함수를 호출한다. 이 함수는 스캔한 정수 값을 반환한다. 각 문자를 차례로 읽고, 유효한 숫자인지 확인한 후 최종 숫자를 만든다. 다음은 그 코드이다:

```c
// 입력 파일에서 정수 리터럴 값을 
// 스캔하고 반환한다
static int scanint(int c) {
  int k, val = 0;

  // 각 문자를 정수 값으로 변환한다
  while ((k = chrpos("0123456789", c)) >= 0) {
    val = val * 10 + k;
    c = next();
  }

  // 정수가 아닌 문자를 만나면 되돌린다
  putback(c);
  return val;
}
```

`val` 값을 0으로 시작한다. `0`부터 `9`까지의 문자를 만날 때마다 `chrpos()`로 이를 `int` 값으로 변환한다. `val`을 10배 늘린 후 이 새로운 숫자를 더한다.

예를 들어 `3`, `2`, `8` 문자가 있다면 다음과 같이 처리한다:

+ `val = 0 * 10 + 3`, 즉 3
+ `val = 3 * 10 + 2`, 즉 32
+ `val = 32 * 10 + 8`, 즉 328

마지막에 `putback(c)` 호출을 눈치챘는가? 이 시점에서 십진수가 아닌 문자를 발견했다. 이를 단순히 버릴 수 없지만, 다행히도 나중에 사용하도록 입력 스트림에 되돌려 놓을 수 있다.

이 시점에서 'c에서 문자 '0'의 ASCII 값을 빼서 정수로 만들지 않는 이유는 무엇인가?'라고 물을 수 있다. 그 답은 나중에 `chrpos("0123456789abcdef")`를 사용하여 16진수도 변환할 수 있기 때문이다.

다음은 `chrpos()` 코드이다:

```c
// 문자열 s에서 문자 c의 위치를 반환한다
// c를 찾지 못하면 -1을 반환한다
static int chrpos(char *s, int c) {
  char *p;

  p = strchr(s, c);
  return (p ? p - s : -1);
}
```

이것으로 현재 `scan.c`의 어휘 스캐너 코드는 여기까지다.

---

[역주] ASCII 값을 이용한 숫자 변환에 대한 추가 설명:

일반적으로 문자로 된 숫자를 정수로 변환할 때는 ASCII 값의 차이를 이용하는 방법을 많이 사용한다. 예를 들어:

```c
char c = '5';    // 문자 '5'의 ASCII 값은 53
int num = c - '0';    // '0'의 ASCII 값은 48
// 따라서 53 - 48 = 5가 된다
```

이러한 ASCII 뺄셈 방식은 0-9까지의 한 자리 숫자를 변환할 때는 간단하고 효율적이다. 하지만 현재 코드가 `chrpos()` 방식을 선택한 데는 다음과 같은 이유가 있다:

1. **확장성**: 
   16진수(0-9, a-f)를 처리할 때 `chrpos("0123456789abcdef")`로 쉽게 확장할 수 있다. ASCII 뺄셈 방식은 a-f 문자를 처리하려면 추가 로직이 필요하다.

2. **유연성**:
   숫자 집합을 쉽게 변경할 수 있으며, 다른 문자 체계나 기수를 지원하기도 쉽다.

3. **가독성**:
   허용된 문자 집합이 문자열에 명시적으로 표현되어 코드의 의도가 더 명확하게 드러난다.

16진수 처리시 두 방식의 차이를 비교해보자:

```c
// ASCII 뺄셈 방식
int convert_hex_digit(char c) {
    if (c >= '0' && c <= '9')
        return c - '0';
    else if (c >= 'a' && c <= 'f')
        return 10 + (c - 'a');
    else if (c >= 'A' && c <= 'F')
        return 10 + (c - 'A');
    return -1;
}

// chrpos 방식
int convert_hex_digit(char c) {
    return chrpos("0123456789abcdefABCDEF", c);
}
```

`chrpos` 방식은 코드가 더 간단하고, 허용된 문자 집합을 한눈에 볼 수 있다. 또한 새로운 문자를 추가하거나 제거하기도 쉽다.

결론적으로 ASCII 뺄셈 방식이 단순 10진수 변환에는 효율적이지만, `chrpos` 방식은 코드 유지보수가 쉽고, 기능 확장이 용이하며, 의도가 명확하게 드러나고, 다양한 문자 집합을 지원하기 쉽다는 장점이 있다. 이러한 이유로 현재 코드에서는 `chrpos` 방식을 선택했다.

## 스캐너 실행하기

`main.c` 파일에는 앞서 구현한 스캐너를 실행하는 코드가 포함되어 있다. `main()` 함수는 파일을 열고 토큰을 검색하는 작업을 수행한다:

```c
void main(int argc, char *argv[]) {
  ...
  init();
  ...
  Infile = fopen(argv[1], "r");
  ...
  scanfile();
  exit(0);
}
```

그리고 `scanfile()` 함수는 새로운 토큰이 있는 동안 반복적으로 실행되며, 각 토큰의 세부 정보를 출력한다:

```c
// 출력 가능한 토큰 목록
char *tokstr[] = { "+", "-", "*", "/", "intlit" };

// 입력 파일의 모든 토큰을 스캔하는 반복문
// 발견된 각 토큰의 세부 정보를 출력한다
static void scanfile() {
  struct token T;

  while (scan(&T)) {
    printf("Token %s", tokstr[T.token]);
    if (T.token == T_INTLIT)
      printf(", value %d", T.intvalue);
    printf("\n");
  }
}
```

## 스캐너 예제 입력 파일

스캐너의 동작 방식을 보여주는 몇 가지 예제 파일을 준비했다. 이 예제들은 스캐너가 어떤 입력은 허용하고 어떤 입력은 거부하는지 보여준다.

```bash
$ make
cc -o scanner -g main.c scan.c

$ cat input01
2 + 3 * 5 - 8 / 3

$ ./scanner input01
Token intlit, value 2
Token +
Token intlit, value 3
Token *
Token intlit, value 5
Token -
Token intlit, value 8
Token /
Token intlit, value 3

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./scanner input04
Token intlit, value 23
Token +
Token intlit, value 18
Token -
Token intlit, value 45
Unrecognised character . on line 3
```

## 결론 및 다음 단계

지금까지 간단한 구현을 시작으로, 네 가지 주요 수학 연산자와 정수 리터럴 값을 인식하는 기본적인 어휘 분석기(lexical scanner)를 만들었다. 이 과정에서 공백 문자를 건너뛰고 입력값을 너무 많이 읽었을 때 문자를 되돌려 놓아야 하는 필요성도 발견했다.

단일 문자 토큰은 쉽게 분석할 수 있지만, 여러 문자로 구성된 토큰은 좀 더 복잡한 처리가 필요하다. 하지만 결과적으로 `scan()` 함수는 입력 파일에서 다음 토큰을 읽어 `struct token` 변수로 반환한다:

```c
struct token {
  int token;
  int intvalue;
};
```

컴파일러 제작 여정의 다음 단계에서는 재귀 하향 파서(recursive descent parser)를 구축할 것이다. 이 파서는 입력 파일의 문법을 해석하고, 각 파일에 대한 최종 값을 계산하여 출력하는 역할을 수행한다. 

[다음 단계](../02_Parser/Readme.md)

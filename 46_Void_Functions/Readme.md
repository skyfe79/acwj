# 46부: void 함수 매개변수와 스캐너 변경 사항

이번 파트에서는 컴파일러 개발 과정에서 스캐너와 파서에 관련된 여러 변경 사항을 다룬다.


## Void Function Parameters

C 언어에서 함수가 매개변수를 가지지 않음을 나타내기 위해 일반적으로 사용하는 구문은 다음과 같다:

```c
int fred(void);         // Void는 매개변수가 없음을 의미
int fred();             // 매개변수가 없음도 매개변수가 없음을 의미
```

매개변수가 없음을 나타내는 방법이 이미 있음에도 불구하고, 이는 흔히 사용되는 관행이므로 이를 지원해야 한다.

문제는 왼쪽 괄호를 만나면 `decl.c` 파일의 `declaration_list()` 함수로 진입한다는 점이다. 이 함수는 타입과 그 뒤에 오는 식별자를 파싱하도록 설계되어 있다. 타입은 있지만 식별자가 없는 경우를 처리하도록 수정하기는 쉽지 않다. 따라서 `param_declaration_list()` 함수로 돌아가서 'void'와 ')' 토큰을 파싱해야 한다.

이미 `scan.c` 파일에 `reject_token()`이라는 함수가 있다. 토큰을 스캔하고, 이를 검토한 후 원하지 않으면 거부할 수 있어야 한다. 그러면 다음에 스캔되는 토큰은 거부한 토큰이 된다.

이 함수를 사용해 본 적은 없는데, 알고 보니 고장난 상태였다. 그래서 한 발 물러서서 다음 토큰을 미리 확인(*peek*)하는 것이 더 쉬울 것이라고 판단했다. 원하는 토큰이면 일반적으로 스캔하고, 원하지 않으면 아무것도 하지 않으면 된다. 그러면 다음 실제 토큰 스캔 시에 해당 토큰이 스캔될 것이다.

이 기능이 필요한 이유는 매개변수 목록에서 'void'를 처리하기 위한 의사 코드가 다음과 같기 때문이다:

```
  '('를 파싱
  다음 토큰이 'void'인지 확인
  'void' 다음 토큰을 미리 확인
  'void' 다음 토큰이 ')'이면 매개변수가 없음을 반환
  실제 매개변수를 얻기 위해 declaration_list() 호출
  이때 'void'가 여전히 현재 토큰으로 유지되도록
```

이러한 미리 확인이 필요한 이유는 다음 두 가지 경우가 모두 유효하기 때문이다:

```c
int fred(void);
int jane(void *ptr, int x, int y);
```

'void' 다음 토큰을 스캔하고 파싱한 후 그 토큰이 '*'인 경우, 'void' 토큰을 잃게 된다. 그런 다음 `declaration_list()`를 호출하면, 이 함수가 처음 보는 토큰은 '*'가 되고 이는 문제를 일으킬 것이다. 따라서 현재 토큰을 그대로 유지하면서 다음 토큰을 미리 확인할 수 있는 기능이 필요하다.


## 새로운 스캐너 코드

`data.h` 파일에 새로운 토큰 변수를 추가한다:

```c
extern_ struct token Token;             // 마지막으로 스캔한 토큰
extern_ struct token Peektoken;         // 미리 보는 토큰
```

`Peektoken.token`은 `main.c` 파일의 코드에서 0으로 초기화된다. `scan.c` 파일의 주요 `scan()` 함수를 다음과 같이 수정한다:

```c
// 입력에서 다음 토큰을 스캔하여 반환한다.
// 토큰이 유효하면 1을 반환하고, 토큰이 더 이상 없으면 0을 반환한다.
int scan(struct token *t) {
  int c, tokentype;

  // 미리 보는 토큰이 있으면 이 토큰을 반환한다
  if (Peektoken.token != 0) {
    t->token = Peektoken.token;
    t->tokstr = Peektoken.tokstr;
    t->intvalue = Peektoken.intvalue;
    Peektoken.token = 0;
    return (1);
  }
  ...
}
```

`Peektoken.token`이 0으로 유지되면 다음 토큰을 가져온다. 하지만 `Peektoken`에 무언가 저장되면, 그 토큰이 다음에 반환될 토큰이 된다.


## 선언문 수정

이제 다음 토큰을 미리 확인할 수 있게 되었으니, 이를 실제 코드에 적용해 보자. `param_declaration_list()` 함수의 코드를 다음과 같이 수정한다:

```c
  // 매개변수를 계속해서 가져오는 루프
  while (Token.token != T_RPAREN) {

    // 첫 번째 토큰이 'void'인 경우
    if (Token.token == T_VOID) {
      // 다음 토큰을 미리 확인한다. 만약 ')'라면,
      // 함수에 매개변수가 없으므로 루프를 종료한다.
      scan(&Peektoken);
      if (Peektoken.token == T_RPAREN) {
        // Peektoken을 Token으로 이동시킨다
        paramcnt= 0; scan(&Token); break;
      }
    }
    ...
    // 다음 매개변수의 타입을 가져온다
    type = declaration_list(&ctype, C_PARAM, T_COMMA, T_RPAREN, &unused);
    ...
  }
```

'void'를 이미 스캔했다고 가정하자. 이제 `scan(&Peektoken);`을 통해 현재 `Token`을 변경하지 않고 다음 토큰을 확인한다. 만약 다음 토큰이 오른쪽 괄호라면, 'void' 토큰을 건너뛰고 `paramcnt`를 0으로 설정한 뒤 루프를 종료한다.

그러나 다음 토큰이 오른쪽 괄호가 아니라면, `Token`은 여전히 'void'로 설정된 상태이며, 이제 `declaration_list()`를 호출해 실제 매개변수 목록을 가져올 수 있다.


## 16진수와 8진수 정수 상수

이 문제는 컴파일러의 소스 코드를 컴파일러 자체에 입력하면서 발견했다. 'void' 매개변수 문제를 해결한 후, 다음으로 발견한 문제는 컴파일러가 `0x314A`나 `0073` 같은 16진수와 8진수 상수를 파싱하지 못한다는 점이었다.

다행히도 Nils M Holm이 작성한 [SubC](http://www.t3x.org/subc/) 컴파일러에는 이 기능을 구현한 코드가 있어, 이를 그대로 가져와 우리 컴파일러에 추가할 수 있다. 이를 위해 `scan.c` 파일의 `scanint()` 함수를 다음과 같이 수정한다:

```c
// 입력 파일에서 정수 리터럴 값을 스캔하고 반환한다.
static int scanint(int c) {
  int k, val = 0, radix = 10;

  // 새로운 코드: 기본 진수는 10이지만, 0으로 시작하면
  if (c == '0') {
    // 다음 문자가 'x'이면 진수는 16
    if ((c = next()) == 'x') {
      radix = 16;
      c = next();
    } else
      // 그렇지 않으면 진수는 8
      radix = 8;

  }

  // 각 문자를 정수 값으로 변환
  while ((k = chrpos("0123456789abcdef", tolower(c))) >= 0) {
    if (k >= radix)
      fatalc("정수 리터럴에 잘못된 숫자가 있습니다", c);
    val = val * radix + k;
    c = next();
  }

  // 정수가 아닌 문자를 만나면, 이를 되돌린다.
  putback(c);
  return (val);
}
```

기존 함수에는 `k = chrpos("0123456789")` 코드가 있어 10진수 리터럴 값을 처리했다. 위의 새로운 코드는 이제 선행 '0' 숫자를 스캔한다. 이를 발견하면 다음 문자를 확인한다. 'x'이면 진수는 16이고, 그렇지 않으면 진수는 8이다.

또 다른 변경점은 이전 값에 상수 10 대신 진수를 곱한다는 것이다. 이는 이 문제를 해결하는 매우 우아한 방법이며, Nils에게 코드를 작성해 준 것에 대해 깊이 감사한다.


## 추가 문자 상수 처리

다음으로 마주친 문제는 컴파일러에서 다음과 같은 코드를 처리하는 부분이었다:

```c
   if (*posn == '\0')
```

이 코드는 우리 컴파일러가 인식하지 못하는 문자 리터럴이다. `scan.c` 파일에 있는 `scanch()` 함수를 수정하여 8진수로 지정된 문자 리터럴을 처리할 필요가 있다. 하지만 16진수로 지정된 문자 리터럴도 가능하다. 예를 들어 '\0x41' 같은 경우다. 이번에도 SubC의 코드가 도움이 된다:

```c
// 입력에서 16진수 상수를 읽어온다
static int hexchar(void) {
  int c, h, n = 0, f = 0;

  // 문자를 계속 읽어온다
  while (isxdigit(c = next())) {
    // 문자를 정수 값으로 변환한다
    h = chrpos("0123456789abcdef", tolower(c));
    // 현재 16진수 값에 더한다
    n = n * 16 + h;
    f = 1;
  }
  // 16진수가 아닌 문자를 만나면 다시 넣는다
  putback(c);
  // 플래그를 통해 16진수 문자를 전혀 보지 못했는지 확인한다
  if (!f)
    fatal("missing digits after '\\x'");
  if (n > 255)
    fatal("value out of range after '\\x'");
  return n;
}

// 문자 또는 문자열 리터럴에서 다음 문자를 반환한다
static int scanch(void) {
  int i, c, c2;

  // 다음 입력 문자를 가져오고 백슬래시로 시작하는 메타 문자를 해석한다
  c = next();
  if (c == '\\') {
    switch (c = next()) {
      ...
      case '0':
      case '1':
      case '2':
      case '3':
      case '4':
      case '5':
      case '6':
      case '7':			// SubC의 코드
        for (i = c2 = 0; isdigit(c) && c < '8'; c = next()) {
          if (++i > 3)
            break;
          c2 = c2 * 8 + (c - '0');
        }
        putback(c);             // 첫 번째 8진수가 아닌 문자를 다시 넣는다
        return (c2);
      case 'x':
        return hexchar();	// SubC의 코드
      default:
        fatalc("unknown escape sequence", c);
    }
  }
  return (c);                   // 일반 문자를 반환한다
}
```

이 코드 역시 깔끔하고 우아하다. 하지만 이제 16진수 변환을 위한 코드가 두 개, 진수 변환을 위한 코드가 세 개가 되었으므로 여전히 리팩토링의 여지가 있다.


# 결론 및 다음 단계

이번 단계에서는 주로 스캐너에 변화를 주었다. 혁신적인 변화는 아니었지만, 컴파일러가 스스로 컴파일할 수 있도록 하기 위해 필요한 작은 작업들이었다.

앞으로 해결해야 할 두 가지 중요한 과제는 정적 함수와 변수, 그리고 `sizeof()` 연산자다.

다음 컴파일러 작성 단계에서는 아직 조금 두려운 `static`보다는 `sizeof()` 연산자 작업을 먼저 진행할 것 같다! [다음 단계](../47_Sizeof/Readme.md)



# 48장: `static`의 부분 집합

실제 C 컴파일러에서는 세 가지 유형의 `static` 요소가 존재한다:

1. **정적 함수**: 함수가 선언된 소스 파일 내에서만 선언이 보인다.
2. **정적 전역 변수**: 변수가 선언된 소스 파일 내에서만 선언이 보인다.
3. **정적 지역 변수**: 전역 변수와 유사하지만, 변수가 선언된 함수 내에서만 보인다.

첫 두 가지는 구현하기 비교적 간단하다:

1. 선언 시 전역 변수로 추가한다.
2. 해당 소스 코드 파일이 닫힐 때 전역 심볼 테이블에서 제거한다.

세 번째 유형은 훨씬 복잡하다. 예를 들어, 두 개의 비공개 카운터를 유지하고 이를 증가시키는 함수를 만들어 보자:

```c
int inc_counter1(void) {
  static int counter= 0;
  return(counter);
}

int inc_counter2(void) {
  static int counter= 0;
  return(counter);
}
```

두 함수는 각각 자신의 `counter` 변수를 확인한다. 그리고 두 카운터의 값은 함수 호출 간에 유지된다. 변수의 지속성은 이들을 "전역"처럼 만드지만, 하나의 함수 내에서만 보이기 때문에 일종의 "지역" 변수로도 볼 수 있다.

여기서 [클로저](https://en.wikipedia.org/wiki/Closure_(computer_programming))에 대한 언급을 덧붙일 수 있지만, 이론적인 측면은 다루지 않는다. 주된 이유는 세 번째 유형의 `static` 요소를 구현하지 않기 때문이다.

왜 구현하지 않을까? 주된 이유는 전역과 지역 특성을 동시에 가지는 무언가를 구현하기가 어렵기 때문이다. 또한, 현재 컴파일러에는 정적 지역 변수가 없어서(코드를 다시 작성한 이후로) 이 기능이 필요하지 않다.

대신, 정적 전역 함수와 정적 전역 변수에 집중할 수 있다.


## 새로운 키워드와 토큰

새로운 키워드 `static`과 새로운 토큰 T_STATIC이 추가되었다. 항상 그렇듯이, 변경 사항을 확인하려면 `scan.c` 파일을 살펴보면 된다.


## `static` 키워드 파싱

`static` 키워드는 `extern`과 동일한 위치에서 파싱된다. 또한 로컬 컨텍스트에서 `static` 키워드를 사용하려는 시도를 거부해야 한다. 이를 위해 `decl.c` 파일에서 `parse_type()` 함수를 수정한다:

```c
// 현재 토큰을 파싱하고, 기본 타입 enum 값을 반환한다.
// 복합 타입에 대한 포인터를 반환할 수도 있으며, 타입의 클래스를 수정할 수 있다.
int parse_type(struct symtable **ctype, int *class) {
  int type, exstatic = 1;

  // 클래스가 extern 또는 static으로 변경되었는지 확인
  while (exstatic) {
    switch (Token.token) {
      case T_EXTERN:
        if (*class == C_STATIC)
          fatal("extern과 static을 동시에 사용할 수 없음");
        *class = C_EXTERN;
        scan(&Token);
        break;
      case T_STATIC:
        if (*class == C_LOCAL)
          fatal("컴파일러는 로컬 선언에서 static을 지원하지 않음");
        if (*class == C_EXTERN)
          fatal("extern과 static을 동시에 사용할 수 없음");
        *class = C_STATIC;
        scan(&Token);
        break;
      default:
        exstatic = 0;
    }
  }
  ...
}
```

`static` 또는 `extern`을 발견하면, 먼저 현재 선언 클래스에서 이 사용이 적법한지 확인한다. 그런 다음 `class` 변수를 업데이트한다. 두 토큰 중 어느 것도 발견되지 않으면 루프를 빠져나온다.

이제 `static` 선언을 위한 타입이 표시되었으므로, 이것을 전역 심볼 테이블에 어떻게 추가할까? 컴파일러의 거의 모든 부분에서 C_GLOBAL 클래스를 사용하는 곳을 C_STATIC도 포함하도록 변경해야 한다. 이는 여러 파일에서 여러 번 발생하지만, 다음과 같은 코드를 주의 깊게 살펴봐야 한다:

```c
    if (class == C_GLOBAL || class == C_STATIC) ...
```

이러한 코드는 `cg.c`, `decl.c`, `expr.c`, `gen.c` 파일에서 찾을 수 있다.


## `static` 선언 제거하기

`static` 선언을 모두 파싱한 후에는 전역 심볼 테이블에서 이를 제거해야 한다. `main.c` 파일의 `do_compile()` 함수에서 입력 파일을 닫은 직후에 다음과 같이 처리한다:

```c
  genpreamble();                // 프리앰블 출력
  global_declarations();        // 전역 선언 파싱
  genpostamble();               // 포스트앰블 출력
  fclose(Outfile);              // 출력 파일 닫기
  freestaticsyms();             // 파일 내 모든 static 심볼 해제
```

이제 `sym.c` 파일의 `freestaticsyms()` 함수를 살펴보자. 전역 심볼 테이블을 순회하며 `static` 노드를 발견하면 리스트에서 해당 노드를 제거하도록 연결을 재구성한다. 연결 리스트 코드에 익숙하지 않아서 가능한 모든 경우의 수를 종이에 적어가며 다음 코드를 작성했다:

```c
// 전역 심볼 테이블에서 모든 static 심볼 제거
void freestaticsyms(void) {
  // g는 현재 노드를, prev는 이전 노드를 가리킴
  struct symtable *g, *prev= NULL;

  // 전역 테이블을 순회하며 static 항목 탐색
  for (g= Globhead; g != NULL; g= g->next) {
    if (g->class == C_STATIC) {

      // 이전 노드가 존재하면 prev 포인터를 재조정하여 현재 노드를 건너뛰도록 함
      // 이전 노드가 없으면 g가 헤드이므로 Globhead에 동일한 작업 수행
      if (prev != NULL) prev->next= g->next;
      else Globhead->next= g->next;

      // g가 테일이면 Globtail을 이전 노드(존재할 경우) 또는 Globhead로 설정
      if (g == Globtail) {
        if (prev != NULL) Globtail= prev;
        else Globtail= Globhead;
      }
    }
  }

  // 다음 노드로 이동하기 전에 prev를 g로 설정
  prev= g;
}
```

이 코드의 전체적인 효과는 `static` 선언을 전역 선언처럼 처리하되, 입력 파일 처리가 끝나면 심볼 테이블에서 제거하는 것이다.


## 변경 사항 테스트

변경 사항을 테스트하기 위해 `tests/input116.c`부터 `tests/input118.c`까지 세 개의 프로그램이 있다. 첫 번째 프로그램을 살펴보자:

```c
#include <stdio.h>

static int counter=0;
static int fred(void) { return(counter++); }

int main(void) {
  int i;
  for (i=0; i < 5; i++)
    printf("%d\n", fred());
  return(0);
}
```

이 코드의 어셈블리 출력 일부를 살펴보자:

```
        ...
        .data
counter:
        .long   0
        .text
fred:
        pushq   %rbp
        movq    %rsp, %rbp
        addq    $0,%rsp
        ...
```

일반적으로 `counter`와 `fred`는 `.globl` 마킹으로 장식되었을 것이다. 하지만 이제 이들이 `static`으로 선언되었기 때문에 라벨은 있지만, 어셈블러에게 이들을 전역적으로 보이지 않도록 지시한다.


## 결론 및 다음 단계

`static` 키워드에 대해 고민이 많았지만, 가장 복잡한 세 번째 대안을 구현하지 않기로 결정하니 그리 어렵지 않았다. 실제로 힘들었던 부분은 코드 전체를 살펴보며 `C_GLOBAL`을 사용한 부분을 찾고, 적절한 `C_STATIC` 코드를 추가하는 작업이었다.

이제 컴파일러 개발의 다음 단계로, [삼항 연산자](https://en.wikipedia.org/wiki/%3F:)를 다룰 차례라고 생각한다. [다음 단계](../49_Ternary/Readme.md)로 넘어가자.



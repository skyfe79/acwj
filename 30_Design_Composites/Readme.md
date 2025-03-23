# 30장: 구조체, 공용체, 열거형 설계

이번 장에서는 컴파일러 개발 여정의 일환으로 구조체, 공용체, 열거형을 구현하기 위한 설계 아이디어를 간략히 소개한다. 함수를 구현할 때와 마찬가지로, 이 모든 것을 완성하려면 여러 단계를 거쳐야 한다.

또한 심볼 테이블을 단일 배열에서 여러 개의 단일 연결 리스트로 재작성하기로 결정했다. 이전에도 이 의도를 언급한 바 있다: 복합 타입을 구현하는 방법에 대한 아이디어를 고려했을 때, 이 시점에서 심볼 테이블 구현을 재작성하는 것이 중요하다고 판단했다.

코드 변경 사항을 살펴보기 전에, 복합 타입이 정확히 무엇인지 먼저 알아보자.


## 복합 타입, 열거형, 그리고 타입 정의

C 언어에서 [구조체](https://en.wikipedia.org/wiki/Struct_(C_programming_language))와 [공용체](https://en.wikipedia.org/wiki/Union_type#C/C++)는 *복합 타입*으로 알려져 있다. 구조체나 공용체 변수는 여러 멤버를 포함할 수 있다. 차이점은 구조체의 경우 멤버들이 메모리에서 겹치지 않도록 보장되지만, 공용체는 모든 멤버가 동일한 메모리 위치를 공유하도록 설계된다.

구조체 타입의 예시는 다음과 같다:

```c
struct foo {
  int  a;
  int  b;
  char c;
};

struct foo fred;
```

변수 `fred`는 `struct foo` 타입이며, 세 개의 멤버 `a`, `b`, `c`를 가지고 있다. 이제 `fred`에 대해 다음과 같이 세 가지 할당을 수행할 수 있다:

```c
  fred.a= 4;
  fred.b= 7;
  fred.c= 'x';
```

이렇게 하면 세 값이 각각 `fred`의 멤버에 저장된다.

반면, 공용체 타입의 예시는 다음과 같다:

```c
union bar {
  int  a;
  int  b;
  char c;
};

union bar jane;
```

다음과 같은 문장을 실행하면:

```c
  jane.a= 5;
  printf("%d\n", jane.b);
```

값 5가 출력된다. 이는 `jane` 공용체에서 `a`와 `b` 멤버가 동일한 메모리 위치를 공유하기 때문이다.


### 열거형(Enums)

열거형은 구조체나 공용체처럼 복합 타입을 정의하지 않지만, 여기서 설명하겠다. C 언어에서 [열거형](https://en.wikipedia.org/wiki/Enumerated_type#C)은 기본적으로 정수 값에 이름을 부여하는 방법이다. 열거형은 이름이 붙은 정수 값들의 목록을 나타낸다.

예를 들어, 다음과 같이 새로운 식별자를 정의할 수 있다:

```c
enum { apple=1, banana, carrot, pear=10, peach, mango, papaya };
```

이제 다음과 같이 이름이 붙은 정수 값을 얻는다:

|  이름   | 값  |
|:------:|:---:|
| apple  |  1  |
| banana |  2  |
| carrot |  3  |
| pear   | 10  |
| peach  | 11  |
| mango  | 12  |
| papaya | 13  |

열거형과 관련해 몇 가지 흥미로운 문제가 있는데, 아래에서 이를 다룰 것이다.


### 타입 정의(Typedefs)

이 시점에서 타입 정의(typedef)에 대해 간략히 언급할 필요가 있다. 비록 컴파일러가 스스로를 컴파일하는 데는 타입 정의를 구현할 필요가 없지만, 타입 정의는 기존 타입에 새로운 이름을 부여하는 방법이다. 주로 구조체(struct)와 공용체(union)의 이름을 간결하게 만들기 위해 사용한다.

이전 예제를 활용해 다음과 같이 작성할 수 있다:

```c
typedef struct foo Husk;
Husk kim;
```

여기서 `kim`은 `Husk` 타입이며, 이는 `kim`이 `struct foo` 타입과 동일하다는 의미이다.


## 타입 vs 심볼

구조체(struct), 공용체(union), typedef는 새로운 타입을 정의한다. 그렇다면 이들이 변수와 함수 정의를 담는 심볼 테이블과 어떤 관련이 있을까? 열거형(enum)은 정수 리터럴에 이름을 붙인 것일 뿐, 변수나 함수가 아니다.

핵심은 이들 모두 *이름*을 가진다는 점이다. 구조체나 공용체의 이름, 그 멤버들의 이름, 멤버들의 타입, 열거형 값들의 이름, 그리고 typedef로 정의된 이름들 모두가 해당된다.

이러한 이름들을 어딘가에 저장하고, 필요할 때 찾을 수 있어야 한다. 구조체/공용체 멤버의 경우, 그들의 기본 타입을 찾아야 한다. 열거형 이름의 경우, 해당하는 정수 리터럴 값을 조회해야 한다.

이러한 이유로, 나는 심볼 테이블을 사용해 이 모든 것을 저장할 것이다. 하지만 심볼 테이블을 여러 개의 특정 목록으로 나누어, 원하는 것을 찾고 원하지 않는 것을 피할 수 있도록 해야 한다.


## 심볼 테이블 구조 재설계

먼저, 다음과 같은 구조로 시작한다:

+ 전역 변수와 함수를 위한 단일 연결 리스트
+ 현재 함수의 지역 변수를 위한 단일 연결 리스트
+ 현재 함수의 매개변수를 위한 단일 연결 리스트

기존의 배열 기반 심볼 테이블에서는 전역 변수와 함수를 검색할 때 함수 매개변수를 건너뛰어야 했다. 따라서 함수의 매개변수를 위한 별도의 리스트를 추가한다:

```c
struct symtable {
  char *name;                   // 심볼의 이름
  int stype;                    // 심볼의 구조적 타입
  ...
  struct symtable *next;        // 한 리스트에서 다음 심볼
  struct symtable *member;      // 함수의 첫 번째 매개변수
};
```

다음 코드 조각을 예로 들어, 이 구조가 어떻게 동작하는지 시각적으로 살펴보자:

```c
  int a;
  char b;
  void func1(int x, int y);
  void main(int argc, char **argv) {
    int loc1;
    int loc2;
  }
```

이 코드는 세 개의 심볼 테이블 리스트에 다음과 같이 저장된다:

![](Figs/newsymlists.png)

세 리스트의 시작점을 가리키는 세 개의 "헤드"가 있다. 이제 전역 심볼 리스트를 순회할 때 매개변수를 건너뛰지 않아도 된다. 각 함수는 매개변수를 자체 리스트로 관리하기 때문이다.

함수의 본문을 파싱할 때, 매개변수 리스트를 함수의 매개변수 리스트로 지정한다. 그런 다음, 지역 변수가 선언되면 단순히 지역 변수 리스트에 추가된다.

함수의 본문 파싱과 어셈블리 코드 생성이 완료되면, 매개변수와 지역 리스트를 다시 비운다. 이 과정은 전역적으로 보이는 함수의 매개변수 리스트를 방해하지 않는다. 이것이 심볼 테이블 재작성의 현재 진행 상황이다. 하지만 구조체, 공용체, 열거형을 어떻게 구현할 수 있는지는 아직 보여주지 않는다.


## 흥미로운 문제와 고려사항

기존 심볼 테이블 노드와 단일 연결 리스트를 구조체(struct), 공용체(union), 열거형(enum)을 지원하도록 확장하는 방법을 살펴보기 전에, 먼저 이들과 관련된 몇 가지 흥미로운 문제를 고려해야 한다.


### 유니언

유니언부터 시작해보자. 첫째, 구조체 안에 유니언을 넣을 수 있다. 둘째, 유니언은 이름이 필요하지 않다. 셋째, 유니언을 담기 위해 구조체에 별도의 변수를 선언할 필요가 없다. 예를 들어:

```c
#include <stdio.h>
struct fred {
  int x;
  union {
    int a;
    int b;
  };            // 이 유니언 타입의 변수를 선언할 필요 없음
};

int main() {
  struct fred foo;
  foo.x= 5;
  foo.a= 12;                            // a는 구조체 멤버처럼 취급됨
  foo.b= 13;                            // b는 구조체 멤버처럼 취급됨
  printf("%d %d\n", foo.x, foo.a);      // 5와 13 출력
}
```

이를 지원할 수 있어야 한다. 익명 유니언(그리고 구조체)은 쉽다: 심볼 테이블 노드의 `name`을 NULL로 설정하면 된다. 하지만 이 유니언을 위한 변수 이름이 없다: 구조체의 멤버 이름도 NULL로 설정함으로써 이를 구현할 수 있을 것 같다. 즉:

![](Figs/structunion1.png)


### 열거형(Enums)

열거형을 사용해 본 적은 있지만, 실제로 구현에 대해 깊이 생각해 본 적은 없었다. 그래서 열거형을 "깨뜨릴" 수 있는지 확인하기 위해 다음과 같은 C 프로그램을 작성했다:

```c
#include <stdio.h>

enum fred { bill, mary, dennis };
int fred;
int mary;
enum fred { chocolate, spinach, glue };
enum amy { garbage, dennis, flute, amy };
enum fred x;
enum { pie, piano, axe, glyph } y;

int main() {
  x= bill;
  y= pie;
  y= bill;
  x= axe;
  x= y;
  printf("%d %d %ld\n", x, y, sizeof(x));
}
```

이 프로그램을 통해 다음과 같은 질문에 대한 답을 찾아보려고 한다:

1. 열거형 목록을 다른 요소로 재선언할 수 있는가? 예를 들어 `enum fred`와 `enum fred`.
2. 열거형 목록과 같은 이름의 변수를 선언할 수 있는가? 예를 들어 `fred`.
3. 열거형 값과 같은 이름의 변수를 선언할 수 있는가? 예를 들어 `mary`.
4. 한 열거형 목록에서 사용한 열거형 값의 이름을 다른 열거형 목록에서 재사용할 수 있는가? 예를 들어 `dennis`와 `dennis`.
5. 한 열거형 목록의 값을 다른 열거형 목록으로 선언된 변수에 할당할 수 있는가?
6. 서로 다른 열거형 목록으로 선언된 변수 간에 값을 할당할 수 있는가?

`gcc`가 출력한 오류와 경고는 다음과 같다:

```c
z.c:4:5: error: ‘mary’ redeclared as different kind of symbol
 int mary;
     ^~~~
z.c:2:19: note: previous definition of ‘mary’ was here
 enum fred { bill, mary, dennis };
                   ^~~~
z.c:5:6: error: nested redefinition of ‘enum fred’
 enum fred { chocolate, spinach, glue };
      ^~~~
z.c:5:6: error: redeclaration of ‘enum fred’
z.c:2:6: note: originally defined here
 enum fred { bill, mary, dennis };
      ^~~~
z.c:6:21: error: redeclaration of enumerator ‘dennis’
 enum amy { garbage, dennis, flute, amy };
                     ^~~~~~
z.c:2:25: note: previous definition of ‘dennis’ was here
 enum fred { bill, mary, dennis };
                         ^~~~~~
```

위 프로그램을 수정하고 컴파일해 본 결과, 다음과 같은 결론을 얻었다:

1. `enum fred`를 재선언할 수 없다. 이는 열거형 목록의 이름을 기억해야 하는 유일한 경우로 보인다.
2. 열거형 목록 식별자 `fred`를 변수 이름으로 재사용할 수 있다.
3. 열거형 값 식별자 `mary`를 다른 열거형 목록이나 변수 이름으로 재사용할 수 없다.
4. 열거형 값은 어디에나 할당할 수 있다. 이들은 단순히 정수 리터럴 값에 대한 이름으로 취급되는 것으로 보인다.
5. `enum`과 `enum X`를 타입으로 사용하는 대신 `int`로 대체할 수 있는 것으로 보인다.


## 설계 고려 사항

이제 우리가 원하는 사항을 나열할 준비가 되었다:

+ 이름이 있는 구조체와 없는 구조체의 목록. 각 구조체의 멤버 이름과 멤버의 타입 세부 정보를 포함한다. 또한, 구조체의 '기준' 위치에서 멤버의 메모리 오프셋도 필요하다.
+ 이름이 있는 구조체와 없는 구조체에 대해서도 동일한 정보가 필요하지만, 오프셋은 항상 0이 된다.
+ 열거형 목록의 이름과 실제 열거형 이름, 그리고 그에 해당하는 값을 포함한 목록.
+ 심볼 테이블에서, 비복합 타입에 대한 기존 `type` 정보가 필요하다. 하지만 심볼이 구조체나 공용체인 경우, 관련된 복합 타입에 대한 포인터도 필요하다.
+ 구조체가 자신을 가리키는 포인터를 멤버로 가질 수 있으므로, 멤버의 타입이 동일한 구조체를 다시 가리킬 수 있어야 한다.


## 심볼 테이블 노드 구조 변경

현재 단일 연결 리스트로 구현된 심볼 테이블 노드에 다음과 같은 변경 사항을 적용했다. 변경된 부분은 굵은 글씨로 표시했다.

<pre>
struct symtable {
  char *name;                   // 심볼의 이름
  int type;                     // 심볼의 기본 타입
  <b>struct symtable *ctype;       // 필요한 경우, 복합 타입에 대한 포인터</b>
  int stype;                    // 심볼의 구조적 타입
  int class;                    // 심볼의 저장 클래스
  union {
    int size;                   // 심볼의 요소 개수
    int endlabel;               // 함수의 경우, 종료 레이블
    <b>int intvalue;               // enum 심볼의 경우, 연관된 값</b>
  };
  union {
    int nelems;                 // 함수의 경우, 매개변수 개수
    int posn;                   // 로컬 변수의 경우, 스택 베이스 포인터로부터의 음수 오프셋
  };
  struct symtable *next;        // 하나의 리스트에서 다음 심볼
  struct symtable *member;      // 함수, 구조체, 공용체, 혹은 enum의 첫 번째 멤버
};                              
</pre>

이 새로운 노드 구조와 함께 총 여섯 개의 연결 리스트를 사용한다:

 + 전역 변수와 함수를 위한 단일 연결 리스트
 + 현재 함수의 로컬 변수를 위한 단일 연결 리스트
 + 현재 함수의 매개변수를 위한 단일 연결 리스트
 + 정의된 구조체 타입을 위한 단일 연결 리스트
 + 정의된 공용체 타입을 위한 단일 연결 리스트
 + 정의된 enum 이름과 열거형 값을 위한 단일 연결 리스트

이 변경 사항은 심볼 테이블의 유연성을 높이고, 복합 타입과 enum 값을 더 효과적으로 관리할 수 있도록 한다. 특히 `ctype` 포인터와 `intvalue` 필드의 추가는 타입 시스템의 표현력을 크게 향상시킨다.


## 새로운 심볼 테이블 노드의 활용 사례

앞서 열거한 여섯 가지 리스트에서 위 구조체의 각 필드가 어떻게 활용되는지 살펴보자.


### 새로운 타입 소개

P_STRUCT와 P_UNION이라는 두 가지 새로운 타입을 소개한다. 이 타입들에 대해 아래에서 자세히 설명한다.


### 전역 변수와 함수, 매개변수 변수, 지역 변수

+ *name*: 변수나 함수의 이름
+ *type*: 변수의 타입 또는 함수의 반환 값과 4비트 간접 참조 수준
+ *ctype*: 변수가 P_STRUCT 또는 P_UNION 타입일 경우, 이 필드는 관련 단일 연결 리스트에 있는 구조체 또는 공용체 정의를 가리킴
+ *stype*: 변수나 함수의 구조적 타입: S_VARIABLE, S_FUNCTION, S_ARRAY 중 하나
+ *class*: 변수의 저장 클래스: C_GLOBAL, C_LOCAL, C_PARAM 중 하나
+ *size*: 변수의 경우 전체 바이트 크기. 배열의 경우 배열의 요소 수. 나중에 `sizeof()`를 구현할 때 사용
+ *endlabel*: 함수의 경우, `return`으로 돌아갈 수 있는 끝 레이블
+ *nelems*: 함수의 경우, 매개변수의 수
+ *posn*: 지역 변수와 매개변수의 경우, 스택 베이스 포인터로부터의 음수 오프셋
+ *next*: 이 리스트에서 다음 심볼을 가리킴
+ *member*: 함수의 경우, 첫 번째 매개변수의 노드를 가리킴. 변수의 경우 NULL


### 구조체 타입

 + *name*: 구조체 타입의 이름. 익명 구조체인 경우 NULL이다.
 + *type*: 항상 P_STRUCT이다. 실제로는 필요하지 않다.
 + *ctype*: 사용되지 않는다.
 + *stype*: 사용되지 않는다.
 + *class*: 사용되지 않는다.
 + *size*: 구조체의 전체 크기(바이트 단위). 나중에 `sizeof()`에서 사용된다.
 + *nelems*: 구조체 내 멤버의 개수.
 + *next*: 정의된 다음 구조체 타입을 가리킨다.
 + *member*: 첫 번째 구조체 멤버의 노드를 가리키는 포인터.


### 유니온 타입

 + *name*: 유니온 타입의 이름. 익명일 경우 NULL이다.
 + *type*: 항상 P_UNION이다. 실제로는 필수적이지 않다.
 + *ctype*: 사용되지 않는다.
 + *stype*: 사용되지 않는다.
 + *class*: 사용되지 않는다.
 + *size*: `sizeof()`에서 사용할 유니온의 총 바이트 크기.
 + *nelems*: 유니온의 멤버 수.
 + *next*: 정의된 다음 유니온 타입을 가리킨다.
 + *member*: 첫 번째 유니온 멤버의 노드를 가리키는 포인터.


### 구조체와 공용체 멤버

각 멤버는 기본적으로 변수와 유사한 특성을 가진다. 일반 변수와 크게 다르지 않다.

+ *name*: 멤버의 이름.
+ *type*: 변수의 타입과 4비트 간접 참조 수준을 포함.
+ *ctype*: 멤버가 P_STRUCT 또는 P_UNION 타입일 경우, 이 필드는 관련 단일 연결 리스트에서 해당 구조체나 공용체 정의를 가리킴.
+ *stype*: 멤버의 구조적 타입: S_VARIABLE 또는 S_ARRAY.
+ *class*: 사용되지 않음.
+ *size*: 변수의 경우 전체 바이트 크기. 배열의 경우 배열의 요소 개수. 이 값을 나중에 `sizeof()` 구현에 사용.
+ *posn*: 구조체/공용체의 시작점부터 멤버까지의 양수 오프셋.
+ *next*: 구조체/공용체 내 다음 멤버.
+ *member*: NULL.


### 열거형 목록 이름과 값 저장

아래 심볼과 암시적 값을 모두 저장하려고 한다:

```c
  enum fred { chocolate, spinach, glue };
  enum amy  { garbage, dennis, flute, couch };
```

`fred`와 `amy`를 연결한 다음, `fred`의 `member` 필드를 `chocolate`, `spinach`, `glue` 목록에 사용할 수 있다. 마찬가지로 `garbage` 등도 동일하게 처리할 수 있다.

하지만 실제로는 `fred`와 `amy` 이름이 열거형 목록 이름으로 재사용되지 않도록 하는 데 관심이 있다. 우리가 진정으로 관심 있는 것은 실제 열거형 이름과 그 값이다.

따라서 두 가지 "더미" 타입 값을 제안한다: `P_ENUMLIST`와 `P_ENUMVAL`. 그런 다음 다음과 같은 단일 차원의 목록을 구성한다:

```c
     fred  -> chocolate-> spinach ->   glue  ->    amy  -> garbage -> dennis -> ...
  P_ENUMLIST  P_ENUMVAL  P_ENUMVAL  P_ENUMVAL  P_ENUMLIST  P_ENUMVAL  P_ENUMVAL
```

이렇게 하면 `glue`라는 단어를 사용할 때 하나의 목록만 탐색하면 된다. 그렇지 않으면 `fred`를 찾고, `fred`의 멤버 목록을 탐색한 다음, `amy`에 대해서도 동일한 작업을 수행해야 한다. 하나의 목록을 사용하는 것이 더 쉬울 것이다.


## 지금까지 변경된 내용

이 문서의 앞부분에서, 심볼 테이블을 단일 배열에서 여러 개의 단일 연결 리스트로 재구성했다고 언급했다. 이 과정에서 `struct symtable` 노드에 다음과 같은 새로운 필드가 추가되었다:

```c
  struct symtable *next;        // 한 리스트 내의 다음 심볼
  struct symtable *member;      // 함수의 첫 번째 파라미터
```

이제 이러한 변경 사항을 간단히 살펴보자. 가장 먼저, 기능적인 변화는 전혀 없다는 점을 명심하자.


### 세 가지 심볼 테이블 리스트

이제 `data.h` 파일에 세 가지 심볼 테이블 리스트가 있다:

```c
// 심볼 테이블 리스트
struct symtable *Globhead, *Globtail;   // 전역 변수와 함수
struct symtable *Loclhead, *Locltail;   // 지역 변수
struct symtable *Parmhead, *Parmtail;   // 지역 매개변수
```

그리고 `sym.c` 파일의 모든 함수가 이 리스트를 사용하도록 재작성되었다. 리스트에 노드를 추가하는 일반적인 함수도 작성했다:

```c
// head 또는 tail이 가리키는 단일 연결 리스트에 노드를 추가한다
void appendsym(struct symtable **head, struct symtable **tail,
               struct symtable *node) {

  // 유효한 포인터인지 확인
  if (head == NULL || tail == NULL || node == NULL)
    fatal("Either head, tail or node is NULL in appendsym");

  // 리스트에 추가
  if (*tail) {
    (*tail)->next = node; *tail = node;
  } else *head = *tail = node;
  node->next = NULL;
}
```

이제 `newsym()` 함수가 있다. 이 함수는 심볼 테이블 노드의 모든 필드 값을 받아 새로운 노드를 `malloc()`으로 할당하고, 값을 채운 뒤 반환한다. 여기서 코드는 생략한다.

각 리스트에 대해 노드를 생성하고 리스트에 추가하는 함수가 있다. 한 가지 예는 다음과 같다:

```c
// 전역 심볼 리스트에 심볼을 추가한다
struct symtable *addglob(char *name, int type, int stype, int class, int size) {
  struct symtable *sym = newsym(name, type, stype, class, size, 0);
  appendsym(&Globhead, &Globtail, sym);
  return (sym);
}
```

리스트에서 심볼을 찾는 일반적인 함수도 있다. 이때 `list` 포인터는 리스트의 헤드를 가리킨다:

```c
// 특정 리스트에서 심볼을 검색한다.
// 찾은 노드의 포인터를 반환하거나, 없으면 NULL을 반환한다.
static struct symtable *findsyminlist(char *s, struct symtable *list) {
  for (; list != NULL; list = list->next)
    if ((list->name != NULL) && !strcmp(s, list->name))
      return (list);
  return (NULL);
}
```

그리고 리스트별로 `findXXX()` 함수가 있다.

`findsymbol()` 함수는 먼저 함수의 매개변수 리스트에서 심볼을 찾고, 그다음 함수의 지역 변수 리스트에서 찾고, 마지막으로 전역 변수 리스트에서 찾는다.

`findlocl()` 함수는 함수의 매개변수 리스트와 지역 변수 리스트에서만 심볼을 검색한다. 지역 변수를 선언할 때 재선언을 방지하기 위해 이 함수를 사용한다.

마지막으로, `clear_symtable()` 함수는 세 리스트의 헤드와 테일을 모두 NULL로 초기화하여 모든 리스트를 비운다.


### 파라미터와 로컬 리스트

글로벌 심볼 리스트는 각 소스 코드 파일을 파싱할 때 한 번만 초기화된다. 하지만 함수 본문 파싱을 시작할 때마다 a) 파라미터 리스트를 설정하고, b) 로컬 심볼 리스트를 초기화해야 한다.

이 과정은 다음과 같이 동작한다. `expr.c` 파일의 `param_declaration()` 함수에서 파라미터 리스트를 파싱할 때, 각 파라미터마다 `var_declaration()`을 호출한다. 이는 심볼 테이블 노드를 생성하고 파라미터 리스트(`Parmhead`와 `Parmtail`)에 추가한다. `param_declaration()`이 반환되면, `Parmhead`는 파라미터 리스트를 가리키게 된다.

전체 함수(이름, 파라미터 리스트, 함수 본문)를 파싱하는 `function_declaration()` 함수에서는 파라미터 리스트를 함수의 심볼 테이블 노드에 복사한다:

```c
    newfuncsym->nelems = paramcnt;
    newfuncsym->member = Parmhead;

    // 파라미터 리스트 초기화
    Parmhead = Parmtail = NULL;
```

`Parmhead`와 `Parmtail`을 `NULL`로 설정하여 파라미터 리스트를 초기화한다. 이렇게 하면 글로벌 파라미터 리스트를 통해 더 이상 이들을 검색할 수 없게 된다.

해결책은 전역 변수 `Functionid`를 함수의 심볼 테이블 엔트리로 설정하는 것이다:

```c
  Functionid = newfuncsym;
```

따라서 함수 본문을 파싱하기 위해 `compound_statement()`를 호출할 때, `Functionid->member`를 통해 파라미터 리스트에 여전히 접근할 수 있다. 이를 통해 다음과 같은 작업을 수행할 수 있다:

 + 파라미터 이름과 동일한 로컬 변수 선언을 방지
 + 파라미터 이름을 일반 로컬 변수로 사용

마지막으로, `function_declaration()`은 전체 함수를 포함하는 AST 트리를 반환한다. 이 트리는 `global_declarations()`로 전달되고, `gen.c`의 `genAST()`로 전달되어 어셈블리 코드를 생성한다. `genAST()`가 반환되면, `global_declarations()`는 `freeloclsyms()`를 호출하여 로컬 및 파라미터 리스트를 초기화하고 `Functionid`를 다시 `NULL`로 설정한다.


### 주목할 만한 다른 변경 사항

심볼 테이블을 여러 연결 리스트로 변경하면서 상당량의 코드를 다시 작성해야 했다. 전체 코드베이스를 일일이 살펴보지는 않겠지만, 몇 가지 눈에 띄는 변경 사항을 쉽게 확인할 수 있다. 예를 들어, 심볼 노드를 참조하던 코드가 `Symtable[n->id]`에서 `n->sym`으로 바뀌었다.

또한 `cg.c` 파일에 있는 많은 코드가 심볼 이름을 참조하는데, 이제는 `n->sym->name` 형태로 표시된다. 마찬가지로 `tree.c` 파일에서 AST 트리를 출력하는 코드에도 `n->sym->name`이 많이 포함되어 있다.


## 결론 및 다음 단계

이번 파트는 디자인과 재구현이 혼합된 과정이었다. 구조체, 공용체, 열거형을 구현할 때 어떤 문제에 직면할지 오랜 시간 고민했다. 그런 다음 이 새로운 개념을 지원하기 위해 심볼 테이블을 재설계했다. 마지막으로 이 새로운 개념을 구현하기 위해 심볼 테이블을 세 개의 연결 리스트로 다시 작성했다.

컴파일러 작성 여정의 다음 파트에서는 구조체 타입의 선언을 구현할 예정이다. 하지만 실제로 사용할 수 있는 코드를 작성하는 것은 그 다음 파트로 미룰 것이다. 이 두 단계가 완료되면, 세 번째 파트에서 공용체를 구현할 수 있을 것이다. 그다음 네 번째 파트에서 열거형을 다룰 것이다. 기대해 주세요! [다음 단계](../31_Struct_Declarations/Readme.md)



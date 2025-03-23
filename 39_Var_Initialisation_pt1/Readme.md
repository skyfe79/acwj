# 39장: 변수 초기화, 1부

우리 컴파일러가 지원하는 언어에서 변수를 선언할 수 있지만, 동시에 초기화할 수는 없다. 이번 장(그리고 다음 장들)에서는 이 문제를 해결하기 위해 작업할 것이다.

실제 구현에 들어가기 전에 이 문제에 대해 생각해보는 것이 중요하다. 코드 일부를 공유할 수 있는 방법을 고안할 수 있기를 바란다. 그래서 아래에 내 생각을 정리해보려 한다.

현재, 우리는 세 곳에서 변수를 선언할 수 있다:

  + 전역 변수는 함수 외부에서 선언된다.
  + 함수 매개변수는 매개변수 목록에서 선언된다.
  + 지역 변수는 함수 내부에서 선언된다.

각 선언은 변수의 타입과 이름을 포함한다.

초기화 측면에서:

  + 함수 매개변수는 초기화할 수 없다. 함수 호출자의 인자로부터 값이 복사되기 때문이다.
  + 전역 변수는 표현식으로 초기화할 수 없다. 표현식의 어셈블리 코드를 실행할 함수가 없기 때문이다.
  + 지역 변수는 표현식으로 초기화할 수 있다.

또한 타입 정의 뒤에 변수 이름 목록을 가지길 원한다. 이는 처리해야 할 유사점과 차이점이 있음을 의미한다. 반-BNF 문법으로 표현하면 다음과 같다:

```
global_declaration: type_definition global_var_list ';' ;

global_var_list: global_var
               | global_var ',' global_var_list  ;

global_var: variable_name
          | variable_name '=' literal_value ;

local_declaration: type_definition local_var_list ';' ;

local_var_list: local_var
              | local_var ',' local_var_list  ;

local_var: variable_name
         | variable_name '=' expression ;

parameter_list: parameter
              | parameter ',' parameter_list ;

parameter: type_definition variable_name ;
```

우리 컴파일러에서 지원하고자 하는 예제 세트는 다음과 같다.


### 전역 변수 선언

```c
  int   x= 5;
  int   a, b= 7, c, d= 6;
  char *e, f;                           // e는 포인터지만 f는 아니다!
  char  g[]= "Hello", *h= "foo";
  int   j[]= { 1, 2, 3, 4, 5 };
  char *k[]= { "fish", "cat", "ball" };
  int   l[70];
```

추가한 주석은 중요한 의미를 담고 있다. 앞에 있는 타입을 파싱하고, 각 변수에 대해 접두사 '*' 또는 접미사 '[ ]'를 파싱하여 포인터인지 배열인지 결정해야 한다.

위 예제에서 보여준 것처럼 단일 차원의 초기화 값 목록만 다룬다.


### 지역 변수 선언

위 예제들도 동일하게 적용되지만, 다음과 같은 지역 변수 선언도 가능해야 한다:

```c
  int u= x + 3;
  char *v= k[0];
  char *w= k[b-6];
  int y= 2*b+c, z= l[d] + j[2*x+5];
```

원래는 `int list[]= { x+2, a+b, c*d, u+j[3], j[x] + j[a] };`와 같은 구문을 파싱하려고 했지만, 이는 처리하기에 상당히 복잡해 보인다. 따라서 리터럴 값들의 목록만 허용하거나, 지역 범위에서 배열 초기화를 아예 허용하지 않는 방향으로 진행할 생각이다.


## 이제 어떻게 해야 할까?

위의 예제를 살펴본 후, 약간 두려움을 느낄 수 있다. 전역 변수 초기화는 할 수 있지만, 리스트 내 각 변수의 타입을 파싱하는 방식을 다시 작성해야 한다. 그런 다음 '='를 파싱할 수 있다.

만약 전역 스코프에 있다면, 리터럴 값을 파싱하는 함수를 호출한다.

로컬 스코프에 있다면, 기존의 `binexpr()` 함수를 사용할 수 없다. 왜냐하면 이 함수는 왼쪽의 변수 이름을 파싱하고 내부적으로 lvalue AST 노드를 생성하기 때문이다. 아마도 이 lvalue AST 노드를 직접 생성하고, 이 노드의 포인터를 `binexpr()`에 전달할 수 있다. 그리고 `binexpr()`에 다음과 같은 코드를 추가한다:

```
  if we got an lvalue pointer {
    set left to this pointer
  } else {
    left = prefix();
    deal with the operator token
  }
  rest of the existing code
```

좋다. 이제 어느 정도 계획이 생겼다. 먼저 리팩토링을 진행할 것이다. 그리고 첫 번째 작업은 타입과 변수 이름을 파싱하는 방식을 다시 작성하여 리스트 형태로 파싱할 수 있게 하는 것이다.


## 리팩토링 살펴보기

방금 코드 리팩토링을 마쳤다. 단순히 코드를 재배열한 것처럼 보일 수 있지만, 사실 그렇지 않다. 이제 새로운 함수들이 어떻게 서로 호출되는지 보여주고, 각 함수가 어떤 역할을 하는지 설명하겠다.

새로운 `decl.c` 파일의 호출 그래프를 그려보았다:

![](Figs/decl_call_graph.png)

가장 상위에는 `global_declarations()`가 있다. 이 함수는 전역적으로 선언된 모든 것을 파싱한다. 단순히 반복문을 돌면서 `declaration_list()`를 호출한다. 또는 함수 내부에서 타입 토큰(`int`, `char` 등)을 만나면, 변수를 파싱하기 위해 `declaration_list()`를 호출한다.

`declaration_list()`는 새로 추가된 함수다. 이 함수는 `parse_type()`를 호출해 타입(`int`, `char`, 구조체, 공용체, typedef 등)을 가져온다. 이 타입은 리스트의 기본 타입이지만, 리스트의 각 항목은 이 타입을 수정할 수 있다. 예를 들면 다음과 같다:

```c
  int a, *b, c[40], *d[100];
```

`declaration_list()`에서는 리스트의 각 선언에 대해 반복문을 실행한다. 각 선언에 대해 `parse_stars()`를 호출해 기본 타입이 어떻게 수정되었는지 확인한다. 이 시점에서 개별 선언의 식별자를 파싱할 수 있으며, 이 작업은 `symbol_declaration()`에서 수행된다. 다음 토큰에 따라 다음 함수 중 하나를 호출한다:

  + 함수인 경우 `function_declaration()`
  + 배열인 경우 `array_declaration()`
  + 스칼라 변수인 경우 `scalar_declaration()`

함수 선언에는 매개변수가 있을 수 있으므로, `parameter_declaration_list()`를 호출해 이를 처리한다. 물론 매개변수 리스트도 선언이기 때문에, `declaration_list()`를 호출해 처리한다!

왼쪽에는 `parse_type()`가 있다. 이 함수는 `int`와 `char` 같은 일반적인 타입을 가져오지만, 구조체, 공용체, 열거형, typedef 같은 새로운 타입도 여기서 파싱한다.

`typedef_declaration()`에서 typedef를 파싱하는 것은 기존 타입에 별칭을 붙이는 것이기 때문에 쉬워야 한다. 하지만 다음과 같은 경우도 있다:

```c
typedef char * charptr;
```

`parse_type()`는 `*` 토큰을 처리하지 않기 때문에, `typedef_declaration()`은 별칭을 만들기 전에 기본 타입이 어떻게 수정되었는지 확인하기 위해 `parse_stars()`를 직접 호출해야 한다.

열거형 선언은 `enum_declaration()`이 처리한다. 구조체와 공용체의 경우 `composite_declaration()`을 호출한다. 그리고 짐작했겠지만, 새로운 구조체나 공용체 내부의 멤버들은 멤버 선언 리스트를 형성하므로, 이를 파싱하기 위해 `declaration_list()`를 호출한다!


## 리그레션 테스트

이제 약 80개의 개별 테스트를 보유하고 있어서 매우 기쁘다. `decl.c`를 안전하게 리팩토링하려면 새 코드가 이전과 동일한 에러나 어셈블리 출력을 생성하는지 확인할 수 있어야 하기 때문이다.


## 새로운 기능

이번 단계는 주로 변수 초기화를 준비하기 위한 리디자인이 중심이지만, 이제 전역 및 지역 변수 선언에서 리스트를 지원한다. 따라서 새로운 테스트 케이스를 추가했다:

```c
// tests/input84.c, 지역 변수
int main() {
  int x, y;
  x=2; y=3;
  ..
}

//input88.c, 전역 변수
struct foo {
  int x;
  int y;
} fred, mary;
```


## 결론 및 다음 단계

타입 뒤에 변수 목록을 파싱하는 기능을 추가했기 때문에 조금 더 기분이 좋다. 예를 들어 `int a, *b, **c;` 같은 구문을 처리할 수 있게 됐다. 또한 선언문과 함께 할당 기능을 구현해야 하는 부분에 코드 주석도 추가해 뒀다.

컴파일러 개발의 다음 단계에서는 전역 변수 선언과 할당 기능을 추가해 볼 예정이다. [다음 단계](../40_Var_Initialisation_pt2/Readme.md)



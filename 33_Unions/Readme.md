# 33장. 유니온과 멤버 접근 구현하기

유니온을 구현하는 일은 생각보다 쉬웠다. 그 이유는 유니온이 구조체와 비슷하지만, 유니온의 모든 멤버가 유니온의 시작 지점에서 오프셋 0에 위치하기 때문이다. 또한 유니온 선언 문법은 "union" 키워드를 제외하면 구조체와 동일하다.

이러한 특성 덕분에 기존에 구현한 구조체 코드를 재사용하고 수정하여 유니온을 처리할 수 있다.


## 새로운 키워드: "union"

`scan.c` 파일의 스캐너에 "union" 키워드와 T_UNION 토큰을 추가했다. 스캐닝을 수행하는 코드는 여기서 생략한다.


## 유니온 심볼 리스트

구조체와 마찬가지로, 유니온을 저장하기 위해 단일 연결 리스트를 사용한다(`data.h`에 정의됨):

```c
extern_ struct symtable *Unionhead, *Uniontail;   // 유니온 타입 리스트
```

`sym.c`에서는 새로운 유니온 타입 노드를 리스트에 추가하는 `addunion()` 함수와 주어진 이름으로 유니온 타입을 검색하는 `findunion()` 함수를 작성했다.

> 구조체와 유니온 리스트를 하나의 복합 타입 리스트로 병합하는 것을 고려 중이지만, 아직 실행하지는 않았다. 리팩토링을 진행할 때 함께 처리할 계획이다.


## 유니온 선언 파싱

기존의 구조체 파싱 코드를 수정하여 구조체와 유니온을 모두 파싱할 수 있도록 변경한다. 전체 함수가 아닌 변경된 부분만 설명한다.

`parse_type()` 함수에서 이제 T_UNION 토큰을 스캔하고, 구조체와 유니온 타입을 모두 파싱하는 함수를 호출한다:

```c
  case T_STRUCT:
    type = P_STRUCT;
    *ctype = composite_declaration(P_STRUCT);
    break;
  case T_UNION:
    type = P_UNION;
    *ctype = composite_declaration(P_UNION);
    break;
```

이 `composite_declaration()` 함수는 이전에 `struct_declaration()`으로 불렸던 함수이다. 이제 파싱할 타입을 인자로 받는다.


```c
#include <stdio.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdio.h>
```


## 유니온 표현식 파싱

유니온 선언과 마찬가지로, 표현식에서 구조체를 다루는 코드를 재사용할 수 있다. 실제로 `expr.c` 파일에서는 거의 변경할 부분이 없다.

```c
// 구조체나 유니온의 멤버 참조를 파싱하고 AST 트리를 반환한다.
// withpointer가 true면, 멤버에 대한 접근은 포인터를 통해 이뤄진다.
static struct ASTnode *member_access(int withpointer) {
  ...
  if (withpointer && compvar->type != pointer_to(P_STRUCT)
      && compvar->type != pointer_to(P_UNION))
    fatals("Undeclared variable", Text);
  if (!withpointer && compvar->type != P_STRUCT && compvar->type != P_UNION)
    fatals("Undeclared variable", Text);
```

이것이 전부다. 나머지 코드는 충분히 일반적이어서 유니온에도 그대로 사용할 수 있다. 주요 변경 사항은 `types.c` 파일의 한 함수뿐이다:

```c
// 타입과 복합 타입 포인터를 받아, 이 타입의 크기를 바이트 단위로 반환한다.
int typesize(int type, struct symtable *ctype) {
  if (type == P_STRUCT || type == P_UNION)
    return (ctype->size);
  return (genprimsize(type));
}
```


## 유니온 코드 테스트

다음은 테스트 프로그램 `test/input62.c`의 내용이다:

```c
int printf(char *fmt);

union fred {
  char w;
  int  x;
  int  y;
  long z;
};

union fred var1;
union fred *varptr;

int main() {
  var1.x= 65; printf("%d\n", var1.x);
  var1.x= 66; printf("%d\n", var1.x); printf("%d\n", var1.y);
  printf("The next two depend on the endian of the platform\n");
  printf("%d\n", var1.w); printf("%d\n", var1.z);

  varptr= &var1; varptr->x= 67;
  printf("%d\n", varptr->x); printf("%d\n", varptr->y);

  return(0);
}
```

이 코드는 유니온의 네 멤버가 모두 동일한 위치에 있는지 확인한다. 즉, 하나의 멤버를 변경하면 다른 모든 멤버에도 동일한 변경이 반영되는지 테스트한다. 또한 유니온에 대한 포인터 접근이 제대로 동작하는지도 확인한다.


## 결론 및 다음 단계

이번 장은 컴파일러 작성 여정에서 또 다른 간단하고 쉬운 부분이었다. 다음 단계에서는 열거형(enum)을 추가할 예정이다. [다음 단계](../34_Enums_and_Typedefs/Readme.md)



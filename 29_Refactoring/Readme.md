# 29장: 코드 리팩토링

구조체(struct), 공용체(union), 열거형(enum)을 컴파일러에 구현하는 과정에서 디자인 측면을 고민하다가, 심볼 테이블을 개선할 좋은 아이디어가 떠올랐다. 이 아이디어를 바탕으로 컴파일러 코드를 약간 리팩토링했다. 이번 장에서는 새로운 기능을 추가하지는 않았지만, 컴파일러의 일부 코드가 더 나아져서 만족스럽다.

구조체, 공용체, 열거형에 대한 디자인 아이디어가 더 궁금하다면 다음 장으로 넘어가도 좋다.


## 심볼 테이블 리팩토링

컴파일러를 작성하기 시작했을 때, 나는 방금 [SubC](http://www.t3x.org/subc/) 컴파일러의 코드를 읽고 나만의 주석을 추가한 상태였다. 그래서 초기 아이디어 중 상당수를 이 코드베이스에서 차용했다. 그 중 하나는 심볼 테이블을 배열로 구성하고, 한쪽 끝에는 전역 심볼을, 다른 쪽 끝에는 지역 심볼을 배치하는 방식이었다.

함수 프로토타입과 매개변수를 다룰 때, 함수의 프로토타입을 전역 끝에서 지역 끝으로 복사해야 한다는 것을 확인했다. 이렇게 해야 함수가 지역 매개변수 변수를 가질 수 있다. 또한 심볼 테이블의 한쪽 끝이 다른 쪽 끝과 충돌할 가능성도 염두에 두어야 한다.

따라서 언젠가는 심볼 테이블을 단일 연결 리스트(singly-linked list)로 변환해야 한다. 최소한 전역 심볼을 위한 하나의 리스트와 지역 심볼을 위한 또 다른 리스트가 필요하다. 열거형(enum)을 구현할 때가 되면, 열거형 값을 위한 세 번째 리스트도 추가할 것이다.

아직 이 부분에서 리팩토링을 진행하지는 않았다. 변경 사항이 상당히 많아 보이기 때문에, 실제로 필요할 때까지 기다리기로 했다. 하지만 앞으로 할 변경 사항 중 하나는 다음과 같다. 각 심볼 노드는 단일 연결 리스트를 형성하기 위한 `next` 포인터를 가질 뿐만 아니라, `param` 포인터도 가질 것이다. 이렇게 하면 함수가 매개변수를 위한 별도의 단일 연결 리스트를 가질 수 있다. 이 리스트는 전역 심볼을 검색할 때 건너뛸 수 있다. 그리고 함수의 프로토타입을 매개변수 목록으로 "복사"해야 할 때, 단순히 프로토타입 매개변수 리스트에 대한 포인터를 복사하면 된다. 어쨌든, 이 변경은 미래의 작업이다.


## 타입 재고찰

SubC에서 가져온 또 다른 개념은 타입 열거다(`defs.h`에 정의됨):

```c
// 기본 타입
enum {
  P_NONE, P_VOID, P_CHAR, P_INT, P_LONG,
  P_VOIDPTR, P_CHARPTR, P_INTPTR, P_LONGPTR
};
```

SubC는 한 단계의 간접 참조만 허용하므로 위와 같은 타입 목록을 사용한다. 여기서 아이디어를 얻어, 기본 타입 값에 간접 참조 수준을 인코딩하면 어떨까 생각했다. 그래서 코드를 변경해 `type` 정수의 하위 4비트는 간접 참조 수준을 나타내고, 상위 비트는 실제 타입을 인코딩하도록 했다:

```c
// 기본 타입. 하위 4비트는 간접 참조 수준을 나타내는 정수 값이다.
// 예: 0= 포인터 없음, 1= 포인터, 2= 포인터의 포인터 등.
enum {
  P_NONE, P_VOID=16, P_CHAR=32, P_INT=48, P_LONG=64
};
```

이 변경으로 기존 코드에서 모든 `P_XXXPTR` 참조를 완전히 리팩토링할 수 있었다. 어떤 변화가 있었는지 살펴보자.

먼저, `types.c`에서 스칼라 타입과 포인터 타입을 처리해야 한다. 이제 코드가 이전보다 더 간결해졌다:

```c
// 타입이 어떤 크기의 정수 타입이면 true를 반환하고, 그렇지 않으면 false를 반환한다.
int inttype(int type) {
  return ((type & 0xf) == 0);
}

// 타입이 포인터 타입이면 true를 반환한다.
int ptrtype(int type) {
  return ((type & 0xf) != 0);
}

// 기본 타입이 주어지면, 해당 타입의 포인터 타입을 반환한다.
int pointer_to(int type) {
  if ((type & 0xf) == 0xf)
    fatald("Unrecognised in pointer_to: type", type);
  return (type + 1);
}

// 기본 포인터 타입이 주어지면, 해당 포인터가 가리키는 타입을 반환한다.
int value_at(int type) {
  if ((type & 0xf) == 0x0)
    fatald("Unrecognised in value_at: type", type);
  return (type - 1);
}
```

`modify_type()`은 전혀 변경되지 않았다.

`expr.c`에서 리터럴 문자열을 처리할 때, 이전에는 `P_CHARPTR`를 사용했지만 이제는 다음과 같이 작성할 수 있다:

```c
   n = mkastleaf(A_STRLIT, pointer_to(P_CHAR), id);
```

`P_XXXPTR` 값이 사용된 또 다른 주요 영역은 `cg.c`의 하드웨어 의존 코드다. 먼저 `cgprimsize()`를 `ptrtype()`을 사용하도록 다시 작성했다:

```c
// P_XXX 타입 값이 주어지면, 기본 타입의 크기를 바이트 단위로 반환한다.
int cgprimsize(int type) {
  if (ptrtype(type)) return (8);
  switch (type) {
    case P_CHAR: return (1);
    case P_INT:  return (4);
    case P_LONG: return (8);
    default: fatald("Bad type in cgprimsize:", type);
  }
  return (0);                   // Keep -Wall happy
}
```

이 코드를 통해 `cg.c`의 다른 함수들은 이제 특정 타입을 참조하는 대신 필요에 따라 `cgprimsize()`, `ptrtype()`, `inttype()`, `pointer_to()`, `value_at()`을 호출할 수 있다. `cg.c`의 예를 살펴보자:

```c
// 포인터를 역참조해 가리키는 값을 동일한 레지스터로 가져온다.
int cgderef(int r, int type) {

  // 가리키는 타입을 얻는다.
  int newtype = value_at(type);

  // 이 타입의 크기를 얻는다.
  int size = cgprimsize(newtype);

  switch (size) {
  case 1:
    fprintf(Outfile, "\tmovzbq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  case 2:
    fprintf(Outfile, "\tmovslq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  case 4:
  case 8:
    fprintf(Outfile, "\tmovq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  default:
    fatald("Can't cgderef on type:", type);
  }
  return (r);
}
```

`cg.c`를 빠르게 읽어보고 `cgprimsize()` 호출을 찾아보자.


### 이중 포인터 사용 예제

이제 최대 16단계의 간접 참조가 가능해졌으므로, 이를 확인하기 위해 테스트 프로그램 `tests/input55.c`를 작성했다:

```c
int printf(char *fmt);

int main(int argc, char **argv) {
  int i;
  char *argument;
  printf("Hello world\n");

  for (i=0; i < argc; i++) {
    argument= *argv; argv= argv + 1;
    printf("Argument %d is %s\n", i, argument);
  }
  return(0);
}
```

현재 `argv++`와 `argv[i]`는 아직 동작하지 않는다. 하지만 위 예제에서 보듯이, 이러한 기능이 없더라도 문제를 해결할 수 있다.


## 심볼 테이블 구조 변경

심볼 테이블을 리스트로 리팩토링하지는 않았지만, 심볼 테이블 구조 자체를 약간 수정했다. 이제는 union을 사용할 수 있고, union에 변수 이름을 부여할 필요가 없다는 것을 깨달았다.

```c
// 심볼 테이블 구조
struct symtable {
  char *name;                   // 심볼의 이름
  int type;                     // 심볼의 기본 타입
  int stype;                    // 심볼의 구조적 타입
  int class;                    // 심볼의 저장 클래스
  union {
    int size;                   // 심볼의 요소 개수
    int endlabel;               // 함수의 경우, 종료 라벨
  };
  union {
    int nelems;                 // 함수의 경우, 매개변수 개수
    int posn;                   // 지역 변수의 경우, 스택 베이스 포인터에서의 음수 오프셋
  };
};
```

이전에는 `nelems`를 위해 `#define`을 사용했지만, 위 구조는 동일한 결과를 제공하면서도 `nelems`의 전역 정의가 네임스페이스를 오염시키는 것을 방지한다. 또한 `size`와 `endlabel`이 구조체 내에서 같은 위치를 차지할 수 있다는 것을 깨닫고, 해당 union을 추가했다. `addglob()` 함수의 매개변수에 몇 가지 미관상의 변경이 있지만, 그 외에는 큰 변화가 없다.


## AST 구조 변경

마찬가지로, AST 노드 구조를 수정하여 공용체(union)에 변수 이름을 사용하지 않도록 했다:

```c
// 추상 구문 트리 구조
struct ASTnode {
  int op;                       // 이 트리에서 수행할 "연산"
  int type;                     // 이 트리가 생성하는 표현식의 타입
  int rvalue;                   // 노드가 rvalue인지 여부
  struct ASTnode *left;         // 왼쪽, 중간, 오른쪽 자식 트리
  struct ASTnode *mid;
  struct ASTnode *right;
  union {                       // A_INTLIT인 경우 정수 값
    int intvalue;               // A_IDENT인 경우 심볼 슬롯 번호
    int id;                     // A_FUNCTION인 경우 심볼 슬롯 번호
    int size;                   // A_SCALE인 경우 스케일링할 크기
  };                            // A_FUNCCALL인 경우 심볼 슬롯 번호
};
```

이 변경으로 인해, 예를 들어 첫 번째 줄 대신 두 번째 줄과 같이 코드를 작성할 수 있다:

```c
    return (cgloadglob(n->left->v.id, n->op));    // 이전 코드
    return (cgloadglob(n->left->id,   n->op));    // 새로운 코드
```


## 결론 및 다음 단계

컴파일러 작성 여정의 이번 파트는 여기까지다. 몇 군데 작은 코드 변경을 더 했을 수도 있지만, 큰 변화는 없었다.

다음 단계에서는 심볼 테이블을 링크드 리스트로 변경할 예정이다. 이 작업은 열거형 값을 구현하는 부분에서 진행할 것이다.

컴파일러 작성 여정의 다음 파트에서는 이번 파트에서 다루고 싶었던 주제인 구조체(struct), 공용체(union), 열거형(enum)을 컴파일러에 구현하는 설계 측면으로 돌아갈 것이다. [다음 단계](../30_Design_Composites/Readme.md)



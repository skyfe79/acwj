# 17장: 더 나은 타입 검사와 포인터 오프셋

몇 장 전에 포인터를 소개하고 타입 호환성을 확인하는 코드를 구현했다. 당시 다음과 같은 코드를 작성할 때:

```c
  int   c;
  int  *e;

  e= &c + 1;
```

`&c`로 계산된 포인터에 1을 더하면, 메모리에서 `c` 다음에 오는 `int`로 이동하기 위해 `c`의 크기만큼 변환해야 한다는 것을 깨달았다. 즉, 정수를 스케일링해야 한다.

이 작업은 포인터뿐만 아니라 나중에 배열에서도 필요하다. 다음 코드를 생각해 보자:

```c
  int list[10];
  int x= list[3];
```

이를 위해서는 `list[]`의 베이스 주소를 찾고, `int`의 크기의 세 배를 더해 인덱스 위치 3에 있는 요소를 찾아야 한다.

당시 `types.c` 파일에 `type_compatible()` 함수를 작성해 두 타입이 호환되는지 확인하고, 작은 정수 타입을 더 큰 정수 타입과 같은 크기로 "확장"해야 하는지 여부를 결정했다. 하지만 이 확장 작업은 다른 곳에서 수행되었다. 실제로 이 작업은 컴파일러 내 세 군데에서 이루어졌다.


## `type_compatible()` 대체하기

`type_compatible()`이 지시한 대로라면, 더 큰 정수 타입과 일치하도록 트리를 A_WIDEN으로 확장했다. 이제는 타입의 크기에 따라 값을 스케일링하기 위해 A_SCALE을 사용해야 한다. 또한 중복된 확장 코드를 리팩토링하고자 한다.

이를 위해 `type_compatible()`을 제거하고 새로운 함수로 대체했다. 이 작업은 상당한 고민이 필요했으며, 앞으로도 수정하거나 확장해야 할 가능성이 있다. 이제 디자인을 살펴보자.

기존 `type_compatible()`의 기능은 다음과 같았다:
 + 두 타입 값을 인자로 받고, 선택적으로 방향을 지정할 수 있었다.
 + 타입이 호환되면 true를 반환했다.
 + 한쪽이 확장이 필요하면 왼쪽 또는 오른쪽에 A_WIDEN을 반환했다.
 + 실제로 트리에 A_WIDEN 노드를 추가하지는 않았다.
 + 타입이 호환되지 않으면 false를 반환했다.
 + 포인터 타입은 처리하지 않았다.

이제 타입 비교의 사용 사례를 살펴보자:

 + 두 표현식에 이진 연산을 수행할 때, 타입이 호환되는지 확인하고 확장 또는 스케일링이 필요한지 판단한다.
 + `print` 문을 실행할 때, 표현식이 정수인지 확인하고 확장이 필요한지 판단한다.
 + 할당문을 실행할 때, 표현식이 확장이 필요한지 확인하고 lvalue 타입과 일치하는지 판단한다.
 + `return` 문을 실행할 때, 표현식이 확장이 필요한지 확인하고 함수의 반환 타입과 일치하는지 판단한다.

이 사용 사례 중 단 하나만 두 표현식을 다룬다. 따라서, 새로운 함수를 작성해 하나의 AST 트리와 변환하려는 타입을 인자로 받도록 했다. 이진 연산의 경우, 이 함수를 두 번 호출하고 각 호출의 결과를 확인한다.


## `modify_type()` 소개

`types.c` 파일에 있는 `modify_type()` 함수는 `type_compatible()` 함수를 대체하는 코드이다. 이 함수의 API는 다음과 같다:

```c
// AST 트리와 원하는 타입이 주어졌을 때,
// 트리를 확장하거나 크기를 조정하여 주어진 타입과 호환되도록 수정한다.
// 변경이 없으면 원래 트리를 반환하고, 수정된 트리 또는 호환되지 않으면 NULL을 반환한다.
// 이 연산이 바이너리 연산의 일부라면 AST 연산자는 0이 아니다.
struct ASTnode *modify_type(struct ASTnode *tree, int rtype, int op);
```

여기서 의문이 생길 수 있다. 왜 트리와 다른 트리 간의 바이너리 연산이 필요한가? 그 이유는 포인터에 대해 더하거나 빼는 연산만 가능하기 때문이다. 다른 연산은 불가능하다. 예를 들어:

```c
  int x;
  int *ptr;

  x= *ptr;	   // 가능
  x= *(ptr + 2);   // ptr이 가리키는 위치에서 두 int만큼 이동
  x= *(ptr * 4);   // 의미 없음
  x= *(ptr / 13);  // 의미 없음
```

현재 코드는 다음과 같다. 다양한 특정 테스트가 포함되어 있으며, 모든 가능한 테스트를 합리화할 방법은 아직 보이지 않는다. 또한 나중에 확장이 필요할 것이다.

```c
struct ASTnode *modify_type(struct ASTnode *tree, int rtype, int op) {
  int ltype;
  int lsize, rsize;

  ltype = tree->type;

  // 스칼라 int 타입 비교
  if (inttype(ltype) && inttype(rtype)) {

    // 두 타입이 같으면 아무것도 할 필요 없음
    if (ltype == rtype) return (tree);

    // 각 타입의 크기 가져오기
    lsize = genprimsize(ltype);
    rsize = genprimsize(rtype);

    // 트리의 크기가 너무 크면
    if (lsize > rsize) return (NULL);

    // 오른쪽으로 확장
    if (rsize > lsize) return (mkastunary(A_WIDEN, rtype, tree, 0));
  }

  // 왼쪽이 포인터 타입인 경우
  if (ptrtype(ltype)) {
    // 오른쪽 타입이 같고 바이너리 연산이 아니면 가능
    if (op == 0 && ltype == rtype) return (tree);
  }

  // A_ADD 또는 A_SUBTRACT 연산에서만 크기 조정 가능
  if (op == A_ADD || op == A_SUBTRACT) {

    // 왼쪽이 int 타입, 오른쪽이 포인터 타입이고
    // 원래 타입의 크기가 1보다 크면 왼쪽 크기 조정
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1) 
        return (mkastunary(A_SCALE, rtype, tree, rsize));
    }
  }

  // 여기까지 왔다면 타입이 호환되지 않음
  return (NULL);
}
```

AST의 A_WIDEN과 A_SCALE 연산을 추가하는 코드는 이제 한 곳에서만 처리된다. A_WIDEN 연산은 자식의 타입을 부모의 타입으로 변환한다. A_SCALE 연산은 자식의 값을 크기만큼 곱하며, 이 크기는 새로운 `struct ASTnode`의 union 필드에 저장된다 (`defs.h` 파일에 정의됨):

```c
// 추상 구문 트리 구조
struct ASTnode {
  ...
  union {
    int size;                   // A_SCALE 연산에서 크기 조정을 위한 값
  } v;
};
```


## 새로운 `modify_type()` API 사용하기

이 새로운 API를 사용하면 `stmt.c`와 `expr.c`에 중복된 A_WIDEN 코드를 제거할 수 있다. 하지만 이 새로운 함수는 단 하나의 트리만 인자로 받는다. 실제로 하나의 트리만 다룰 때는 문제가 없다. 현재 `stmt.c`에는 `modify_type()`을 호출하는 세 군데가 있다. 이들은 모두 유사하므로, `assignment_statement()`에서 사용된 예제를 살펴보자:

```c
  // 할당문의 좌변을 위한 AST 노드 생성
  right = mkastleaf(A_LVIDENT, Gsym[id].type, id);

  ...
  // 다음 표현식 파싱
  left = binexpr(0);

  // 두 타입이 호환되는지 확인
  left = modify_type(left, right->type, 0);
  if (left == NULL) fatal("할당문에서 호환되지 않는 표현식");
```

이전 코드보다 훨씬 깔끔해졌다.


### 그리고 `binexpr()` 함수에서...

`expr.c` 파일의 `binexpr()` 함수에서는 이제 두 개의 AST 트리를 덧셈, 곱셈 등의 이항 연산으로 결합해야 한다. 여기서 각 트리를 다른 트리의 타입으로 `modify_type()`을 통해 수정하려고 시도한다. 이때 한 쪽이 확장될 수 있지만, 이는 다른 쪽이 실패하고 NULL을 반환할 가능성이 있음을 의미한다. 따라서 `modify_type()`의 결과가 하나만 NULL인지 확인하는 것으로는 부족하다. 타입이 호환되지 않는다고 판단하려면 두 결과가 모두 NULL인지 확인해야 한다. 다음은 `binexpr()` 함수의 새로운 비교 코드이다:

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  struct ASTnode *ltemp, *rtemp;
  int ASTop;
  int tokentype;

  ...
  // 왼쪽 트리를 가져온다.
  // 동시에 다음 토큰을 가져온다.
  left = prefix();
  tokentype = Token.token;

  ...
  // 현재 토큰의 우선순위를 사용해
  // 재귀적으로 binexpr()를 호출해 서브 트리를 만든다.
  right = binexpr(OpPrec[tokentype]);

  // 두 타입이 호환되는지 확인하기 위해
  // 각 트리를 다른 트리의 타입으로 수정하려고 시도한다.
  ASTop = arithop(tokentype);
  ltemp = modify_type(left, right->type, ASTop);
  rtemp = modify_type(right, left->type, ASTop);
  if (ltemp == NULL && rtemp == NULL)
    fatal("이항 표현식에서 호환되지 않는 타입");

  // 확장되거나 스케일링된 트리를 업데이트한다.
  if (ltemp != NULL) left = ltemp;
  if (rtemp != NULL) right = rtemp;
```

코드가 약간 복잡해 보이지만, 이전보다 더 나쁘지는 않다. 이제 A_SCALE과 A_WIDEN을 모두 처리할 수 있다.


## 스케일링 구현

`defs.h`에 AST 노드 연산으로 A_SCALE을 추가했다. 이제 이를 구현해야 한다.

앞서 언급했듯이, A_SCALE 연산은 자식 노드의 값에 새로운 `struct ASTnode` 유니온 필드에 저장된 크기를 곱한다. 모든 정수 타입에서 이 크기는 2의 배수다. 이 특징을 활용해 자식 노드의 값에 특정 비트 수만큼 왼쪽 시프트를 하여 곱셈을 수행할 수 있다.

나중에는 크기가 2의 거듭제곱이 아닌 구조체를 다룰 것이다. 따라서 스케일 계수가 적절할 때는 시프트 최적화를 사용하고, 더 일반적인 스케일 계수에 대해서는 곱셈을 구현해야 한다.

`gen.c`의 `genAST()` 함수에 이를 구현한 새로운 코드는 다음과 같다:

```c
    case A_SCALE:
      // 작은 최적화: 스케일 값이 알려진 2의 거듭제곱일 때 시프트 사용
      switch (n->v.size) {
        case 2: return(cgshlconst(leftreg, 1));
        case 4: return(cgshlconst(leftreg, 2));
        case 8: return(cgshlconst(leftreg, 3));
        default:
          // 크기를 레지스터에 로드하고
          // leftreg에 이 크기를 곱함
          rightreg= cgloadint(n->v.size, P_INT);
          return (cgmul(leftreg, rightreg));
```


## x86-64 코드에서 왼쪽 시프트 구현

이제 레지스터 값을 상수만큼 왼쪽으로 시프트하는 `cgshlconst()` 함수가 필요하다. 나중에 C의 '<<' 연산자를 추가할 때 더 일반적인 왼쪽 시프트 함수를 작성할 것이다. 지금은 정수 리터럴 값을 사용해 `salq` 명령어를 활용할 수 있다:

```c
// 레지스터 값을 상수만큼 왼쪽으로 시프트
int cgshlconst(int r, int val) {
  fprintf(Outfile, "\tsalq\t$%d, %s\n", val, reglist[r]);
  return(r);
}
```


## 작동하지 않는 테스트 프로그램

스케일링 기능을 테스트하기 위해 작성한 프로그램은 `tests/input16.c`이다:

```c
int   c;
int   d;
int  *e;
int   f;

int main() {
  c= 12; d=18; printint(c);
  e= &c + 1; f= *e; printint(f);
  return(0);
}
```

어셈블러가 다음 어셈블리 지시문을 생성할 때 `d`가 `c` 바로 뒤에 위치하기를 기대했다:

```
        .comm   c,1,1
        .comm   d,4,4
```

하지만 어셈블리 출력을 컴파일하고 확인해 보니, 두 변수가 인접하지 않았다:

```
$ cc -o out out.s lib/printint.c
$ nm -n out | grep 'B '
0000000000201018 B d
0000000000201020 B b
0000000000201028 B f
0000000000201030 B e
0000000000201038 B c
```

`d`가 실제로 `c`보다 앞에 위치했다! 변수가 인접하도록 보장하는 방법을 찾아야 했다. 그래서 *SubC*가 생성하는 코드를 살펴보고, 컴파일러를 수정하여 다음 코드를 생성하도록 변경했다:

```
        .data
        .globl  c
c:      .long   0	# 네 바이트 정수
        .globl  d
d:      .long   0
        .globl  e
e:      .quad   0	# 여덟 바이트 포인터
        .globl  f
f:      .long   0
```

이제 `input16.c` 테스트를 실행하면, `e= &c + 1; f= *e;`가 `c`의 다음 정수 주소를 가져와 그 값을 `f`에 저장한다. 선언한 대로:

```c
  int   c;
  int   d;
  ...
  c= 12; d=18; printint(c);
  e= &c + 1; f= *e; printint(f);
```

두 숫자를 출력한다:

```
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
./comp1 tests/input16.c
cc -o out out.s lib/printint.c
./out
12
18
```


## 결론과 다음 단계

타입 변환 코드를 작성하면서 만족스러운 결과를 얻었다. 내부적으로는 `modify_type()` 함수에 가능한 모든 타입 값을 전달하는 테스트 코드를 작성했다. 이때 바이너리 연산과 연산에 사용할 0도 함께 제공했다. 출력 결과를 직접 확인해보니 원하는 대로 동작하는 것 같다. 시간이 지나면 더 확실히 알 수 있을 것이다.

컴파일러 작성 여정의 다음 단계에서는 무엇을 할지 아직 결정하지 못했다! [다음 단계](../18_Lvalues_Revisited/Readme.md)



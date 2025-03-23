# 51부: 배열 심화 학습

컴파일러 개발 여정의 이전 부분에서, 배열을 완벽하게 구현하지 못했다는 사실을 깨달았다. 이번 단계에서는 그 문제를 해결하려고 한다.

먼저, 배열과 포인터에 대해 다시 생각해보았다. 배열은 포인터와 비슷하지만 몇 가지 중요한 차이점이 있다:

1. 배열 식별자를 그대로 rvalue로 사용할 수 없다.
2. 배열의 크기는 모든 요소의 크기를 포함한다. 포인터의 크기는 포인터가 가리키는 배열의 요소 크기를 포함하지 않는다.
3. 배열의 주소(예: `&ary`)는 유용한 의미를 가지지 않는다. 반면 포인터의 주소(예: `&ptr`)는 유용한 정보를 제공한다.

첫 번째 차이점을 예를 들어 설명하면 다음과 같다:

```c
int ary[5];
int *ptr;

int main() {
   ptr= ary;            // OK, ary의 기본 주소를 ptr에 할당
   ary= ptr;            // Bad, ary의 기본 주소를 변경할 수 없음
```

C 언어를 깊이 이해한 사람이라면 세 번째 차이점이 완전히 사실은 아니라는 것을 알고 있을 것이다. 하지만 `&ary`를 사용할 계획이 없기 때문에, 컴파일러가 이를 거부하도록 만들면 이 기능을 구현할 필요가 없다.

그렇다면 정확히 무엇을 변경해야 할까?

+ '[' 토큰 앞에 스칼라 또는 배열 식별자를 허용한다.
+ 배열 식별자를 그대로 허용하되 rvalue로 표시한다.
+ 배열을 잘못 사용할 때 추가적인 에러를 발생시킨다.

이 정도가 주요 변경 사항이다. 이 변경 사항을 컴파일러에 반영했다. 이제 배열 관련 문제가 모두 해결되길 바라지만, 아마도 또 다른 문제를 놓쳤을 가능성이 있다. 그렇다면 다시 이 문제를 다루게 될 것이다.


## `postfix()` 함수 변경 사항

이전 부분에서 `expr.c` 파일의 `postfix()` 함수에 임시적인 수정을 적용했다. 이제 제대로 고칠 차례다. 배열 식별자를 그대로 사용할 수 있게 하되, 이를 rvalue로 표시해야 한다. 다음은 변경 사항이다:

```c
static struct ASTnode *postfix(void) {
  ...
  int rvalue=0;
  ...
  // 식별자를 확인하고 존재하는지 검사한다. 배열인 경우 rvalue를 1로 설정한다.
  if ((varptr = findsymbol(Text)) == NULL)
    fatals("Unknown variable", Text);
  switch(varptr->stype) {
    case S_VARIABLE: break;
    case S_ARRAY: rvalue= 1; break;
    default: fatals("Identifier not a scalar or array variable", Text);
  }

  switch (Token.token) {
    // 후위 증가: 토큰을 건너뛴다. 후위 감소도 동일하다.
  case T_INC:
    if (rvalue == 1)
      fatals("Cannot ++ on rvalue", Text);
  ...
    // 단순 변수 참조. 배열이 lvalue로 처리되지 않도록 한다.
  default:
    if (varptr->stype == S_ARRAY) {
      n = mkastleaf(A_ADDR, varptr->type, varptr, 0);
      n->rvalue = rvalue;
    } else
      n = mkastleaf(A_IDENT, varptr->type, varptr, 0);
  }
  return (n);
}
```

이제 스칼라 변수나 배열 변수를 그대로 사용할 수 있지만, 배열은 lvalue가 될 수 없다. 또한 배열은 전위 또는 후위 증가 연산자를 사용할 수 없다. 배열의 경우 배열 베이스의 주소를 로드하거나, 스칼라 변수의 값을 로드한다.


## `array_access()` 함수 변경 사항

이제 `expr.c` 파일의 `array_access()` 함수를 수정해 포인터가 '[' ']' 인덱싱과 함께 사용될 수 있도록 해야 한다. 다음은 변경 사항이다:

```c
static struct ASTnode *array_access(void) {
  struct ASTnode *left, *right;
  struct symtable *aryptr;

  // 식별자가 배열 또는 포인터로 정의되었는지 확인
  if ((aryptr = findsymbol(Text)) == NULL)
    fatals("Undeclared variable", Text);
  if (aryptr->stype != S_ARRAY &&
        (aryptr->stype == S_VARIABLE && !ptrtype(aryptr->type)))
    fatals("Not an array or pointer", Text);
  
  // 배열의 베이스 주소를 가리키는 리프 노드 생성
  // 또는 포인터 변수의 값을 rvalue로 로드
  if (aryptr->stype == S_ARRAY)
    left = mkastleaf(A_ADDR, aryptr->type, aryptr, 0);
  else {
    left = mkastleaf(A_IDENT, aryptr->type, aryptr, 0);
    left->rvalue= 1;
  }
  ...
}
```

이제 심볼이 존재하는지 확인하고, 배열이거나 포인터 타입의 스칼라 변수인지 검사한다. 이 검사를 통과하면 배열의 베이스 주소를 로드하거나 포인터 변수의 값을 로드한다.


## 코드 변경 사항 테스트

모든 테스트를 하나씩 살펴보기보다는 주요 내용을 요약한다:

+ `tests/input124.c`는 배열에 `ary++` 연산을 수행할 수 없는지 확인한다.
+ `tests/input125.c`는 `ptr = ary`와 같은 할당이 가능하고, 포인터를 통해 배열에 접근할 수 있는지 검증한다.
+ `tests/input126.c`는 `&ary` 연산이 불가능한지 확인한다.
+ `tests/input127.c`는 `fred(ary)`와 같이 함수를 호출하고, 이를 포인터 매개변수로 받을 수 있는지 검사한다.


## 결론 및 다음 단계

배열이 제대로 동작하도록 하려면 코드를 완전히 다시 작성해야 할까봐 걱정했었다. 다행히 기존 코드가 거의 완벽했고, 필요한 모든 기능을 지원하기 위해 약간의 추가 조정만 필요했다.

컴파일러 개발의 다음 단계에서는 [다음 단계](../52_Pointers_pt2/Readme.md)로 넘어가며 마무리 작업을 진행할 예정이다.



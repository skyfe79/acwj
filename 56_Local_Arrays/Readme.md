# 56장. 지역 배열

정말 놀라운 일이다. 지역 배열을 구현하는 것이 전혀 어렵지 않았다. 우리 컴파일러에 필요한 모든 조각이 이미 있었고, 단지 그것들을 연결하기만 하면 되었다.


## 로컬 배열 파싱

파싱 측면부터 시작해 보자. 로컬 배열 선언을 허용하되, 값 할당 없이 요소의 개수만 지정할 수 있도록 구현하려고 한다.

선언 부분은 간단하다. `decl.c` 파일의 `array_declaration()` 함수에 다음 코드를 추가한다:

```c
  // 이 배열을 알려진 배열로 추가한다. 배열을 요소 타입에 대한 포인터로 취급한다.
  switch (class) {
    ...
    case C_LOCAL:
      sym = addlocl(varname, pointer_to(type), ctype, S_ARRAY, 0);
      break;
    ...
  }
```

이제 로컬 배열에 대한 할당을 방지해야 한다:

```c
  // 배열 초기화
  if (Token.token == T_ASSIGN) {
    if (class != C_GLOBAL && class != C_STATIC)
      fatals("Variable can not be initialised", varname);
```

추가로 몇 가지 오류 검사를 더했다:

```c
  // 배열의 크기와 요소 개수를 설정한다.
  // extern만이 요소가 없을 수 있다.
  if (class != C_EXTERN && nelems<=0)
    fatals("Array must have non-zero elements", sym->name);
```

이것으로 로컬 배열 선언에 대한 작업은 완료된다.


## 코드 생성

`cg.c` 파일에는 `newlocaloffset()` 함수가 있다. 이 함수는 로컬 변수의 오프셋을 스택 프레임의 상단을 기준으로 계산한다. 이전에는 컴파일러가 int와 포인터 타입만 로컬 변수로 허용했기 때문에 인자가 기본 타입이었다.

이제 각 심볼이 자신의 크기를 가지고 있고(`sizeof()`가 사용하는 크기), 이 함수의 코드를 심볼의 크기를 사용하도록 변경할 수 있다:

```c
// 새로운 로컬 변수의 위치를 생성한다.
static int newlocaloffset(int size) {
  // 오프셋을 최소 4바이트만큼 감소시키고
  // 스택에 할당한다
  localOffset += (size > 4) ? size : 4;
  return (-localOffset);
}
```

함수의 프리앰블을 생성하는 코드인 `cgfuncpreamble()`에서는 다음과 같이 변경한다:

```c
  // 레지스터에 있는 파라미터를 스택으로 복사한다. 최대 6개까지.
  // 나머지 파라미터는 이미 스택에 있다.
  for (parm = sym->member, cnt = 1; parm != NULL; parm = parm->next, cnt++) {
    if (cnt > 6) {
      parm->st_posn = paramOffset;
      paramOffset += 8;
    } else {
      parm->st_posn = newlocaloffset(parm->size);       // 여기
      cgstorlocal(paramReg--, parm);
    }
  }

  // 나머지 변수들은 파라미터라면 이미 스택에 있다.
  // 로컬 변수라면 스택 위치를 생성한다.
  for (locvar = Loclhead; locvar != NULL; locvar = locvar->next) {
    locvar->st_posn = newlocaloffset(locvar->size);     // 여기
  }
```

이게 전부다! 이 변경으로 구조체와 공용체도 로컬 변수로 허용할 수 있게 될 가능성이 있다. 아직 이 부분은 고려하지 않았지만, 나중에 탐구해볼 만한 주제다.


## 변경 사항 테스트

`test/input140.c` 파일은 다음과 같이 선언한다:

```c
int main() {
  int  i;
  int  ary[5];
  char z;
  ...
```

배열은 FOR 루프를 통해 채워지며, `i`가 인덱스로 사용된다. `z` 로컬 변수도 초기화된다. 이 테스트는 변수들이 서로의 영역을 침범하지 않는지 확인한다. 또한 배열의 모든 요소를 할당하고 그 값을 다시 가져올 수 있는지 검증한다.

`test/input141.c`와 `test/input142.c` 파일은 컴파일러가 매개변수로 배열을 사용하는 경우와 요소가 없는 배열 선언을 감지하고 거부하는지 확인한다.


## 결론 및 다음 단계

컴파일러 작성 여정의 다음 파트에서는 미처 다루지 못한 부분을 정리한다. [다음 단계](../57_Mop_up_pt3/Readme.md)



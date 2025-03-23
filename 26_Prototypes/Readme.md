# 26부: 함수 프로토타입

이번 파트에서는 컴파일러 작성 과정에서 함수 프로토타입을 추가할 수 있는 기능을 구현했다. 이 과정에서 이전 파트에서 작성한 코드 일부를 다시 작성해야 했는데, 이 부분에 대해 사과드린다. 미리 충분히 예상하지 못했던 부분이다!

함수 프로토타입을 통해 우리가 원하는 기능은 다음과 같다:

+ 본문이 없는 함수 프로토타입을 선언할 수 있어야 한다.
+ 나중에 완전한 함수를 선언할 수 있어야 한다.
+ 프로토타입을 전역 심볼 테이블 섹션에 유지하고, 매개변수는 로컬 심볼 테이블 섹션에 로컬 변수로 유지해야 한다.
+ 이전 함수 프로토타입과 매개변수의 개수 및 타입을 검사하는 오류 체크 기능이 필요하다.

그리고 다음 기능들은 **아직** 구현하지 않을 것이다:

+ `function(void)`: 이는 `function()` 선언과 동일하게 처리할 것이다.
+ 타입만으로 함수를 선언하는 방식, 예를 들어 `function(int ,char, long);`과 같은 형태는 파싱을 더 복잡하게 만들 수 있다. 이 부분은 나중에 구현할 예정이다.


## 어떤 기능을 재작성해야 하는가

최근 작업 과정에서 매개변수와 전체 함수 본문을 가진 함수 선언을 추가했다. 각 매개변수를 파싱할 때, 전역 심볼 테이블(프로토타입을 구성하기 위해)과 지역 심볼 테이블(함수의 매개변수가 되기 위해)에 즉시 추가했다.

이제 함수 프로토타입을 구현하려면, 매개변수 목록이 항상 실제 함수의 매개변수가 되는 것은 아니다. 다음 함수 프로토타입을 생각해 보자:

```c
  int fred(char a, int foo, long bar);
```

여기서 `fred`를 함수로 정의하고, `a`, `foo`, `bar`를 전역 심볼 테이블의 세 매개변수로만 정의할 수 있다. 전체 함수 선언이 나올 때까지 `a`, `foo`, `bar`를 지역 심볼 테이블에 추가할 수 없다.

따라서 전역 심볼 테이블과 지역 심볼 테이블에서 C_PARAM 엔트리의 정의를 분리해야 한다.


## 새로운 파싱 메커니즘 설계

아래는 프로토타입도 처리하는 새로운 함수 파싱 메커니즘에 대한 빠른 설계이다.

```
식별자와 '('를 가져온다.
심볼 테이블에서 식별자를 검색한다.
존재하면 프로토타입이 이미 있음: 함수의 ID 위치와 매개변수 개수를 가져온다.

매개변수를 파싱하는 동안:
  - 이전 프로토타입이 있으면, 기존 매개변수 타입과 현재 매개변수 타입을 비교한다. 
    완전한 함수인 경우 심볼 이름을 업데이트한다.
  - 이전 프로토타입이 없으면, 심볼 테이블에 매개변수를 추가한다.

기존 프로토타입과 매개변수 개수가 일치하는지 확인한다.
')'를 파싱한다. 다음이 ';'이면 완료.

'{'가 다음이면, 전역 심볼 테이블에서 지역 심볼 테이블로 매개변수 리스트를 복사한다. 
로컬 심볼 테이블에 역순으로 배치되도록 루프를 돌며 복사한다.
```

지난 몇 시간 동안 이 작업을 마쳤고, 코드 변경 사항은 다음과 같다.


## `sym.c` 파일 변경 사항

`sym.c` 파일 내 몇 가지 함수의 매개변수 목록을 변경했다:

```c
int addglob(char *name, int type, int stype, int class, int endlabel, int size);
int addlocl(char *name, int type, int stype, int class, int size);
```

이전에는 `addlocl()` 함수가 `addglob()`을 호출해 C_PARAM 심볼을 두 심볼 테이블에 추가했다. 이제 이 기능을 분리하면서, 두 함수에 실제 심볼 클래스를 전달하는 것이 더 적합하다.

이 함수들은 `main.c`와 `decl.c`에서 호출된다. `decl.c`의 변경 사항은 나중에 다룰 예정이다. `main.c`의 변경 사항은 간단하다.

실제 함수의 선언을 만나면, 전역 심볼 테이블에서 로컬 심볼 테이블로 매개변수 목록을 복사해야 한다. 이 작업은 심볼 테이블에 특화된 기능이므로, `sym.c`에 다음 함수를 추가했다:

```c
// 함수의 슬롯 번호를 받아, 전역 심볼 테이블의 프로토타입에서
// 로컬 심볼 테이블로 매개변수를 복사한다
void copyfuncparams(int slot) {
  int i, id = slot + 1;

  for (i = 0; i < Symtable[slot].nelems; i++, id++) {
    addlocl(Symtable[id].name, Symtable[id].type, Symtable[id].stype,
            Symtable[id].class, Symtable[id].size);
  }
}
```


## `decl.c` 파일의 변경 사항

컴파일러의 거의 모든 변경 사항은 `decl.c` 파일에 한정되어 있다. 작은 변경 사항부터 시작해 점차 큰 변경 사항으로 나아가며 살펴본다.


### `var_declaration()` 함수

`sym.c` 함수들과 동일한 방식으로 `var_declaration()` 함수의 매개변수 목록을 변경했다:

```c
void var_declaration(int type, int class) {
  ...
  addglob(Text, pointer_to(type), S_ARRAY, class, 0, Token.intvalue);
  ...
  if (addlocl(Text, type, S_VARIABLE, class, 1) == -1)
  ...
  addglob(Text, type, S_VARIABLE, class, 0, 1);
}
```

이제 클래스를 전달할 수 있는 기능을 다른 `decl.c` 함수들에서도 활용할 것이다.


### `param_declaration()`

여기서는 큰 변화가 있다. 이미 전역 심볼 테이블에 프로토타입으로 파라미터 리스트가 존재할 수 있다. 만약 그렇다면, 새로운 리스트의 파라미터 개수와 타입을 프로토타입과 비교해야 한다.

```c
// 함수 이름 뒤의 괄호 안에 있는 파라미터를 파싱한다.
// 심볼 테이블에 심볼로 추가하고 파라미터 개수를 반환한다.
// id가 -1이 아니면, 기존 함수 프로토타입이 존재하며,
// 함수는 이 심볼 슬롯 번호를 가진다.
static int param_declaration(int id) {
  int type, param_id;
  int orig_paramcnt;
  int paramcnt = 0;

  // id에 1을 더해, 0(프로토타입 없음)이거나
  // 심볼 테이블에서 기존 파라미터의 첫 번째 위치를 가리키도록 한다.
  param_id = id + 1;

  // 기존 프로토타입의 파라미터 개수를 가져온다.
  if (param_id)
    orig_paramcnt = Symtable[id].nelems;

  // 마지막 닫는 괄호가 나올 때까지 반복한다.
  while (Token.token != T_RPAREN) {
    // 타입과 식별자를 가져와 심볼 테이블에 추가한다.
    type = parse_type();
    ident();

    // 기존 프로토타입이 존재한다면,
    // 이 타입이 프로토타입과 일치하는지 확인한다.
    if (param_id) {
      if (type != Symtable[id].type)
        fatald("Type doesn't match prototype for parameter", paramcnt + 1);
      param_id++;
    } else {
      // 새로운 프로토타입에 파라미터를 추가한다.
      var_declaration(type, C_PARAM);
    }
    paramcnt++;

    // 이 시점에서 ',' 또는 ')'가 있어야 한다.
    switch (Token.token) {
    case T_COMMA:
      scan(&Token);
      break;
    case T_RPAREN:
      break;
    default:
      fatald("Unexpected token in parameter list", Token.token);
    }
  }

  // 이 리스트의 파라미터 개수가 기존 프로토타입과 일치하는지 확인한다.
  if ((id != -1) && (paramcnt != orig_paramcnt))
    fatals("Parameter count mismatch for function", Symtable[id].name);

  // 파라미터 개수를 반환한다.
  return (paramcnt);
}
```

첫 번째 파라미터의 전역 심볼 테이블 슬롯 위치는 함수 이름 심볼의 슬롯 바로 다음이다. 기존 프로토타입의 슬롯 위치를 전달받거나, 프로토타입이 없으면 -1을 받는다.

여기에 1을 더하면 첫 번째 파라미터의 슬롯 번호를 얻거나, 프로토타입이 없음을 나타내는 0이 된다는 것은 우연히도 편리한 일이다.

여전히 각 새로운 파라미터를 파싱하며 반복하지만, 이제는 기존 프로토타입과 비교하거나 전역 심볼 테이블에 파라미터를 추가하는 새로운 코드가 있다.

반복문을 빠져나오면, 이 리스트의 파라미터 개수를 기존 프로토타입의 개수와 비교할 수 있다.

현재 코드는 다소 지저분해 보인다. 시간이 지나면 리팩토링할 방법을 찾을 수 있을 것 같다.


### `function_declaration()`

이전에는 이 함수가 비교적 단순했다. 타입과 이름을 가져오고, 전역 심볼을 추가하며, 파라미터를 읽고, 함수의 본문을 가져와 함수 코드에 대한 AST 트리를 생성하는 것이었다.

이제는 이 함수가 프로토타입일 수도 있고, 완전한 함수일 수도 있다는 사실을 처리해야 한다. 그리고 ';' (프로토타입) 또는 '{' (완전한 함수)를 파싱할 때까지 이를 알 수 없다. 따라서 코드를 단계별로 살펴보자.

```c
// 함수 선언을 파싱한다.
// 식별자는 이미 스캔되었고, 타입을 가지고 있다.
struct ASTnode *function_declaration(int type) {
  struct ASTnode *tree, *finalstmt;
  int id;
  int nameslot, endlabel, paramcnt;

  // Text에는 식별자의 이름이 있다. 이 이름이 존재하고 함수라면 id를 가져온다. 그렇지 않으면 id를 -1로 설정한다.
  if ((id = findsymbol(Text)) != -1)
    if (Symtable[id].stype != S_FUNCTION)
      id = -1;

  // 새로운 함수 선언이라면, end label을 위한 label-id를 가져오고, 함수를 심볼 테이블에 추가한다.
  if (id == -1) {
    endlabel = genlabel();
    nameslot = addglob(Text, type, S_FUNCTION, C_GLOBAL, endlabel, 0);
  }
  // '(', 파라미터, ')'를 스캔한다. 기존 함수 프로토타입 심볼 슬롯 번호를 전달한다.
  lparen();
  paramcnt = param_declaration(id);
  rparen();
```

이 코드는 이전 버전과 거의 동일하지만, 이제 `id`는 이전 프로토타입이 없을 때 -1로 설정되거나, 이전 프로토타입이 있을 때 양수로 설정된다. 함수 이름이 아직 전역 심볼 테이블에 없을 때만 추가한다.

```c
  // 새로운 함수 선언이라면, 함수 심볼 항목을 파라미터 개수로 업데이트한다.
  if (id == -1)
    Symtable[nameslot].nelems = paramcnt;

  // 선언이 세미콜론으로 끝나면, 이는 프로토타입이다.
  if (Token.token == T_SEMI) {
    scan(&Token);
    return (NULL);
  }
```

파라미터 개수를 얻었다. 이전 프로토타입이 없다면 이 프로토타입을 이 개수로 업데이트한다. 이제 파라미터 리스트 끝 이후의 토큰을 확인할 수 있다. 세미콜론이라면 이는 단순히 프로토타입이다. 이제 반환할 AST 트리가 없으므로 토큰을 건너뛰고 NULL을 반환한다. `global_declarations()`에서 이 NULL 값을 처리하기 위해 코드를 약간 수정해야 했다: 큰 변화는 없다.

계속 진행한다면, 이제 본문이 있는 완전한 함수 선언을 다루고 있다.

```c
  // 이는 단순히 프로토타입이 아니다.
  // 전역 파라미터를 로컬 파라미터로 복사한다.
  if (id == -1)
    id = nameslot;
  copyfuncparams(id);
```

이제 프로토타입에서 로컬 심볼 테이블로 파라미터를 복사해야 한다. `id = nameslot` 코드는 우리가 직접 전역 심볼을 추가하고 이전 프로토타입이 없을 때를 위한 것이다.

`function_declaration()`의 나머지 코드는 이전과 동일하며 생략한다. 이 코드는 void가 아닌 함수가 값을 반환하는지 확인하고, A_FUNCTION 루트 노드로 AST 트리를 생성한다.


## 새로운 기능 테스트

`tests/runtests` 스크립트의 단점 중 하나는 컴파일러가 반드시 어셈블리 출력 파일 `out.s`를 생성한다고 가정한다는 점이다. 이 파일은 어셈블되고 실행될 수 있다. 이로 인해 컴파일러가 구문 오류와 의미론적 오류를 감지하는지 테스트하기 어렵다.

`decl.c` 파일을 간단히 *grep* 해보면 새로운 오류가 감지되는 것을 확인할 수 있다:

```c
fatald("Type doesn't match prototype for parameter", paramcnt + 1);
fatals("Parameter count mismatch for function", Symtable[id].name);
```

따라서 `tests/runtests` 스크립트를 다시 작성해 컴파일러가 잘못된 입력에서 이러한 오류를 감지하는지 확인해야 한다.

현재 두 개의 새로운 테스트 프로그램 `input29.c`와 `input30.c`가 작동하고 있다. 첫 번째 프로그램인 `input29.c`는 `input28.c`와 동일하지만, 모든 함수의 프로토타입을 프로그램 상단에 배치했다:

```c
int param8(int a, int b, int c, int d, int e, int f, int g, int h);
int fred(int a, int b, int c);
int main();
```

이 프로그램과 이전의 모든 테스트 프로그램은 여전히 정상적으로 작동한다. 하지만 `input30.c`는 아마도 우리 컴파일러가 받은 첫 번째 비단순 프로그램일 것이다. 이 프로그램은 자신의 소스 파일을 열어 표준 출력으로 출력한다:

```c
int open(char *pathname, int flags);
int read(int fd, char *buf, int count);
int write(int fd, void *buf, int count);
int close(int fd);

char *buf;

int main() {
  int zin;
  int cnt;

  buf= "                                                             ";
  zin = open("input30.c", 0);
  if (zin == -1) {
    return (1);
  }
  while ((cnt = read(zin, buf, 60)) > 0) {
    write(1, buf, cnt);
  }
  close(zin);
  return (0);
}
```

아직 전처리기를 호출할 수 없으므로 `open()`, `read()`, `write()`, `close()` 함수의 프로토타입을 수동으로 추가했다. 또한 `open()` 호출에서 `O_RDONLY` 대신 0을 사용해야 한다.

현재 컴파일러는 `char buf[60];`과 같은 선언을 허용하지만 `buf` 자체를 char 포인터로 사용할 수 없다. 그래서 60자 길이의 리터럴 문자열을 char 포인터에 할당하고 이를 버퍼로 사용했다.

또한 IF와 WHILE 본문을 '{' ... '}'로 감싸 복합문으로 만들어야 한다. 아직 'dangling else' 문제를 해결하지 못했다. 마지막으로, `main()` 함수의 인자로 `char *argv[]`를 선언할 수 없으므로 입력 파일 이름을 하드코딩했다.

그럼에도 불구하고, 이제 우리 컴파일러가 컴파일할 수 있는 매우 기본적인 *cat(1)* 프로그램을 갖게 되었다. 이는 분명한 진전이다.


## 결론과 다음 단계

컴파일러 작성 여정의 다음 단계에서는 앞서 언급한 내용을 바탕으로 컴파일러 기능 테스트를 개선할 예정이다. [다음 단계](../27_Testing_Errors/Readme.md)



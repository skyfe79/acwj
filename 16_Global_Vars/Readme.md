# 16장: 전역 변수 올바르게 선언하기

포인터에 오프셋을 추가하는 문제를 살펴보겠다고 약속했지만, 그 전에 먼저 깊이 생각할 필요가 있다. 그래서 함수 선언 외부에 전역 변수 선언을 옮기기로 결정했다. 사실, 함수 내부의 변수 선언 파싱도 그대로 두었다. 나중에 이 부분을 지역 변수 선언으로 변경할 예정이기 때문이다.

또한 문법을 확장해 동일한 타입의 여러 변수를 한 번에 선언할 수 있도록 하고 싶다. 예를 들어, 다음과 같은 방식이다:

```c
  int x, y, z;
```


## 새로운 BNF 문법

전역 선언(함수와 변수 모두)을 위한 새로운 BNF 문법은 다음과 같다:

```
 global_declarations : global_declarations 
      | global_declaration global_declarations
      ;

 global_declaration: function_declaration | var_declaration ;

 function_declaration: type identifier '(' ')' compound_statement   ;

 var_declaration: type identifier_list ';'  ;

 type: type_keyword opt_pointer  ;
 
 type_keyword: 'void' | 'char' | 'int' | 'long'  ;
 
 opt_pointer: <empty> | '*' opt_pointer  ;
 
 identifier_list: identifier | identifier ',' identifier_list ;
```

`function_declaration`과 `global_declaration`은 모두 `type`으로 시작한다. 이제 `type`은 `type_keyword`와 `opt_pointer`로 구성되며, `opt_pointer`는 0개 이상의 '*' 토큰을 포함한다. 이후 `function_declaration`과 `global_declaration`은 하나의 `identifier`가 뒤따라야 한다.

그러나 `type` 이후 `var_declaration`은 `identifier_list`가 따라오며, 이는 ',' 토큰으로 구분된 하나 이상의 `identifier`로 구성된다. 또한 `var_declaration`은 ';' 토큰으로 끝나야 하지만, `function_declaration`은 `compound_statement`로 끝나며 ';' 토큰이 필요하지 않다.


## 새로운 토큰

이제 `scan.c` 파일에서 ',' 문자를 위한 T_COMMA 토큰을 추가했다.


## `decl.c`에 대한 변경 사항

위의 BNF 문법을 재귀 하강 함수 집합으로 변환한다. 하지만 반복문을 사용할 수 있기 때문에, 재귀의 일부를 내부 반복문으로 바꿀 수 있다.


### `global_declarations()` 함수

하나 이상의 전역 선언이 있을 때, 각 선언을 반복해서 파싱할 수 있다. 토큰이 더 이상 없으면 반복문을 종료한다.

```c
// 하나 이상의 전역 선언을 파싱한다. 
// 변수 또는 함수 선언이 가능하다.
void global_declarations(void) {
  struct ASTnode *tree;
  int type;

  while (1) {

    // 타입과 식별자를 읽은 후, 
    // 함수 선언인 경우 '('를 확인하거나
    // 변수 선언인 경우 ',' 또는 ';'를 확인한다.
    // ident() 호출로 텍스트가 채워진다.
    type = parse_type();
    ident();
    if (Token.token == T_LPAREN) {

      // 함수 선언을 파싱하고 
      // 해당 함수에 대한 어셈블리 코드를 생성한다.
      tree = function_declaration(type);
      genAST(tree, NOREG, 0);
    } else {

      // 전역 변수 선언을 파싱한다.
      var_declaration(type);
    }

    // EOF에 도달하면 종료한다.
    if (Token.token == T_EOF)
      break;
  }
}
```

현재는 전역 변수와 함수만 존재한다는 것을 알고 있으므로, 여기서 타입과 첫 번째 식별자를 스캔할 수 있다. 그런 다음, 다음 토큰을 확인한다. 만약 '('라면 `function_declaration()`을 호출한다. 그렇지 않으면 `var_declaration()`이라고 가정한다. 두 함수 모두 `type`을 전달한다.

이제 `function_declaration()`에서 AST `tree`를 받아오므로, 즉시 AST 트리에서 코드를 생성할 수 있다. 이 코드는 원래 `main()`에 있었지만, 이제 여기로 이동했다. `main()`은 이제 `global_declarations()`만 호출하면 된다:

```c
  scan(&Token);                 // 입력에서 첫 번째 토큰을 가져온다.
  genpreamble();                // 프리앰블을 출력한다.
  global_declarations();        // 전역 선언을 파싱한다.
  genpostamble();               // 포스트앰블을 출력한다.
```


### `var_declaration()`

함수 파싱은 이전과 거의 동일하다. 단, 타입과 식별자를 스캔하는 코드가 다른 곳에서 처리되며, `type`을 인자로 받는다.

변수 파싱 역시 타입과 식별자 스캔 코드가 사라졌다. 식별자를 전역 심볼에 추가하고 이를 위한 어셈블리 코드를 생성할 수 있다. 하지만 이제 루프를 추가해야 한다. 만약 다음 토큰이 ','라면, 동일한 타입의 다음 식별자를 얻기 위해 루프를 돌아야 한다. 그리고 다음 토큰이 ';'라면, 변수 선언이 끝난 것이다.

```c
// 변수 목록의 선언을 파싱한다.
// 식별자는 이미 스캔되었으며, 타입을 가지고 있다
void var_declaration(int type) {
  int id;

  while (1) {
    // Text에 식별자 이름이 있다.
    // 이를 알려진 식별자로 추가하고
    // 어셈블리에서 공간을 생성한다
    id = addglob(Text, type, S_VARIABLE, 0);
    genglobsym(id);

    // 다음 토큰이 세미콜론이면,
    // 이를 건너뛰고 반환한다.
    if (Token.token == T_SEMI) {
      scan(&Token);
      return;
    }
    // 다음 토큰이 쉼표면, 이를 건너뛰고
    // 식별자를 얻은 후 루프를 반복한다
    if (Token.token == T_COMMA) {
      scan(&Token);
      ident();
      continue;
    }
    fatal("식별자 이후 , 또는 ;가 없습니다");
  }
}
```


## 완전히 지역 변수는 아님

`var_declaration()` 함수는 이제 여러 변수 선언 목록을 파싱할 수 있지만, 타입과 첫 번째 식별자를 미리 스캔해야 한다.

그래서 `stmt.c` 파일의 `single_statement()` 함수 내에서 `var_declaration()` 호출을 그대로 두었다. 나중에 이 부분을 수정해 지역 변수를 선언할 것이다. 하지만 지금은 이 예제 프로그램의 모든 변수가 전역 변수다:

```c
int   d, f;
int  *e;

int main() {
  int a, b, c;
  b= 3; c= 5; a= b + c * 10;
  printint(a);

  d= 12; printint(d);
  e= &d; f= *e; printint(f);
  return(0);
}
```


## 변경 사항 테스트

위 코드는 `tests/input16.c` 파일이다. 이전과 마찬가지로 테스트를 진행할 수 있다:

```
$ make test16
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c scan.c
      stmt.c sym.c tree.c types.c
./comp1 tests/input16.c
cc -o out out.s lib/printint.c
./out
53
12
12
```


## 결론과 앞으로의 계획

컴파일러 개발 여정의 다음 단계에서는 포인터에 오프셋을 추가하는 문제를 다룰 예정이다. [다음 단계](../17_Scaling_Offsets/Readme.md)



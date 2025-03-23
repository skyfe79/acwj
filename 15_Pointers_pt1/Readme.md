# 15부: 포인터, 1부

컴파일러 개발 여정의 이번 파트에서는 언어에 포인터를 추가하는 작업을 시작한다. 특히 다음 기능을 구현할 계획이다:

+ 포인터 변수 선언
+ 포인터에 주소 할당
+ 포인터가 가리키는 값을 역참조

이 작업은 아직 진행 중이므로, 지금은 단순한 버전으로 구현할 것이다. 하지만 나중에 더 일반적으로 확장하거나 변경해야 할 부분이 생길 것이다.


## 새로운 키워드와 토큰

이번에는 새로운 키워드가 없고, 두 가지 새로운 토큰만 추가되었다:

 + '&', T_AMPER, 그리고
 + '&&', T_LOGAND

T_LOGAND는 아직 필요하지 않지만, `scan()` 함수에 이 코드를 미리 추가해두는 것이 좋다:

```c
    case '&':
      if ((c = next()) == '&') {
        t->token = T_LOGAND;
      } else {
        putback(c);
        t->token = T_AMPER;
      }
      break;
```


## 새로운 타입 코드 추가

언어에 몇 가지 새로운 기본 타입을 추가했다(`defs.h` 파일에 정의):

```c
// 기본 타입
enum {
  P_NONE, P_VOID, P_CHAR, P_INT, P_LONG,
  P_VOIDPTR, P_CHARPTR, P_INTPTR, P_LONGPTR
};
```

새로운 단항 접두사 연산자도 추가된다:

 + '&'는 식별자의 주소를 가져오는 연산자이고,
 + '*'는 포인터를 역참조하여 가리키는 값을 가져오는 연산자이다.

각 연산자가 생성하는 표현식의 타입은 연산자가 작동하는 타입과 다르다. 이 타입 변화를 처리하기 위해 `types.c` 파일에 두 함수를 추가한다:

```c
// 주어진 기본 타입에 대한 포인터 타입을 반환
int pointer_to(int type) {
  int newtype;
  switch (type) {
    case P_VOID: newtype = P_VOIDPTR; break;
    case P_CHAR: newtype = P_CHARPTR; break;
    case P_INT:  newtype = P_INTPTR;  break;
    case P_LONG: newtype = P_LONGPTR; break;
    default:
      fatald("Unrecognised in pointer_to: type", type);
  }
  return (newtype);
}

// 주어진 기본 포인터 타입이 가리키는 타입을 반환
int value_at(int type) {
  int newtype;
  switch (type) {
    case P_VOIDPTR: newtype = P_VOID; break;
    case P_CHARPTR: newtype = P_CHAR; break;
    case P_INTPTR:  newtype = P_INT;  break;
    case P_LONGPTR: newtype = P_LONG; break;
    default:
      fatald("Unrecognised in value_at: type", type);
  }
  return (newtype);
}
```

이제 이 함수들을 어디에서 사용할 것인지가 궁금할 것이다.


## 포인터 변수 선언

스칼라 변수와 포인터 변수를 선언할 수 있도록 한다. 예를 들어 다음과 같다:

```c
  char  a; char *b;
  int   d; int  *e;
```

이미 `decl.c` 파일에 `parse_type()` 함수가 있다. 이 함수는 타입 키워드를 타입으로 변환한다. 다음 토큰을 스캔하고, 토큰이 '*'라면 타입을 변경하도록 확장한다:

```c
// 현재 토큰을 파싱하고 기본 타입 enum 값을 반환한다. 
// 또한 다음 토큰을 스캔한다
int parse_type(void) {
  int type;
  switch (Token.token) {
    case T_VOID: type = P_VOID; break;
    case T_CHAR: type = P_CHAR; break;
    case T_INT:  type = P_INT;  break;
    case T_LONG: type = P_LONG; break;
    default:
      fatald("Illegal type, token", Token.token);
  }

  // 하나 이상의 '*' 토큰을 스캔하고 올바른 포인터 타입을 결정한다
  while (1) {
    scan(&Token);
    if (Token.token != T_STAR) break;
    type = pointer_to(type);
  }

  // 다음 토큰을 이미 스캔한 상태로 종료한다
  return (type);
}
```

이제 프로그래머가 다음과 같은 코드를 작성할 수 있다:

```c
   char *****fred;
```

현재 `pointer_to()` 함수가 P_CHARPTR를 P_CHARPTRPTR로 변환할 수 없기 때문에 이 코드는 실패한다. 하지만 `parse_type()` 함수는 이를 처리할 준비가 되어 있다!

`var_declaration()` 함수의 코드는 이제 포인터 변수 선언을 문제없이 파싱한다:

```c
// 변수 선언을 파싱한다
void var_declaration(void) {
  int id, type;

  // 변수의 타입을 가져온다. 
  // 이 과정에서 식별자도 스캔한다
  type = parse_type();
  ident();
  ...
}
```


### 접두사 연산자 '*'와 '&'


선언 부분을 마쳤으니, 이제 표현식 앞에 오는 '*'와 '&' 연산자를 파싱하는 방법을 살펴보자. BNF 문법은 다음과 같다:

```
 prefix_expression: primary
     | '*' prefix_expression
     | '&' prefix_expression
     ;
```

기술적으로 이 문법은 다음과 같은 표현을 허용한다:

```
   x= ***y;
   a= &&&b;
```

이 두 연산자의 불가능한 사용을 방지하기 위해, 의미 검사를 추가한다. 다음은 그 코드다:

```c
// 접두사 표현식을 파싱하고 이를 나타내는 서브 트리를 반환한다.
struct ASTnode *prefix(void) {
  struct ASTnode *tree;
  switch (Token.token) {
    case T_AMPER:
      // 다음 토큰을 가져오고 접두사 표현식으로 재귀적으로 파싱한다.
      scan(&Token);
      tree = prefix();

      // 식별자인지 확인한다.
      if (tree->op != A_IDENT)
        fatal("& 연산자 뒤에는 식별자가 와야 한다");

      // 연산자를 A_ADDR로 변경하고 타입을 원래 타입의 포인터로 변경한다.
      tree->op = A_ADDR; tree->type = pointer_to(tree->type);
      break;
    case T_STAR:
      // 다음 토큰을 가져오고 접두사 표현식으로 재귀적으로 파싱한다.
      scan(&Token); tree = prefix();

      // 현재로서는, 또 다른 역참조 또는 식별자인지 확인한다.
      if (tree->op != A_IDENT && tree->op != A_DEREF)
        fatal("* 연산자 뒤에는 식별자 또는 *가 와야 한다");

      // 트리에 A_DEREF 연산을 추가한다.
      tree = mkastunary(A_DEREF, value_at(tree->type), tree, 0);
      break;
    default:
      tree = primary();
  }
  return (tree);
}
```

여전히 재귀 하강 파싱을 사용하지만, 입력 오류를 방지하기 위해 오류 검사를 추가했다. 현재 `value_at()`의 제한으로 인해 연속된 '*' 연산자를 하나 이상 사용할 수 없지만, 나중에 `value_at()`을 변경할 때 `prefix()`를 다시 수정할 필요는 없다.

`prefix()`는 '*' 또는 '&' 연산자를 발견하지 못했을 때 `primary()`를 호출한다는 점에 주목하자. 이는 `binexpr()`에서 기존 코드를 변경할 수 있게 해준다:

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int lefttype, righttype;
  int tokentype;

  // 왼쪽 트리를 가져온다.
  // 동시에 다음 토큰을 가져온다.
  // 이전에는 primary() 호출이었다.
  left = prefix();
  ...
}
```


## 새로운 AST 노드 타입

`prefix()` 함수에서 두 가지 새로운 AST 노드 타입을 도입했다. 이 타입들은 `defs.h` 파일에 선언되어 있다:

+ **A_DEREF**: 자식 노드의 포인터를 역참조한다.
+ **A_ADDR**: 이 노드에 있는 식별자의 주소를 가져온다.

A_ADDR 노드는 부모 노드가 아니다. 예를 들어 `&fred`라는 표현식에서 `prefix()` 함수는 "fred" 노드의 A_IDENT를 A_ADDR 노드 타입으로 교체한다.


## 새로운 어셈블리 코드 생성하기

일반적인 코드 생성기인 `gen.c`에서 `genAST()` 함수에 추가된 코드는 몇 줄에 불과하다:

```c
    case A_ADDR:
      return (cgaddress(n->v.id));
    case A_DEREF:
      return (cgderef(leftreg, n->left->type));
```

A_ADDR 노드는 `n->v.id` 식별자의 주소를 레지스터에 로드하는 코드를 생성한다. A_DEREF 노드는 `leftreg`에 있는 포인터 주소와 해당 타입을 가져와 이 주소에 있는 값을 담은 레지스터를 반환한다.


### x86-64 구현

다른 컴파일러가 생성한 어셈블리 코드를 검토하여 다음 어셈블리 출력을 작성했다. 이 코드가 정확하지 않을 수도 있다.

```c
// 전역 식별자의 주소를 변수에 로드하는 코드를 생성한다. 새로운 레지스터를 반환한다.
int cgaddress(int id) {
  int r = alloc_register();

  fprintf(Outfile, "\tleaq\t%s(%%rip), %s\n", Gsym[id].name, reglist[r]);
  return (r);
}

// 포인터를 역참조하여 가리키는 값을 동일한 레지스터에 로드한다.
int cgderef(int r, int type) {
  switch (type) {
    case P_CHARPTR:
      fprintf(Outfile, "\tmovzbq\t(%s), %s\n", reglist[r], reglist[r]);
      break;
    case P_INTPTR:
    case P_LONGPTR:
      fprintf(Outfile, "\tmovq\t(%s), %s\n", reglist[r], reglist[r]);
      break;
  }
  return (r);
}
```

`leaq` 명령어는 지정된 식별자의 주소를 로드한다. `cgderef` 함수에서 `(%r8)` 구문은 `%r8` 레지스터가 가리키는 값을 로드한다.


## 새로운 기능 테스트

새로운 테스트 파일인 `tests/input15.c`와 이를 컴파일한 결과는 다음과 같다:

```c
int main() {
  char  a; char *b; char  c;
  int   d; int  *e; int   f;

  a= 18; printint(a);
  b= &a; c= *b; printint(c);

  d= 12; printint(d);
  e= &d; f= *e; printint(f);
  return(0);
}

```

```
$ make test15
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c
   scan.c stmt.c sym.c tree.c types.c
./comp1 tests/input15.c
cc -o out out.s lib/printint.c
./out
18
18
12
12
```

이제 테스트 파일들이 실제 C 프로그램이 되었으므로 파일 확장자를 `.c`로 변경했다. 또한 `tests/mktests` 스크립트를 수정해 "실제" 컴파일러를 사용해 테스트 파일을 컴파일하고 올바른 결과를 생성하도록 했다.


## 결론과 다음 단계

지금까지 포인터의 기본적인 구현을 마쳤다. 하지만 아직 완벽하지는 않다. 예를 들어, 다음과 같은 코드를 작성하면:

```c
int main() {
  int x; int y;
  int *iptr;
  x= 10; y= 20;
  iptr= &x + 1;
  printint( *iptr);
}
```

이 코드는 20을 출력해야 한다. 왜냐하면 `&x + 1`은 `x`의 다음 `int` 위치, 즉 `y`를 가리켜야 하기 때문이다. `y`는 `x`에서 8바이트 떨어져 있다. 하지만 현재 컴파일러는 단순히 `x`의 주소에 1을 더할 뿐이다. 이는 잘못된 동작이다. 이 문제를 해결하는 방법을 찾아야 한다.

컴파일러 작성 여정의 다음 단계에서는 이 문제를 해결해 보자. [다음 단계](../16_Global_Vars/Readme.md)



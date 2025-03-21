# 7부: 비교 연산자

다음으로 IF 문을 추가하려고 했지만, 먼저 비교 연산자를 추가하는 것이 더 나을 것 같다는 생각이 들었다. 비교 연산자는 기존의 이항 연산자와 비슷하기 때문에 구현이 상대적으로 쉬웠다.

이제 여섯 가지 비교 연산자(`==`, `!=`, `<`, `>`, `<=`, `>=`)를 추가하기 위해 어떤 변경이 필요한지 간단히 살펴보자.


## 새로운 토큰 추가

여섯 개의 새로운 토큰을 추가할 예정이므로, `defs.h` 파일에 이를 포함시키자:

```c
// 토큰 타입
enum {
  T_EOF,
  T_PLUS, T_MINUS,
  T_STAR, T_SLASH,
  T_EQ, T_NE,
  T_LT, T_GT, T_LE, T_GE,
  T_INTLIT, T_SEMI, T_ASSIGN, T_IDENT,
  // 키워드
  T_PRINT, T_INT
};
```

우선순위가 있는 토큰을 낮은 순서부터 높은 순서로 재배치했다. 이렇게 하면 우선순위가 없는 토큰보다 앞에 위치하게 된다.


## 토큰 스캐닝

이제 토큰을 스캔해야 한다. 여기서는 `=`와 `==`, `<`와 `<=`, `>`와 `>=`를 구별해야 한다. 따라서 입력에서 추가 문자를 읽어야 하며, 필요하지 않은 경우 다시 되돌려 놓아야 한다. 다음은 `scan.c` 파일의 `scan()` 함수에 추가된 새로운 코드다:

```c
  case '=':
    if ((c = next()) == '=') {
      t->token = T_EQ;
    } else {
      putback(c);
      t->token = T_ASSIGN;
    }
    break;
  case '!':
    if ((c = next()) == '=') {
      t->token = T_NE;
    } else {
      fatalc("Unrecognised character", c);
    }
    break;
  case '<':
    if ((c = next()) == '=') {
      t->token = T_LE;
    } else {
      putback(c);
      t->token = T_LT;
    }
    break;
  case '>':
    if ((c = next()) == '=') {
      t->token = T_GE;
    } else {
      putback(c);
      t->token = T_GT;
    }
    break;
```

또한 `=` 토큰의 이름을 T_ASSIGN으로 변경해 새로운 T_EQ 토큰과 혼동하지 않도록 했다.


## 새로운 표현식 코드

이제 6개의 새로운 토큰을 스캔할 수 있다. 이제 이 토큰들이 표현식에서 나타날 때 이를 파싱하고, 연산자 우선순위를 적용해야 한다.

지금쯤이면 다음 사항을 파악했을 것이다:

  + 나는 자체 컴파일이 가능한 컴파일러를 만들고 있다
  + C 언어를 사용한다
  + SubC 컴파일러를 참고로 한다.

이것은 내가 C 언어의 부분 집합을 위한 컴파일러를 작성 중이라는 의미다. SubC와 마찬가지로, 이 컴파일러는 스스로를 컴파일할 수 있어야 한다. 따라서 일반적인 [C 연산자 우선순위](https://en.cppreference.com/w/c/language/operator_precedence)를 사용해야 한다. 이는 비교 연산자의 우선순위가 곱셈과 나눗셈보다 낮다는 것을 의미한다.

또한, 토큰을 AST 노드 타입으로 매핑하기 위해 사용하던 switch 문이 점점 더 커질 것이라는 사실을 깨달았다. 그래서 모든 이항 연산자에 대해 1:1 매핑이 가능하도록 AST 노드 타입을 재구성하기로 결정했다 (`defs.h` 파일에서):

```c
// AST 노드 타입. 처음 몇 개는 관련 토큰과 일치
enum {
  A_ADD=1, A_SUBTRACT, A_MULTIPLY, A_DIVIDE,
  A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE,
  A_INTLIT,
  A_IDENT, A_LVIDENT, A_ASSIGN
};
```

이제 `expr.c` 파일에서 토큰을 AST 노드로 변환하는 과정을 단순화할 수 있으며, 새로운 토큰의 우선순위도 추가할 수 있다:

```c
// 이항 연산자 토큰을 AST 연산으로 변환.
// 토큰과 AST 연산 간 1:1 매핑에 의존
static int arithop(int tokentype) {
  if (tokentype > T_EOF && tokentype < T_INTLIT)
    return(tokentype);
  fatald("Syntax error, token", tokentype);
}

// 각 토큰의 연산자 우선순위. defs.h 파일의 토큰 순서와 일치해야 함
static int OpPrec[] = {
  0, 10, 10,                    // T_EOF, T_PLUS, T_MINUS
  20, 20,                       // T_STAR, T_SLASH
  30, 30,                       // T_EQ, T_NE
  40, 40, 40, 40                // T_LT, T_GT, T_LE, T_GE
};
```

이것으로 파싱과 연산자 우선순위에 대한 작업은 끝났다!


## 코드 생성

새로 추가된 6개의 연산자는 모두 바이너리 연산자이므로, `gen.c` 파일의 일반적인 코드 생성기를 수정하여 이들을 처리하는 것은 간단하다:

```c
  case A_EQ:
    return (cgequal(leftreg, rightreg));
  case A_NE:
    return (cgnotequal(leftreg, rightreg));
  case A_LT:
    return (cglessthan(leftreg, rightreg));
  case A_GT:
    return (cggreaterthan(leftreg, rightreg));
  case A_LE:
    return (cglessequal(leftreg, rightreg));
  case A_GE:
    return (cggreaterequal(leftreg, rightreg));
```


## x86-64 코드 생성

이제 조금 까다로운 부분이다. C 언어에서 비교 연산자는 값을 반환한다. 비교 결과가 참이면 1을, 거짓이면 0을 반환한다. 이를 x86-64 어셈블리 코드로 구현해야 한다.

다행히 x86-64에는 이를 위한 명령어가 있다. 하지만 몇 가지 문제를 해결해야 한다. 다음 x86-64 명령어를 살펴보자:

```
    cmpq %r8,%r9
```

위의 `cmpq` 명령어는 %r9 - %r8 연산을 수행하고, 결과에 따라 여러 상태 플래그(negative, zero 등)를 설정한다. 따라서 이 플래그 조합을 통해 비교 결과를 확인할 수 있다:

| 비교      | 연산      | 참일 때 플래그           |
|-----------|-----------|--------------------------|
| %r8 == %r9 | %r9 - %r8 | Zero                    |
| %r8 != %r9 | %r9 - %r8 | Not Zero                |
| %r8 > %r9  | %r9 - %r8 | Not Zero, Negative      |
| %r8 < %r9  | %r9 - %r8 | Not Zero, Not Negative  |
| %r8 >= %r9 | %r9 - %r8 | Zero or Negative        |
| %r8 <= %r9 | %r9 - %r8 | Zero or Not Negative    |

이에 대응하는 x86-64 명령어는 `sete`, `setne`, `setg`, `setl`, `setge`, `setle`로, 위 테이블의 각 행에 해당한다.

문제는 이 명령어들이 레지스터의 최하위 바이트만 설정한다는 점이다. 만약 레지스터의 최하위 바이트 외에 다른 비트가 이미 설정되어 있다면, 이 비트들은 그대로 유지된다. 따라서 변수에 1을 설정하려 했지만, 이미 1000(10진수) 값이 있었다면 결과는 1001이 되어버린다. 이는 우리가 원하는 결과가 아니다.

해결 방법은 `setX` 명령어 실행 후 `andq`를 사용해 원치 않는 비트를 제거하는 것이다. `cg.c`에는 이를 수행하는 일반적인 비교 함수가 있다:

```c
// 두 레지스터를 비교한다.
static int cgcompare(int r1, int r2, char *how) {
  fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  fprintf(Outfile, "\t%s\t%s\n", how, breglist[r2]);
  fprintf(Outfile, "\tandq\t$255,%s\n", reglist[r2]);
  free_register(r1);
  return (r2);
}
```

여기서 `how`는 `setX` 명령어 중 하나다. 주목할 점은 다음과 같은 연산을 수행한다는 것이다:

```
   cmpq reglist[r2], reglist[r1]
```

이는 실제로 `reglist[r1] - reglist[r2]` 연산을 수행하는데, 이는 우리가 원하는 결과와 일치한다.


## x86-64 레지스터

여기서 잠시 x86-64 아키텍처의 레지스터에 대해 알아보자. x86-64에는 여러 개의 64비트 범용 레지스터가 있으며, 이 레지스터의 하위 부분에 접근하고 작업하기 위해 다른 레지스터 이름을 사용할 수 있다.

![](https://i.stack.imgur.com/N0KnG.png)

위 이미지(*stack.imgur.com*에서 가져옴)는 64비트 *r8* 레지스터의 하위 32비트에 접근하기 위해 "*r8d*" 레지스터를 사용하는 방법을 보여준다. 마찬가지로, "*r8w*" 레지스터는 하위 16비트를, "*r8b*" 레지스터는 하위 8비트를 나타낸다.

`cgcompare()` 함수에서 코드는 두 개의 64비트 레지스터를 비교하기 위해 `reglist[]` 배열을 사용하지만, `breglist[]` 배열에 있는 이름을 사용해 두 번째 레지스터의 8비트 버전에 플래그를 설정한다. x86-64 아키텍처는 `setX` 명령어가 8비트 레지스터 이름에서만 작동하도록 허용하기 때문에 `breglist[]` 배열이 필요하다.


## 여러 비교 명령어 생성하기

이제 일반 함수를 만들었으니, 여섯 가지 실제 비교 함수를 작성할 수 있다:

```c
int cgequal(int r1, int r2) { return(cgcompare(r1, r2, "sete")); }
int cgnotequal(int r1, int r2) { return(cgcompare(r1, r2, "setne")); }
int cglessthan(int r1, int r2) { return(cgcompare(r1, r2, "setl")); }
int cggreaterthan(int r1, int r2) { return(cgcompare(r1, r2, "setg")); }
int cglessequal(int r1, int r2) { return(cgcompare(r1, r2, "setle")); }
int cggreaterequal(int r1, int r2) { return(cgcompare(r1, r2, "setge")); }
```

다른 이항 연산자 함수와 마찬가지로, 하나의 레지스터는 해제되고 다른 레지스터는 결과를 반환한다.


# 실습해 보기

`input04` 입력 파일을 살펴보자:

```c
int x;
x= 7 < 9;  print x;
x= 7 <= 9; print x;
x= 7 != 9; print x;
x= 7 == 7; print x;
x= 7 >= 7; print x;
x= 7 <= 7; print x;
x= 9 > 7;  print x;
x= 9 >= 7; print x;
x= 9 != 7; print x;
```

이 비교 연산들은 모두 참이므로 1을 아홉 번 출력해야 한다. `make test` 명령어로 이를 확인해 보자.

첫 번째 비교 연산에서 생성된 어셈블리 코드를 살펴보자:

```
        movq    $7, %r8
        movq    $9, %r9
        cmpq    %r9, %r8        # %r8 - %r9 연산 수행, 즉 7 - 9
        setl    %r9b            # 7이 9보다 작으면 %r9b를 1로 설정
        andq    $255,%r9        # %r9의 다른 비트 제거
        movq    %r9, x(%rip)    # 결과를 x에 저장
        movq    x(%rip), %r8
        movq    %r8, %rdi
        call    printint        # x 출력
```

위 어셈블리 코드는 다소 비효율적이다. 아직 최적화된 코드를 고려하지 않았다. 도널드 크누스의 말을 인용하자면:

> **조급한 최적화는 프로그래밍에서 모든 악의 근원이다(적어도 대부분은).**


## 결론 및 다음 단계

컴파일러에 간단히 추가한 기능이 잘 동작한다. 다음 단계는 훨씬 더 복잡한 작업이 될 것이다.

컴파일러 개발의 다음 단계에서는 IF 문을 추가하고, 방금 구현한 비교 연산자를 활용할 것이다. [다음 단계](../08_If_Statements/Readme.md)



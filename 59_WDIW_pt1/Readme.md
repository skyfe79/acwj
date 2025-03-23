# 59장: 왜 작동하지 않을까? (1부)

이제 **WDIW**(왜 작동하지 않을까?) 단계에 도달했다. 이 첫 번째 파트에서는 비교적 쉽게 발견할 수 있는 버그 몇 가지를 찾아서 수정한다. 이는 아직 발견하지 못한 더 미묘한 버그들이 남아 있음을 의미한다.


## `*argv[i]`의 잘못된 코드 생성

`cwj`(Gnu C 컴파일 버전)를 사용해 `cwj0`를 빌드하고 있다. `cwj0`의 어셈블리 코드는 Gnu C가 생성한 것이 아니라, 우리가 직접 작성한 어셈블리 코드다. 따라서 `cwj0`를 실행할 때 발생하는 모든 오류는 우리의 어셈블리 코드가 정확하지 않기 때문이다.

처음 발견한 버그는 `*argv[i]`가 `(*argv)[i]`처럼 코드를 생성한다는 점이었다. 즉, `argv[i]`의 첫 번째 문자가 아니라 항상 `*argv`의 i번째 문자를 가리키는 코드가 생성됐다.

처음에는 이 문제가 구문 분석 오류라고 생각했지만, 사실은 아니었다. 실제로는 `argv[i]`를 역참조하기 전에 rvalue로 설정하지 않았던 것이 원인이었다. `cwj`와 `cwj0`의 AST 트리를 덤프해서 비교해보며 이 사실을 알아냈다. 해결책은 `*` 토큰 뒤의 표현식을 rvalue로 표시하는 것이다. 이제 `expr.c`의 `prefix()` 함수에서 이 작업을 수행한다:

```c
static struct ASTnode *prefix(int ptp) {
  struct ASTnode *tree;
  switch (Token.token) {
  ...
  case T_STAR:
    // 다음 토큰을 가져와 접두사 표현식으로
    // 재귀적으로 파싱한다.
    // 이를 rvalue로 설정한다.
    scan(&Token);
    tree = prefix(ptp);
    tree->rvalue= 1;
```


## 외부 심볼도 전역 심볼이다

이 문제는 계속 나를 괴롭힐 것 같다. 외부 심볼을 전역 심볼로 취급하지 않는 또 다른 경우를 발견했다. 이 문제는 `gen.c` 파일의 `genAST()` 함수에서 할당 어셈블리 코드를 생성할 때 발생했다. 해결 방법은 다음과 같다:

```c
      // 이제 할당 코드로 이동
      // 식별자에 할당하는지, 포인터를 통해 할당하는지 확인
      switch (n->right->op) {
        case A_IDENT:
          if (n->right->sym->class == C_GLOBAL ||
              n->right->sym->class == C_EXTERN ||
              n->right->sym->class == C_STATIC)
            return (cgstorglob(leftreg, n->right->sym));
          else
            return (cgstorlocal(leftreg, n->right->sym));
```


## 스캐닝이 동작 중

현재 `cwj0` 컴파일러는 소스 코드를 읽어들이지만 아직 어떤 출력도 생성하지 못한다. 이를 위한 새로운 `Makefile` 규칙은 다음과 같다:

```
# 삼중 테스트 시도
triple: cwj1

cwj1: cwj0 $(SRCS) $(HSRCS)
        ./cwj0 -o cwj1 $(SRCS)

cwj0: install $(SRCS) $(HSRCS)
        ./cwj -o cwj0 $(SRCS)
```

`$ make triple` 명령을 실행하면 Gnu C로 `cwj`를 빌드한 후, `cwj`를 사용해 `cwj0`을 빌드하고, 마지막으로 `cwj0`을 사용해 `cwj1`을 빌드한다. 이에 대해 아래에서 자세히 설명한다.

현재 `cwj1`은 어셈블리 출력이 없어 생성할 수 없다! 문제는 컴파일러가 어디까지 진행되는지 확인하는 것이다. 이를 알아보기 위해 `scan()` 함수의 끝에 `printf()`를 추가했다:

```c
  // 토큰을 찾았음
  t->tokstr = Tstring[t->token];
  printf("Scanned %d\n", t->token);
  return (1);
```

이를 추가한 후, `cwj`와 `cwj0` 모두 50,404개의 토큰을 스캔하고, 결과 토큰 스트림이 동일함을 확인했다. 따라서 `scan()`까지는 정상적으로 동작한다고 결론지을 수 있다.

그러나 `./cwj0 -S -T cg.c`의 출력 결과는 AST 트리를 보여주지 않는다. `gdb cwj0`을 실행하고, `dumpAST()`에 중단점을 설정한 후 `-S -T cg.c` 인자와 함께 실행하면, `dumpAST()`에 도달하기 전에 종료된다. 또한 `function_declaration()`에도 도달하지 못한다. 그렇다면 왜 동작하지 않는 걸까?

아, `0(%rbp)`에 대한 메모리 접근을 발견했다. 이는 절대 발생해서는 안 된다. 모든 지역 변수는 프레임 포인터를 기준으로 음수 위치에 있어야 한다. `cg.c`의 `cgaddress()`에서 또 다른 외부 테스트를 놓쳤다. 이제 다음과 같이 수정했다:

```c
int cgaddress(struct symtable *sym) {
  int r = alloc_register();

  if (sym->class == C_GLOBAL ||
      sym->class == C_EXTERN ||
      sym->class == C_STATIC)
    fprintf(Outfile, "\tleaq\t%s(%%rip), %s\n", sym->name, reglist[r]);
  else
    fprintf(Outfile, "\tleaq\t%d(%%rbp), %s\n", sym->st_posn, reglist[r]);
  return (r);
}
```

이런 외부 문제들 때문에 골치가 아프다! 모두 내 잘못이니, 여기서 책임을 져야 한다.


## 잘못된 비교

위의 변경 사항을 추가한 후, 다음과 같은 오류가 발생한다:

```
$ ./cwj0 -S tests/input001.c 
invalid digit in integer literal:e on line 1 of tests/input001.c
```

이 문제는 `scan.c` 파일의 `scanint()` 함수 내부에 있는 다음 루프 때문에 발생했다:

```c
static int scanint(int c) {
  int k;
  ...
  // 각 문자를 정수 값으로 변환
  while ((k = chrpos("0123456789abcdef", tolower(c))) >= 0) {
```

여기서 `k=` 할당은 메모리에 결과를 저장할 뿐만 아니라 표현식으로도 사용된다. 이 경우 `k` 결과가 비교에 사용된다. 즉, `k >= 0`이다. `k`는 `int` 타입이며, 메모리에 값을 저장하기 위해 다음과 같은 어셈블리 코드가 생성된다:

```
    movl    %r10d, -8(%rbp)
```

`chrpos()`가 `-1`을 반환하면, 이 값은 32비트(`0xffffffff`)로 잘려 `-8(%rbp)` 즉, `k`에 저장된다. 하지만 이후 비교에서:

```
    movslq  -8(%rbp), %r10    # k의 값을 다시 로드
    movq    $0, %r11          # 0 로드
    cmpq    %r11, %r10        # k의 값을 0과 비교
```

`k`의 *32비트* 값을 `%r10`에 로드한 후, *64비트* 비교를 수행한다. 64비트 값으로 `0xffffffff`는 양수이므로, 루프 비교가 참으로 유지되고, 루프를 빠져나와야 할 때 빠져나가지 못한다.

이 문제를 해결하려면 비교 연산의 피연산자 크기에 따라 다른 `cmp` 명령어를 사용해야 한다. `cg.c` 파일의 `cgcompare_and_set()` 함수를 다음과 같이 수정했다:

```c
int cgcompare_and_set(int ASTop, int r1, int r2, int type) {
  int size = cgprimsize(type);
  ...
  switch (size) {
  case 1:
    fprintf(Outfile, "\tcmpb\t%s, %s\n", breglist[r2], breglist[r1]);
    break;
  case 4:
    fprintf(Outfile, "\tcmpl\t%s, %s\n", dreglist[r2], dreglist[r1]);
    break;
  default:
    fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  }
  ...
}
```

이제 올바른 비교 명령어가 사용된다. 비슷한 기능을 하는 `cgcompare_and_jump()` 함수도 있으며, 나중에 이 두 함수를 리팩토링하고 병합해야 한다.


# 거의 다 왔다!

우리는 이제 비공식적으로 **트리플 테스트**라고 불리는 것을 통과하기 직전이다. 트리플 테스트에서는 기존 컴파일러를 사용해 소스 코드로부터 우리의 컴파일러를 빌드한다(1단계). 그런 다음 이 컴파일러를 사용해 스스로를 빌드한다(2단계). 이제 컴파일러가 스스로를 컴파일할 수 있음을 증명하기 위해, 2단계 컴파일러를 사용해 스스로를 다시 빌드한다. 이 결과물이 3단계 컴파일러다.

현재 우리는 다음 작업을 수행할 수 있다:

 + Gnu C 컴파일러를 사용해 `cwj`를 빌드한다(1단계)
 + `cwj` 컴파일러를 사용해 `cwj0`를 빌드한다(2단계)
 + `cwj0` 컴파일러를 사용해 `cwj1`를 빌드한다(3단계)

그러나 `cwj0`와 `cwj1`의 바이너리 크기가 일치하지 않는다:

```
$ size cwj[01]
   text    data     bss     dec     hex filename
 109636    3028      48  112712   1b848 cwj0
 109476    3028      48  112552   1b7a8 cwj1
```

이 두 크기는 *정확히* 일치해야 한다. 컴파일러가 스스로를 여러 번 연속해서 컴파일하고 동일한 결과를 생성할 때만, 우리는 컴파일러가 제대로 스스로를 컴파일하고 있다고 확신할 수 있다.

결과가 정확히 일치하지 않는 한, 2단계와 3단계 사이에 미묘한 동작 차이가 존재한다는 의미다. 따라서 컴파일러가 일관되게 스스로를 컴파일하고 있지 않다는 것을 알 수 있다.


## 결론과 다음 단계

이번 단계에서 `cwj`, `cwj0`, `cwj1`을 한 번에 빌드할 수 있게 될 줄은 몰랐다. 이 시점까지 오기 전에 수많은 버그를 해결해야 할 거라 예상했었다.

다음 문제는 왜 stage 2와 stage 3 컴파일러의 크기가 다른지 알아내는 것이다. `size` 출력을 보면 데이터와 bss 섹션은 동일하지만, 두 컴파일러 간의 어셈블리 코드 양이 다르다.

컴파일러 작성 여정의 다음 단계에서는 각 단계 간의 어셈블리 출력을 나란히 비교해 보며, 무엇이 차이를 만드는지 알아볼 것이다.

> P.S. 이번 단계에서는 `gdb`가 멈춘 소스 코드 라인 번호를 확인할 수 있도록 일부 어셈블리 출력을 추가하기 시작했다. 아직 작동하지는 않지만, `cg.c` 파일에 새로운 함수 `cglinenum()`이 추가된 것을 볼 수 있다. 이 기능이 작동하게 되면 관련 내용을 정리해 공유할 것이다. [다음 단계](../60_TripleTest/Readme.md)



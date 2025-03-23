# 28장: 런타임 플래그 추가하기

이번 장은 컴파일러 개발 과정 중 스캐닝, 파싱, 의미 분석, 코드 생성과는 직접적인 관련이 없다. 대신 전통적인 유닉스 C 컴파일러와 비슷한 동작을 하도록 `-c`, `-S`, `-o` 런타임 플래그를 추가한다.

이 내용이 흥미롭지 않다면 다음 장으로 넘어가도 좋다.


## 컴파일 단계

지금까지 우리의 컴파일러는 어셈블리 파일만 출력했다. 하지만 고수준 언어로 작성된 소스 코드 파일을 실행 가능한 파일로 변환하려면 더 많은 단계가 필요하다:

+ 소스 코드 파일을 스캔하고 파싱하여 어셈블리 출력을 생성
+ 어셈블리 출력을 [오브젝트 파일](https://en.wikipedia.org/wiki/Object_file)로 어셈블
+ 하나 이상의 오브젝트 파일을 [링크](https://en.wikipedia.org/wiki/Linker_(computing))하여 실행 파일 생성

지금까지는 마지막 두 단계를 수동으로 또는 Makefile을 통해 처리했다. 하지만 이제 컴파일러를 수정해 외부 어셈블러와 링커를 호출하여 마지막 두 단계를 수행하도록 할 것이다.

이를 위해 `main.c`의 일부 코드를 재구성하고, 어셈블링과 링킹을 수행하는 함수를 추가로 작성할 것이다. 대부분의 코드는 C 언어에서 일반적으로 사용되는 문자열 및 파일 처리 코드이므로, 이 코드를 살펴보겠지만 이런 종류의 코드를 처음 보는 경우에만 흥미로울 수 있다.


## 커맨드라인 플래그 파싱

프로젝트 이름을 반영하기 위해 컴파일러 이름을 `cwj`로 변경했다. 이제 아무런 커맨드라인 인자 없이 실행하면 다음과 같은 사용법 메시지를 출력한다:

```
$ ./cwj 
사용법: ./cwj [-vcST] [-o outfile] file [file ...]
       -v 컴파일 단계를 자세히 출력
       -c 오브젝트 파일을 생성하지만 링크하지 않음
       -S 어셈블리 파일을 생성하지만 링크하지 않음
       -T 각 입력 파일의 AST 트리를 덤프
       -o outfile, 실행 파일을 outfile로 생성
```

이제 여러 개의 소스 코드 파일을 입력으로 받을 수 있다. 네 가지 불리언 플래그 `-v`, `-c`, `-S`, `-T`를 제공하며, 출력 실행 파일의 이름을 지정할 수도 있다.

`main()` 함수 내의 `argv[]` 파싱 코드는 이를 처리하기 위해 변경되었으며, 결과를 저장하기 위한 여러 옵션 변수가 추가되었다.

```c
  // 변수 초기화
  O_dumpAST = 0;        // true이면 AST 트리 덤프
  O_keepasm = 0;        // true이면 어셈블리 파일 유지
  O_assemble = 0;       // true이면 어셈블리 파일을 어셈블
  O_dolink = 1;         // true이면 오브젝트 파일 링크
  O_verbose = 0;        // true이면 컴파일 단계 정보 출력

  // 커맨드라인 옵션 스캔
  for (i = 1; i < argc; i++) {
    // '-'로 시작하지 않으면 옵션 스캔 중지
    if (*argv[i] != '-')
      break;

    // 각 인자의 옵션 처리
    for (int j = 1; (*argv[i] == '-') && argv[i][j]; j++) {
      switch (argv[i][j]) {
      case 'o':
        outfilename = argv[++i]; break;         // 다음 인자로 이동하여 저장
      case 'T':
        O_dumpAST = 1; break;
      case 'c':
        O_assemble = 1; O_keepasm = 0; O_dolink = 0; break;
      case 'S':
        O_keepasm = 1; O_assemble = 0; O_dolink = 0; break;
      case 'v':
        O_verbose = 1; break;
      default:
        usage(argv[0]);
      }
    }
  }
```

일부 옵션은 상호 배타적이다. 예를 들어 `-S`로 어셈블리 출력만 원한다면, 링크하거나 오브젝트 파일을 생성할 필요가 없다.


## 컴파일 단계 수행하기

커맨드라인 플래그를 파싱한 후, 이제 컴파일 단계를 실행할 수 있다. 각 입력 파일을 쉽게 컴파일하고 어셈블할 수 있지만, 마지막에 여러 오브젝트 파일을 링크해야 할 수도 있다. 따라서 `main()` 함수 내에 오브젝트 파일 이름을 저장할 지역 변수를 선언한다:

```c
#define MAXOBJ 100
  char *objlist[MAXOBJ];        // 오브젝트 파일 이름 목록
  int objcnt = 0;               // 다음 이름을 삽입할 위치
```

먼저 모든 입력 소스 파일을 순차적으로 처리한다:

```c
  // 각 입력 파일을 순차적으로 처리
  while (i < argc) {
    asmfile = do_compile(argv[i]);      // 소스 파일 컴파일

    if (O_dolink || O_assemble) {
      objfile = do_assemble(asmfile);   // 오브젝트 형식으로 어셈블
      if (objcnt == (MAXOBJ - 2)) {
        fprintf(stderr, "컴파일러가 처리할 수 있는 오브젝트 파일 수를 초과했습니다\n");
        exit(1);
      }
      objlist[objcnt++] = objfile;      // 오브젝트 파일 이름을 목록에 추가
      objlist[objcnt] = NULL;           // 오브젝트 파일 목록의 끝을 표시
    }

    if (!O_keepasm)                     // 어셈블리 파일을 유지할 필요가 없으면
      unlink(asmfile);                  // 어셈블리 파일 삭제
    i++;
  } 
```

`do_compile()` 함수에는 이전에 `main()`에 있던 코드가 포함되어 있다. 이 코드는 파일을 열고 직접 파싱한 후 어셈블리 파일을 생성한다. 하지만 이전처럼 하드코딩된 파일명 `out.s`를 열 수는 없다. 이제 `filename.c`를 `filename.s`로 변환해야 한다.


## 입력 파일명 변경하기

파일명을 변경하는 헬퍼 함수를 살펴보자.

```c
// '.'과 최소 1글자 이상의 접미사가 있는 문자열이 주어졌을 때,
// 접미사를 주어진 문자로 변경한다.
// 원본 문자열을 수정할 수 없는 경우 NULL을 반환한다.
char *alter_suffix(char *str, char suffix) {
  char *posn;
  char *newstr;

  // 문자열 복제
  if ((newstr = strdup(str)) == NULL) return (NULL);

  // '.' 위치 찾기
  if ((posn = strrchr(newstr, '.')) == NULL) return (NULL);

  // 접미사가 있는지 확인
  posn++;
  if (*posn == '\0') return (NULL);

  // 접미사 변경 및 문자열 종료
  *posn++ = suffix; *posn = '\0';
  return (newstr);
}
```

실제 작업은 `strdup()`, `strrchr()`, 그리고 마지막 두 줄에서만 이루어진다. 나머지는 오류 검사다.


## 컴파일 작업 수행

다음은 기존 코드를 새로운 함수로 재구성한 예제이다.

```c
// 입력 파일명을 받아서 그 파일을 어셈블리 코드로 컴파일한다.
// 새로 생성된 파일의 이름을 반환한다.
static char *do_compile(char *filename) {
  Outfilename = alter_suffix(filename, 's');
  if (Outfilename == NULL) {
    fprintf(stderr, "Error: %s has no suffix, try .c on the end\n", filename);
    exit(1);
  }
  // 입력 파일 열기
  if ((Infile = fopen(filename, "r")) == NULL) {
    fprintf(stderr, "Unable to open %s: %s\n", filename, strerror(errno));
    exit(1);
  }
  // 출력 파일 생성
  if ((Outfile = fopen(Outfilename, "w")) == NULL) {
    fprintf(stderr, "Unable to create %s: %s\n", Outfilename,
            strerror(errno));
    exit(1);
  }

  Line = 1;                     // 스캐너 초기화
  Putback = '\n';
  clear_symtable();             // 심볼 테이블 초기화
  if (O_verbose)
    printf("compiling %s\n", filename);
  scan(&Token);                 // 입력에서 첫 번째 토큰 가져오기
  genpreamble();                // 프리앰블 출력
  global_declarations();        // 전역 선언 파싱
  genpostamble();               // 포스트앰블 출력
  fclose(Outfile);              // 출력 파일 닫기
  return (Outfilename);
}
```

여기서 새로운 코드는 거의 없으며, `alter_suffix()`를 호출해 올바른 출력 파일명을 얻는 부분만 추가되었다.

중요한 변경 사항 하나는 어셈블리 출력 파일이 이제 `Outfilename`이라는 전역 변수로 관리된다는 점이다. 이 변경으로 인해 `misc.c`의 `fatal()` 함수와 관련 함수들이 어셈블리 파일을 완전히 생성하지 못한 경우 해당 파일을 삭제할 수 있게 되었다. 예를 들면 다음과 같다.

```c
// 치명적 오류 메시지 출력
void fatal(char *s) {
  fprintf(stderr, "%s on line %d\n", s, Line);
  fclose(Outfile);
  unlink(Outfilename);
  exit(1);
}
```


## 위 출력 결과 조립

이제 조립 출력 파일을 얻었으니, 외부 어셈블러를 호출할 수 있다. 이는 `defs.h`에서 ASCMD로 정의되어 있다. 다음은 이를 수행하는 함수다:

```c
#define ASCMD "as -o "
// 입력 파일명을 받아 해당 파일을 오브젝트 코드로 조립한다.
// 오브젝트 파일명을 반환한다.
char *do_assemble(char *filename) {
  char cmd[TEXTLEN];
  int err;

  char *outfilename = alter_suffix(filename, 'o');
  if (outfilename == NULL) {
    fprintf(stderr, "Error: %s has no suffix, try .s on the end\n", filename);
    exit(1);
  }
  // 조립 명령어를 만들고 실행한다.
  snprintf(cmd, TEXTLEN, "%s %s %s", ASCMD, outfilename, filename);
  if (O_verbose) printf("%s\n", cmd);
  err = system(cmd);
  if (err != 0) { fprintf(stderr, "Assembly of %s failed\n", filename); exit(1); }
  return (outfilename);
}
```

`snprintf()`를 사용해 실행할 조립 명령어를 만든다. 사용자가 `-v` 커맨드라인 플래그를 사용했다면, 이 명령어가 화면에 표시된다. 그런 다음 `system()`을 사용해 이 리눅스 명령어를 실행한다. 예를 들면 다음과 같다:

```
$ ./cwj -v -c tests/input54.c 
compiling tests/input54.c
as -o  tests/input54.o tests/input54.s
```


## 오브젝트 파일 연결하기

`main()` 함수에서는 `do_assemble()`이 반환한 오브젝트 파일 목록을 구성한다:

```c
      objlist[objcnt++] = objfile;      // 오브젝트 파일 이름을 목록에 추가
      objlist[objcnt] = NULL;           // 오브젝트 파일 목록의 끝을 표시
```

이 모든 파일을 연결할 때는 이 목록을 `do_link()` 함수에 전달해야 한다. 이 코드는 `do_assemble()`과 유사하게 `snprintf()`와 `system()`을 사용한다. 차이점은 커맨드 버퍼에서 현재 위치를 추적하고, 남은 공간을 계산해 더 많은 `snprintf()` 작업을 수행해야 한다는 점이다.

```c
#define LDCMD "cc -o "
// 오브젝트 파일 목록과 출력 파일 이름을 받아 모든 오브젝트 파일을 연결한다.
void do_link(char *outfilename, char *objlist[]) {
  int cnt, size = TEXTLEN;
  char cmd[TEXTLEN], *cptr;
  int err;

  // 링커 커맨드와 출력 파일로 시작
  cptr = cmd;
  cnt = snprintf(cptr, size, "%s %s ", LDCMD, outfilename);
  cptr += cnt; size -= cnt;

  // 각 오브젝트 파일을 추가
  while (*objlist != NULL) {
    cnt = snprintf(cptr, size, "%s ", *objlist);
    cptr += cnt; size -= cnt; objlist++;
  }

  if (O_verbose) printf("%s\n", cmd);
  err = system(cmd);
  if (err != 0) { fprintf(stderr, "Linking failed\n"); exit(1); }
}
```

한 가지 불편한 점은 여전히 외부 C 컴파일러 `cc`를 호출해 링크를 수행한다는 것이다. 이렇게 다른 컴파일러에 의존하지 않고도 작업을 수행할 수 있어야 한다.

오래전에는 다음과 같이 수동으로 오브젝트 파일 세트를 연결할 수 있었다:

```
  $ ln -o out /lib/crt0.o file1.o file.o /usr/lib/libc.a
```

현재 리눅스에서도 비슷한 명령어를 사용할 수 있을 것이라 추측하지만, 아직은 이를 확인할 만한 정보를 찾지 못했다. 이 글을 읽고 답을 알고 있다면 알려주기 바란다!


## `printint()`와 `printchar()` 제거

이제 컴파일 가능한 프로그램에서 `printf()`를 직접 호출할 수 있게 되면서, 수작업으로 만든 `printint()`와 `printchar()` 함수는 더 이상 필요하지 않다. `lib/printint.c` 파일을 삭제했고, `tests/` 디렉토리에 있는 모든 테스트 코드를 `printf()`를 사용하도록 업데이트했다.

또한 `tests/mktests`와 `tests/runtests` 스크립트를 새로운 컴파일러 커맨드라인 인자를 사용하도록 수정했고, 최상위 `Makefile`도 동일하게 업데이트했다. 이제 `make test`를 실행하면 여전히 회귀 테스트가 정상적으로 동작한다.


## 결론과 다음 단계

여기까지가 우리의 여정 중 이번 파트의 끝이다. 이제 우리의 컴파일러는 내가 익숙한 전통적인 유닉스 컴파일러처럼 느껴진다.

이번 단계에서 외부 전처리기 지원을 추가하겠다고 약속했지만, 그렇게 하지 않기로 결정했다. 주된 이유는 전처리기가 출력에 포함하는 파일 이름과 줄 번호를 파싱해야 하기 때문이다. 예를 들어 다음과 같은 내용을 처리해야 한다.

```c
# 1 "tests/input54.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "tests/input54.c"
int printf(char *fmt);

int main()
{
  int i;
  for (i=0; i < 20; i++) {
    printf("Hello world, %d\n", i);
  }
  return(0);
}
```

컴파일러 작성 여정의 다음 파트에서는 구조체(struct) 지원을 추가하는 방법을 살펴볼 것이다. 아마도 변경 사항을 구현하기 전에 또 다른 설계 단계를 거쳐야 할 것 같다. [다음 단계](../29_Refactoring/Readme.md)



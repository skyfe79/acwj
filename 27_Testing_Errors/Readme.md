# 27장: 회귀 테스트와 뜻밖의 성과

최근 컴파일러 개발 과정에서 몇 가지 큰 진전이 있었기 때문에, 이번 단계에서는 잠시 숨을 돌릴 필요가 있다고 생각했다. 속도를 조금 늦추고 지금까지의 진행 상황을 되돌아보자.

지난 단계에서 구문 오류와 의미 오류 검사가 제대로 작동하는지 확인할 방법이 없다는 것을 깨달았다. 그래서 `tests/` 폴더에 있는 스크립트를 다시 작성해 이 문제를 해결했다.

1980년대 후반부터 Unix를 사용해 왔기 때문에, 나의 자동화 도구는 주로 쉘 스크립트와 Makefile이다. 더 복잡한 도구가 필요하면 Python이나 Perl로 스크립트를 작성한다(그렇다, 나는 그 정도로 오래되었다).

이제 `tests/` 디렉토리에 있는 `runtest` 스크립트를 간단히 살펴보자. Unix 스크립트를 오랫동안 사용해 왔지만, 나는 절대 최고 수준의 스크립트 작가는 아니다.


## `runtest` 스크립트

이 스크립트의 역할은 입력 프로그램들을 받아 컴파일러로 컴파일하고, 실행 파일을 실행한 후 그 결과를 미리 준비된 정답과 비교한다. 결과가 일치하면 테스트는 성공이고, 그렇지 않으면 실패로 처리한다.

최근에는 입력과 관련된 "error" 파일이 있는 경우, 컴파일러를 실행해 오류 출력을 캡처하도록 기능을 확장했다. 이 오류 출력이 예상된 오류 출력과 일치하면, 컴파일러가 잘못된 입력을 올바르게 감지한 것으로 간주해 테스트를 성공으로 처리한다.

이제 `runtest` 스크립트의 각 부분을 단계별로 살펴보자.

```
# 필요한 경우 컴파일러를 빌드
if [ ! -f ../comp1 ]
then (cd ..; make)
fi
```

여기서 '( ... )' 구문을 사용해 *서브셸*을 생성한다. 이렇게 하면 원래 셸의 작업 디렉토리에 영향을 주지 않고 상위 디렉토리로 이동해 컴파일러를 다시 빌드할 수 있다.

```
# 각 입력 소스 파일을 사용해 테스트 시도
for i in input*
# 비교할 파일이 없으면 테스트를 실행할 수 없음
do if [ ! -f "out.$i" -a ! -f "err.$i" ]
   then echo "Can't run test on $i, no output file!"
```

'[' 기호는 실제로 외부 유닉스 도구인 *test(1)*를 의미한다. 이 구문을 처음 보는 사람을 위해 설명하자면, *test(1)*는 매뉴얼 페이지의 섹션 1에 해당하는 *test*의 매뉴얼 페이지를 의미한다. 따라서 다음과 같이 명령어를 실행해 매뉴얼을 읽을 수 있다.

```
$ man 1 test
```

`/usr/bin/[` 실행 파일은 보통 `/usr/bin/test`에 링크되어 있다. 따라서 셸 스크립트에서 '['를 사용하는 것은 *test* 명령어를 실행하는 것과 동일하다.

`[ ! -f "out.$i" -a ! -f "err.$i" ]` 라인은 "out.$i" 파일과 "err.$i" 파일이 모두 존재하지 않는지 테스트한다. 두 파일이 모두 없으면 오류 메시지를 출력한다.

```
   # 출력 파일: 소스를 컴파일하고 실행한 후
   # 결과를 캡처해 정답과 비교
   else if [ -f "out.$i" ]
        then
          # 테스트 이름을 출력하고 컴파일러로 컴파일
          echo -n $i
          ../comp1 $i

          # 출력을 어셈블하고 실행한 후
          # 결과를 trial.$i에 저장
          cc -o out out.s ../lib/printint.c
          ./out > trial.$i

          # 정답과 비교
          cmp -s "out.$i" "trial.$i"

          # 다르면 실패를 알리고 차이점 출력
          if [ "$?" -eq "1" ]
          then echo ": failed"
            diff -c "out.$i" "trial.$i"
            echo

          # 실패가 없으면 성공을 알림
          else echo ": OK"
          fi
```

이 부분이 스크립트의 핵심이다. 주석으로 설명이 되어 있지만, 몇 가지 세부 사항을 더 살펴보자. `cmp -s`는 두 텍스트 파일을 비교한다. `-s` 플래그는 출력을 생성하지 않고, `cmp`가 종료할 때 반환하는 종료 값을 설정한다.

> 입력이 같으면 0, 다르면 1, 문제가 발생하면 2를 반환 (매뉴얼 페이지 참조)

`if [ "$?" -eq "1" ]` 라인은 마지막 명령어의 종료 값이 1인지 확인한다. 따라서 컴파일러의 출력이 정답과 다르면 이를 알리고 `diff` 도구를 사용해 두 파일의 차이점을 보여준다.

```
   # 오류 파일: 소스를 컴파일하고 오류 메시지를 캡처
   # 정답과 비교. 이전과 동일한 메커니즘
   else if [ -f "err.$i" ]
        then
          echo -n $i
          ../comp1 $i 2> "trial.$i"
          cmp -s "err.$i" "trial.$i"
          ...
```

이 부분은 "err.$i" 오류 파일이 있을 때 실행된다. 이번에는 셸 구문 `2>`를 사용해 컴파일러의 표준 오류 출력을 "trial.$i" 파일로 캡처하고, 이를 정답 오류 출력과 비교한다. 이후 로직은 이전과 동일하다.


## 우리가 하는 작업: 회귀 테스트

지금까지 테스트에 대해 많이 언급하지 않았지만, 이제 그때가 왔다. 과거에 소프트웨어 개발을 가르친 경험이 있기 때문에, 테스트를 다루지 않는다면 큰 실수를 저지르는 셈이다.

우리가 여기서 진행하는 것은 [**회귀 테스트**](https://en.wikipedia.org/wiki/Regression_testing)다. 위키피디아에서는 이를 다음과 같이 정의한다:

> 회귀 테스트는 이전에 개발되고 테스트된 소프트웨어가 변경 후에도 여전히 정상적으로 동작하는지 확인하기 위해 기능적 및 비기능적 테스트를 다시 실행하는 작업이다.

우리의 컴파일러는 각 단계마다 변화하므로, 새로운 변경 사항이 이전 단계의 기능(및 오류 검사)을 망가뜨리지 않도록 해야 한다. 따라서 변경 사항을 도입할 때마다, 나는 하나 이상의 테스트를 추가하여 a) 해당 기능이 작동하는지 확인하고 b) 이후 변경 사항에서 이 테스트를 다시 실행한다. 모든 테스트가 통과하는 한, 새로운 코드가 기존 코드를 망가뜨리지 않았다고 확신할 수 있다.


### 기능 테스트

`runtests` 스크립트는 `out` 접두사가 붙은 파일을 찾아 기능 테스트를 수행한다. 현재 다음과 같은 파일들이 있다:

```
tests/out.input01.c  tests/out.input12.c   tests/out.input22.c
tests/out.input02.c  tests/out.input13.c   tests/out.input23.c
tests/out.input03.c  tests/out.input14.c   tests/out.input24.c
tests/out.input04.c  tests/out.input15.c   tests/out.input25.c
tests/out.input05.c  tests/out.input16.c   tests/out.input26.c
tests/out.input06.c  tests/out.input17.c   tests/out.input27.c
tests/out.input07.c  tests/out.input18a.c  tests/out.input28.c
tests/out.input08.c  tests/out.input18.c   tests/out.input29.c
tests/out.input09.c  tests/out.input19.c   tests/out.input30.c
tests/out.input10.c  tests/out.input20.c   tests/out.input53.c
tests/out.input11.c  tests/out.input21.c   tests/out.input54.c
```

총 33개의 별도 테스트가 컴파일러의 기능을 검증한다. 현재 우리 컴파일러는 다소 취약한 상태다. 이 테스트들은 컴파일러를 크게 부담하지 않는다. 각 테스트는 몇 줄로 구성된 간단한 예제다. 나중에 컴파일러를 강화하고 더 견고하게 만들기 위해 일부 부하 테스트를 추가할 예정이다.


### 비기능적 테스트

`runtests` 스크립트는 `err` 접두사가 붙은 파일을 찾아 기능 테스트를 수행한다. 현재 다음과 같은 파일들이 있다:

```
tests/err.input31.c  tests/err.input39.c  tests/err.input47.c
tests/err.input32.c  tests/err.input40.c  tests/err.input48.c
tests/err.input33.c  tests/err.input41.c  tests/err.input49.c
tests/err.input34.c  tests/err.input42.c  tests/err.input50.c
tests/err.input35.c  tests/err.input43.c  tests/err.input51.c
tests/err.input36.c  tests/err.input44.c  tests/err.input52.c
tests/err.input37.c  tests/err.input45.c
tests/err.input38.c  tests/err.input46.c
```

이 단계에서 컴파일러의 오류 검사를 위해 22개의 테스트를 만들었다. 컴파일러 내부의 `fatal()` 호출을 찾아 각각을 트리거할 수 있는 작은 입력 파일을 작성했다. 해당 소스 파일을 읽어보고 각 테스트가 어떤 문법 또는 의미론적 오류를 발생시키는지 확인해 보자.


## 다양한 테스트 방식

이 강의는 소프트웨어 개발 방법론에 대한 내용이 아니므로, 테스트에 대해 깊이 다루지는 않는다. 하지만 여러분이 꼭 알아두면 좋을 몇 가지 주제에 대한 링크를 공유한다:

+ [유닛 테스트](https://en.wikipedia.org/wiki/Unit_testing)
+ [테스트 주도 개발](https://en.wikipedia.org/wiki/Test-driven_development)
+ [지속적 통합](https://en.wikipedia.org/wiki/Continuous_integration)
+ [버전 관리](https://en.wikipedia.org/wiki/Version_control)

현재 우리의 컴파일러에는 유닛 테스트를 적용하지 않았다. 주요 이유는 함수들의 API가 매우 유동적이기 때문이다. 전통적인 폭포수 모델을 따르지 않기 때문에, 모든 함수의 최신 API에 맞춰 유닛 테스트를 다시 작성하는 데 너무 많은 시간을 소모하게 된다. 따라서 어느 정도는 위험한 상황에 놓여 있다고 할 수 있다. 아직 발견하지 못한 잠재적 버그가 코드 내에 존재할 가능성이 크다.

그러나 컴파일러가 C 언어를 받아들이는 것처럼 보이지만 실제로는 그렇지 않은 경우, 훨씬 더 많은 버그가 존재할 수밖에 없다. 컴파일러는 [최소 놀람의 원칙](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)을 충족하지 못하고 있다. 일반적인 C 프로그래머가 기대하는 기능을 추가하는 데 시간을 투자해야 할 것이다.


## 그리고 좋은 깜짝 선물

컴파일러를 개발하면서 좋은 깜짝 선물을 하나 발견했다. 한동안 함수 호출 시 인자의 개수와 타입이 함수 프로토타입과 일치하는지 확인하는 코드를 일부러 추가하지 않았다(`expr.c` 파일 참조):

```
  // XXX 각 인자의 타입을 함수 프로토타입과 비교하는 코드 추가 필요
```

한 번에 너무 많은 코드를 추가하고 싶지 않아 이 부분을 일부러 생략했다.

이제 프로토타입을 지원하니, `printf()` 함수를 추가하여 우리가 직접 만든 `printint()`와 `printchar()` 함수를 대체하고 싶었다. 하지만 아직은 불가능하다. `printf()`는 [가변 인자 함수](https://ko.wikipedia.org/wiki/가변_인자_함수)이기 때문이다. 현재 우리 컴파일러는 고정된 개수의 인자만 허용한다.

*그런데* (이것이 바로 깜짝 선물이다), 함수 호출 시 인자 개수를 확인하지 않기 때문에, 프로토타입만 존재한다면 `printf()`에 *임의의* 개수의 인자를 전달할 수 있다. 따라서 현재 이 코드(`tests/input53.c`)가 정상적으로 동작한다:

```c
int printf(char *fmt);

int main()
{
  printf("Hello world, %d\n", 23);
  return(0);
}
```

이것은 정말 좋은 점이다!

하지만 주의할 점이 있다. 위의 `printf()` 프로토타입을 사용하면, `cgcall()`의 정리 코드는 함수가 반환될 때 스택 포인터를 조정하지 않는다. 프로토타입에 정의된 인자 개수가 6개 미만이기 때문이다. 그러나 `printf()`를 10개의 인자로 호출할 수도 있다. 이 경우 4개의 인자를 스택에 푸시하지만, `printf()`가 반환될 때 `cgcall()`은 이 4개의 인자를 정리하지 않는다.


## 결론 및 다음 단계

이번 단계에서는 새로운 컴파일러 코드를 추가하지 않았다. 대신 컴파일러의 오류 검사 기능을 테스트하고 있다. 현재 54개의 회귀 테스트를 통해 새로운 기능을 추가할 때 컴파일러가 깨지지 않도록 보장한다. 또한, 운 좋게도 이제 `printf()`와 같은 외부 고정 인자 수 함수를 사용할 수 있게 되었다.

컴파일러 개발의 다음 단계에서는 다음과 같은 작업을 시도할 계획이다:

+ 외부 전처리기 지원 추가
+ 커맨드라인에 지정된 여러 파일을 컴파일할 수 있도록 기능 확장
+ `-o`, `-c`, `-S` 플래그를 추가해 일반적인 C 컴파일러처럼 작동하도록 개선 [다음 단계](../28_Runtime_Flags/Readme.md)



## 2024년 5월 16일 목요일 오전 10:41:14 (AEST)


6809용 cwj 작업을 시작했다. 간단한 접근 방식을 사용 중이다:
메모리 위치를 레지스터로 활용한다. 나중에 코드 품질을 개선할 계획이다.

또한 컴파일러를 단계별로 분리해야 한다. 하나는 파싱을 통해 AST와 심볼 테이블을 생성하는 단계이고, 다른 하나는 AST와 심볼 테이블을 기반으로 어셈블리 코드를 출력하는 단계다.

모든 것이 64K 안에 들어가길 바란다!

현재는 다음과 같은 코드를 처리할 수 있다:

```
int p=3; int q=4;
int main() { p= p + q; return(0); }
```


2024년 5월 16일 목요일 오후 2시 55분 39초 (AEST)

함수 호출 후 인자를 푸시하고 스택을 정리하는 코드를 추가했다. `printint()` 함수를 호출하면 작동한다. 하지만 컴파일러가 앞에 언더스코어를 추가하지 않는 문제가 있다. 이를 해결해야 한다.


## 2024년 5월 16일 목요일 15:13:22 AEST

`cg.c` 파일 내 모든 `fprintf()`에 `_`를 추가하여 문제를 해결했다.


2024년 5월 16일 목요일 16:49:54 AEST

흠. 작은 정수 리터럴은 문자 리터럴로 처리되지만, `printf()`를 사용하려면 정수 크기로 다뤄야 한다.

아, 맞다. 지난번에 `cwj`를 사용해 6809 컴파일러를 작성하려고 했을 때 이미 이 문제를 해결했었다. 2023년 6월에 작성한 `cloud/Nine_E/Old/Compiler` 코드를 확인해보면 된다!

그 코드 중 일부를 임포트해서 도움을 받았다.


## 2024년 5월 18일 토요일 10:00:44 AEST

`cg.c`를 6809로 변환하는 작업을 마쳤다. 아직 테스트는 진행하지 않았다. 이제 `crt0`로 전환하려고 한다. `crt0`는 stdio를 지원하는데, 모든 테스트가 printf를 사용하기 때문이다. 현재는 `fcc`로 컴파일한 `libc`를 사용하고 있다.


## 2024년 5월 18일 토요일 13:44:47 AEST

아직 완료하지 못했다. long 타입 비교를 깜빡했다. 
이제 input009.c까지 비교를 마쳤다. 결과는 OK이다.


## 2024년 5월 18일 토요일 14:33:42 AEST


현재 input022.c까지 완료: 성공.


## 2024년 5월 18일 토요일 15:11:34 AEST


현재 input054.c까지 진행: 완료


2024년 5월 18일 토요일 15:24:25 AEST

이제 input114.c까지 완료: 성공
와우!


## 2024년 5월 20일 월요일 10:42:50 AEST

테스트 115는 `sizeof` 테스트였는데, 이제 6809와 amd64에서 결과가 다르다. 그래서 이제 input135.c까지 확인 완료했다.

테스트 136은 기본적으로 다음과 같다:

```
result= 3 * add(2,3) - 5 * add(4,6);
```

디버그 출력을 확인해 보니 `add()` 함수는 정상적으로 동작한다. 하지만 첫 번째 `add()` 반환 시 다음과 같은 코드를 볼 수 있다:

```
       lbsr _add
        leas 4,s
        puls d
        std     R0
        std     R1
```

이 코드는 `R0`에 `3*`의 결과인 3이 저장되어 있어서 문제가 된다. 문제를 파악했다. 함수 호출 전에 사용 중인 로컬 레지스터를 스택에 푸시하고, 반환 시 이를 팝하여 복원한다. 그런 다음 함수의 반환 값을 저장한다.

하지만 함수가 반환 값을 보관하기 위해 `Y,D`를 사용하는데, 이 레지스터들이 레지스터 팝으로 인해 파괴되고 있다.

약간 번거로운 방법으로 문제를 해결했다. 이제 input143.c까지 확인 완료했다.


## 2024년 5월 20일 월요일 12:33:02 AEST

이제 모든 테스트를 통과했다! Fuzix의 헤더 파일에서 필요한 부분을 내 헤더 파일로 임포트했다. 이제 컴파일러를 단계별로 분리하는 작업을 시작할 수 있다.

총 8단계로 나눌 계획이다.

1. C 전처리기: `#include`, `#ifdef`, 그리고 전처리 매크로를 처리한다.
2. 렉서: 소스 코드를 읽어 토큰 스트림을 생성한다.
3. 파서: 토큰 스트림을 읽어 심볼 테이블과 AST(Abstract Syntax Tree) 트리 집합을 만든다.
4. AST 최적화기: AST 트리를 읽고 최적화한다.
5. 코드 생성기: 최적화된 AST 트리와 심볼 테이블을 읽어 어셈블리 코드를 생성한다.
6. 피홀 최적화기: 생성된 어셈블리 코드를 개선한다.
7. 어셈블러: 오브젝트 파일을 생성한다.
8. 링커: `crt0.o`, 오브젝트 파일, 그리고 여러 라이브러리를 결합해 실행 파일을 만든다.

디버깅 도구도 함께 개발할 예정이다:
- 토큰 스트림 덤프
- AST 트리 덤프
- 심볼 테이블 덤프

심볼 테이블은 전역 변수, 프로토타입, 구조체/공용체/타입 정의를 포함하는 파일로 구성된다. 그리고 각 함수별 심볼 정보를 담은 섹션들이 뒤따른다.

AST 파일은 여러 개의 독립적인 AST 트리를 포함하며, 각 함수마다 하나씩 존재한다.

이전에 PipeC 작업을 하면서 이미 일부를 구현했기 때문에, 그 코드를 재활용할 수 있다. 좋은 소식이다!


## 2024년 5월 20일 월요일 14:55:17 AEST

PipeC에서 빌린 코드로 스캐너와 detok 프로그램을 분리했다. 파일 이름과 줄 번호가 변경될 때 이를 저장하는 토큰도 추가했다. 이상한 점은 다음과 같은 입력에서:

```
# 0 "scan.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "scan.c"
# 1 "defs.h" 1
# 1 "/tmp/include/stdlib.h" 1 3 4




# 4 "/tmp/include/stdlib.h" 3 4
void exit(int status);
...
```

다음과 같은 토큰을 얻는다는 것이다:

```
1E: void
43: filename "/tmp/include/stdlib.h"
44: linenum 4
36: exit
39: (
20: int
36: status
3A: )
35: ;
```

`void`가 파일 이름보다 먼저 나온다! 이건 `scan()` 재귀 호출 때문이다. 이제 문제를 해결했다. 하지만 줄 번호가 약간 어긋나는 것 같다.


## 2024년 5월 21일 화요일 09:57:34 AEST

stdin에서 토큰 스트림을 읽고 `global_declarations()`를 호출해 컴파일을 시작하는 프론트엔드로 컴파일러를 다시 빌드했다. 현재는 대부분 동일한 어셈블리 코드를 출력하지만, 다음과 같은 파일에서 차이가 발생했다:

```
input021.s가 트리에서 변경됨
input058.s가 트리에서 변경됨
input084.s가 트리에서 변경됨
input089.s가 트리에서 변경됨
input090.s가 트리에서 변경됨
input134.s가 트리에서 변경됨
input140.s가 트리에서 변경됨
input148.s가 트리에서 변경됨
```

이제 살펴볼 것들이 생겼다!

아, 토큰 디코더에서 CHARLIT를 잊고 있었다. 수정했다. 이제 컴파일러가 모든 테스트에서 동일한 어셈블리 파일을 생성한다. 좋다!


## 2024년 5월 21일 화요일 11:08:53 AEST

현재 파싱 단계 작업 중이다. AST를 모두 직렬화하여 stdout으로 출력하는 코드를 작성했고, `detree`를 통해 트리를 출력할 수 있다.

작은 문제가 하나 있는데, 이전에는 파싱 중에 문자열 리터럴과 레이블에 대한 어셈블리를 함께 생성했다. 현재는 이 부분을 주석 처리했지만, 나중에 수정해야 한다.

이제 `gen.c`를 파서에서 분리해야 하고, 심볼 테이블을 덤프하는 방법도 고민해야 한다!


## 2024년 5월 21일 화요일 11:24:39 (AEST)


파서에서 `gen.c`와 `cg.c`를 분리했다. 이 과정에서 이 파일들에 있던 몇 가지 함수를 새로운 `target.c` 파일로 추상화했다. 이제 이 파일은 파서와 코드 생성기가 공유할 수 있다.

이제 심볼 테이블 직렬화에 대해 고민할 차례다.

다음과 같은 상황이 발생할까 봐 걱정된다:

```
<전역 변수>
<함수1>
<전역 변수>
<함수2>
```

만약 `function1` 이전에 전역 심볼 테이블을 덤프하면 두 번째 전역 변수를 놓치게 된다.

심볼을 선언될 때마다 직렬화할 수 있을까? 아니다. 예를 들어, 선언된 요소가 있는 배열처럼 심볼이 수정될 때도 고려해야 한다. 혹시 마지막으로 덤프한 심볼에 대한 포인터를 유지하고, 함수를 완료할 때마다 그 지점부터 덤프하는 방법은 어떨까?


## 2024년 5월 21일 화요일 16:53:06 AEST

좋다, 직렬화 코드의 시작 부분은 완성했다. 이제 역직렬화 코드를 작성하고 버그가 어디에 있는지 확인해야 한다.


## 2024년 5월 22일 수요일 08:28:42 AEST

시리얼라이징 코드를 약간 수정했다. 실행이 되고, 크래시도 발생하지 않는다. 이제 디시리얼라이저를 작성할 차례다!

좋다, 심볼 파일에 무엇이 있는지 확인할 수 있도록 시작 부분을 작성했다. 아직 완벽하지는 않다.

몇 가지 버그를 발견하고 수정했다. 현재는 심볼만 덤프하고 있으며, 심볼 테이블을 재구성하지는 않는다. 심볼 테이블과 AST 디시리얼라이징을 처리할 새로운 프로그램인 코드 생성기가 필요하다. 이 프로그램은 다음과 같은 파일로 구성될 것이다:

```
gen.c misc.c sym.c tree.c type.c cg.c
```


## 2024년 5월 22일 수요일 15:26:54 AEST

이제 코드 생성기를 만들었고, 실제로 동작한다. 문자열 리터럴을 AST에 저장해 나중에 생성할 수 있도록 하는 등 여러 문제를 해결해야 했다. 하지만 이제 첫 번째 테스트 12개를 통과했다! 야호!

현재 input025.c까지 진행 중이다: 통과. 와우. 테스트 26은 실패 중인데, 매개변수나 지역 변수에 프레임 오프셋이 없기 때문이다.


실제로 그들은 작동하지만, 심볼 ID로 찾는 코드에 버그가 있었다.  
이제 input057.c까지 정상적으로 작동한다.


## 2024년 5월 23일 목요일 10:17:43 AEST

아, 나는 전역 변수를 심볼 테이블에서 먼저 직렬화했지만,  
이들이 구조체 타입일 수 있어서 전역 변수보다 구조체, 열거형, 공용체를 먼저 출력해야 한다.  
이제 input073.c까지는 문제없이 처리했다.

이 문제는 리터럴 세그먼트에서 텍스트 세그먼트로 전환하고,  
코드 세그먼트로 돌아가지 않고 데이터 세그먼트로 이동했기 때문이다.  
이제 input088.c까지는 문제없이 처리했다.

문제는 다음과 같다:

```
char *z= "Hello world";
```

원본 코드에서는 문자열 리터럴을 생성하고, 레이블을 얻은 다음  
그 레이블을 사용해 전역 변수 `z`를 생성할 수 있었다.  
하지만 지금은 문자열이 유실되고 `z`가 0으로 초기화된다.  
또한 `char *z=NULL;`도 처리해야 한다.

나는 문자열의 문자들을 심볼의 `initlist`에 넣고 `nelems`를 문자열 길이로 설정하는 것을 고민했지만,  
초기값이 NULL인 경우는 어떻게 처리할까? `nelems`를 0으로 설정할 수 있을까?  
하지만 `char *z= "";`처럼 NULL은 아니지만 문자가 없는 경우는 어떻게 할까?  
흠... 이건 해결책이 아니다.

또한 다음과 같은 경우도 지원해야 한다:

```
char *fred[]= { "Hello", "there", "Warren", "piano", NULL };
```

원래 `cwj` 컴파일러는 이를 지원했다.  
아마도 문자열 리터럴을 위한 또 다른 심볼 테이블이 필요할지도 모른다.  
그러면 이를 직렬화하고, 백엔드에서 문자열과 레이블을 생성할 수 있을까?


## 2024년 5월 24일 금요일 06:32:06 AEST

내 솔루션은 문자열 리터럴 심볼 테이블을 사용할 것이다. `char *` 전역 변수의 초기화 리스트에는 심볼 ID가 심볼 파일에 저장된다. 생성기에서 심볼 테이블을 로드한다. 문자열 리터럴에 대한 어셈블리 코드를 생성하고 레이블을 만든다. 이후 `char *` 전역 변수를 만나면 초기화 리스트의 심볼 ID를 해당 레이블로 대체한다.


## 2024년 5월 24일 금요일 09:42:04 AEST

구현 완료. 작동하는 것 같다. 현재 input098.c까지 진행했고, 문제없다.  
사실 문자열 배열을 잊고 있었다. 이제 수정했고, input129.c까지 진행했으며, 문제없다.  

다음과 같은 문자열 리터럴을 표현식으로 처리하지 않는 것 같다:  

```
"Hello " "world" "\n"
```


## 2024년 5월 25일 토요일 07:49:15 AEST

`primary()` 함수에서 `litlast`를 올바르게 증가시키지 않아 문제가 발생했었다. 이제 수정했고 모든 테스트를 통과한다! 일부 오류 보고가 정확하지 않지만, 큰 문제는 없다. 드디어 중요한 단계를 달성했다.

QBE 백엔드를 다시 도입할 생각이다. 두 개의 백엔드를 사용하면 더 많은 버그를 발견하는 데 도움이 될 것이다. 기존 백엔드를 재구성된 컴파일러에 맞게 수정하는 데 큰 어려움이 없기를 바란다. 또한 모든 단계를 올바르게 실행할 수 있도록 `wcc.c`라는 프론트엔드를 작성해야 한다.

다음은 할 일 목록이다:

- QBE 백엔드를 생성하고 정상적으로 작동하도록 한다.
- 각 백엔드로 컴파일러를 빌드할 수 있도록 설정한다.
- 최종 설치 위치를 `/opt/wcc`로 변경한다. (`fcc`와 유사하게)
- 트리 최적화를 별도의 단계로 분리하고, `SubC`의 일부 최적화 기법을 추가한다.
- `fcc`에서 피홀 최적화기를 가져와 별도의 단계로 만든다.
- 6809 코드 생성을 개선한다.
- `register`, `volatile`, `unsigned`, `float`, `double`, `const` 키워드를 추가한다.
- 마지막으로 6809 libc를 컴파일하기 시작하고, 추가로 필요한 언어 기능을 확인한다.

이 작업은 상당한 시간이 소요될 것이다!


## 2024년 5월 25일 토요일 08:30:19 AEST

파일 이름을 재정렬하고 Makefile을 수정하여 이제 "parse"와 "gen"에 대한 6809 전용 실행 파일을 만들었다. QBE 백엔드를 임포트했다. 이전 컴파일러는 타입을 `cg` 함수에 전달했지만, 새 컴파일러는 이를 수행하지 않는다. 6809와 QBE 외의 다른 백엔드에도 도움이 될 것 같아 이전 방식으로 돌아갈 수 있을 것 같다.

거의 모든 작업을 마쳤다. 새 코드는 함수 인자를 하나의 `cg` 함수로 푸시하고, 다른 `cg` 함수로 호출을 수행한다. 기존 QBE 백엔드는 이를 하나의 함수에서 처리한다. 따라서 이를 분리해야 한다. 하지만 테스트를 제외하면 이것이 남은 유일한 작업이다.

사실 QBE의 경우 변경된 이유는 QBE가 호출을 먼저 하고 인자 목록을 나중에 처리하기 때문이다. 아마도 6809 버전도 이 방식을 사용하도록 변경할 수 있을 것이다.


2024년 5월 25일 토요일 17:14:01 AEST

변경 사항을 적용하고 그 과정에서 버그를 몇 가지 수정했다. 테스트 `input135.c`까지 진행했으며, 이제 거의 완료 단계에 이르렀다!

모든 테스트가 통과된다. 함수 호출 전에 레지스터를 비워야 한다는 것을 잊고 있었다.


## 2024년 5월 26일 일요일 08:09:25 AEST


현재 `cgqbe.c` 작업 중이다. 거의 완료되었으며, QBE 파일에 `cgswitch()`를 작성해야 한다. 이전 QBE 버전에서는 `gen.c`에 있었다.


## 2024년 5월 26일 일요일 11:01:30 AEST

`cgqbe.c`에 `cgswitch()`를 작성했고, 코드 이후에 문자열 리터럴을 출력하도록 지연시키는 코드도 추가했다. 현재 `input010.c`까지 테스트를 통과했고, 작은 이슈를 해결한 후 `input057.c`까지 테스트를 통과했다.


## 2024년 5월 26일 일요일 11:42:03 AEST

구조체 멤버에 접근할 때 사용되는 INTLIT의 크기는 QBE에서는 long 타입이고, 6809에서는 int 타입이다. 또 다른 타겟 함수가 추가되었다.

현재 endianness 테스트인 input062까지 진행했다. 당분간 이 테스트를 제거해야 할 것 같다. 이제 첫 번째 switch 테스트인 input074.c까지 진행했는데, 현재 실패하고 있다.


## 2024년 5월 26일 일요일 12:19:05 AEST

`cgqbe.c` 파일의 버그를 수정하고 스위치 생성 코드를 재구성했다. 이제 6809와 QBE 양쪽에서 모든 테스트가 통과한다. 두 가지 백엔드를 가진 컴파일러를 완성했다!


## 2024년 5월 28일 화요일 09:23:11 AEST

프론트엔드 `wcc`를 작성했고, 이제 QBE와 6809에서 모두 동작한다. 설치 경로를 `/opt/wcc`로 옮겼다. `runtests`와 `onetest`를 다시 작성했다. 현재 어떤 이유에서인지 QBE 테스트 중 하나가 실패하고 있다. 아!


## 2024년 5월 28일 화요일 오전 10시 34분 14초 (AEST)

아, `include`에서 6809와 QBE 간에 차이가 있는 부분이 몇 가지 있네요. 그래서 이제 각 플랫폼별로 `include` 디렉터리를 두 개로 나눴습니다. 이제 테스트를 다시 통과합니다!


## 2024년 5월 28일 화요일 11:12:20 AEST

방금 전 프론트엔드에 peephole 최적화 기능을 추가했다. 첫 번째 규칙이 실패했다! 이유가 궁금하다. 아, 규칙을 잘못 작성했구나, 수정했다. 이제 6809 테스트가 peephole 최적화 기능과 함께 통과된다.


## 2024년 5월 28일 화요일 11:50:54 AEST

AST 최적화기를 파서에서 독립적인 프로그램으로 분리했다. 아직 파서 내에 남아 있지만 작동은 잘 되는 것 같다. 그런데 파서에서 최적화기를 완전히 제거하려고 시도했을 때 문제가 발생했다. 현재 코드는 다음과 같다:

```
decl.c:  // 표현식을 파싱하고 결과 AST 트리를 최적화
decl.c:  tree = optimise(binexpr(0));
```

이 코드를 `tree=binexpr(0);`로 변경하자, 테스트 112에서 `Cannot initialise globals with a general expression` 오류가 발생했다. 문제는 이 코드 때문이다:

```
int x= 10 + 6;
```

이런 표현식은 파싱 시점에 해결되어야 한다. 아마도 파서 내에 일부 트리 최적화 코드를 남겨둘 필요가 있을지도 모르겠다. 아직 확실하지 않다.

현재 amd64 바이너리 크기를 살펴보면:

```
   text	   data	    bss	    dec	    hex	filename
   9973	   1008	   4200	  15181	   3b4d	wcc
   8902	    676	    680	  10258	   2812	cscan
  13695	    752	   1328	  15775	   3d9f	cpeep
  38772	   1880	    936	  41588	   a274	cparse6809
  47917	   1456	   1064	  50437	   c505	cgen6809
  38796	   1880	    936	  41612	   a28c	cparseqbe
  35088	   1264	    968	  37320	   91c8	cgenqbe
```

6809 생성기가 파서보다 다이어트가 더 필요한 것 같다.

컴파일러 코드 자체를 6809 컴파일러로 컴파일해 보기로 했다:

```
$ for i in *.c; do wcc -c -m 6809 $i; done
Expecting a primary expression, got token:void on line 27 of cgen.c
Type mismatch: literal vs. variable on line 20 of cgqbe.c
Unrecognised character \
Unrecognised character \
Unrecognised character \
Unrecognised character \
Unrecognised character \
Type mismatch: literal vs. variable on line 35 of cpeep.c
Expecting a primary expression, got token:void on line 23 of ctreeopt.c
Expecting a primary expression, got token:void on line 25 of desym.c
Unrecognised character \
Unrecognised character \
Expecting a primary expression, got token:void on line 22 of detok.c
Expecting a primary expression, got token:void on line 25 of detree.c
Expecting a primary expression, got token:void on line 27 of parse.c
& operator must be followed by an identifier on line 604 of scan.c
Unrecognised character \
Unknown variable or function:s on line 143 of tree.c
Expecting a primary expression, got token:} on line 17 of wcc.h

$ ls *.o
 71859 May 28 13:30 cg6809.o
   688 May 28 13:26 crt0.o
 33009 May 28 13:30 decl.o
 29981 May 28 13:30 expr.o
 22461 May 28 13:30 gen.o
  1342 May 28 13:30 misc.o
  3639 May 28 13:30 opt.o
 12545 May 28 13:30 stmt.o
 22795 May 28 13:30 sym.o
  1053 May 28 13:30 targ6809.o
  1393 May 28 13:30 targqbe.o
   126 May 28 13:30 tstring.o
  6732 May 28 13:30 types.o
```

흥미롭다. 스캐너/파서가 왜 죽는지 알아내야 한다. 또한 `cg6809.o` 파일이 너무 크다!


## 2024년 5월 28일 화요일 14:03:18 AEST

문자열 리터럴 내부에 `\"`가 포함된 경우 스캐너가 처리하지 못하는 버그를 수정했다. 현재 다음과 같은 오류 메시지가 나타난다:

```
cgen.c 파일의 27번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
cgqbe.c 파일의 20번째 줄에서 리터럴과 변수 간 타입 불일치가 발생했습니다.
cpeep.c 파일의 35번째 줄에서 리터럴과 변수 간 타입 불일치가 발생했습니다.
ctreeopt.c 파일의 23번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
desym.c 파일의 25번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
detok.c 파일의 22번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
detree.c 파일의 25번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
parse.c 파일의 27번째 줄에서 기본 표현식이 필요하지만, void 토큰을 받았습니다.
scan.c 파일의 612번째 줄에서 & 연산자 뒤에 식별자가 필요합니다.
wcc.h 파일의 17번째 줄에서 기본 표현식이 필요하지만, } 토큰을 받았습니다.
```

문제는 라인 번호가 정확하지 않다는 점이다 :-(

아, 버그의 원인은 내 컴파일러가 `return <expression>;` 형식을 허용하지 않고, `return(<expression>);` 형식만 지원하기 때문이다.


## 2024년 5월 28일 화요일 14:26:43 AEST

문제를 해결하려고 시도했지만 쉽지 않다. 다음과 같은 경우를 처리할 수 있어야 한다:

```
    return( (void *)0 );
그리고
    return (void *)0 ;
```

일단 내 코드만 수정하기로 했다. 작업 완료.


## 2024년 5월 28일 화요일 14:47:59 AEST

`& operator must be followed by an identifier` 오류는 아직 다음과 같은 코드를 사용할 수 없기 때문이다:

```
mary= &fred.x;
```

이제 다음 명령을 실행해 보자:

```
$ for i in *.c; do wcc -S -m6809 $i; done
cgqbe.c 파일 20번째 줄에서 타입 불일치: 리터럴 vs. 변수
cpeep.c 파일 35번째 줄에서 타입 불일치: 리터럴 vs. 변수
```


## 2024년 5월 28일 화요일 15:14:56 AEST

이제 소스 파일이 하나만 남았다:

```
$ for i in *.c; do wcc -c -m6809 $i; done
cpeep.c 파일의 95번째 줄에서 바이너리 표현식에 호환되지 않는 타입이 발견됨

$ ls *.o
 71859 5월 28 15:14 cg6809.o
 12497 5월 28 15:14 cgen.o
 37028 5월 28 15:14 cgqbe.o
   688 5월 28 15:13 crt0.o
  8427 5월 28 15:14 ctreeopt.o
 33009 5월 28 15:14 decl.o
  4832 5월 28 15:14 desym.o
  3093 5월 28 15:14 detok.o
  3336 5월 28 15:14 detree.o
 29981 5월 28 15:14 expr.o
 22461 5월 28 15:14 gen.o
  1342 5월 28 15:14 misc.o
  3639 5월 28 15:14 opt.o
  8853 5월 28 15:14 parse.o
 23115 5월 28 15:14 scan.o
 12545 5월 28 15:14 stmt.o
 22795 5월 28 15:14 sym.o
  1053 5월 28 15:14 targ6809.o
  1393 5월 28 15:14 targqbe.o
  9685 5월 28 15:14 tree.o
   126 5월 28 15:14 tstring.o
  6732 5월 28 15:14 types.o
 19125 5월 28 15:14 wcc.o
```

`fcc`로 코드를 컴파일하기로 결정했다. 결과는 다음과 같다:

```
$ ls *.o
 35320 5월 28 15:21 cg6809.o
  5587 5월 28 15:21 cgen.o
 16838 5월 28 15:21 cgqbe.o
 17642 5월 28 15:21 cpeep.o
  3965 5월 28 15:21 ctreeopt.o
 14467 5월 28 15:21 decl.o
  2308 5월 28 15:21 desym.o
  1569 5월 28 15:21 detok.o
  1520 5월 28 15:21 detree.o
 11534 5월 28 15:21 expr.o
  8617 5월 28 15:21 gen.o
   742 5월 28 15:21 misc.o
  1521 5월 28 15:21 opt.o
  4228 5월 28 15:21 parse.o
 11414 5월 28 15:21 scan.o
  5617 5월 28 15:21 stmt.o
 10648 5월 28 15:21 sym.o
   761 5월 28 15:21 targ6809.o
   903 5월 28 15:21 targqbe.o
  5152 5월 28 15:21 tree.o
  2012 5월 28 15:21 tstring.o
  2515 5월 28 15:21 types.o
 10225 5월 28 15:21 wcc.o
```

크기가 약 절반으로 줄었다 :-)

재미 삼아 `fcc` 컴파일러를 사용해 `cgen6809`를 빌드해봤다.
크기는 `8A91 B __end`이다. 그리고 `cparse6809`는 `765A B __end`이다 :-)

amd64 버전의 크기는 다음과 같다:

```
$ size cparse6809 cgen6809
   text	   data	    bss	    dec	    hex	filename
  38793	   1880	    936	  41609	   a289	cparse6809
  47942	   1456	   1064	  50462	   c51e	cgen6809
```

따라서 코드 생성기가 `fcc`만큼 좋아진다면 크로스 컴파일이 가능할 것이다. 할 일이 많다.


## 2024년 5월 28일 화요일 16:34:46 AEST

코드 개선에 대해 브레인스토밍 중이다. QBE를 지원하기 위해 `gen.c`에 있는 레지스터 개념을 유지해야 한다. 다음과 같은 아이디어를 고려 중이다:

- `cg6809.c`가 Location 구조체 배열을 유지한다.
- 배열의 인덱스가 "레지스터" 역할을 한다.
- 전역 변수 `d_free`를 사용해 D 레지스터를 로드할 수 있는 시점을 파악한다.
- Location 구조체는 B/D/Y 연산의 피연산자로 사용할 수 있는 충분한 정보를 담는다.

현재 코드를 살펴보자:

```
ldd #0\n");
ldd 0,x\n");
ldd #1\n");
ldd 2,x\n");
ldd #%d\n", offset);
ldd #%d\n", offset & 0xffff);
ldd #%d\n", val & 0xff);
ldd #%d\n", value & 0xffff);
ldd #%d\n", (value>>16) & 0xffff);
ldd %d,u\n", 2+sym->st_posn);
ldd %d,u\n", sym->st_posn);
ldd #L%d\n", label);
ldd _%s+2\n", sym->name);
ldd #_%s\n", sym->name);
ldd _%s\n", sym->name);
```

다음 정보를 기록해야 한다:

- 위치 정보가 포함된 심볼 이름
- 스택 프레임의 오프셋
- 상수 값
- 레이블 ID
- 심볼 이름의 주소

다음과 같은 구조를 생각해볼 수 있다:

```
enum {
  L_SYMBOL,
  L_LOCAL,
  L_CONST,
  L_LABEL,
  L_SYMADDR,
  L_DREG
};

struct Location {
  int type;
  char *name;
  int intval;		// 오프셋, 상수 값, 레이블 ID
};
```

Location을 출력하는 함수도 필요하다. 레지스터 할당과 해제를 유지하고, 모든 레지스터를 해제할 때 `d_free`를 true로 설정한다. 레지스터 스필링은 더 간단해질 것이다. 레지스터를 할당하는 `cg` 함수는 이제 Location 엘리먼트를 할당하고 그 값을 채운다.

그런 다음 다음과 같은 코드를 작성할 수 있다:

```
int cgadd(int r1, int r2, int type) {
  int size= cgprimsize(type);

  // r1이 이미 L_DREG이면 아무 작업도 하지 않는다.
  // 그렇지 않으면 기존 r1 위치를 D에 로드하고 L_DREG으로 표시한다.
  // 이 작업은 B, D 또는 Y,D를 로드할 수 있다.
  load_d(r1, size);	

  switch (size) {
    case 1: fprintf(Outfile, "\taddb"); printlocation(r2,0); break;
    case 2: fprintf(Outfile, "\taddd"); printlocation(r2,0); break;
    case 4: fprintf(Outfile, "\taddd"); printlocation(r2,2);
	    // Y를 업데이트하는 코드 추가
  }
  return(r1);
}
```


## 2024년 5월 29일 수요일 09:01:58 AEST

어제 밤에 작업을 시작했고 꽤 좋은 결과를 얻었다. 몇 가지 개선점을 생각해 냈고, 지금 바로 적용해 보려고 한다.

지금까지 다음과 같은 코드를 컴파일할 수 있다:

```
int x, y, z;

int main() {
  int result; x=2; y=3; z=4;
  result= x + y + z; printf("%d\n", result); return(result);
}
```

이 코드를 어셈블리로 변환하면 다음과 같다(일부 생략):

```
_main:
	pshs u
	tfr s,u
	leas -2,s
	ldd #2
	std _x
	ldd #3
	std _y
	ldd #4
	std _z
	ldd _x+0
	addd _y+0
	addd _z+0
	std -2,u

	ldd -2,u	; 이 부분은 개선할 여지가 있다!
	pshs d
	ldd #L2
	pshs d
	lbsr _printf
	leas 4,s

	ldd -2,u
	leas 2,s
	puls u
	rts
```

R0, R1 등을 거치는 것보다 훨씬 깔끔한 결과물이다.


2024년 5월 29일 수요일 12:48:15 AEST

진도가 더딘 상태다. 현재 input009.c까지 진행했고, 문제는 없다. 이제 input010.c도 확인 완료했다.

long 타입을 다루는 게 쉽지 않을 것 같다는 걸 깨달았다.


2024년 5월 30일 목요일 오후 12시 51분 23초 (AEST)

Fait가 오늘 Liz와 함께 떠났다 :-(
현재 input026.c까지 진행 완료: OK


2024년 5월 30일 목요일 13:19:48 AEST

현재 input090.c까지 완료.


2024년 5월 30일 목요일 14:05:55 AEST

현재 input139.c까지 완료: 정상


## 2024년 5월 30일 목요일 15:09:40 AEST

테스트가 모두 통과했다. 정말 대단하다.

재미로 컴파일러 소스 코드를 자기 자신으로 컴파일해봤다:

```
$ for i in *.c; do wcc -c -m6809 $i; done
Incompatible argument type in function call on line 58 of cg6809.c
Out of locations in cgalloclocn on line 241 of (null)
Incompatible types in binary expression on line 95 of cpeep.c
Out of locations in cgalloclocn on line 284 of (null)
Out of locations in cgalloclocn on line 41 of (null)
child phase didn't Exit
Out of locations in cgalloclocn on line 69 of (null)
child phase didn't Exit
Out of locations in cgalloclocn on line 48 of (null)
```

몇 파일은 컴파일되지 않았지만, 이전 코드 생성기와 새로운 코드 생성기의 크기 변화를 확인해보았다:

```
Old Size              New Size		Fcc Size
------------------------------------------------
 71859 cg6809.o
 12497 cgen.o		7890		5587
 37028 cgqbe.o
   688 crt0.o
  8427 ctreeopt.o	5530		3965
 33009 decl.o
  4832 desym.o		3464		2308
  3093 detok.o		2242		1569
  3336 detree.o		2313		1520
 29981 expr.o
 22461 gen.o		11608		8617
  1342 misc.o		834		742
  3639 opt.o		2132		1521
  8853 parse.o		5897		4228
 23115 scan.o
 12545 stmt.o
 22795 sym.o
  1053 targ6809.o	806		761
  1393 targqbe.o	1062		903
  9685 tree.o
   126 tstring.o	126		2012
  6732 types.o		4385		2515
 19125 wcc.o		13238		10225
```

상당히 개선된 것 같다. 하지만 `fcc`가 여전히 훨씬 나으므로 더 많은 작업이 필요하다.


## 2024년 5월 30일 목요일 17:43:35 AEST

`gen.c`에서 버그를 발견했다. QBE/amd64에서 포인터와 정수의 크기가 다르기 때문에 발생한 문제다. `Locn[l].type = 23;` 이 줄이 실패한 이유는 `l`이 정수로 구조체 크기를 곱한 후 `Locn`의 주소에 더해졌기 때문이다( long vs. word). 이를 수정했다.

그런데 아직 완전히 해결되지 않았다. 나중에 long을 long으로 확장하려고 할 때 QBE가 이를 처리하지 못해 문제가 발생한다(이유는 모르겠다 :-)).


2024년 5월 30일 목요일 20:02:32 AEST

몇 가지 문제를 수정했습니다. 현재 상태는 다음과 같습니다:

```
$ for i in *.c; do wcc -c $i; done
cpeep.c 파일의 95번째 줄에서 이진 표현식의 타입이 호환되지 않음
qbe:decl.c_qbe:2530: copy에서 첫 번째 피연산자 %class의 타입이 유효하지 않음
```

`cpeep.c`의 문제는 포인터를 빼고 그 결과를 int에 할당하는 부분에서 발생합니다:

```
int main() {
  char *a, *b;
  int x;
  x= a - b;
  return(x);
}
```

다른 문제는 `class`가 주소를 가져야 하는 것으로 표시되어야 하지만 그렇지 않은 것 같습니다. 아직 정확한 이유는 파악하지 못했습니다.


## 2024년 5월 31일 금요일 08:56:58 AEST


컴파일러를 더 가볍게 만들기 위한 아이디어들.

 - 더 많은 AST 최적화를 적용한다. 예를 들어, 한쪽이 정수 리터럴 0인 덧셈/뺄셈/곱셈 연산을 최적화한다.  
   하지만 #ifdef를 사용해 생성 단계의 최적화와 파싱 단계의 최적화를 분리한다.  
   또한, 교환 법칙이 성립하는 연산에서 왼쪽/오른쪽을 바꿔서 D 레지스터가 이미 값을 가지고 있도록 한다.  
 - 가능한 경우 free()를 사용한다.  
 - AST 연산이 트리의 최상단에 위치할 때, 결과를 D 레지스터에 로드하지 않는다. 예: `i++;`.  
 - 변수를 사용하지 않는다. 예: `int primtype= ...; switch(primtype)`  
 - 어떤 정수들을 문자로 바꿀 수 있는지 확인한다.  
 - 현재 P_POINTER가 P_INT와 동일하더라도 유지해야 한다. `unsigned`를 지원할 때, 포인터는 부호 없는 정수로 처리되지만, 정수는 부호 있는 정수일 수 있다.  
 - 중복된 문자열 리터럴을 찾아 전역 변수로 만들어 한 번만 선언되도록 한다.  
 - 6809 코드 개선이 확실히 필요하다.  
 - 더 많은 피홀 최적화를 적용한다.  
 - 코드 커버리지 분석을 수행한다?  
 - 스택에 있는 임시 변수들을 이동시킨다. 각 함수의 시작/끝에서 `leas` 명령어를 사용해 스택을 조정한다. 이를 통해 임시 변수를 스택에 저장할 필요를 줄인다.


## 2024년 5월 31일 금요일 09:22:30 AEST

포인터 간의 덧셈과 뺄셈을 수행할 수 있어야 한다. 하지만 결과를 스케일링 해제해야 한다. 예를 들어 (amd64 gcc에서):

```
#include <stdio.h>
int main() {
  int a, b; int *x, *y; long z;
  a=2; b=3; x=&a; y=&b;
  printf("x is %lx y is %lx\n", x, y);
  z= x - y;     printf("z is %lx\n", z);
  z= x - y + 1; printf("z is %lx\n", z);
  return(0);
}
```

이 코드는 다음과 같은 결과를 출력한다:

```
x is 7ffc7b25b244 y is 7ffc7b25b240
z is 1
z is 2
```

x와 y는 4바이트 떨어져 있지만, 뺄셈 결과는 1이다. 그런데 `+1`은 long 타입의 덧셈으로 처리된다. 흠.

A_DESCALE AST 연산과 `cgshrconst()` 함수가 필요할 것 같다.


## 2024년 5월 31일 금요일 14:43:58 AEST

`decl.c` 파일의 QBE `%class` 문제를 살펴보고 있다. 문제가 되는 함수 `declaration_list()`만 포함된 새 파일 `d.c`를 만들었다. 이 파일은 아무 문제 없이 컴파일된다! 심지어 `decl.c`와 동일한 함수 프로토타입을 넣어도 문제가 없다. 흥미롭다.

`sym` 파일에서 `class`는 다음과 같다:

```
{name = 0x555555566860 "class", id = 575, type = 48, ctype = 0x0, 
  ctypeid = 0, stype = 0, class = 3, size = 4, nelems = 1, st_hasaddr = 0, 
  st_posn = 0, initlist = 0x0, next = 0x555555566880, member = 0x0}
```

여기서 `st_hasaddr`는 0으로 설정되어 있다. 하지만 파서를 `gdb`로 실행하면서 이 값이 1로 설정되는 것을 확인했다. 그런데 어떻게 다시 0으로 리셋되었는지 잘 모르겠다.

다시 파서를 실행해 보았다. 아니, 아무것도 이 값을 0으로 리셋하지 않는다. `desym`을 수정해 `hasaddr`를 표시하도록 했는데, 어떤 파라미터도 이 값을 1로 설정하지 않았다.

파서에 `printf`를 추가해 보니 다음과 같은 출력을 확인했다:

```
In declaration_list set hasaddr 1 on class stype 0 class 3
```

이는 변수(0) 파라미터(3)를 의미한다. 그렇다면 왜 이 값이 제대로 덤프되지 않는 걸까?


## 2024년 6월 1일 토요일 10:03:16 AEST

문제를 파악한 것 같다. `hasaddr`가 설정되기 전에 심볼이 덤프되고 있다:

```
$ wcc -S -X -v decl.c
...
Serialising class stype 0 class 3 id 575 hasaddr 0
...
In declaration_list set hasaddr 1 on class stype 0 class 3 id 575
```

프로토타입에 언급된 `class`가 `hasaddr 0` 상태일 때 직렬화되는 것 같다. 나중에 `hasaddr`가 설정되지만, 심볼이 이미 직렬화된 후라서 변경 사항이 반영되지 않는다. 문제가 발생한다.

함수가 프로토타입으로만 표현된 함수를 호출할 수 있기 때문에 심볼 테이블에서 프로토타입을 출력해야 한다.

현재 해결책은 `cgqbe.c`에서 모든 매개변수를 주소를 가진 것으로 표시하는 것이다. 프로토타입에 나타나는 매개변수만 처리하면 된다. 모든 테스트는 여전히 통과한다. 이제 다음과 같은 오류만 남았다:

```
Incompatible types in binary expression on line 95 of cpeep.c
```

하지만 QBE 버전의 컴파일러에서는 이를 사용하지 않으므로, QBE 컴파일러를 스스로 컴파일해볼 수 있을 것이다.


## 2024년 6월 1일 토요일 13:26:27 AEST

`wcc` 프론트엔드가 작동하지 않지만, 시스템 호출 문제로 의심된다. `cscan`은 정상적으로 작동한다.

파서가 제대로 동작하지 않아 다음과 같은 결과를 확인했다:

```
$ ls *ast ; ls *sym
 235105 Jun  1 13:28 decl.c_ast
 216017 Jun  1 13:28 fred_ast
 59119 Jun  1 13:28 decl.c_sym
 56399 Jun  1 13:28 fred_sym
```

출력 결과에서 "fred"로 시작하는 파일들은 `wcc`로 컴파일된 `cparseqbe`에서 생성된 것이다.

구조체 크기가 다르다는 것을 발견했다. `wcc`와 `gcc`로 컴파일된 `detree`를 실행해 차이점을 확인했다:

```
<       STRLIT rval \"Can't have static/extern in a typedef declaration\"
---
>       STRLIT rval "Can't have static/extern in a typedef declaration"
1641c1641
<       STRLIT rval \"redefinition of typedef\"
---
>       STRLIT rval "redefinition of typedef"
1669c1669
<       STRLIT rval \"unknown type\"
---
>       STRLIT rval "unknown type"
...
```

이것이 문제인지 확실하지 않다. 덤프된 심볼 테이블의 경우, `desymqbe`는 동일한 출력을 생성하지만 `wcc`로 컴파일된 버전은 마지막에 세그먼트 오류를 발생시킨다. 

`.s` 파일을 생성하고 `cc -o desymqbe -g -Wall *.s`를 실행했지만, 어디서 충돌이 발생했는지 알 수 없었다.

다음과 같은 방법으로 테스트를 진행했다:

```
$ ./wscan < decl.c_cpp > fred.tok
$ ./wparseqbe fred_sym fred_ast < fred.tok
$ ./wgenqbe fred_sym fred_ast > fred_qbe
```

이 과정에서는 충돌이 발생하지 않았다. `diff`를 실행해 차이점을 확인했다:

```
$ paste fred_qbe decl.c_qbe

@L2     		  @L2
  %ctype =l alloc8 1      %ctype =l alloc8 1
  storel %.pctype, %ctype storel %.pctype, %ctype
  %class =l alloc8 1      %class =l alloc8 1
  storel %.pclass, %class storel %.pclass, %class
  %.t1 =w copy 0          %.t1 =w copy 0
  %type =w copy %.t1      %type =w copy %.t1
  %.t2 =w copy 1          %.t2 =w copy 1
  %exstatic =w copy %.t2  %exstatic =w copy %.t2
  %.t3 =w copy 0          %.t3 =w copy 0
  %.t5 =l extsw %.t3      %.t5 =l extsw %.t3
  %.t6 =l loadl $ctype    %.t6 =l loadl %ctype	<===
  storel %.t5, %.t6       storel %.t5, %.t6
```

`ctype`은 로컬 변수이므로 항상 `%ctype`이어야 하지만, 컴파일러가 전역 변수인 `$ctype`을 생성하고 있다.


`desymqbe`로 돌아가서, 몇 가지 `printf`를 추가했다.

```
int main() {
  struct symtable sym;

  while (1) {
printf("A\n");
    if (deserialiseSym(&sym, stdin)== -1) break;
printf("B\n");
    dumpsym(&sym, 0);
printf("C\n");
  }
printf("D\n");
  return(0);
}
```

그리고 우리 컴파일러로 컴파일한 버전을 실행했을 때 결과는 다음과 같았다.

```
C
A
D
Segmentation fault
```

`return(0)`에서 세그먼테이션 폴트가 발생하는 것 같다. 이를 `exit(0)`로 변경하면 문제가 해결된다. 일단은 이렇게 처리하자.


## 2024년 6월 1일 토요일 15:29:20 AEST

요약: 우리는 여러 패스를 연결하고 실행할 수 있지만, 프론트엔드 `wcc`는 실행할 수 없다.
`decl.c`를 어셈블리로 컴파일할 때 다음과 같은 문제가 발생했다:

- 토큰 파일은 동일하다.
- `desym` 관점에서 덤프된 심볼 파일도 동일하다.
- AST 파일의 문자열 리터럴이 다르다. 자체 컴파일된 버전은 각 `"` 앞에 `\`를 추가한다.
- `cgenqbe`가 여기저기서 지역 변수를 전역 변수처럼 처리하는 것 같다.


## 2024년 6월 2일 일요일 오전 9:42:19 AEST

심볼 덤프 코드를 `sym.c`에서 `desym.c`로 옮겼고, 모든 내용을 출력하기 위해 더 많은 코드를 추가했다.

두 종류의 바이너리 파일에 적합한 이름이 필요하다. G 바이너리는 `gcc`로 컴파일했고, W 바이너리는 `wcc`로 컴파일했다.

G와 W 토크나이저는 동일한 토큰 스트림을 생성한다.  
G와 W 파서는 서로 다른 심볼 테이블 파일을 생성하지만, 이 파일들에 G와 W `desym`을 실행하면 동일한 결과를 얻는다.

G와 W `detree` 출력에서 STRLIT 문제가 여전히 존재한다. AST 파일은 서로 다르지만, `hd` 명령어로 확인한 결과는 다음과 같다:

```
G 버전
00030440  00 00 00 00 44 75 70 6c  69 63 61 74 65 20 73 74  |....Duplicate st|
00030450  72 75 63 74 2f 75 6e 69  6f 6e 20 6d 65 6d 62 65  |ruct/union membe|
00030460  72 20 64 65 63 6c 61 72  61 74 69 6f 6e 00 1c 00  |r declaration...|

W 버전
00034880  00 00 00 00 44 75 70 6c  69 63 61 74 65 20 73 74  |....Duplicate st|
00034890  72 75 63 74 2f 75 6e 69  6f 6e 20 6d 65 6d 62 65  |ruct/union membe|
000348a0  72 20 64 65 63 6c 61 72  61 74 69 6f 6e 00 1c 00  |r declaration...|
```

리터럴은 동일하다. 단지 `detree`가 이를 다르게 출력하는 것 같다.


## Sun 02 Jun 2024 10:03:32 AEST

흠. `cgqbe.c` 파일에 코드를 추가해서 로컬 `%` 또는 글로벌 `$` 문자를 사용하는 결정 과정을 출력해 봤다. 결과가 잘못 나온다:

```
< loadvar ctype 3 -> %
< loadvar exstatic 2 -> %
---
> loadvar ctype 3 -> $
> loadvar exstatic 2 -> $

< loadvar class 3 -> %
< loadvar class 3 -> %
---
> loadvar class 3 -> $
> loadvar class 3 -> $
```

이를 위한 코드는 다음과 같다:

```
  // 심볼에 적합한 QBE 접두사를 가져옴
  qbeprefix = ((sym->class == C_GLOBAL) || (sym->class == C_STATIC) ||
               (sym->class == C_EXTERN)) ? (char)'$' : (char)'%';
```

문제는 LOGOR 또는 삼항 연산자 또는 둘의 조합 때문일 수 있다! 테스트 프로그램을 작성해 모든 class 값에 대해 `'$'`만 나오고 `'%'`는 나오지 않는다. 흠.

`acwj 63` 컴파일러의 QBE 출력과 비교 중이다. LOGOR 코드는 동일하다. 삼항 연산자 코드는 다르다.


## 2024년 6월 3일 월요일 13:47:00 AEST

임의의 표현식에서 불리언 값을 얻어야 하는 경우와, 비교 후 점프하거나 불리언 값에 따라 점프해야 하는 경우를 충분히 고려하지 못했다는 사실을 깨달았다. 문제를 해결한 방법은 다음과 같다.

이미 IF와 WHILE 문에서 다음과 같은 코드를 사용하고 있었다.

```
  // 다음 표현식을 파싱하고
  // 뒤따르는 ')'를 처리한다. 비교 연산이 아닌 경우
  // 불리언 값으로 강제 변환한다.
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    condAST =
      mkastunary(A_TOBOOL, condAST->type, condAST->ctype, condAST, NULL, 0);
  rparen();
```

그리고 `gen.c`와 `cg.c`에서는 다음과 같이 처리했다.

```
  case A_TOBOOL:
    // 부모 AST 노드가 A_IF나 A_WHILE인 경우,
    // 비교 후 점프를 생성한다. 그렇지 않으면
    // 레지스터의 값이 0인지 아닌지에 따라 0 또는 1로 설정한다.
    return (cgboolean(leftreg, parentASTop, iflabel, type));
...
  case A_EQ:
  case A_NE:
  case A_LT:
  case A_GT:
  case A_LE:
  case A_GE:
    // 부모 AST 노드가 A_IF, A_WHILE 또는 A_TERNARY인 경우,
    // 비교 후 점프를 생성한다. 그렇지 않으면 레지스터를 비교하고
    // 비교 결과에 따라 하나의 레지스터를 1 또는 0으로 설정한다.
    if (parentASTop == A_IF || parentASTop == A_WHILE ||
        parentASTop == A_TERNARY)
      return (cgcompare_and_jump
              (n->op, leftreg, rightreg, iflabel, n->left->type));
    else
      return (cgcompare_and_set(n->op, leftreg, rightreg, n->left->type));
```

따라서, 문제를 해결하기 위해 a) 삼항 연산자 코드에 `mkastunary(A_TOBOOL...)`을 추가하고, b) `case A_TOBOOL`에 `parentASTop == A_TERNARY`를 추가했다. 이 작업을 마치고 나니 모든 테스트가 통과했으며, 추가로 만든 테스트도 모두 통과했다. 다행이다!


## 2024년 6월 3일 월요일 14:00:23 AEST

이제 우리가 만든 컴파일러로 `decl.c`를 컴파일해 보자. 결과는 다음과 같다:

```
$ md5sum decl.c_qbe fred.qbe
0f299257e088b3de96b68430e9d1f123  decl.c_qbe	G 버전
0f299257e088b3de96b68430e9d1f123  fred.qbe	W 버전
```


W 바이너리를 사용하여 패스를 QBE 코드로 컴파일하는 셸 스크립트를 통해 삼중 테스트를 통과할 수 있을 것 같다!

```
18fe9843b22b0ad06acfbc2011864619  cg6809.c_qbe	G 버전
18fe9843b22b0ad06acfbc2011864619  fred.qbe	W 버전
===
f7db53fe1cb35bd6ed19e633a5c618a6  cgen.c_qbe
f7db53fe1cb35bd6ed19e633a5c618a6  fred.qbe
===
9bfc78b1ef9b08eecf9dde1046bc4ab6  cgqbe.c_qbe
9bfc78b1ef9b08eecf9dde1046bc4ab6  fred.qbe
===
===
76ec1c53e8362a69ef1935a5efd7a1e3  ctreeopt.c_qbe
76ec1c53e8362a69ef1935a5efd7a1e3  fred.qbe
===
0f299257e088b3de96b68430e9d1f123  decl.c_qbe
0f299257e088b3de96b68430e9d1f123  fred.qbe
===
62d30bbd1b3efc61a021fffe76fff670  desym.c_qbe
62d30bbd1b3efc61a021fffe76fff670  fred.qbe
===
28e5d1e283ee25a6b7e08f1d69816de8  detok.c_qbe
28e5d1e283ee25a6b7e08f1d69816de8  fred.qbe
===
c1cdef1a287717a429281f6c439475d4  detree.c_qbe
c1cdef1a287717a429281f6c439475d4  fred.qbe
===
4c72e8a009e3e6730969b7d55a30b9b4  expr.c_qbe
4c72e8a009e3e6730969b7d55a30b9b4  fred.qbe
===
ea63b78d3fa59c8d540639d83df8cf75  gen.c_qbe
ea63b78d3fa59c8d540639d83df8cf75  fred.qbe
===
ccb643e15e8cc51969ee41aac2a691e7  misc.c_qbe
ccb643e15e8cc51969ee41aac2a691e7  fred.qbe
===
58fd815735a4ff3586a460884e58e700  opt.c_qbe
58fd815735a4ff3586a460884e58e700  fred.qbe
===
784b5c65469c655868761f5c4501739c  parse.c_qbe
784b5c65469c655868761f5c4501739c  fred.qbe
===
453d1ccb4e334b9e2852fac87a87bcff  scan.c_qbe
453d1ccb4e334b9e2852fac87a87bcff  fred.qbe
===
8d66ab6b83506b92ab7e747c9d645fa3  stmt.c_qbe
8d66ab6b83506b92ab7e747c9d645fa3  fred.qbe
===
d91591705e6e99733269781f4f44bf47  sym.c_qbe
d91591705e6e99733269781f4f44bf47  fred.qbe
===
30e39b30644aead98e47ccd5ebf6171c  targ6809.c_qbe
30e39b30644aead98e47ccd5ebf6171c  fred.qbe
===
0be7b46eb6add099be91cd3721ec4f09  targqbe.c_qbe
0be7b46eb6add099be91cd3721ec4f09  fred.qbe
===
73bd8faa39878c98f40397e0cf103408  tree.c_qbe
73bd8faa39878c98f40397e0cf103408  fred.qbe
===
7cf6e7e9ad7f587e31e3163aad1a40f3  tstring.c_qbe
7cf6e7e9ad7f587e31e3163aad1a40f3  fred.qbe
===
392531419455d54d333922f37570cb61  types.c_qbe
392531419455d54d333922f37570cb61  fred.qbe
===
d7a3ddeafccf98d03d2fe594e78f2689  wcc.c_qbe
d7a3ddeafccf98d03d2fe594e78f2689  fred.qbe
```

모든 체크섬이 동일하다.


## 2024년 6월 4일 화요일

`Makefile`을 재구성하여 QBE 백엔드로 트리플 테스트를 실행할 수 있도록 설정했다. 레벨 0 바이너리는 `gcc`로 빌드한다. `L1/` 디렉토리의 레벨 1 바이너리는 레벨 0 바이너리로 빌드한다. `L2/` 디렉토리의 레벨 2 바이너리는 레벨 1 바이너리로 빌드된다. 따라서 `L1/`과 `L2/`의 파일은 `wcc`를 제외하고는 동일해야 한다. BINDIR이 다르기 때문이다.

`wcc`에는 여전히 문제가 있다. 다음 명령어는 실행할 수 있다:

```
$ L1/wcc -S wcc.c
$ L1/wcc -c wcc.c
```

그러나 링크 단계를 실행하려고 하면 다음과 같은 루프에 빠진다:

```
$ L1/wcc -o wcc -v wcc.c
Doing: cpp -nostdinc -isystem /usr/local/src/Cwj6809/include/qbe wcc.c 
  redirecting stdout to wcc.c_cpp
Doing: /usr/local/src/Cwj6809/L1/cscan 
  redirecting stdin from wcc.c_cpp
  redirecting stdout to wcc.c_tok
Doing: /usr/local/src/Cwj6809/L1/cparseqbe wcc.c_sym wcc.c_ast 
  redirecting stdin from wcc.c_tok
Doing: /usr/local/src/Cwj6809/L1/cgenqbe wcc.c_sym wcc.c_ast 
  redirecting stdout to wcc.c_qbe
Doing: qbe -o wcc.c_s wcc.c_qbe 
Doing: as -o wcc.c_o wcc.c_s 
Doing: cpp -nostdinc -isystem /usr/local/src/Cwj6809/include/qbe wcc.c 
  redirecting stdout to wcc.c_cpp
Doing: /usr/local/src/Cwj6809/L1/cscan 
  redirecting stdin from wcc.c_cpp
  redirecting stdout to wcc.c_tok
...
```

문제를 찾았다. 테스트 코드는 다음과 같다:

```
#include <stdio.h>
int i;
int main() {
  for (i=1; i<=10; i++) {
    if (i==4) { printf("I don't like 4 very much\n"); continue; }
    printf("i is %d\n", i);
  }
  return(0);
}
```

`continue`는 `i++`를 실행한 후 `i<=10` 조건을 검사하는 코드로 이동해야 한다. 그러나 컴파일러는 `i<=10` 조건 검사로 바로 이동하여 `i`가 4인 상태에서 무한 루프에 빠진다.

이 문제를 해결하는 방법이 명확하지 않다. `for` 루프를 파싱할 때, 나는 루프 본문 끝에 postop 트리를 붙이고 `while` 루프처럼 처리한다. 따라서 조건 검사 전과 루프 끝에만 레이블이 있고, 본문과 postop 코드 사이에는 레이블이 없다.

일단 `wcc`를 재작성하여 문제를 피할 수 있다. 그리고 이제 트리플 테스트를 통과했다:

```
md5sum L1/* L2/* | sort
424d006522f88a6c8750888380c48dbe  L1/desym
424d006522f88a6c8750888380c48dbe  L2/desym
5da0fd17d14f35f19d1e1001c4ffa032  L2/wcc
6459d5698068115890478e4498bad693  L1/wcc
74ce22e789250c3c406980dab1c37df1  L1/detok
74ce22e789250c3c406980dab1c37df1  L2/detok
9cd8c07f0b66df2c775cfa348dfac4f7  L1/cscan
9cd8c07f0b66df2c775cfa348dfac4f7  L2/cscan
9fbe13d2b8120797ace55a37045f2a48  L1/cgenqbe
9fbe13d2b8120797ace55a37045f2a48  L2/cgenqbe
a9f109370f44ce15b9245d01b7b03597  L1/cparseqbe
a9f109370f44ce15b9245d01b7b03597  L2/cparseqbe
ebed2d69321e600bc3f5a634eb1ac1f8  L1/detree
ebed2d69321e600bc3f5a634eb1ac1f8  L2/detree
```

성공이다! 방금 `bettercode` 브랜치를 `master` 브랜치로 병합했다.


## 2024년 6월 4일 화요일 10:12:29 (AEST)

이제 트리플 테스트를 통과했으니 6809로 돌아왔다. 현재 오브젝트 파일 크기는 다음과 같다:

```
   806 Jun  4 10:11 targ6809.o
   834 Jun  4 10:11 misc.o
  1062 Jun  4 10:11 targqbe.o
  2012 Jun  4 10:11 tstring.o
  2112 Jun  4 10:11 opt.o
  2658 Jun  4 10:11 detok.o
  2733 Jun  4 10:11 detree.o
  4399 Jun  4 10:11 types.o
  5513 Jun  4 10:11 ctreeopt.o
  5759 Jun  4 10:11 tree.o
  5858 Jun  4 10:11 parse.o
  7373 Jun  4 10:11 desym.o
  7638 Jun  4 10:11 stmt.o
  7868 Jun  4 10:11 cgen.o
  8801 Jun  4 10:11 sym.o
 11590 Jun  4 10:11 gen.o
 13256 Jun  4 10:11 wcc.o
 13942 Jun  4 10:11 scan.o
 17670 Jun  4 10:11 expr.o
 19156 Jun  4 10:11 decl.o
 21114 Jun  4 10:11 cgqbe.o
 34268 Jun  4 10:11 cg6809.o
```


## 2024년 6월 4일 화요일 11:38:09 AEST

SubC 트리 최적화 작업 중 일부를 적용했다.  
몇 가지는 동작하지 않아 주석 처리했다. 전체 코드 크기는 거의 줄어들지 않았다.  

아차. 나중에 확인하지 않았는데, 이 변경사항들이 6809 테스트 중 일부를 깨뜨렸다.  
아쉽다.


2024년 6월 4일 화요일 11:44:46 (AEST)

이 아이디어에 대해 생각 중이다. 기존 코드를 보자:

```
L2:
        ldd -2,u   <-- d_holds "-2,u"
        cmpd #8
        bge L3
;		   <- now NOREG due to free all locns
        ldd -2,u
```

하지만 이 부분을 그대로 유지하면 D를 다시 로드할 필요가 없지 않을까? 점프 시 모든 위치를 비워야 한다. 이걸 시도해볼 수 있을지 확인해보려고 한다.

아, 테스트는 통과했지만 코드 크기가 더 나빠졌다!! 같은 위치에 `std` 후에 `ldd`를 피하기 위해 간단한 피홀 최적화를 추가했는데 말이다.


## 2024년 6월 4일 화요일 13:25:58 AEST

방금 fcc와 wcc의 크기를 다시 비교했다:

```
                 fcc     wcc
cg6809.o        28682   33364   1.16
cgen.o          5587    7742    1.39
cgqbe.o         16867   20999   1.24
ctreeopt.o      3965    5504    1.39
decl.o          14499   19058   1.31
desym.o         5881    7340    1.25
detok.o         1894    2609    1.38
detree.o        1848    2700    1.46
expr.o          11646   17527   1.50
gen.o           8745    11482   1.31
misc.o          742     834     1.12
opt.o           1519    2051    1.35
parse.o         4265    5797    1.36
scan.o          11312   13822   1.22
stmt.o          5649    7596    1.34
sym.o           6732    8781    1.30
targ6809.o      761     806     1.06
targqbe.o       903     1046    1.16
tree.o          5152    5745    1.12
tstring.o       2012    2012    1.00
types.o         2516    4397    1.75
wcc.o           10282   12897   1.25
                        Average 1.29
```

현재 wcc는 fcc보다 29% 더 크다.

`types.c` 파일을 살펴보고 있다. 첫 번째 함수는 다음과 같다:

```
int inttype(int type) {
  return (((type & 0xf) == 0) && (type >= P_CHAR && type <= P_LONG));
}
```

이 함수의 AST는 다음과 같다:

```
FUNCTION inttype
  RETURN
    LOGAND
      EQ
        AND
          IDENT rval type
          INTLIT 15
        INTLIT 0
      LOGAND
        GE
          IDENT rval type
          INTLIT 32
        LE
          IDENT rval type
          INTLIT 64
```

wcc가 생성한 코드는 매우 비효율적이다! 하지만 다음 부분이 잘못되었다:

```
        ldd 4,u
        anda #15	<- #00이어야 함
        andb #15
```

이 문제는 `printlocation()` 함수가 리터럴을 다룰 때 8비트, 16비트, 32비트 연산 중 어떤 것을 수행하는지 알 수 없기 때문이다. 아마도 A, B, D, Y 레지스터 중 어떤 것을 사용하는지 나타내는 인수를 추가해야 할 것 같다. 이 문제를 해결했고, 이제 제대로 동작한다.


2024년 6월 6일 목요일 오전 8시 20분 21초 (AEST)

LOGAND와 LOGOR 코드를 개선하는 데 어려움을 겪고 있다. 현재 코드는 많은 #0과 #1을 생성한 후 이를 테스트한다. 대신 `cgcompare_and_jump()`처럼 레이블로 바로 점프할 수 있어야 한다. 두 함수로 나눴지만, 합칠 수 있을 것 같다. 예를 들면:

```
                 x || y                   x && y
--------------------------------------------------------------
        if (lhs true)  goto Ltrue   if (lhs false) goto Lfalse
        if (rhs false) goto Lfalse  if (rhs false) goto Lfalse
Ltrue:  ldd #1; goto Lend
Lfalse: ldd #0
Lend:
```

실제로 첫 번째 줄만 다르다. 첫 번째 줄에 다른 AST 연산을 전달하면 된다.


2024년 6월 6일 목요일 11:33:16 AEST

오랜 고민 끝에 드디어 문제를 해결했다. 모든 테스트가 통과할 뿐만 아니라, 삼중 테스트도 완벽하게 통과한다.


## 2024년 6월 7일 금요일 08:40:20 AEST

다음 코드를 최적화할 수 있다는 사실을 깨달았다:

```
#11
        ldx %1,u
        ldd 0,x
```

이 코드는 다음과 같이 최적화할 수 있다:

```
        ldd [%1,u]
```

D 저장, B 로드 및 저장 연산도 동일하게 최적화할 수 있다. 또한 피홀 최적화 규칙이 공백에 민감하다는 사실을 깨달았다. 이제 최적화가 제대로 동작하는지 확인할 수 있는 테스트 입력 파일을 만들었다.

코드 생성의 일부를 피홀 규칙 파일로 옮길 수 있다는 사실도 깨달았다. 이렇게 하면 코드 생성기에서 C 코드의 양을 줄일 수 있다. 예를 들어, `a >> 24`를 다음과 같이 생성할 수 있다:

```
	ldd _a
	; cgshr 24
```

그러면 최적화기가 이를 몇 번의 Y/D/A/B 교환과 클리어 연산으로 대체할 수 있다!


## 2024년 6월 7일 금요일 08:53:54 AEST

`fcc`와 `.o` 파일 크기를 비교하기 위해 Perl 스크립트를 작성했다:

```
    cg6809.o:	1.07
      cgen.o:	1.30  *
     cgqbe.o:	1.15
  ctreeopt.o:	1.33
      decl.o:	1.05
     desym.o:	1.25
     detok.o:	1.38
    detree.o:	1.46
      expr.o:	1.21  *
       gen.o:	1.34  *
      misc.o:	1.12
       opt.o:	1.21
     parse.o:	1.33  *
      scan.o:	1.13
      stmt.o:	1.22  *
       sym.o:	1.17
  targ6809.o:	1.06
   targqbe.o:	1.16
      tree.o:	1.12
   tstring.o:	1.00
     types.o:	0.99
       wcc.o:	1.25
     Average:	1.19
```

중요한 항목에는 별표(*)를 표시했다.


## 2024년 6월 7일 금요일 09:53:22 AEST

몇 가지 피홀 규칙을 더 작성하고 있었다. 그러다 이 코드를 발견했다:

```
char *fred() { return(NULL); }

int main() {
  if (fred() != NULL) printf("fred was not NULL\n");
  return(0);
}
---
FUNCTION main
  IF
    NE
      FUNCCALL fred
      CAST 17
        INTLIT 0
---
        lbsr _fred	; fred 호출
        std R0+0	; 결과 저장
        ldd #0
        std R1+0	; 임시 변수에 0 저장
        ldd R0+0	; 비교 수행?!
        cmpd R1+0
        beq L3
```

왜 그냥 `cmpd #0, beq L3`를 사용할 수 없는 걸까? 그렇게 해도 동작한다. 이제 이 코드가 생성되는 이유를 알아내야 한다!

아, 캐스트 때문이었다. `cgwiden()`에 코드를 추가했다: 기본 크기가 같으면 아무것도 하지 않는다.

이로 인해 `wcc/fcc` 비율이 줄어들었다:

```
      cgen.o:	1.30  이제 1.23
      expr.o:	1.21  이제 1.12
       gen.o:	1.34  이제 1.30
     parse.o:	1.33  이제 1.28
      stmt.o:	1.22  이제 1.10
     Average:	1.19  이제 1.15
```


## 2024년 6월 7일 금요일 16:40:13 AEST

`objcompare`를 작성해 함수 크기를 비교했다. `fcc`와 `wcc` 간 가장 차이가 큰 부분은 다음과 같다:

```
enumerateAST:
   F/parse.o     85
   W/parse.o    151 1.78 (비율)

genlabel:
     F/gen.o     15
     W/gen.o     26 1.73

mkastnode:
    F/tree.o    133
    W/tree.o    215 1.62
```

이 부분을 먼저 확인해보기로 했다.


## 2024년 6월 8일 토요일


앨런 콕스로부터 `fcc`에 대한 최근 개선 사항에 대해 이메일을 받았다. 나는 이 프로젝트에 대해 이야기하고 코드를 보내겠다고 제안했다.

아레나에 있는 동안 `cg6809.c` 파일 전체를 살펴봤다. 다음과 같은 코드가 굉장히 많이 보였다:

```
  int primtype= cgprimtype(type);
  ...
  switch(primtype) {
    case P_CHAR:
    ...
    case P_INT:
    ...
    case P_POINTER:
    ...
    case P_LONG:
    ...
  }
```

`gen.c`에서 `primtype`을 보내도록 할 수도 있지만, 나는 기본 타입을 나타내는 문자를 받는 방식을 고민하고 있다. 그렇게 하면 다음과 같이 코드를 작성할 수 있다:

```
int cgadd(int l1, int l2, char primchar) {

  load_d(l1);
  // 의사 명령어 출력
  fprintf(Outfile, "\tadd%c ", primchar); printlocation(l2, 0, primchar);
  cgfreelocn(l2);
  Locn[l1].type= L_DREG;
  d_holds= l1;
  return(l1);
}
```

이렇게 하면 피홀 최적화기가 의사 명령어를 실제 코드로 확장할 수 있다. 이 방식은 코드 생성기에서 많은 코드를 줄일 수 있다! 하지만 INTLITS와 관련된 문제가 생길 수 있다. 만약 다음과 같은 코드를 생성한다면:

```
    ; 의사 명령어
    addl #1234567
```

피홀 최적화기가 `>>8` 등을 처리해야 한다. 이는 번거로울 수 있다. 아, 가능한 해결책이 있다. 바이트와 정수 리터럴의 경우 그냥 출력하면 된다. 예를 들어 `#2345`처럼. 긴 리터럴의 경우 상위 바이트를 10진수로, 두 번째 바이트를 10진수로, 하위 16비트를 10진수로 출력하면 된다. 예를 들어:

```
    ; 의사 명령어
    addl #0 #18 #54919
```

그러면 최적화기가 이를 매칭할 수 있다. 하지만 이 경우 많은 규칙이 필요할 것 같다.


## 2024년 6월 8일 토요일 10:11:45 AEST


다른 메모를 작성했다. 이제 가져올 시간이다.


## for와 continue 문제 해결

현재 우리는 다음과 같은 코드를 사용한다:

```
for (PRE; COND; POST)
  BODY
```

이 코드는 다음과 같이 변환된다:

```
  PRE
Lstart:
  COND  (Lend로 점프할 수 있음)
  BODY
  POST
Lend:
```

그리고 AST 트리는 다음과 같다:

```
  GLUE
  / \
PRE WHILE
     /  \
   COND  \
        GLUE
        /  \
     BODY POST
```

하지만 WHILE 노드의 중간 자식을 사용하지 않는다. 따라서 다음과 같은 트리를 구성할 수 있다:

```
  GLUE
  / \
PRE WHILE
   /  |  \
COND  | BODY
    POST
```

`genWHILE()` 함수에서 중간 자식의 존재를 확인한다. 만약 중간 자식이 있다면 다음과 같은 코드를 생성한다:

```
  PRE	(다른 곳에서 처리됨)
  jump Lcond
Lstart:
  POST
Lcond:
  COND  (Lend로 점프할 수 있음)
  BODY
Lend:
```

이제 `break`는 `Lend`로 이동하고, `continue`는 `Lstart`로 이동하면서 POST 작업을 수행한다.


## 추가 아이디어

`x_holds` 트래커를 도입하는 것이 유용할까? 현재 `cg6809.c`에서는 사용하지 않기 때문에 코드 양이 크게 늘어나지는 않겠지만, 코드 출력을 몇 퍼센트 줄이는 데 도움이 될 수 있다.

`cgshrconst()` 함수가 필요하다.

시프트 연산을 할 때, 오른쪽 값이 INTLIT인지 확인하고 특정 값에 대해 `cgshlconst()`와 `cgshrconst()`를 사용해야 한다.

값이 8, 16, 24인 경우에는 어셈블리 코드에 주석을 추가한다. 예를 들어 `leftshift_8`과 같이 작성한다. 피홀 최적화기(peephole optimiser)를 사용해 코드를 삽입한다. 이렇게 하면 코드 생성기에서 코드를 제거하고 피홀 최적화기로 옮길 수 있다. 심지어 1과 2도 처리할 수 있다!

8, 16, 24의 경우, 레지스터 교환과 클리어 작업을 통해 원하는 결과를 얻을 수 있다.


## 2024년 6월 9일 일요일 07:44:45 AEST

앨런이 답장을 보내며, 매크로를 어셈블러에 추가하는 것이 peephole 최적화기에 pseudo-op 확장을 밀어붙이는 것보다 낫다고 제안했다. 특히 최적화기가 더 많은 최적화를 적용할 수 있는지 반복적으로 확인하기 때문에 속도가 느릴 것이라는 점에서 그 말이 맞는 것 같다. 앨런은 테이블에 대해 언급했는데, 이것을 활용할 수 있을지 궁금하다.

각 "연산"에 대해 여러 크기 규칙이 있을 것이다. 각 규칙은 문자열과 오프셋을 가진다. 오프셋이 UNUSED인 경우 단순히 문자열을 출력하고, 그렇지 않으면 문자열과 오프셋을 적용한 위치를 출력한다.

단순히 큰 테이블과 사용할 첫 번째 항목과 줄 수를 보관하는 두 번째 테이블을 사용해야 할까?

어쨌든, 그 전에 코드 밀도를 개선하고 wcc/fcc 크기 비율을 줄이는 작업이 필요하다.


몇 가지 추가적인 peephole 규칙을 적용했다. 현재 결과는 다음과 같다:

```
      cgen.o:	1.30  현재 1.21
      expr.o:	1.21  현재 1.10
       gen.o:	1.34  현재 1.14
     parse.o:	1.33  현재 1.27
      stmt.o:	1.22  현재 1.06
     Average:	1.19  현재 1.12
```


## 2024년 6월 9일 일요일 14:02:30 AEST

Bunnings에 다녀오는 길에, 임시 변수가 스택에 할당되는 방식에 대해 생각했다. 확인해보니, 임시 변수는 항상 증가하는 순서로 할당되고, 완전히 해제되거나 마지막 하나가 해제된 후 모두 해제된다. 이는 할당 알고리즘을 단순히 숫자를 증가시키는 방식으로 변경할 수 있음을 의미한다.

임시 변수가 스택에 올라갈 때, `localOffset`을 조정해야 한다. 하지만 함수가 끝날 때까지 최종 값이 무엇인지 알 수 없다.

`cgfuncpreamble()`을 수행할 때, 출력 파일을 읽기/쓰기 모드로 열어두는 것을 생각했다. 스택을 조정하기 위해 `leas`를 출력할 위치에서 `ftell()`을 사용해 오프셋을 저장하고 다음과 같이 출력한다:

```
fprintf(Outfile, ";\tleas XXXXX,s\n");
```

나중에 `localOffset`이 0이 아니면, `cgfuncpostamble()`에서 `fseek()`을 사용해 이 위치로 돌아가서 실제 값으로 해당 줄을 덮어쓸 수 있다. 약간 번거롭지만 동작할 것이다.

일단 임시 변수 할당을 단순히 증가시키는 방식으로 변경할 것이다. 이 방식은 잘 동작한다. 새로운 코드는 다음과 같다:

```
static int next_free_temp;

static int cgalloctemp() {
  return(next_free_temp++);
}
```

나중에 `highest_free_temp`를 유지할 것이다. 이를 넘어서 할당하면 `localOffset`을 4씩 증가시킬 수 있다.

`u`를 프레임 포인터로 사용하는 방식을 반드시 대체해야 한다. 이 방식은 매우 간단한 함수에 대해 많은 추가 코드를 생성하기 때문이다. 이것이 `fcc`가 `parse.c`에서 더 나은 성능을 보이는 이유다.

아이디어: `sp_adjust` 변수를 유지하고, 초기값을 0으로 설정한다. `pshs`를 할 때마다 적절한 크기만큼 증가시킨다. `puls` 또는 `leas`를 할 때는 감소시킨다. `fcc`와 마찬가지로, 함수의 포스트앰블에 도달했을 때 이 값이 0인지 확인할 수 있다.

화요일에 `sp_adjust`를 구현하고 `u` 프레임 포인터를 제거할 것이다. 이것이 동작하면 임시 변수를 스택으로 옮길 수 있다.


## 2024년 6월 9일 일요일 14:48:20 AEST

`sp_adjust`를 추가하기로 결정했지만, 초기 단계로 `u` 프레임 포인터를 유지하기로 했다. 확인해본 결과, `sp_adjust`는 내가 테스트할 수 있는 모든 코드의 포스트앰블에서 항상 0이었다.


2024년 6월 9일 일요일 14:57:49 AEST

아, `u` 프레임 포인터를 제거하는 코드를 작업하기 시작했다.  
input026.c까지는 괜찮았는데, 테스트 27에서 실패했다.


## 2024년 6월 9일 일요일 15:30:10 AEST

완료! 생각보다 쉬웠다. 몇 가지 간단한 수정이 필요했지만 모두 해결했다. 모든 테스트가 정상적으로 통과했다.

이제 비교를 위해 다음과 같은 결과를 얻었다:

```
      cgen.o:	1.18 size   6438
      expr.o:	1.04 size  12204
       gen.o:	1.06 size  10552
     parse.o:	1.23 size   5120
      stmt.o:	1.03 size   6047
     Average:	1.09
```

가장 큰 `.o` 파일은 다음과 같다:

```
    cg6809.o:	1.03 size  29498
      decl.o:	0.97 size  13901
      expr.o:	1.04 size  12204
       gen.o:	1.06 size  10552
      scan.o:	1.09 size  12365
       wcc.o:	1.17 size  12055
```

`wcc`는 괜찮지만, 코드 생성기는 `cg6809.o`와 `gen.o`를 비롯한 여러 파일로 구성되어 있어 이 부분이 문제가 될 것이다.


## 2024년 6월 10일 월요일 오전 7시 39분 53초 (AEST)

기존 함수 프리앰블에 있는 `leas` 명령어를 사용해 스택 변경을 패치하는 `ftell()`과 `fseek()` 코드를 시도해 보기로 했다. 성공했다! 이제 스택에 임시 변수를 올리는 작업을 진행할 수 있다.

하지만 문제가 생겼다. 스택에 다음과 같은 구조를 만들어야 하는데:

```
  매개변수
  반환 주소
  지역 변수
  임시 변수
              <- sp
```

이 모든 요소의 오프셋을 미리 알아야 한다. 하지만 모든 임시 변수를 연속적으로 할당하기 전까지는 필요한 공간을 알 수 없다. 이는 지역 변수와 매개변수의 오프셋을 계산할 수 없다는 의미다. 임시 변수를 지역 변수 위에 배치하더라도 매개변수의 오프셋을 계산할 수 없다.

임시 변수를 할당할 때, 그 크기를 `sp_adjust`에 추가하고 임시 변수의 오프셋으로 0을 반환하는 방법은 어떨까? 이렇게 하면 임시 변수를 스택에 푸시하는 효과를 얻을 수 있다. 그리고 모든 임시 변수를 해제할 때 `sp_adjust`를 줄일 수 있다.


2024년 6월 10일 월요일 09:45:43 AEST

`cg6809.c` 파일을 살펴보니, spill 코드가 주석 처리되어 있다. 따라서 임시 변수를 스택에 저장할 필요가 없다. 이는 스택에 임시 변수를 넣을 필요가 없음을 의미한다. 즉, 현재 R0, R1 임시 변수를 그대로 유지할 수 있다. 동시에 `fseek()` 코드도 제거할 수 있다.


## 2024년 6월 10일 월요일 09:59:13 AEST

이제 온도 할당 코드를 간소화한 것과 같은 방식으로 위치 할당 코드를 간소화해 보자. 하지만 `gen.c`가 종종 하나의 레지스터를 제외한 나머지를 해제하기 때문에, 그 하나를 계속 추적해야 한다. 이게 문제다.

유지할 레지스터를 R0로 복사할 수는 있지만, 이는 상당한 작업이 필요하다. 새로운 레지스터의 위치를 받도록 `gen.c` 코드를 다시 작성해야 한다. 일단 이 문제는 잠시 보류하기로 한다.


## 2024년 6월 10일 월요일 10:24:44 AEST

이제 출력 라인 테이블 아이디어로 돌아가 보자. printlocation의 오프셋과 "레지스터"를 인코딩해야 한다:

```
      1 printlocation(0, 'a');
      8 printlocation(0, 'b');
     10 printlocation(0, 'd');
      3 printlocation(0, 'e');
      3 printlocation(0, 'y');
      1 printlocation(1, 'b');
      3 printlocation(1, 'f');
      1 printlocation(2, 'a');
      6 printlocation(2, 'd');
      1 printlocation(2, 'y');
      1 printlocation(3, 'b');
```

특정 라인에서 printlocation을 실행해야 할 경우를 고려해야 한다.


## 2024년 6월 10일 월요일 오전 11:45:59 AEST

`cg6809.c` 파일에 있는 함수 프로토타입을 다시 살펴보면, 테이블로 정리할 수 있는 것들은 하나 또는 두 개의 레지스터 인자, 타입, 그리고 점프할 레이블을 가지고 있다. `printlocation` 함수를 수정해 필요하지 않을 때는 아무것도 출력하지 않고, 레이블이 필요한 경우에만 출력하도록 변경할 수 있을 것 같다.

또한 `gen.c`에서 `cgXXX.c` 함수로 보내는 `type`을 변경해야 한다. 이렇게 하면 계속해서 타입을 변환할 필요가 없어진다. 이미 작업을 시작했지만 몇 가지 테스트를 진행한 후에 오류가 발생한다.


## 2024년 6월 11일 화요일 10:14:16 (AEST)

모든 6809 테스트가 이제 통과했다. 아직 QBE 백엔드를 다시 작성하지는 않았다.  
`cgprimtype()` 함수를 `gen.c`로 옮기면서 6809 백엔드에서 180바이트를 절약했다. 큰 차이는 아니다. 특히 테이블 기반 접근 방식을 사용할 경우, `cgprimtype()`을 두 번 수행한 후 여러 작업에 테이블을 활용할 수 있기 때문에 이 변경은 그다지 가치가 없을 수도 있다.  
이 변경 사항은 사이드 브랜치에 보관해 두기로 한다.


## 2024년 6월 12일 수요일 13:47:19 (AEST)

`cgen.c`에 AST 트리를 정리하기 위해 `free()` 코드를 추가했다.  
가끔 왼쪽과 오른쪽 노드가 동일한 경우가 있는데, 이유는 잘 모르겠다.  
어쨌든 이 부분은 잘 동작한다.  
로컬과 파라미터 심볼 리스트도 해제하려고 시도했지만, QBE 백엔드가 다른 코드를 생성하게 된다.  
그래서 이 변경사항은 일단 주석 처리해 두었다.


2024년 6월 12일 수요일 14:12:56 AEST

다음과 같은 시도를 해봤다:

```
$ sh -x z
wcc -m6809 -o wcc wcc.c
wcc -m6809 -o cscan scan.c misc.c
wcc -m6809 -o detok detok.c tstring.c
wcc -m6809 -o detree detree.c misc.c
wcc -m6809 -o desym desym.c
wcc -m6809 -o cpeep cpeep.c
cpeep.c 파일 95번 줄에서 바이너리 표현에 호환되지 않는 타입이 발견됨
wcc -m6809 -o ctreeopt ctreeopt.c tree.c misc.c
wcc -m6809 -o cparse6809 decl.c expr.c misc.c opt.c parse.c stmt.c sym.c
              tree.c targ6809.c tstring.c types.c
wcc -m6809 -o cgen6809 cg6809.c cgen.c gen.c misc.c sym.c targ6809.c
              tree.c types.c
```

결과는 다음과 같다:

```
-rwxr-xr-x 1 wkt wkt 12280 Jun 12 14:16 wcc
-rwxr-xr-x 1 wkt wkt 10583 Jun 12 14:16 cscan
-rwxr-xr-x 1 wkt wkt  7536 Jun 12 14:16 detok
-rwxr-xr-x 1 wkt wkt  8984 Jun 12 14:16 detree
-rwxr-xr-x 1 wkt wkt  8434 Jun 12 14:16 desym
-rwxr-xr-x 1 wkt wkt  7941 Jun 12 14:16 ctreeopt
-rwxr-xr-x 1 wkt wkt 27267 Jun 12 14:16 cparse6809
-rwxr-xr-x 1 wkt wkt 29615 Jun 12 14:16 cgen6809
```

흥미롭다.

문제는 `emu6809 ./cscan < detok.c_cpp > fred` 명령을 실행했을 때 빈 출력 파일이 생성된다는 점이다.


2024년 6월 12일 수요일 15:34:38 (AEST)

방금 아이디어가 떠올랐다. 문제는 내가 만든 어셈블리 출력이나 libc와의 상호작용, 또는 에뮬레이터가 제대로 작동하지 않을 가능성이 있다. 하지만 우리에게는 또 다른 6809 컴파일러인 `fcc`가 있다. 그래서 `fcc`로 바이너리를 빌드해보고 결과를 확인할 수 있다.

좋다, 이제 `fcc`로 빌드한 바이너리를 얻었다. 이번에는 `cscan`이 출력을 생성하지만, `detok.c_cpp` 입력 파일을 사용해 `detok.c`의 82번째 줄 근처에서 루프에 빠진다. 적어도 6809 `detok` 바이너리는 작동한다 :-) 아, `detok.c`는 82줄밖에 없는데, 파일의 끝을 제대로 감지하지 못하는 것 같다.

그렇다면 내 에뮬레이터가 EOF를 올바르게 보내지 않는 것일까?

또한 `fcc`가 내 테스트를 통과하는지 확인한다. 아니, test67은 실패하지만 나머지는 괜찮다.


## 2024년 6월 13일 목요일 09:26:47 AEST

`scan.c`에 디버그 코드를 추가했고, `fcc` 버그를 발견한 것 같다:

```
  switch (c) {
    case EOF:
      t->token = T_EOF;
fprintf(stderr, "c is EOF, t->token is %d not %d\n", t->token, T_EOF);
      return (0);
```

이 코드는 다음과 같은 어셈블리를 생성한다:

```
        ldx #Sw67
        bra __switch
Sw67_1:
        clr [0,s]	; t->token을 0으로 설정하려 했지만,
        clr [1,s]	; [0,s]는 아래의 [8,s]와 다르다
...
        clra
        clrb
        pshs d		; T_EOF(0)를 푸시
        ldd [8,s]
        pshs d		; t->token을 푸시
        ldd #T176+0
        pshs d		; 문자열 포인터를 푸시
        ldd #_stderr+0
        pshs d		; FILE *를 푸시
        lbsr _fprintf+0
```

`wcc`로 `cscan`을 컴파일하고 빌드한 결과, `fgetc()`가 첫 호출에서 EOF를 반환한다. 매우 성가신 문제다.


## 2024년 6월 13일 목요일 09:56:36 AEST

`fcc`와 bintools를 최신 Github 저장소 버전으로 업데이트하려고 고민 중이다. 그 전에 현재 사용 중인 커밋을 확인해 보자:

 - bintools: bdb0076b5e3d4745aa08289d61e39f646d75805e
 - compiler: ffda85a94ce900423dc25a020fe62609ddcd46db

두 프로젝트의 최신 버전을 가져왔고, 컴파일러는 8a4b65b4d18be9528f3e5a6402b8e392e5ecc341 커밋 상태다. `wtests`는 정상적으로 실행되지만, Fuzemsys 라이브러리에 대해 잘못된 코드를 생성하고 있다. 예를 들어:

```
$ /opt/fcc/bin/fcc -m6809 -S -O -D__m6809__ clock_gettime.c
$ vi +28 clock_gettime.s
        lbsr __shrul
        add 2,s
        adc 1,s
        adc ,s
```

위 코드는 `addd, adcb, adca`로 출력되어야 한다. 이 문제를 Alan에게 보고해야 할까? 문제를 일으킨 커밋을 찾아야 하기 때문에 고민된다.


`wcc`에서 `fgetc()` 문제는 `stdin`을 `FILE` 구조체 하나가 아닌 포인터로 정의했기 때문에 발생했다. 이제 `cscan`은 문자를 읽지만 다른 부분에서 실패한다.


2024년 6월 14일 금요일 12:01:11 (AEST)

아차! 6809 `cgswitch()`에서 스위치 값을 담고 있는 레지스터(위치)는 주어졌지만, 이를 D 레지스터로 로드하지 않았다. 간단히 `load_d(reg)`를 추가해 이 문제를 해결했다. 테스트 160을 추가했다.


## 2024년 6월 14일 금요일 12:17:20 AEST

진전이 있다. `cscan`과 `detok`가 작동하며, 올바른 토큰 스트림 파일을 생성하는 것으로 보인다. 유일한 문제는 Fuxiz의 `printf()`가 Linux의 것과 다르게 동작하는 것 같다는 점이다. 다음과 같은 차이점이 보인다:

```
6464c6375
< 36: struct
---
> 27: struct
6469d6379
< 43: filename \"decl.c\"
6472c6382
< 36: void
---
> 1e: void
6476d6385
< 43: filename \"decl.c\"
6480c6389
< 36: struct
---
> 27: struct
```

토큰 번호가 다르게 출력되지만, 사용된 Tstring은 정확하다. 그리고 파일명의 큰따옴표가 이스케이프 처리되고 있다!

사실 이건 정확한 진실은 아니다. `detok.c`에서 다음과 같은 변경을 해야 했다:

```
<     *s++ = (char) ch;
---
>     *s = (char) ch; s++
```

그래서 원래 라인으로도 QBE 삼중 테스트를 통과할 수 있으니, 왜 이런 변경이 필요한지 조사해야 한다. 다행히도 이 문제는 쉽게 해결되었다.


2024년 6월 14일 금요일 12:42:40 AEST

아, 토큰 번호가 잘못된 이유를 알아냈다. `cscan`이 키워드를 감지하지 못하고 모든 것을 STRLIT로 변환하는 것 같다. 그래서 다음과 같은 결과가 나온다:

```
$ emu6809 cparse6809 decl.c_sym decl.c_ast < decl.c_tok 
unknown type:void on line 4 of /opt/wcc/include/6809/stdlib.h
```

이 오류는 "void"가 T_VOID(30)가 아닌 일반 문자열로 처리되었기 때문이다.


`if (!strcmp(...))`가 동작하지 않아서 수정했다.


## 2024년 6월 15일 토요일 09:32:15 AEST

문제가 해결되지 않았다. 스위치 케이스에서 반환하기 때문이다. 새로운 테스트 케이스를 작성했다:

```
int keyword(char *s) {

  switch (*s) {
    case 'b': return(1);
    case 'c': if (!strcmp(s, "case")) return(2);
              return(3);
    case 'v': if (!strcmp(s, "void")) return(4);
              return(5);
    default: return(6);
  }
  return(0);
}
```

이 코드는 QBE에서는 잘 동작하지만 6809 백엔드에서는 항상 6을 반환한다. 이제 문제를 해결했다. 스위치 케이스를 처리하는 어셈블리 코드는 인자로 `int` 타입을 기대하지만, 우리는 A 레지스터에 쓰레기 값이 들어간 `char` 타입을 보내고 있었다. `stmt.c` 파일을 수정해 P_CHAR 트리를 필요한 경우 P_INT로 확장하도록 변경했다.

이제 토큰 스트림이 "void"와 같은 T_VOID를 올바르게 처리하는 것 같다.

다음 단계로 넘어간다:

```
$ emu6809 cparse6809 decl.c_sym decl.c_ast < decl.c_tok
unknown struct/union type:FILE on line 35 of /opt/wcc/include/6809/stdio.h
```

심볼 테이블이 어딘가에서 망가진 것 같다:

```
Searching for struct __stdio_file: missing!
Adding struct __stdio_file
Searching for struct __stdio_file: found it
Searching for struct __stdio_file: missing!
```

더 많은 디버그 출력을 추가했다:

```
Searching for __stdio_file in list 778e class 0
Comparing against __stdio_file
  (and found it)
...
Searching for __stdio_file in list 778e class 0
  Did not find __stdio_file
```

여기서 778e는 헤드의 값(16진수)이며, 이름 포인터가 위치한 곳이기도 하다(구조체의 첫 번째 멤버). 디버그 코드는 다음과 같다:

```
fprintf(stderr, "Searching for %s in list %lx class %d\n",
					s, (long)list, class);
  for (; list != NULL; list = list->next) {
    if (list->name != NULL)
      fprintf(stderr, "Comparing against %s\n", list->name);
    if ((list->name != NULL) && !strcmp(s, list->name))
      if (class == 0 || class == list->class)
        return (list);
  }
fprintf(stderr, "  Did not find %s\n", s);
  return (NULL);
```

이것은 `list->name`이 어딘가에서 NULL로 설정되고 있다는 것을 의미하는가?


2024년 6월 15일 토요일 11시 14분 39초 AEST

에뮬레이터와 쓰기 중단점을 사용해 보니, 현재 `scalar_declaration()` 함수의 시작 부분에서 `*tree = NULL;`을 실행하고 있다. 이로 인해 의문이 생긴다: 여기서 우리가 이중 역참조를 하지 않는가? 아니면 어떻게 구조체 테이블에 대한 포인터를 얻고 있는가?

더 심각한 문제는, `tree`가 심볼 테이블 포인터가 아니라 `struct ASTnode **` 타입이라는 점이다. 이런!


2024년 6월 15일 토요일 11시 42분 43초 (AEST)

`debug` 파일을 분석 중이며 778E를 검색하고 있다. `newsym()`이 생성되는 것을 확인할 수 있다. `composite_declaration()` 함수로 진입하고, `rbrace()`를 찾아 구조체에 멤버를 추가하는 과정을 볼 수 있다.


## 2024년 6월 15일 토요일 12:24:59 AEST

조금 뒤로 돌아가서, 이 프로그램을 6809 바이너리 단계로 컴파일할 수 있다:

```
void printint(int x);
int main() { printint(5); return(0); }
```

그렇다면 테스트 프로그램을 컴파일해 보는 것이 좋을 것 같다.


## 2024년 6월 15일 토요일 12:45:54 AEST

이제 이에 대한 테스트 스크립트가 준비되었다. 테스트 1과 2는 정상적으로 통과했지만, 테스트 3은 실패했다.

6809 `cparse6809`가 실행되어 출력을 생성한다. 컴파일러의 네이티브 버전도 마찬가지로 동작한다. 후자는 다음과 같은 심볼 테이블을 생성한다:

```
int printf() id 1: global, 1 params, ctypeid 0, nelems 1 st_posn 0
    char *fmt id 2: param offset 0, size 2, ctypeid 0, nelems 1 st_posn 0
void main() id 3: global, 0 params, ctypeid 0, nelems 0 st_posn 0
int x id 4: local offset 0, size 2, ctypeid 0, nelems 1 st_posn 0
unknown type x id 0: unknown class, size 0, ctypeid 0, nelems 0 st_posn 0
```

마지막 줄은 하나의 AST 트리의 끝을 표시하는 빈 심볼이다. 트리는 다음과 같이 보인다:

```
FUNCTION main
  ASSIGN
    INTLIT 1
    IDENT x
  FUNCCALL printf
    STRLIT rval "%d
"
    IDENT rval x
  ASSIGN
    ADD
...
```

이제 6809 도구로 동일한 작업을 수행하면:

```
int printf() id 1: global, 1 params, ctypeid 0, nelems 1 st_posn 0
    char *fmt id 2: param offset 0, size 2, ctypeid 512, nelems 1 st_posn 0
void main() id 3: global, 0 params, ctypeid 0, nelems 0 st_posn 0
int x id 4: local offset 0, size 2, ctypeid 4, nelems 1 st_posn 0
unknown type  id 0: unknown class, size 0, ctypeid 0, nelems 0 st_posn 0
```

그리고

```
FUNCTION main
Unknown dumpAST operator:8745 on line 1 of
```

디버그 코드를 추가하여 두 `detree`의 동작을 확인하면:

```
      Native		    6809 binary
Next ASTnode op 32      Next ASTnode op 32
About to read in a name About to read in a name
We got main     	We got main
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op 29
Next ASTnode op 29      Next ASTnode op -31959
```

확인해보니, 연속으로 `op 29` 노드가 9개 있다. 왜 이 부분을 읽지 못할까? 아마도 stdio 문제일 수 있다. 이 부분은 우리가 읽은 32바이트 레코드 중 9번째이며, 다음과 같이 읽은 것으로 보인다:

```
2B3E: D7 29 D2 28 D2 28 D2 29 D2 00 03 00 5D 00 00 00   .).(.(.)....]...
2B4E: 00 00 00 00 00 00 00 24 85 00 53 00 00 00 00 00   .......$..S.....
이전에는 다음과 같이 읽었다:
2B62: 00 1D 00 00 00 00 00 00 74 A8 00 00 73 F4 00 09   ........t...s...
2B72: 00 0A 00 00 00 13 00 00 00 00 00 00 00 00 00 00   ................
```

그리고 0x001D는 op 29이다.


## 2024년 6월 15일 토요일 14:09:01 AEST


입력 문제로 확인됨. `hd`를 사용한 두 레코드는 다음과 같다:

```
00000100                    1d 00  00 00 00 00 00 74 a8 00
00000110  00 73 f4 00 09 00 0a 00  00 00 13 00 00 00 00 00
00000120  00 00 00 00 00 00 1d 00  00 00 00 00 00 6f 90 00
00000130  00 74 cc 00 0a 00 0b 00  00 00 0e 00 00 00 00 00
00000140  00 00 00 00 00 00
```

파일의 덤프는 다음과 같다:

```
op 1d type 0 ctype 0 rvalue 0 left 74a8 mid 0 right 73f4 nodeid 9
leftid a midid 0 righid 13 sym 0 name NULL symid 0 size 0 linenm 0

op     5823 type   5322 ...
```


## 2024년 6월 16일 일요일 10:24:16 AEST

오늘도 감기 몸살로 다시 시작한다. 버퍼를 헥사 값으로 덤프하도록 코드를 변경했다. `fcc`와 `wcc`를 모두 사용해 봤는데, 여전히 이상한 문자를 출력하는 동일한 문제가 발생한다. 흥미롭게도 `fgetstr()` 함수를 호출하는 코드를 제거하면 정상적으로 동작하며 이상한 문자를 출력하지 않는다.

하. 코드를 처음부터 다시 작성했는데도 여전히 실패한다. 그런 다음 `fgetc()`를 제거하고 `fread()`로 대체했더니 이제 작동한다. 따라서 `fgetc()` 자체가 문제이거나, `fcc`로 컴파일된 코드가 문제일 수 있다. 아니면 `fgetc()`와 `fread()` 간의 상호작용 문제일 수도 있다.


## 2024년 6월 16일 일요일 10:59:40 AEST


아래 내용을 확인하세요...

덕분에 많은 도움이 되었습니다! 이제 더 많은 테스트를 통과할 수 있게 되었습니다. 테스트 1부터 10까지는 성공했고, 11번은 실패했습니다. 12번부터 21번까지는 성공했고, 22번은 실패했습니다. 23번부터 25번까지는 성공했습니다.

어셈블리 파일을 비교해서 차이점을 확인할 수 있을 것 같습니다. 실제로 AST와 심볼 파일도 확인할 수 있습니다. AST 파일은 문제가 없지만, 심볼 파일에서는 다음과 같은 차이를 발견했습니다:

```
$ diff sym 6sym
2c2
<     char *fmt id 2: param offset 0, size 2, ctypeid 0, nelems 1 st_posn 0
---
>     char *fmt id 2: param offset 0, size 2, ctypeid 512, nelems 1 st_posn 0
4,7c4,7
< int i id 4: local offset 0, size 2, ctypeid 0, nelems 1 st_posn 0
< char j id 5: local offset 0, size 1, ctypeid 0, nelems 1 st_posn 0
< long k id 6: local offset 0, size 4, ctypeid 0, nelems 1 st_posn 0
< unknown type k id 0: unknown class, size 0, ctypeid 0, nelems 0 st_posn 0
---
> int i id 4: local offset 0, size 2, ctypeid 4, nelems 1 st_posn 0
> char j id 5: local offset 0, size 1, ctypeid 5, nelems 1 st_posn 0
> long k id 6: local offset 0, size 4, ctypeid 6, nelems 1 st_posn 0
> unknown type  id 0: unknown class, size 0, ctypeid 0, nelems 0 st_posn 0
```

그리고 ctypeid가 모두 잘못되었습니다. 실제 심볼 파일에서도 이 문제를 확인했습니다. `newsym()` 함수에 디버그 코드를 추가해서 `ctype` 포인터가 어떻게 설정되는지 확인했습니다. 네이티브 컴파일러에서는 다음과 같습니다:

```
newsym printf ctype (nil) ctypeid 0
newsym fmt ctype (nil) ctypeid 0
newsym main ctype (nil) ctypeid 0
newsym i ctype (nil) ctypeid 0
newsym j ctype (nil) ctypeid 0
newsym k ctype (nil) ctypeid 0
```

6809 버전에서는 다음과 같습니다:

```
newsym printf ctype 00000 ctypeid 0
newsym fmt ctype 00003 ctypeid 512
newsym main ctype 00000 ctypeid 0
newsym i ctype 00002 ctypeid 4
newsym j ctype 00002 ctypeid 5
newsym k ctype 00002 ctypeid 6
```

그리고 2와 3이 실제 메모리 포인터일 리가 없다고 생각합니다!


## 2024년 6월 16일 일요일 오전 11:52:03 AEST

몇 가지 printf 출력을 확인한 결과:

```
d_l B2 ctype 00000
s_d ctype 00002
newsym i ctype 00002 ctypeid 4
```

`ctype`을 `symbol_declaration()`에 제대로 전달하지 못하고 있는 것으로 보인다. 다음 코드가 문제가 있어 보인다:

```
        leax 6,s
        tfr x,d
        ldd [10,s]
        pshs d
        ldd 14,s
        pshs d
        pshs d		<-- D 값을 변경하지 않고 두 번 push?!
        ldd 8,s
        pshs d
        lbsr _symbol_declaration
        leas 8,s
```

C 코드는 다음과 같다:

```
sym = symbol_declaration(type, *ctype, class, &tree);
```

`*ctype` 값을 제대로 로드하지 않고 push하고 있는 것 같다. AST 트리는 다음과 같다:

```
      FUNCCALL symbol_declaration
        IDENT rval type
        DEREF rval
          IDENT rval ctype
        IDENT rval class
        ADDR tree
```

트리 내에 다른 DEREF IDENT가 많이 있지만, 이 경우에만 deref를 수행하지 않는다.

`cgcall()`에 두 개의 위치가 D_REGS로 표시된 상태로 들어가는 것 같다:

```
(gdb) p Locn[0]
$4 = {type = 7, name = 0x0, intval = 0, primtype = 3}   <===
(gdb) p Locn[1]
$5 = {type = 2, name = 0x0, intval = 12, primtype = 2}
(gdb) p Locn[2]
$6 = {type = 7, name = 0x0, intval = 0, primtype = 3}   <===
(gdb) p Locn[3]
$7 = {type = 2, name = 0x0, intval = 2, primtype = 2}
```

문제를 파악한 것 같다. `gen_funccal()`에서 모든 인자 값을 얻기 위한 코드를 생성한다:

```
  for (i = 0, gluetree = n->left; gluetree != NULL; gluetree = gluetree->left) {
    // 표현식의 값을 계산
    arglist[i] =
      genAST(gluetree->right, NOLABEL, NOLABEL, NOLABEL, gluetree->op);
    typelist[i++] = gluetree->right->type;
  }
```

하지만 `cg6809`의 여러 함수가 L_DREG 위치를 할당한다: `cgwiden()`, `cgaddress()`, `cgderef()`. 그리고 다음 코드에서:

```
sym = symbol_declaration(type, *ctype, class, &tree);
```

`tree`의 주소를 얻고 `ctype`을 dereference하고 있다! 따라서 두 개의 위치가 L_DREG라고 생각하고 있다.


## 2024년 6월 16일 일요일 13:23:23 AEST

위의 내용에서 `fgets()`를 `fread()`로 변경한 부분이 테스트를 실패하게 만들었기 때문에, 일단 변경 사항을 되돌리고 새 코드를 RCS에 보관해 두었다.

`cgderef()`에 `stash_d()`를 추가했고, 이제 어셈블리 코드는 다음과 같다:

```
        leax 6,s
        tfr x,d
        std R0+0	&tree를 R0에 저장
        ldx 10,s
        ldd 0,x
        std R1+0	*ctype를 R1에 저장
        ldd R0+0	&tree를 스택에 푸시
        pshs d
        ldd 14,s	class를 스택에 푸시
        pshs d
        ldd R1+0	*ctype를 스택에 푸시
        pshs d
        ldd 8,s		type를 스택에 푸시
        pshs d
        lbsr _symbol_declaration
```

이 코드는 다음과 같이 최적화된다:

```
        leax 6,s
        stx R0+0	&tree를 R0에 저장
        ldd [10,s]
        std R1+0	*ctype를 R1에 저장
        ldd R0+0
        pshs d		&tree를 스택에 푸시
        ldd 14,s
        pshs d		class를 스택에 푸시
        ldd R1+0
        pshs d		*ctype를 스택에 푸시
        ldd 8,s
        pshs d		type를 스택에 푸시
        lbsr _symbol_declaration
```


## 2024년 6월 16일 일요일 13:54:28 (AEST)

`ctype` 버그는 해결했다. 하지만 `input011` 문제는 여전히 남아 있다.  
적어도 이제 트리와 심볼 테이블은 동일하다.

컴파일러가 long 타입 리터럴의 하위 값을 올바르게 생성하지 않는 것 같다:

```
$ diff goodqbe badqbe 
49c49
< 	ldy #0
---
> 	ldy #30
150c150
< 	ldy #0
---
> 	ldy #1
158c158
< 	cmpy #0
---
> 	cmpy #5
189,190c189,190
< 	adcb #0
< 	adca #0
---
> 	adcb #1
> 	adca #1
```

특히, 상위 절반을 사용해야 할 때 하위 절반을 사용하는 것으로 보인다.

이 문제는 `printlocation()`에서 상수 처리를 할 때 발생하며, 여기서만 오른쪽 시프트를 수행한다.  
바이트 단위 처리를 시도해 보겠다. 이미 종이에 작성해 둔 내용을 바탕으로 수정했다.  
이제 테스트 31까지는 통과하지만, 실제로는 오류 테스트다.  
현재 테스트 58에서 실패한다 :-)


## 2024년 6월 16일 일요일 15:09:58 AEST

에러 메시지는 `main의 15번째 줄에서 cgprimtype::0에 잘못된 타입이 있습니다`이다.  
심볼 테이블은 정상이지만, 트리 구조에서 INTLIT 값이 다르게 나타난다:

```
  정상		   문제
FUNCTION main   FUNCTION main
  ASSIGN          ASSIGN
    INTLIT 12       INTLIT 12
    DEREF           DEREF
      ADD             ADD
        ADDR var2       ADDR var2
        INTLIT 0        INTLIT 48	<==
  FUNCCALL printf FUNCCALL printf
    STRLIT rval "%d"        STRLIT rval "%d"
    DEREF rval      DEREF rval
      ADD             ADD
        ADDR var2       ADDR var2
        INTLIT 0        INTLIT 48	<==
```

이는 구조체의 멤버 오프셋 값이다.

이 값들은 파일에 존재하며, detree를 위해 몇 가지 printf를 추가했다:

```
INTLIT 노드 값 12	INTLIT 노드 값 12
INTLIT 노드 값 0	INTLIT 노드 값 48
INTLIT 노드 값 0	INTLIT 노드 값 48
INTLIT 노드 값 99	INTLIT 노드 값 99
INTLIT 노드 값 2	INTLIT 노드 값 48
INTLIT 노드 값 2	INTLIT 노드 값 48
INTLIT 노드 값 4005	INTLIT 노드 값 4005
INTLIT 노드 값 3	INTLIT 노드 값 48
INTLIT 노드 값 3	INTLIT 노드 값 48
INTLIT 노드 값 0	INTLIT 노드 값 48
INTLIT 노드 값 2	INTLIT 노드 값 48
INTLIT 노드 값 3	INTLIT 노드 값 48
INTLIT 노드 값 0	INTLIT 노드 값 48
INTLIT 노드 값 2	INTLIT 노드 값 48
INTLIT 노드 값 3	INTLIT 노드 값 48
INTLIT 노드 값 0	INTLIT 노드 값 0
```

`expr.c`에서 INTLIT 노드를 생성하는 부분을 확인했다:

```
right = mkastleaf(A_INTLIT, cgaddrint(), NULL, NULL, m->st_posn);
```

여기서 네이티브와 6809 버전 모두 올바른 위치를 사용한다.  
그런데 어떻게 값이 48로 변질될까? `mkastleaf()` 이후 값을 출력해보니 48이 나온다!

`mkastleaf()`가 48을 반환하고 있다!

또 다른 이중 푸시 문제가 발견됐다:

```
        ldx 4,s
        ldd 20,x	<- 하지만 m->st_posn을 저장하지 않음
        lbsr _cgaddrint
        pshs d		<- cgaddrint() 결과를 푸시
        ldd #0
        pshs d		<- NULL을 푸시
        ldd #0
        pshs d		<- NULL을 푸시
        pshs d
        ldd #26
        pshs d		<- A_INTLIT을 푸시
        lbsr _mkastleaf
```

질문: 왜 `cgaddrint()` 결과가 NULL보다 먼저 푸시되는가?  
`cgcall()`에서 D 저장을 놓쳤다. 이제 수정했다.


## 2024년 6월 17일 월요일 07:49:35 AEST


이 부분은 input130에서 실패한다. 현재 다음과 같은 코드를 실행 중이다.

```
printf("Hello " "world" "\n");
```

AST 트리는 다음과 같다.

```
FUNCTION 
  FUNCCALL printf
    STRLIT rval \"Hello \"
  RETURN
    INTLIT 0
```

이는 올바르지 않다. 세 개의 문자열이 토큰 스트림에 존재한다.

참고: `printf("My name is \"Warren\" you know!\n");`. `gcc`는 `My name is "Warren" you know!`를 출력한다. 하지만 내 컴파일러는 `My name is \"Warren\" you know!`를 출력한다. 이는 내 컴파일러가 리터럴을 잘못 해석하고 있다는 의미이므로, 이를 수정해야 한다.

이 부분을 고쳤지만, 여전히 테스트 130은 실패한다.


## 2024년 6월 17일 월요일 오전 08:26:02 AEST

네이티브 `wcc`는 문자열을 잘 연결한다:

```
First strlit >foo< totalsize 3 >foo< litend 0x56267086dfe3
First strlit >Hello < totalsize 6 >Hello < litend 0x56267086e196
Next strlit >world< totalsize 11 >Hello world< litend 0x56267086e19b
Next strlit >
< totalsize 12 >Hello world
< litend 0x56267086e19c
```

하지만 6809 버전은 그렇지 않다:

```
First strlit >Hello < totalsize 6 litval 08574 >Hello < litend 0857a
Next strlit >world< totalsize 11 litval 08564 >Hello < litend 0857f
Next strlit >
< totalsize 12 litval 08090 >Hello < litend 08580
```

이제 문제를 해결했다. 하지만 테스트 135가 실패한다. 또한 실패하는 다른 테스트는 162뿐이다.

심볼 테이블과 AST 트리는 정상적으로 보인다. 코드 생성기가 충돌하는 것 같다. 디버그 트레일을 보면 `cgcall()`에 있다. `ptrtype()`은 1을 반환하고 `cgprimsize()`는 2를 반환한다. 그런 다음 `fprintf()`를 실행한다. 따라서 다음 코드를 실행하는 중일 것이다:

```
  // 함수를 호출하고 스택을 조정한다
  fprintf(Outfile, "\tlbsr _%s\n", sym->name);
```

에뮬레이터를 단계별로 실행하면 `sym->name`은 정상이며 "printf"를 가리킨다. 문자열 리터럴도 정상적으로 보인다. 그리고 $765E를 푸시하는데, 맵 파일에 따르면 이 값은 Outfile이다. 따라서 우리 코드가 잘못된 것 같지는 않다.

`vfnprintf()`로 진입하고 '%'에 대한 스위치 문에서 B는 0x73, 즉 's'를 가지고 있다. 다음 코드까지 진행된다:

```
   __switchc+0005: CMPB ,X+         | -FHI-Z-- 00:73 7572 0001 0000 FC89
   __switchc+0007: BEQ $6410        | -FHI-Z-- 00:73 7572 0001 0000 FC89
   __switchc+000F: LDX ,X           | -FHI---- 00:73 0007 0001 0000 FC89
   __switchc+0011: JMP ,X           | -FHI---- 00:73 0007 0001 0000 FC89
0007: LSR <$00         | -FHI---C 00:73 0007 0001 0000 FC89
```

그리고 $0007로의 점프는 분명히 잘못되었다! 다음은 스위치 테이블의 덤프다:

```
7536: 00 14 00 55 5D 2D 55 6B 20 55 73 2B 55 73 23 55
7546: 7C 2A 55 84 2E 55 C2 6C 55 CA 68 55 D2 64 55 D9
7556: 69 55 D9 62 56 27 6F 56 2E 70 56 35 58 56 4E 78
7566: 56 55 75 56 5A 21 57 61 63 57 7F 73 00 07 58 06
7576: FD 96 00 05 FF FF 00 00 00 00 00 00 00 04 00 02
```

이를 조금 다시 작성해 보면:

```
00 14     $14 (20 entries)
00 555D   0: '\0'
2D 556B   1: '-'
20 5573   2: ' '
2B 5573   3: '+'
23 557C   4: '#'
2A 5584   5: '*'
2E 55C2   6: '.'
6C 55CA   7: 'l'
68 55D2   8: 'h'
64 55D9   9: 'd'
69 55D9  10: 'i'
62 5627  11: 'b'
6F 562E  12: 'o'
70 5635  13: 'p'
58 564E  14: 'X'
78 5655  15: 'x'
75 565A  16: 'u'
21 5761  17: '!'
63 577F  18: 'c'
73 0007  19: 's'  <== 덮어쓰인 것 같다
   5806  default
```

점프는 $57AC로 가야 한다. 몇 가지 쓰기 중단점을 추가해 보니... `load_d()`의 끝 부분에 문제가 있는 것 같다:

```
     _load_d+$00C8: LBSR _mul
     _load_d+$00CB: ADDD #$757C     base of Locn[]
     _load_d+$00CE: TFR D,X
     _load_d+$00D0: LDD #$0007      
     _load_d+$00D3: STD ,X

이는   Locn[l].type= L_DREG;
```

디버그 트레이스는 `load_d()`가 -1(NOREG)로 호출되고 있음을 보여준다! 이전에 `genAST()`에서 0x22(A_RETURN)에 대한 `switch`를 실행하고 있다. `genAST()`는 NOREG를 로드하고 `cgreturn()`을 호출한다.

네이티브 버전에서는:

```
Breakpoint 1, cgreturn (l=-1, sym=0x555555566200) at cg6809.c:1211
(gdb) s
load_d (l=-1) at cg6809.c:154
```

`load_d()`가 NOREG를 받고 있다! 그래서 `cgreturn()`에 NOREG일 때 D를 로드하지 않도록 코드를 추가했다. 이제 테스트 135가 통과한다. 그리고 테스트 162도 통과한다. 이제 모든 테스트가 통과한다. 성공!


## 2024년 6월 17일 월요일 10:13:33 AEST

컴파일러를 자체적으로 빌드할 때 남은 문제는 다음과 같다:

- cpeep.c 파일의 95번 줄에서 바이너리 표현식에 호환되지 않는 타입이 발생
- run_command 파일의 180번 줄에서 genAST() 함수 내 A_ASSIGN을 사용할 수 없음, op:1

첫 번째 문제는 포인터 뺄셈과 관련된 것이다.
두 번째 문제는 Fuzix에서 `stdin`이 포인터가 아닌 배열로 정의되어 있어, 배열에 포인터를 할당할 수 없기 때문인 것 같다. `freopen()`이 이미 들어오는 파일 핸들을 재설정하므로 할당할 필요가 없다. 이 문제는 해결되었다.

이제 char 포인터 뺄셈에 대한 임시 해결책을 적용할 수 있다. 이를 구현하고 테스트를 추가했다. 모든 테스트가 통과되었다: triple 테스트, QBE, 6809, 그리고 6809로 컴파일된 컴파일러. 이제 6809에서 triple 테스트를 통과하기만 하면 된다!


`cpeep.c` 파일의 다음 라인에서 컴파일러가 오류를 발생시키는 것 같다:

```
next = &((*next)->o_next);
```

컴파일러는 `&` 연산자 뒤에 식별자가 와야 한다고 지적한다. 이 라인을 다시 작성해야 할 것 같다.

이 문제를 해결하려면, 포인터 역참조와 주소 연산자를 올바르게 사용해야 한다. 현재 코드는 `(*next)->o_next`의 주소를 가져오려고 시도하지만, 이는 올바른 문법이 아니다. 대신, `next` 포인터를 올바르게 업데이트하는 방식으로 코드를 수정할 수 있다.

예를 들어, 다음과 같이 작성할 수 있다:

```
next = &(*next)->o_next;
```

이렇게 하면 `*next`가 가리키는 구조체의 `o_next` 멤버의 주소를 `next`에 할당한다. 이는 컴파일러가 요구하는 문법을 충족시키면서도 원래의 의도를 유지한다.

만약 여전히 문제가 발생한다면, 코드의 컨텍스트를 더 자세히 살펴보고 `next`와 `o_next`의 타입을 확인하는 것이 좋다. 이는 올바른 포인터 연산을 보장하는 데 도움이 될 것이다.


## 2024년 6월 17일 월요일 14:52:55 AEST

이런. `smake`라는 스크립트를 사용해 가장 작은 파일부터 순서대로 작업 중이다. 이제 `types.c`를 실행했더니 다음과 같은 오류가 발생했다:

```
$ ./smake types.c
types.c 처리 중
types.c 파일의 103번째 줄에서 mkastnode() 함수 내에서 메모리 할당(malloc) 실패
```

하지만 다른 파일들은 문제없이 컴파일되었다:
tstring.c, targ6809.c, tree.c, detok.c, opt.c. 단지 사소한 차이점이 하나 있다:

```
$ diff detok.s S/detok.s 
536c536
< 	ldd #65535
---
> 	ldd #-1
```

이 차이는 괜찮을 것 같다. 실제로 두 버전에서 생성된 `.o` 파일의 체크섬이 동일하다.

이제 메모리 절약 문제를 해결해야 한다!


## 2024년 6월 17일 월요일 15:29:42 AEST

이런 아이디어가 떠올랐다. 어떤 서브트리가 완전한 문장이고 유용한 rvalue가 없다는 것을 알 때, 그 서브트리를 직렬화한 다음 해제할 수 있지 않을까?

방금 `dumptree.c`를 작성했다. 이 프로그램은 단순히 AST 노드를 읽어서 덤프한다. `types.c`의 103번째 줄에서 231번째 노드까지 진행 중이며, `modify_type()` 함수는 76번째 노드에서 시작한다. 6809 AST 노드의 크기는 32바이트이므로 총 4,992바이트를 차지한다.

AST 노드를 `free()`하는 방법에 대해서도 고민했다. `serialiseAST()`와 `opt.c` 함수들에서 몇 번 시도해봤지만 성공하지 못했다. 아쉽다.


## 2024년 6월 18일 화요일 08:52:10 AEST

6809 에뮬레이터에 `brk()/sbrk()` 디버그 코드를 추가해 힙이 얼마나 빠르게 증가하는지 확인했다. `malloc()`을 호출하는 부분을 살펴보니, `types.c`를 컴파일할 때 힙이 부족해지기 전까지 `malloc()`이 호출된 횟수는 다음과 같다:

```
    241 mkastnode
    460 newsym
      5 serialiseSymtab
    928 strdup
```

`decl.c`에서 `strdup()`으로 인한 메모리 누수를 줄이기 위해 몇 가지 `free()`를 추가했다. 네이티브 컴파일러로 확인해보니, 심볼 테이블에 460개의 심볼이 있었다. 놀랍다!


## 2024년 6월 18일 화요일 09:27:49 AEST

아! 컴파일러가 구조체 안에 공용체(union)를 포함할 수 있다는 걸 깜빡했네요.

```
typedef union { int x; char c; } fred;
typedef struct { int foo; char *name; fred combo; } jim;
jim mynode;

int main() {
  mynode.foo= 12; mynode.name="Dennis"; mynode.combo.x=17;
  printf("%d %s %d\n", mynode.foo, mynode.name, mynode.combo.x);
  return(0);
}
```

이걸 활용하면 데이터 구조를 좀 더 효율적으로 만들 수 있을지도 모르겠네요. 심볼 테이블에 적용해봤는데 성공하지 못했어요. 아쉽네요.


## 2024년 6월 19일 수요일 10:13:08 (AEST)

심볼 테이블에 대한 좋은 아이디어를 생각해냈다.

먼저, 파싱 중에 비-로컬/파라미터 심볼은 가능한 한 빨리 심볼 테이블 파일에 쓰고, 메모리에서 해제한다.

단일 인-메모리 심볼 테이블을 유지한다. 이 테이블에는 `findXXX()` 연산으로 불러온 심볼 테이블 파일의 심볼만 포함된다.

글로벌 컨텍스트에서 이 인-메모리 테이블은 다른 것들이 사용하는 typedef 등을 보관한다. 예를 들어:

```
extern FILE *stdin;	// FILE의 크기를 알아야 함
```

함수 컨텍스트에서는, 이 테이블에 함수가 사용하는 모든 심볼(파라미터와 로컬 변수 포함)을 보관한다.

각 함수의 시작 부분에서 리스트를 해제하여 글로벌 파싱 중에 쌓인 내용을 제거할 수 있다. 각 함수의 끝에서는 모든 로컬 변수와 파라미터를 파일에 플러시한 다음 리스트를 해제한다.

이렇게 하면 인-메모리 심볼 테이블을 비교적 작게 유지하면서도 파일을 다시 읽어야 하는 횟수를 최소화할 수 있다.

기존 `sym.c` API의 대부분 또는 전부를 유지하면서, free/flush 작업을 수행할 몇 가지 새로운 함수를 추가할 수 있을 것 같다.

이 작업을 위해 새로운 git 브랜치 `symtable_revamp`를 만들었다. 기존 컴파일러 바이너리는 `/opt/wcc`에 설치했다. 이 작업이 많은 부분을 깨뜨릴 것이므로, 새로운 설치 경로 `/opt/nwcc`를 만들었다. 이렇게 하면 새 코드가 생성한 심볼 테이블과 트리를 기존 것과 비교할 수 있다.


## 2024년 6월 19일 수요일 18:00:54 AEST

새로운 코드 작업을 시작했다. 아직 완성되지 않았고 컴파일도 되지 않는다. 하지만 이 접근 방식이 괜찮다고 생각한다. 심볼을 추가하는 코드는 완성했고, 이제 심볼을 찾는 코드를 작성 중이다. 디버깅을 통해 어떤 실수를 했고 어떤 상황을 예상하지 못했는지 알아보는 것이 흥미로울 것이다.


## 2024년 6월 21일 금요일 07:39:23 AEST

코드가 컴파일되고 실행되는 단계까지 왔다. 물론 아직 제대로 작동하지는 않는다. 단지 알려진 정상적인 심볼 테이블을 얻기 위해 다음과 같이 실행했다:

```
$ wcc -m6809 -S -X gen.c
$ /opt/wcc/bin/desym gen.c_sym > goodsym
```

이제 새 코드로 같은 작업을 수행하면 결과를 비교할 수 있다. 코드에는 여러 `fprintf()`를 추가했다.

현재 코드는 함수 매개변수를 함수 심볼의 멤버로 제대로 연결하지 않는다. 이 문제는 해결할 수 있다. 함수 심볼에 지역 변수도 추가하는 것을 고민 중이다. 코드 생성 부분에서 두 리스트를 모두 순회하는 곳이 있기 때문에, 함수 심볼에 연결된 상태로 두는 것이 합리적일 것이다.


2024년 6월 22일 토요일 12:58:09 AEST

어느 정도 진행은 되었지만, 또 다른 문제에 부딪힌 것 같다. 현재 두 개의 메모리 내 심볼 리스트가 존재한다. 하나는 타입(struct, union, enum, typedef)을 위한 것이고, 다른 하나는 변수와 함수를 위한 것이다. 멤버를 가진 것들(struct, union, enum, function)은 관련된 멤버(로컬 변수, 함수의 파라미터)를 임시 리스트에 추가하고 이를 심볼에 연결한다.

이제 문제는 이렇다. 원래 컴파일러에서 다음과 같은 "클래스" 리스트를 정의했다:

```
// 스토리지 클래스
enum {
  C_GLOBAL = 1,                 // 전역적으로 보이는 심볼
  C_LOCAL,                      // 로컬에서 보이는 심볼
  C_PARAM,                      // 로컬에서 보이는 함수 파라미터
  C_EXTERN,                     // 외부 전역 심볼
  C_STATIC,                     // 한 파일 내에서만 보이는 정적 심볼
  C_STRUCT,                     // 구조체
  C_UNION,                      // 공용체
  C_MEMBER,                     // 구조체 또는 공용체의 멤버
  C_ENUMTYPE,                   // 명명된 열거형 타입
  C_ENUMVAL,                    // 명명된 열거형 값
  C_TYPEDEF,                    // 명명된 typedef
  C_STRLIT                      // 클래스가 아님: 문자열 리터럴을 나타냄
};
```

하지만 이는 두 가지 개념을 혼합하고 있다. 예를 들어, `static struct foo { ... };`와 같이 `foo` 타입이 이 파일 내에서만 보이도록 할 수 있다.

각 심볼 테이블에 두 가지 값을 저장해야 한다. 하나는 심볼이 struct/union/enum/typedef인지 여부를 나타내고, 다른 하나는 심볼의 가시성(global, extern, static)을 나타낸다.

이미 구조적 타입은 다음과 같이 정의되어 있다:

```
enum {
  S_VARIABLE, S_FUNCTION, S_ARRAY
};
```

그래서 여기에 struct/union/enum/typedef를 추가할 수 있다. 또한 strlit도 추가할까? 그런 다음 global/extern/static/local/member/param/enumval을 스토리지 클래스로 유지하고, 이를 가시성으로 이름을 바꿀 수 있다.

마지막 네 가지는 항상 `member` 리스트에 속하며, 실제 가시성은 부모 심볼에 의해 결정된다.

또한 열거형에 대해 주의할 점은, 열거형 값은 특정 열거형 타입에 속하지 않는다는 것이다. 또한, 열거형 타입 이름은 실제로 아무 역할을 하지 않지만, 재정의를 방지해야 한다. 따라서 열거형 타입은 타입으로, 열거형 값은 전역 심볼로 메인 심볼 테이블에 저장할 수 있다.


## 2024년 6월 23일 일요일 10:50:15 AEST

위 작업을 진행했고, 천천히 목표에 다가가고 있다. 현재는 전역 프로토타입, 열거형, 구조체 등을 모두 읽어오는 데 성공했다. 하지만 매개변수가 있는 첫 번째 함수를 만나면 오류가 발생한다. 이 문제를 해결하는 데 큰 어려움은 없길 바란다.


## 2024년 6월 24일 월요일 09:41:38 AEST

음... 문제가 있는 것 같다. 예를 들어 보자:

```
int fred(int a, int b, int c);

int fred(int x, int y, int z) {
  return(x+y+x);
}
```

프로토타입은 `a, b, c`로 심볼 테이블 파일에 기록된다. 이제 실제 함수가 다른 매개변수 이름으로 나타난다. 기존 멤버 목록을 제거하고 새로운 매개변수를 추가할 수 있지만, 이는 디스크에 기록되지 않는다. 또는 이를 디스크에 기록하면 동일한 함수에 대해 심볼 테이블에 두 개의 항목이 생긴다. 새로운 변수 이름이 기존 이름보다 길 수 있기 때문에 새로운 이름으로 패치할 수도 없다. 이런!

디스크에 있는 심볼을 무효화하는 함수를 추가하는 것을 고려하고 있다. 기존의 `findSyminfile()`을 사용해 심볼을 다시 찾은 다음, `id`에 -1을 기록해 무효화할 수 있다. 그런 다음 새로운 심볼을 기록할 수 있다. 이 방법은 깔끔하지 않지만, 지금으로서는 더 나은 방법이 떠오르지 않는다.


2024년 6월 24일 월요일 15:14:55 AEST

유효하지 않은 코드를 추가했다. 동작하는 것 같다. 현재 코드는 심볼을 출력하지만 Member 포인터를 NULL로 설정하지 않아서, 구조체의 멤버를 변수로 사용하는 함수가 생성되고 있다 :-)


## 2024년 6월 25일 화요일 오전 9:12:08 (AEST)

디스크에서 심볼과 그 멤버를 로드하고 있었는데, 작업을 마친 후 Memb 포인터를 NULL로 초기화하는 것을 잊었다. 또한 심볼을 디스크에 여러 번 쓰고 있다는 것을 발견했다. 필요할 때 로드한 후, 함수가 끝날 때 다시 디스크에 쓰고 있었다. 이런!

이제 `gen.c` 파일의 이 라인까지 진행했다:

```
type = n->left->type;
```

그리고 다음과 같은 오류 메시지가 발생했다:

```
구조체/공용체에서 멤버를 찾을 수 없음: :type, gen.c 파일 200번째 줄
```

여기까지는 이전과 새로운 파스 트리가 동일해서 다행이다!


## 2024년 6월 25일 화요일 09:35:21 AEST

심볼 ID로 검색 기능을 추가해야 할 것 같다. 메모리 내 심볼 테이블을 덤프하는 코드를 추가했다. 출력 결과는 다음과 같다:

```
struct ASTnode: struct id 302: global, ctypeid 0, nelems 0 st_posn 0
    int op id 303: member, size 2, ctypeid 0, nelems 1 st_posn 0
    int type id 304: member, size 2, ctypeid 0, nelems 1 st_posn 2
!!! struct exit *ctype id 305: member, size 2, ctypeid 287, nelems 1 st_posn 4
    int rvalue id 306: member, size 2, ctypeid 0, nelems 1 st_posn 6
!!! struct nodeid *left id 307: member, size 2, ctypeid 302, nelems 1 st_posn 8
!!! struct nodeid *mid id 308: member, size 2, ctypeid 302, nelems 1 st_posn 10
!!! struct nodeid *right id 309: member, size 2, ctypeid 302, nelems 1 st_posn 1
```

`!!!` 표시는 `ctype` 포인터가 잘못된 타입을 가리키고 있음을 나타낸다. 디스크에서 심볼을 로드할 때, 심볼과 그 멤버들을 함께 로드한다. `ctype`이 NULL이 아니라면, 해당 `ctypeid`와 일치하는 심볼을 찾아 다시 연결해야 한다. 이런 실수를 하다니!


2024년 6월 25일 화요일 11:27:27 AEST

기호를 ID로 로드하는 코드를 추가했고, `ctype` 필드를 해당 기호와 연결했다. 이제 `gen.c` 파일의 마지막 함수까지 진행했으며, 그 시점까지 트리가 동일하다. 아직 기호 파일에 동일한 기호를 반복해서 쓰는 문제는 처리하지 않았다.

`gen.c`의 마지막 함수 문제를 찾았다. 중복 매개변수를 확인하기 전에 Functionid를 NULL로 설정해야 했다. 이렇게 하지 않으면 이전 함수의 매개변수와 비교하게 된다.

이제 디스크 기호 테이블 이전의 컴파일러와 동일한 AST 트리를 생성한다. 성공!


## 2024년 6월 25일 화요일 12:21:45 AEST

디스크 상의 중복 심볼 문제를 해결했다. ID를 순차적으로 할당하기 때문에, 마지막으로 기록한 가장 높은 ID를 저장하고 다음 작업에서 이 ID 이하의 값은 다시 쓰지 않도록 했다.

이제 다른 C 파일들을 파싱하려고 하는데 문제가 발생한다. 한숨만 나온다. 테스트 파일들을 파싱해보려고 했는데, 몇 개만 문제가 생긴다.

문제가 발생한 경우는 함수 호출에 인자가 없을 때였다. 함수에 매개변수는 없지만 지역 변수가 있는 경우였다. 모든 지역 변수는 매개변수 뒤에 있는 멤버 리스트에 있기 때문이다. 첫 번째 지역 변수를 만났을 때 멈추도록 로직을 수정해야 했다.


## 2024년 6월 26일 수요일 10:02:50 AEST

이제 모든 테스트를 파싱할 수 있게 되었으니, 생성기도 실행해 보기로 했다. 생성기 코드를 컴파일하는 데 성공했고, 이제는 이전 컴파일러와 새로운 컴파일러의 어셈블리 출력을 비교 중이다.


## 2024년 6월 26일 수요일 12:33:58 AEST

아, 전역 변수에 대한 데이터를 생성하지 않았군요. 코드는 거의 완성되었지만, 아직 10개의 테스트가 실패하고 있습니다. 정말 답답하네요!


2024년 6월 26일 수요일 14:29:00 AEST

현재 전역 문자열 리터럴을 처리하는 방법을 고민 중이다. 이 리터럴들은 변수에 할당된 후 심볼 테이블에 나타난다. 해결 방법을 찾을 수 있을 것 같다. 문자열 리터럴을 먼저 출력하고, 그 다음에 비 문자열 리터럴을 출력하면 될 것이다. 6809 테스트는 모두 통과했다. QBE 테스트 중 일부는 실패하는데, 이 문제는 문자열 리터럴과 관련이 있어 보인다. 이 부분을 수정해야 한다.


## 2024년 6월 28일 금요일 14:06:37 AEST

`cgqbe.c`에서 문자열 값을 `strdup()`로 복제해야 했다. 이제 QBE와 6809에서 모든 테스트가 통과한다. 하지만 현재 파서는 여전히 컴파일러 자체 코드를 처리하지 못하고 있다.

다음은 `wcc.c`에서 실패하는 코드다:

```
void addtmpname(char *name) {
  struct filelist *this;
  ...
}
```

파서에서 `addtmpname()`의 프로토타입을 메모리 내 심볼 테이블에 추가했다(즉, 매개변수와 함께). 이제 함수 본문에 들어가 있으며, `struct filelist` 타입을 찾고 있다. 이 타입은 메모리에서 제거된 상태라 다시 불러와야 한다.

이 과정에서 `Membhead`와 `Membtail`을 NULL로 설정했다. 왜냐하면 `struct filelist`를 불러온 후 그 멤버들을 읽어야 했기 때문이다. 하지만 `name` 매개변수 뒤에 로컬 변수 `this`를 추가하려면 멤버 리스트가 필요했다. 이런!


## 2024년 6월 29일 토요일 08:15:25 AEST

`loadSym()` 함수가 파일에서 심볼을 읽어올 때 사용할 전용 리스트를 추가해 문제를 해결했다. 이제 다음 문제로 넘어가자.

`scan.c` 파일에 `c = next();`라는 코드가 있다. 여기서 `next` 심볼을 변수로 찾고 있다. 우리는 다음 코드를 호출한다:

```
res = loadSym(sym, name, stype, id, 0, 1); // stype NOTATYPE, id 0
```

그런데 이 함수는 `next`라는 이름의 함수가 아니라 구조체 멤버를 반환하고 있다. 따라서 `loadSym()` 함수에 논리적 버그가 있다는 것을 알 수 있다.

맞다. `loadSym()`을 사용할 때 매칭되는 심볼이 없으면 초기화 리스트를 건너뛴다. 하지만 구조체나 유니온 타입인 경우에는 멤버를 건너뛰지 않는다. 그래서 `loadSym()`을 빠져나갔다가 다시 돌아와 멤버를 읽어와 이름과 비교한다.

따라서 구조체나 유니온 타입을 로드하되 멤버는 건너뛰는 방법을 찾아야 한다. 이 작업은 쉽지 않았다. 만약 이름을 검색 중이라면, 그것은 지역 변수나 매개변수, 멤버가 아니다. 첫 두 가지는 `findlocl()` 함수로 찾을 수 있고, 멤버는 `loadSym()` 함수 자체에서 재귀적으로 로드된다. IF 문에 조건을 하나 더 추가했다. 이제 `scan.c` 파일을 파싱할 수 있지만, 코드 생성기가 중단된다.

현재 발생 중인 오류 목록은 다음과 같다:

```
cg6809.c 6809: 중복된 지역 변수 선언: cmplist, cg6809.c 파일 1028번 줄
cgqbe.c: ID 549에 해당하는 심볼을 찾을 수 없음, cgpreamble 파일 71번 줄
cpeep.c: & 연산자 뒤에 식별자가 와야 함, cpeep.c 파일 155번 줄
decl.c: ID 503에 해당하는 심볼을 찾을 수 없음, (null) 파일 0번 줄
desym.c: 알 수 없는 변수 또는 함수: dumpsym, desym.c 파일 221번 줄
expr.c: 알 수 없는 변수 또는 함수: binexpr, expr.c 파일 156번 줄
gen.c: ID 511에 해당하는 심볼을 찾을 수 없음, Lfalse 파일 30번 줄
scan.c: ID 350에 해당하는 심볼을 찾을 수 없음, (null) 파일 0번 줄
stmt.c: 알 수 없는 변수 또는 함수: if_statement, stmt.c 파일 355번 줄
sym.c: 타입 불일치: 리터럴 vs. 변수, sym.c 파일 26번 줄
```

첫 번째 오류는 6809 아키텍처에만 해당하는 문제다. 나머지 오류들(`cpeep.c` 제외)은 무효화된 심볼이 새로운 정의로 대체되지 않아 발생한 것으로 보인다.


2024년 7월 2일 화요일 오전 10시 47분 44초 (AEST)

짜증이 난다. 파서는 이제 잘 동작하는 것 같은데, 코드 생성기에서 문제가 발생했다. AST 트리는 여러 심볼의 ID를 저장한다. 하지만 함수의 심볼을 무효화할 때 ID를 -1로 설정하고, 새로운 ID를 가진 심볼을 심볼 테이블에 저장한다. 그런데 AST에는 여전히 이전 ID가 남아 있을 수 있다. 예를 들어:

```
cgqbe.c:static void cgmakeglobstrs();
cgqbe.c:  cgmakeglobstrs();
cgqbe.c:static void cgmakeglobstrs() { ... }
```

첫 번째 줄은 심볼 테이블에 심볼을 추가한다. 두 번째 줄은 AST에서 이 ID를 사용한다. 세 번째 줄은 심볼의 ID를 무효화하고 새로운 ID로 대체하지만, AST는 그대로 유지된다.

`decl.c`에서 이전 ID를 가져와 심볼을 무효화하고, 새로운 심볼을 만든 후 이전 ID를 삽입하려고 시도했다. 문제는 `sym.c` 코드가 이전에 기록한 가장 높은 ID를 기억하고 있어, 새로운 심볼을 기록하지 않는다는 점이다. 이미 "기록된" ID로 간주하기 때문이다. 망할!

이 모든 문제는 함수의 변수 이름이 저장되어야 하고, 프로토타입의 이름과 다를 수 있기 때문에 발생한다. 다른 방법이 있을까?

프로토타입의 매개변수 이름이 실제 함수의 매개변수 이름과 일치하도록 강제하는 방법을 생각해봤다. 이렇게 하면 무효화를 피할 수 있을 것 같다.


## 2024년 7월 2일 화요일 11:28:58 AEST

모든 무효화 코드를 제거했다. `cpeep.c` 문제를 제외하면 다른 모든 C 파일이 컴파일되고 테스트를 통과한다.

이제 `wcc.c`에 `-D` 전처리기 처리를 추가해야 한다. 완료했다.


## 2024년 7월 2일 화요일 11:47:40 AEST

`sym.c` 파일에서 `FILE *Symfile = NULL;` 변수가 심볼 테이블에 추가되지 않는 문제가 있었다. 이 문제는 `data.h` 파일에 해당 심볼을 추가함으로써 해결했다. 이 심볼은 때로는 `extern`으로 선언되어야 하고, 때로는 그렇지 않아야 하기 때문에 다른 심볼들과 동일한 방식으로 처리했다.

이제 삼중 테스트를 시도하고 있다. 특정 입력에서 무한 루프에 빠지는 문제가 발생한다:

```
$ L1/nwcc -S gen.c
$ L1/nwcc -S stmt.c
$ L1/nwcc -S cgqbe.c    <---
^C
$ L1/nwcc -S wcc.c      <---
^C
$ L1/nwcc -S cg6809.c   <---
^C
$ L1/nwcc -S expr.c
$ L1/nwcc -S decl.c
```

`wcc.c` 파일의 경우, 심볼 트리와 AST 파일은 `nwcc`와 `L1/nwcc` 간에 차이가 없다. 따라서 문제는 코드 생성기에서 발생하는 것으로 보인다.

`메모리에서 심볼 ID 143을 검색하는 중` 무한 루프에 빠지는 것으로 보인다.


## 2024년 7월 2일 화요일 13:11:14 (AEST)

이 버그에 다시 걸린 것 같다. `cgen.c` 파일에서:

```
  // 이제 문자열 리터럴이 아닌 경우를 처리
  for (sym=Symhead; sym!=NULL; sym=sym->next) {
      if (sym->stype== S_STRLIT) continue;
```

`continue`를 사용해도 `sym=sym->next`가 작동할까? 아니, 안 된다.

코드를 다시 작성했고, 이제야 삼중 테스트를 다시 통과했다. 다행이다!


## 2024년 7월 2일 화요일 13:21:31 AEST


`build6809bins`를 사용해 6809 컴파일러 바이너리를 다시 빌드할 수 있게 되었다.  
이제 2024년 6월 17일 월요일 당시 `smake types.c`를 시도하던 상태로 돌아왔다.

스캐너는 작동하지만 다음과 같은 오류가 발생한다:

```
emu6809 6/cparse6809 misc.c_sym misc.c_ast
New parameter name doesn't match prototype:stream on line 51
of /opt/wcc/include/6809/stdio.h
```

실행 파일 크기는 다음과 같다:

```
6/cgen6809.map:   7BA9 B __end
6/cparse6809.map: 77D6 B __end
6/cscan.map:      2E70 B __end
6/ctreeopt.map:   2258 B __end
6/desym.map:      24AA B __end
6/detok.map:      218E B __end
6/detree.map:     26D2 B __end
6/nwcc.map:       3951 B __end
```


## 2024년 7월 3일 수요일 08:06:08 AEST

문제는 6809 코드에 있다. 새로운 테스트에서 다음 코드를 실행하면:

```
  if (strcmp(c, c)) { printf("%s and %s are different\n", c, c); }
  if (!strcmp(c, c)) { printf("%s and %s are the same\n", c, c); }
```

다음과 같은 결과가 나온다:

```
Fisherman and Fisherman are different
Fisherman and Fisherman are the same
```

이 문제는 심볼 테이블 리팩토링 이전부터 존재했다.

문제를 파악했다. 두 번째 호출의 디버그 실행 결과는 다음과 같다:

```
     _strcmp+003E: LEAS 4,S         | -FHI---- 00:00 01E3 0001 1082 FDAC
     _strcmp+0040: PULS PC,U        | -FHI---- 00:00 01E3 0001 0000 FDB0
       _main+0032: LEAS 4,S         | -FHI---- 00:00 01E3 0001 0000 FDB4
       _main+0034: CMPD #$0000      | -FHI-Z-- 00:00 01E3 0001 0000 FDB4
       _main+0038: BNE $016A        | -FHI-Z-- 00:00 01E3 0001 0000 FDB4
       _main+003A: LDD #$0001       | -FHI---- 00:01 01E3 0001 0000 FDB4
       _main+003D: BRA $016D        | -FHI---- 00:01 01E3 0001 0000 FDB4
```

`strcmp()`가 반환된다. 결과를 0과 비교하고, 0이면 1을 로드한다(부정). 그러나 첫 번째 호출에서는:

```
     _strcmp+003E: LEAS 4,S         | -FHI---- 00:00 01E3 0000 1082 FDAC
     _strcmp+0040: PULS PC,U        | -FHI---- 00:00 01E3 0000 0000 FDB0
       _main+000D: LEAS 4,S         | -FHI---- 00:00 01E3 0000 0000 FDB4
       _main+000F: BEQ $0150        | -FHI---- 00:00 01E3 0000 0000 FDB4
       _main+0011: LDD $1105        | -FHI---- 10:78 01E3 0000 0000 FDB4
```

`strcmp()`가 0을 반환하지만 Z 플래그가 설정되지 않는다. 따라서 `BEQ` 명령어가 실행되지 않는다. 그래서 두 번째 호출에서처럼 `BEQ` 전에 `CMPD #$0000`을 추가해야 한다. 이제 문제를 해결했다.


## 2024년 7월 3일 수요일 08:43:20 AEST

문제가 발생했다. `misc.c` 같은 작은 파일을 컴파일할 때 메모리가 부족하다:

```
misc.c 처리 중
misc.c 파일의 27번 줄에서 loadSym() 함수 내부의 member를 malloc으로 할당할 수 없음
cparse6809 실패
```

이 문제는 심볼 테이블을 개선하기 전보다 더 심각하다. 왜 이런 일이 발생한 걸까?!!


## 2024년 7월 3일 수요일 오전 10:30:58 AEST

오늘은 valgrind를 사용해 작업했다. `sym.c` 파일에서 몇 개의 `free()`를 잊어버린 것을 발견했다. `free()`를 추가했더니 triple 테스트가 실패했다. 문제를 파악해보니 심볼의 `rvalue` 필드를 초기화하지 않았던 것이 원인이었다. 이제 triple 테스트를 다시 통과할 수 있게 되었고, valgrind에서 대부분의 메모리 누수가 AST(AST)와 관련된 것으로 나타났다.

이제 `smake`를 사용해 6809 컴파일러가 스스로를 컴파일하도록 하는 작업으로 돌아갔다. 심볼 테이블을 파일로 관리하면서 작업 속도가 훨씬 느려졌다.

이번에는 메모리가 부족하기 전에 `detree.c`까지 진행할 수 있었다. 이전보다는 나아진 상태다. 하지만 이제는 AST 노드를 해제하려고 시도해야 한다. 이전에 시도했지만 실패했던 부분이다.

아, 코드 생성기에는 추가했던 부분이 동작했지만, 파서에는 추가하지 않았던 것이 문제였다. 지금 추가했고, triple 테스트는 여전히 통과한다.

와우. 파서에서 valgrind가 메모리 누수가 100K에서 약 4K로 줄었다고 알려준다. 이제 `smake`에 도움이 될지 확인해보자.

이번에는 조금 더 진행할 수 있었다. 이제는 생성기에서 `detree.c`에서 실패하지만, `desym.c`는 컴파일할 수 있다. `stmt.c`는 파싱할 수 있지만 생성기에서 실패한다.

흥미롭게도 생성된 어셈블리 파일은 이 부분만 다르다:

```
desym.s
582c582
<       ldd #65535
---
>       ldd #-1
```

이는 좋은 신호다. 이제 코드 생성기에 대해 valgrind를 실행해봐야겠다.

아, 전역 변수를 로드했지만 사용하지 않은 것들을 제대로 해제하지 않았던 것이 문제였다. 수정했고, triple 테스트는 여전히 통과한다.


## 2024년 7월 3일 수요일 11:54:43 AEST

이번 개선 사항을 적용한 후 `smake stmt.c`와 `wcc.c`를 실행할 수 있게 되었다. 하지만 실행 후 다음과 같은 오류가 발생했다:

```
scan.c의 539번 줄에서 mkastnode() 함수 내 malloc 실패, cparse6809 실패
gen.c의 519번 줄에서 mkastnode() 함수 내 malloc 실패, cparse6809 실패
sym.c의 366번 줄에서 ftell 변수 또는 함수를 찾을 수 없음, cparse6809 실패
binexpr의 -1279번 줄에서 알 수 없는 AST 연산자: 14336, cgen6809 실패
decl.c의 86번 줄에서 mkastnode() 함수 내 malloc 실패, cparse6809 실패
```

`malloc()` 관련 오류는 함수가 매우 커서 AST 트리가 거대해질 때 발생한다. 또한 심볼 테이블도 상당히 크게 늘어난 것으로 보인다. 이 문제를 해결하기 위해 구조체의 크기를 줄이는 방법을 고려할 수 있다. 하지만 AST 트리를 부분적으로 직렬화하는 것은 쉽지 않을 것 같다.


## 2024년 7월 3일 수요일 13:10:42 (AEST)


구조체 다이어팅. 심볼 테이블(symtable): 일부 요소를 정수 대신 문자로 표현할 수 있다:
 - 타입(type), 심볼 타입(stype), 클래스(class), 주소 유무(st_hasaddr)

또한 ctype과 ctypeid를 합칠 수 있다.

AST 노드의 경우:
 - 타입(type), rvalue는 문자로 표현할 수 있다
 - left/mid/right 포인터와 id를 합칠 수 있다
 - 심볼 포인터(sym pointer)와 심볼 id(symid)를 합칠 수 있다
 - sym->name을 사용할 수 있다면 name이 필요할까?


## 2024년 7월 3일 수요일 14:48:23 AEST

부분 AST(Abstract Syntax Tree) 트리에 대한 고민... 제너레이터에는 `genfreeregs()`가 있는데, 이 함수는 결과가 필요하지 않음을 알린다. 이 함수는 IF, WHILE, SWITCH, 논리 OR, 논리 AND, 그리고 glue 연산을 생성할 때 호출된다. 파서에서 이 시점에 트리를 직렬화할 수 있는 방법이 있을지도 모른다.

또한 각 AST 노드에는 ID가 있다. 현재 심볼 테이블 트리와 유사한 방식을 적용할 수 있다. `loadASTnode()` 함수를 만들 수 있다. 왼쪽/중간/오른쪽으로 이동할 때마다 ID를 사용해 `loadASTnode()`를 호출할 수 있다.

아마도 다음과 같은 방식이 가능할 것이다:

```
// 주어진 AST 노드 ID를 사용해 AST 파일에서 해당 AST 노드를 로드한다.
// nextfunc가 설정된 경우, 다음 함수인 AST 노드를 찾는다.
// 노드를 할당하고 반환한다.
struct ASTnode *loadASTnode(int id, int nextfunc);
```


## 2024년 7월 3일 수요일 16:27:09 AEST

이제 메모리 내 AST 트리 개념을 완전히 버려도 될 것 같다고 생각한다.  
대신 AST 파일과 left/mid/right id를 디스크에 저장해 암묵적인 트리를 구성할 수 있다.  

제너레이터 측에서는 left/mid/right를 탐색할 때마다 `loadASTnode()`를 사용하면 된다.  
파서 측에서는 여전히 메모리에 노드를 생성하지만, 노드가 완전히 채워지면(즉, left/mid/right, hasaddr 등이 설정되면) 노드를 디스크에 쓰고 메모리에서 해제한다.  

이 방식은 기존 AST 파일을 그대로 사용할 수 있고, 제너레이터 작업을 바로 시작할 수 있다는 장점이 있다.  
제너레이터가 동작하면, 그때 파서 측을 변경할 수 있다.


## 2024년 7월 4일 목요일 16:37:19 AEST

`loadASTnode()` 코드를 작성했지만 아직 테스트하지 않았다. `gen.c`를 이 코드를 사용하도록 변경하기 시작했지만, 세부적인 부분에서 문제가 있다. 잠시 한 발 물러서서 생각을 정리할 필요가 있다고 느낀다. `gen.c`에는 리스트를 순회하는 코드가 있다. 몇몇 함수에서 동일한 리스트를 여러 번 반복한다. 같은 함수 내에서 동일한 노드를 여러 번 `loadASTnode()`로 처리하고 싶지는 않다.

또한 AST 노드를 `free()`하는 최적의 위치에 대해서도 고민해야 한다. 그래서 먼저 몇 가지 `free()`를 추가하고, `loadASTnode()` 변환 작업을 시작하기 전에 이들이 정상적으로 동작하는지 확인할 계획이다.


2024년 7월 4일 목요일 17:36:11 (AEST)

`genAST()` 코드를 수정하여 끝 부분에만 하나의 return 문이 있도록 변경했다.  
`tree.c`에 단일 AST 노드를 해제하는 코드를 추가했다.  
`genAST()`의 마지막 부분에서 왼쪽과 중간 AST 노드를 해제하고, 이렇게 하면 triple 테스트를 통과할 수 있다.  
하지만 오른쪽 노드를 해제할 때는 동작하지 않는다. 이 문제를 조사해야 한다.


## 2024년 7월 5일 금요일 15:38:36 AEST

문제를 해결했다. 어떤 경우에는 right가 left로 설정될 수 있기 때문에, 이를 해제하기 전에 확인해야 한다.

새로운 변수 `nleft`, `nmid`, `nright`를 설정하여 하위 노드를 보관하는 데 시간을 보냈다. 현재는 단순히 포인터를 복사하고 있지만, 나중에는 `loadASTnode()`를 사용할 예정이다. CASE 문과 함수 호출에서는 노드 연결 리스트를 두 번 순회해야 한다. 이를 피할 방법이 없기 때문에, 현재로서는 여러 번의 디스크 읽기를 감수해야 한다. 나중에 캐시를 구축하는 방법을 고려해볼 수도 있을 것이다.


## 2024년 7월 9일 화요일 14:00:37 AEST

3일간의 코미디 크루즈에서 돌아왔다. `detree.c`를 `loadASTnode()` 함수를 사용하도록 다시 작성했다. 몇 가지 사소한 문제를 수정한 후, 이전 버전과 동일한 트리를 출력한다. 이로써 `loadASTnode()` 함수가 정상적으로 동작한다는 확신을 얻었다.

방금 `detree.c`에 `free()`를 추가했고, 모든 AST 노드와 관련된 이름 문자열이 해제된다. 이 점이 매우 만족스럽다!!


## 2024년 7월 9일 화요일 15:38:05 AEST

`gen.c` 파일에 `loadASTnode()` 코드를 추가했고, 첫 번째 SWITCH 테스트까지 잘 동작한다. 꽤 괜찮은 진행이다 :-) 모든 테스트를 통과시키면, 할당된 모든 노드를 해제하는 작업을 다시 진행할 예정이다.


2024년 7월 9일 화요일 15:58:16 AEST

`loadASTnode()`를 통해 로드해야 하는 왼쪽 노드 포인터 참조 하나를 놓쳤었다. 이제 모든 테스트가 통과한다. 야호!!!


## 2024년 7월 10일 수요일 07:39:53 AEST

어제 밤에 valgrind를 사용해 작업하며 코드에 `freeASTnode()`와 관련 함수를 많이 추가했다. 현재 미해제 메모리가 약 3K 정도로 줄었다. 이는 매우 좋은 결과다.

코드 생성 부분은 여기서 마무리하려고 한다. 이제 파싱 측면에서 AST 노드를 어떻게 구성하고, 모든 자식 노드 ID를 얻은 후 디스크에 기록한 다음 메모리를 해제할지 고민해야 한다.

가장 어려운 부분은 이 작업을 수행할 적절한 지점을 찾는 것이다. 현재는 각 함수의 끝에서 처리하고 있지만, 6809에서는 큰 함수의 경우 메모리를 너무 많이 차지한다.


2024년 7월 10일 수요일 10:33:05 AEST

AST 노드의 열거를 `tree.c`로 옮기려고 시도 중이다. 이렇게 하면 노드가 생성될 때 id를 부여할 수 있다. 문제는 `optimise()`와 같은 여러 곳에서 트리 구조를 변경할 때마다 모든 재연결 작업을 찾아서 자식 노드의 id를 동시에 수정해야 한다는 점이다. 현재 테스트 009에서는 자식 노드의 id가 여러 부모 노드에 있는 AST 노드 세트가 있다! 트리가 아니다!

수정 완료. 자식 포인터가 NULL일 때 노드 id를 0으로 초기화하지 않았던 문제를 해결했다.


## 2024년 7월 11일 목요일 14:13:17 AEST

새로운 AST 열거 코드를 사용하여 QBE와 6809에 대한 테스트가 통과했다.  
이제 삼중 테스트를 시도했는데 실패했다. `gcc` 출력(정상)과 L1 출력(비정상)을 비교해 보았다. `cpp` 파일은 문제가 없고, 심볼 테이블도 동일하다.

AST 파일은 문자열 리터럴이 다르다는 점을 제외하면 동일하다. 특히 줄바꿈 문자가 손상된 것 같다. 예를 들어:

```
정상
----
00003d70  00 00 00 00 00 00 00 00  00 4f 75 74 20 6f 66 20  |.........Out of |
00003d80  73 70 61 63 65 20 69 6e  20 63 6d 64 61 72 67 73  |space in cmdargs|
00003d90  0a 00 23 00 00 00 10 00  00 00 00 00 00 00 00 00  |..#.............|

비정상
---
00003870  00 00 00 00 00 00 00 00  00 4f 75 74 20 6f 66 20  |.........Out of |
00003880  73 70 61 63 65 20 69 6e  20 63 6d 64 61 72 67 73  |space in cmdargs|
00003890  6e 00 23 00 00 00 10 00  00 00 00 00 00 00 00 00  |n.#.............|
```

마지막 줄의 첫 번째 바이트를 보면 `0a`와 `6e`이다. `6e`는 ASCII 'n'이다.  
심볼 테이블 수정 전의 컴파일러 출력은 다음과 같다:

```
00003870  00 00 00 00 00 00 00 00  00 4f 75 74 20 6f 66 20  |.........Out of |
00003880  73 70 61 63 65 20 69 6e  20 63 6d 64 61 72 67 73  |space in cmdargs|
00003890  0a 00 23 00 00 00 10 00  00 00 00 00 00 00 00 00  |..#.............|
```

이 출력은 정상이다. 그리고 커밋 6b870ee2d... (날짜: 2024년 7월 10일 10:20:32)도 정상이다.  
이제 문제를 좁힐 수 있을 것 같다(바라본다 :-)).


2024년 7월 11일 목요일 14:59:05 AEST

두 파일 세트 간의 차이점을 살펴보면 문제가 될 만한 부분은 없다. 한숨.


## 2024년 7월 11일 목요일 15:10:40 AEST

이상하다. `gcc`와 `nwcc` 패스에서 문자열 리터럴을 읽고 쓰는 디버그 라인을 확인해 보자:

```
      1 Read in string >Out of space in cmdargs
<
      1 Dumping literal string/id >Out of space in cmdargs
<
```

이것은 `gcc` 파서의 결과다. 줄바꿈이 정상적으로 처리된다. 이제 코드 생성기를 보자:

```
   1509 Read in string >Out of space in cmdargs
<
```

여전히 정상이다. 이제 `nwcc`로 컴파일된 파서와 생성기를 확인해 보자:

```
      1 Read in string >Out of space in cmdargsn<
      1 Dumping literal string/id >Out of space in cmdargsn<
   1509 Read in string >Out of space in cmdargsn<
```

이 결과는 잘못되었다.


2024년 7월 11일 목요일 15:34:54 AEST

아, 스캐너에 문제가 다시 발생했다! Good 스캐너는 다음과 같이 표시한다:

```
34: "Out of space in cmdargs
"
```

Bad 스캐너는 다음과 같이 표시한다:

```
34: "Out of space in cmdargsn"
```


## 2024년 7월 12일 금요일 오전 7시 41분 16초 (AEST)

스캐너 코드 일부를 추상화한 테스트 케이스를 만들었고, 문제를 재현할 수 있었다. 디스크 기반 심볼/AST 변경 이전의 컴파일러와 현재 버전을 비교했을 때, 생성된 AST 트리에서 다음과 같은 차이를 발견했다:

```
28,30d27
<       CASE 110 ()
<         RETURN ()
<           INTLIT 10 ()
```

스위치문의 case 구문이 완전히 누락되었다!

```
    switch (c) {
      case 'n':
        return ('\n');
    }
```

새로 만든 `astdump` 프로그램을 사용해 파일의 SWITCH 노드를 살펴보니, 오른쪽 AST 자식 ID가 누락된 것을 확인했다:

```
31 SWITCH type 0 rval 0
   line 14 symid 0 intval 1
   left 30 mid 0 right 0
```

정상 버전에서는 right 값이 37이다. 문제를 찾았다. `stmt.c`에서 case 서브 트리를 SWITCH 노드에 연결했지만, 오른쪽 ID를 설정하는 것을 잊어버렸다. 이제 수정했다.


## 2024년 7월 12일 금요일 11:40:05 AEST

아직도 삼중 테스트를 통과하지 못하고 있다. 다른 디렉토리에서 삼중 테스트를 통과한 마지막 커밋을 체크아웃했다:  
50b82937c5da569b. 이제 두 버전(Good/Bad)으로 컴파일러 파일을 어셈블리로 컴파일하고, 중간 파일을 비교할 수 있다.

토큰 파일은 동일하다. 이제 `detree` 출력을 살펴보자.  
scan.c, cg6809.c, cgqbe.c 파일에서 차이가 발견된다.

흠, 아직도 누락된 case 문이 있는 것 같다. 이상하다!  
원인을 찾았다. case 트리의 연결 리스트를 만들 때 자식의 nodeid를 추가하는 것을 잊어버렸다. 이제 다시 삼중 테스트를 통과한다!

이제 파서에서 AST 노드를 직렬화하고 해제하는 작업으로 돌아갈 수 있다.


## 2024년 7월 12일 금요일 12:12:08 AEST

시작은 했지만, input001.c 파일에서 일부 AST 노드를 직렬화하지 못하고 있다. 이런! 이제 고쳤다. "No return for function with non-void type"이라는 오류가 발생하는 테스트를 제외하고 모든 테스트를 통과할 수 있다. 이 테스트를 컴파일러에서 주석 처리해야 하는데, 리턴 여부를 확인하기 전에 트리를 해제하고 있기 때문이다!

와, QBE 테스트, 6809 테스트, 그리고 triple 테스트를 모두 통과했다! `valgrind`가 특정 시점에 할당된 최대 메모리 양을 알려줄 수 있을까? 그렇다: https://valgrind.org/docs/manual/ms-manual.html


## 2024년 7월 12일 금요일 16:44:40 AEST

방금 가장 큰 컴파일러 C 파일을 컴파일했다. 스크립트로 AST 파일을 확인한 결과:

```
Seen 165 vs. unseen 2264
```

이 결과는 AST 노드를 읽을 때, 자식 노드가 있는 경우 이미 본 노드보다 보지 못한 노드가 더 많다는 것을 의미한다. 

따라서 파일의 시작 부분으로 되돌아가서 검색하는 대신, 현재 위치에서 검색을 시작하고 동일한 탐색 지점에 도달하면 종료하는 것이 더 효율적이다.

이를 위해 `loadASTnode()` 함수에 몇 가지 카운터를 추가하고, 검색 알고리즘을 변경한 후 어떤 방식이 더 나은지 비교해볼 예정이다.


## 2024년 7월 13일 토요일 09:56:58 AEST

`build6809bins`로 6809 컴파일러 바이너리를 빌드했다. 이제 `smake`를 사용해 모든 컴파일러 C 파일을 6809 컴파일러 바이너리로 어셈블리를 생성하고 있다. 속도가 매우 느리다! 약 10분이 지났는데 `cg6809.c` 파일의 145번째 줄까지만 진행됐다. 어제 작은 파일 몇 개로 시도했을 때 코드 생성기가 중단됐다. 이번에는 모든 파일을 컴파일하고 `*_*` 파일들을 하위 디렉터리로 옮긴 다음, 네이티브 컴파일러로 같은 작업을 할 계획이다. 그렇게 하면 중간 파일들을 비교해 차이점이 있는지, 어디에 있는지 확인할 수 있다.

코드 생성기의 실행 프로파일을 반드시 확인해 주요 병목 현상을 파악해야 할 것 같다.

약 30~40분이 지났는데도 `cg6809.c`의 코드 생성이 절반 정도밖에 진행되지 않았다! 네이티브 컴파일러와 비교했을 때 지금까지 발견한 유일한 차이점은 다음과 같다:

```
<       anda #0
<       andb #0
---
>       anda #255
>       andb #255
```

이는 매우 작은 차이로, 추적하기 어렵지 않을 것이다. 점심 시간이 되어 아이들을 만나러 가야 하니, 이 작업을 몇 시간 동안 계속 실행해 둘 수 있다 :-)


점심을 먹고 돌아왔다. `smake`가 시작할 때 모든 임시 파일을 삭제해서 거의 모든 것이 사라졌다. 코드 생성 단계들도 모두 실패한 것 같다. 그래서 다시 모두 해야 할 것 같다... 이 부분에서 파서가 메모리를 다 쓴 것 같다: `gen.c의 684번째 줄에서 mkastnode() malloc 실패` 하지만 지금까지 다른 문제는 없다.


## 2024년 7월 13일 토요일 16:51:56 AEST

흥미롭다. 대부분의 코드 생성이 성공한 것 같지만, 코드 생성기가 깔끔하게 멈추지 않는 문제가 있다. `main()` 끝에서 `return(0)` 대신 `exit(0)`을 사용하는 게 더 나을까?

또한 다음과 같은 코드를 보았다:

```
<       ldd #65535
---
>       ldd #-1
```

이전에도 언급했던 부분이다.

따라서 다음과 같은 문제를 확인하고 수정해야 한다:

- 65535 / 1
- 그리고 0 / 255
- 코드 생성 종료 시 세그멘테이션 폴트

이 문제들을 해결한 후에는 코드 생성 속도를 개선해야 한다. 현재는 AST 파일 오프셋 목록으로 구성된 임시 파일을 만드는 것을 고려 중이다. 각 노드마다 하나의 오프셋을 저장하는 방식이다. AST 노드 X를 찾으려면 4를 곱하거나(또는 2비트 왼쪽 시프트), 파일에서 해당 지점으로 `fseek()`한 후 그 지점의 long 오프셋을 읽는다. 그런 다음 AST 파일에서 해당 노드를 가져오기 위해 `fseek()`을 사용한다. 가장 큰 AST 파일은 4549개의 노드를 가지고 있으므로, 오프셋 파일은 18K 정도의 크기가 된다. 이 크기는 메모리 배열로 유지하기에는 너무 크다.


## 2024년 7월 14일 일요일 12:40:56 AEST

`parse.c` 파일에 대해 6809 `cgen`의 디버그 실행을 진행했다. 이 파일은 상당한 크기를 가지고 있어 테스트에 적합하다. `main()` 함수가 반환된 후 프로그램이 예상치 못한 상태로 빠지는 것을 확인했다. 동시에 어떤 함수가 가장 많은 명령어를 실행하는지도 살펴보고 있다. 결과는 다음과 같다:

```
242540320 div16x16
87457045 _fread
34695064 _fgetstr
26953031 boolne
20521224 __mul
10534645 _loadASTnode
7960435 __minus
6814620 __divu
6150026 booleq
4794607 boolgt
4735523 boollt
4630436 __syscall
4594719 __not
3479025 boolult
2271540 _read
```

결과를 보면 `loadASTnode()` 함수를 최적화해야 할 필요가 있다는 것이 명확하다.


2024년 7월 14일 일요일, 오후 1시 15분 23초 (AEST)

크래시의 원인을 살펴보니, `crt0.s` 파일의 코드에 문제가 있는 것 같다. 이 코드는 `main()` 함수가 반환된 후 `exit()`를 호출해야 하지만, 다음과 같은 상황이 발생하고 있다:

```
       _main+0156: RTS              | -FHI-Z-- 00:00 8160 0000 0000 FD91
      __code+0028: NEG <$54         | -FHI-Z-- 00:00 8160 0000 0000 FD91
```

`crt0.s`는 $100에서 시작한다. 따라서 $128 주변에 쓰기 브레이크를 설정하고 에뮬레이터를 실행해야 할 것 같다.

흠, `main()` 함수 끝에서 `return(0)` 바로 전에 있는 `fclose()` 연산 중 하나가 문제인 것 같다. 이상하네!


2024년 7월 15일 월요일 07:47:03 (AEST)

방금 `#65535`와 `#-1`이 동일한 머신 코드를 생성한다는 것을 확인했다. 이 부분은 무시해도 될 것 같다. 지금은 두 개의 `fclose()`를 제거하고 cgen.c 파일 끝에 `exit(0)`을 추가했다. 작업 완료.

`and #0` / `and #255` 문제는 컴파일러가 너무 느려서 나중으로 미루기로 했다. 대신 AST 오프셋 인덱스 파일을 생성하는 코드를 작성할 예정이다.


2024년 7월 15일 월요일 11:03:18 AEST

일반 테스트는 통과했지만, 삼중 테스트는 통과하지 못했다. 하지만 45분 동안 작업한 결과치고는 괜찮은 성과다. 그리고 실행 속도도 더 빨라진 것 같다.


## 2024년 7월 15일 월요일 12:09:33 AEST

컴파일러의 `sizeof()`에 버그가 있는 것 같다. 인덱스 생성 코드의 일부는 다음과 같다:

```
void mkASTidxfile(void) {
  struct ASTnode node;
  long offset, idxoff;

  while (1) {
    // 현재 오프셋을 가져옴
    offset = ftell(Infile);
    fprintf(stderr, "A offset %ld\n", offset);

    // 다음 노드를 읽어들임. 없으면 중단
    if (fread(&node, sizeof(struct ASTnode), 1, Infile)!=1) {
      break;
    }
    fprintf(stderr, "B offset %ld\n", offset);
    ...
```

L1 코드 생성기를 실행했을 때 다음과 같은 결과를 확인했다:

```
A offset 133032
B offset -7491291578409943040
A offset 133120
B offset -7491311919375056895
A offset 133208
B offset 1
A offset 133308
B offset 0
```

ASTnode를 읽을 때 필요한 것보다 더 많이 읽고 있는 것 같다. 아니, 실제로는 QBE 문제일 수 있다. 출력을 보면 다음과 같다:

```
C 코드
------
  struct ASTnode node;
  long offset, idxoff;

QBE 코드
--------
  %node =l alloc8 11	// 88바이트 크기
  %offset =l alloc8 1

ASM 코드
--------
subq $40, %rsp		// 최소 96바이트여야 함
movq %rdx, -8(%rbp)	// -8은 스택에서 offset 위치
leaq -24(%rbp), %rdi	// -24는 스택에서 node 위치
```

이것은 `node`가 스택에서 `offset`보다 16바이트 아래에 위치한다는 것을 의미한다. 하지만 `node`는 88바이트 크기다. QBE 리스트에 이 문제에 대한 아이디어를 요청하는 이메일을 보냈다.

흠. `node`를 포인터로 만들고 함수 시작 부분에서 `malloc()`으로 메모리를 할당한 후, 함수 끝에서 `free()`로 해제했다. 이제 세 가지 테스트를 통과한다. 그래서 이 문제는 QBE 버그인 것 같다.

아니다, 내 실수였다. QBE 리스트에서 다음과 같은 답변을 받았다:

> alloc8은 8바이트 경계에 맞춰 할당하는 것이지, 8 * 인자 바이트를 할당하는 것이 아니다. 따라서 alloc8 11은 11바이트를 할당하며, 88바이트가 아니다.

그래서 이 문제를 해결하기 위해 QBE 출력을 수정해야 한다. 수정해야 할 부분이다.


6809 쪽에서 `tree.c` 파일에서 다음과 같이 작성하면:

```
  n->leftid= 0;
  n->midid= 0;
  n->rightid= 0;
```

이렇게 결과가 나온다:

```
        ldx 0,s
        ldd 12,s
        std 10,x
        ldx 0,s
        ldd 14,s
        std 12,x
        ldx 0,s
        ldd #0
        std 16,x
```

하지만 코드를 이렇게 작성하면:

```
  n->leftid= n->midid= n->rightid= 0;
```

이렇게 결과가 나온다:

```
        ldx 0,s
        ldd #0
        std 20,x
        std R0+0	<== rvalue #0 저장
        ldd 0,s
        addd #18
        tfr d,x
        std 0,x		<== R0 다시 로드해야 함
        ldd 0,s
        addd #16
        tfr d,x
        std 0,x		<== R0 다시 로드해야 함
```

이 코드는 잘못되었다. 수정이 필요하다!

이제 모든 컴파일러 파일에 대해 `smake`를 실행 중이다.
확실히 훨씬 빠르다! 아직 크래시는 없다...
그리고 모든 파일을 컴파일하는 데 16분밖에 걸리지 않았다.
이제 파일들을 비교할 차례다.


## 2024년 7월 15일 월요일 15:37:11 AEST

흠. 컴파일되지 않은 유일한 파일은 `gen.c`다.  
파서에서 메모리가 부족해 `gen.c`의 684번째 줄에서 `mkastnode()` 함수 내에서 `malloc`을 할 수 없었다.

어셈블리 파일의 차이점은 무시해도 되는 `#-1 / #65535`와  
반드시 수정해야 하는 `anda #0 / #255` 문제뿐이다.

따라서 `anda #0 / #255` 문제를 해결하고 `gen.c`에서 파서가 메모리를 다 쓰지 않도록 방법을 찾는다면,  
6809에서 트리플 테스트를 통과할 수 있을 것이다.


`printlocation()` 함수에서 `int` 리터럴과 세 번째 인자로 'e' 또는 'f'를 사용할 때 발생하는 `and` 문제는 버그로 보인다. 이 문제는 함수의 논리적 오류나 인자 처리 방식에서 비롯된 것일 수 있다. 코드를 검토하고, 특히 세 번째 인자가 'e' 또는 'f'일 때의 동작을 면밀히 확인해야 한다. 이 문제를 해결하려면 해당 부분의 로직을 수정하거나, 인자 처리 방식을 개선하는 것이 필요하다.


## 2024년 7월 16일 화요일 09:21:25 AEST

문제를 찾은 것 같다. 현재 `Locn[l].intval & 0xffff`를 사용하고 있는데, `intval`이 long 타입이다. 이로 인해 컴파일러가 `0xffff`를 확장하게 된다. 하지만 6809에서는 이 값이 음수로 처리되어 `0xffffffff`로 확장된다. `0x0000ffff`가 아니다. 이를 해결하기 위해 AND 연산을 수행하기 전에 `int` 타입 변수로 캐스팅하거나 저장하는 방법을 시도하고 있다.

이 방법이 효과가 있는 것 같다. 이제 `smake`를 다시 실행 중이다. 이번에는 어셈블리 파일을 오브젝트 파일로 변환하고 있다. 그런 다음 체크섬을 비교해 동일한지 확인할 수 있다.

결과는 매우 좋다. `gen.c`만 컴파일에 실패했다(파서가 메모리를 다 써버림):

```
143856ed08f470c9bc5f4b842dcc27bd  New/opt.o
143856ed08f470c9bc5f4b842dcc27bd  opt.o
18e04acd5b8e0302e95fc3c9cddcdac5  New/tree.o
18e04acd5b8e0302e95fc3c9cddcdac5  tree.o
1d42a151ccf415e102ece78257297cd9  New/tstring.o
1d42a151ccf415e102ece78257297cd9  tstring.o
43b84fc5d30ea22ceac9a21795518fc3  decl.o
43b84fc5d30ea22ceac9a21795518fc3  New/decl.o
45a18eb804fdc0c75f3207482ad8678a  detok.o
45a18eb804fdc0c75f3207482ad8678a  New/detok.o
57d10f0978232603854a6e18bf386cba  New/wcc.o
57d10f0978232603854a6e18bf386cba  wcc.o
76ed2bc3c553568d16880dfdd02053e7  New/parse.o
76ed2bc3c553568d16880dfdd02053e7  parse.o
88bfe3920d8f08d527447fef2c24dc3b  New/scan.o
88bfe3920d8f08d527447fef2c24dc3b  scan.o
8c24c919c06532977a68472c709c5e22  cg6809.o
8c24c919c06532977a68472c709c5e22  New/cg6809.o
8e4fd9f9e9923c20432ec7dae85965c5  expr.o
8e4fd9f9e9923c20432ec7dae85965c5  New/expr.o
8ed5104a4b18a1eb8dea33c5faf6c8bd  New/sym.o
8ed5104a4b18a1eb8dea33c5faf6c8bd  sym.o
9d11b1336597eaa6bcdac3ade6eb13ab  misc.o
9d11b1336597eaa6bcdac3ade6eb13ab  New/misc.o
a5fba53af6d4ca348554336db9455675  New/types.o
a5fba53af6d4ca348554336db9455675  types.o
beb8414b95be6f0de7494c21b16e1c53  New/stmt.o
beb8414b95be6f0de7494c21b16e1c53  stmt.o
c1e56e66055f7868ab20d13585a76eb0  cgen.o
c1e56e66055f7868ab20d13585a76eb0  New/cgen.o
ca8699919c901ed658c0ce5e0eb1d8e8  detree.o
ca8699919c901ed658c0ce5e0eb1d8e8  New/detree.o
d25a34d8dc2bb895b8c279d8946733c3  New/targ6809.o
d25a34d8dc2bb895b8c279d8946733c3  targ6809.o
deca10b552285f2de5c10e70547fd2a6  desym.o
deca10b552285f2de5c10e70547fd2a6  New/desym.o
```

적절한 스크립트를 작성할 수 있다면, 6809 측에서도 트리플 테스트를 통과할 수 있을 것이다 :-)


## 2024년 7월 16일 화요일 10:47:01 AEST

`build6809bins`를 다시 작성하여 6809 바이너리를 생성하고, 각 바이너리에 대해 에뮬레이터를 실행하는 프론트엔드 스크립트도 만들었다. 이를 통해 `native` 실행 파일을 얻을 수 있다. 이 작업이 필요한 이유는 6809 `wcc`가 `emu6809 cscan ...` 대신 `cscan ...`을 실행하기 때문이다.

하지만 이상한 점이 있다. `wcc`가 일부 단계만 실행하고 모든 단계를 실행하지 않는다:

```
$ L1/wcc -m6809 -S -X -v targ6809.c 
Doing: cpp -nostdinc -isystem /usr/local/src/Cwj6809/include/6809 targ6809.c 
  stdout을 targ6809.c_cpp로 리디렉션
Doing: /usr/local/src/Cwj6809/L1/cscan 
  stdin을 targ6809.c_cpp에서 리디렉션
  stdout을 targ6809.c_tok로 리디렉션
Doing: /usr/local/src/Cwj6809/L1/cparse6809 targ6809.c_sym targ6809.c_ast 
  stdin을 targ6809.c_tok에서 리디렉션
```

코드 생성 단계가 없다. 일단은 Perl 버전의 `wcc`를 작성하여 트리플 테스트를 완료할 계획이다.


## 2024년 7월 16일 화요일 오전 11시 35분 05초 (AEST)

`gen.c` 파일에 있는 긴 SWITCH 문을 두 부분으로 나눴다. 이렇게 하면 6809 컴파일러가 메모리 부족 없이 이 코드를 파싱할 수 있을 것이다. 모든 테스트를 통과하는지 확인했다.

이제 `gen.c` 파일이 L1 6809 컴파일러를 사용해 컴파일되며, 생성된 오브젝트 파일은 네이티브 컴파일러로 만든 것과 동일하다. QBE 삼중 테스트를 통과하지 못하는 문제가 발생해서 이 변경 사항을 `#ifdef`로 감쌌다. 이상한 일이다!


## 2024년 7월 16일 화요일 14:09:06 AEST

`wcc` 프론트엔드의 Perl 버전을 작성했다. 기본적으로 코드를 그대로 옮겼다. 이제 이 파일은 `L1` 디렉터리에 들어간다. `build6809bins`를 수정해 `L1`의 6809 컴파일러 바이너리를 사용해 `L2` 바이너리를 빌드하도록 했다. 지금까지 빌드된 `L2` 파일들은 `L1`의 파일과 동일한 체크섬을 가지고 있지만, 아직 진행 중이다...


## 2024년 7월 16일 화요일 14:25:05 AEST

아, 정말 아쉽게도 실패했다!

```
$ md5sum L?/_* | sort 
0778e984e25d407d2067ac43d151d664  L2/_cgen6809  # 다름
e47a9ab1ed9095f1c4784247c72cb1f8  L1/_cgen6809

0caee9118cb7745eaf40970677897dbf  L1/_detree
0caee9118cb7745eaf40970677897dbf  L2/_detree
2d333482ad8b4a886b5b78a4a49f3bb5  L1/_detok
2d333482ad8b4a886b5b78a4a49f3bb5  L2/_detok
d507bd89c0fc1439efe2dffc5d8edfe3  L1/_desym
d507bd89c0fc1439efe2dffc5d8edfe3  L2/_desym
e78da1f3003d87ca852f682adc4214e8  L1/_cscan
e78da1f3003d87ca852f682adc4214e8  L2/_cscan
e9c8b2c12ea5bd4f62091fafaae45971  L1/_cparse6809
e9c8b2c12ea5bd4f62091fafaae45971  L2/_cparse6809
```

이 문제는 링커 단계에서 발생했다:

```
cgen.c_o: 알 수 없는 심볼 '_genglobstr'.
cgen.c_o: 알 수 없는 심볼 '_genglobsym'.
cgen.c_o: 알 수 없는 심볼 '_genpreamble'.
cgen.c_o: 알 수 없는 심볼 '_genAST'.
cgen.c_o: 알 수 없는 심볼 '_genpostamble'.
gen.c_o: 알 수 없는 심볼 '_genAST'.
```

젠장!!

이제 코드 생성기를 만드는 C 파일들에 대해 네이티브와 6809 L1 컴파일러를 사용해 asm 파일을 생성하고 비교해 보겠다.


## 2024년 7월 17일 수요일 13:30:26 AEST

지난 하루 동안 'acwj' 프로젝트의 다음 단계를 위한 Readme.md 파일을 작성했다. 지금까지 약 7,000단어를 작성했고, 앞으로 몇 천 단어를 더 작성해야 한다.

6809 `gen.c` 문제는 `gen.c`를 컴파일할 때 `-DSPLITSWITCH`를 빼먹었던 것 같다. 이제 -1 / 65535 변경을 제외하면 동일한 어셈블리 코드가 생성된다. 다시 삼중 테스트를 시도해보자!


## 2024년 7월 17일 수요일 13:59:56 AEST

좋습니다. 6809 트리플 테스트를 통과한 것 같습니다:

```
$ md5sum L1/_* L2/_* | sort
01c5120e56cb299bf0063a07e38ec2b9  L1/_cgen6809
01c5120e56cb299bf0063a07e38ec2b9  L2/_cgen6809
0caee9118cb7745eaf40970677897dbf  L1/_detree
0caee9118cb7745eaf40970677897dbf  L2/_detree
2d333482ad8b4a886b5b78a4a49f3bb5  L1/_detok
2d333482ad8b4a886b5b78a4a49f3bb5  L2/_detok
d507bd89c0fc1439efe2dffc5d8edfe3  L1/_desym
d507bd89c0fc1439efe2dffc5d8edfe3  L2/_desym
e78da1f3003d87ca852f682adc4214e8  L1/_cscan
e78da1f3003d87ca852f682adc4214e8  L2/_cscan
e9c8b2c12ea5bd4f62091fafaae45971  L1/_cparse6809
e9c8b2c12ea5bd4f62091fafaae45971  L2/_cparse6809
```

모든 바이너리의 체크섬이 일치합니다!! 아직 6809 `wcc` 바이너리가 작동하지 않아 Perl 버전을 사용하고 있습니다. 하지만 컴파일러의 나머지 코드를 스스로 컴파일할 수 있었습니다. 야호!!!


2024년 7월 18일 목요일 오전 9시 52분 53초 (AEST)

6809 `wcc` 바이너리가 실패하는 원인을 파악하려고 노력 중이다. `wcc`는 C 전처리기(x64 네이티브 바이너리)를 정상적으로 실행한다. 그런 다음 포크를 해서 6809 `cscan`을 잘 실행한다. 이후 다시 포크를 해서 6809 `cparse`를 실행한다. 이 과정은 완료되지만, `wcc`는 페이지 0에서 알 수 없는 오퍼레이션으로 인해 크래시가 발생한다. 마지막 반환문 앞에 `exit(0)`을 추가하니 크래시가 해결되었다. 하지만 이제는 피홀 최적화 도구를 실행할 때 크래시가 발생한다.



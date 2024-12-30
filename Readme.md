# 컴파일러 제작 여정

이 깃허브 저장소는 C 언어의 일부분을 자체 컴파일할 수 있는 컴파일러를 만드는 과정을 기록한다. 컴파일러 이론에 대한 참조와 함께 무엇을, 왜, 어떻게 했는지 자세히 설명한다. 독자가 따라하면서 배울 수 있도록 구성했다.

하지만 너무 많은 이론보다는 실용적인 접근 방식을 취한다.

지금까지 진행한 단계는 다음과 같다:

 + [0부](00_Introduction/Readme.md): 여정의 시작
 + [1부](01_Scanner/Readme.md): 어휘 분석 소개
 + [2부](02_Parser/Readme.md): 구문 분석 소개
 + [3부](03_Precedence/Readme.md): 연산자 우선순위
 + [4부](04_Assembly/Readme.md): 실제 컴파일러
 + [5부](05_Statements/Readme.md): 구문
 + [6부](06_Variables/Readme.md): 변수
 + [7부](07_Comparisons/Readme.md): 비교 연산자
 + [8부](08_If_Statements/Readme.md): if 문
 + [9부](09_While_Loops/Readme.md): while 반복문
 + [10부](10_For_Loops/Readme.md): for 반복문
 + [11부](11_Functions_pt1/Readme.md): 함수, 1부
 + [12부](12_Types_pt1/Readme.md): 타입, 1부
 + [13부](13_Functions_pt2/Readme.md): 함수, 2부
 + [14부](14_ARM_Platform/Readme.md): ARM 어셈블리 코드 생성
 + [15부](15_Pointers_pt1/Readme.md): 포인터, 1부
 + [16부](16_Global_Vars/Readme.md): 전역 변수 올바르게 선언하기
 + [17부](17_Scaling_Offsets/Readme.md): 향상된 타입 검사와 포인터 오프셋
 + [18부](18_Lvalues_Revisited/Readme.md): L값과 R값 재검토
 + [19부](19_Arrays_pt1/Readme.md): 배열, 1부
 + [20부](20_Char_Str_Literals/Readme.md): 문자와 문자열 리터럴
 + [21부](21_More_Operators/Readme.md): 추가 연산자
 + [22부](22_Design_Locals/Readme.md): 지역 변수와 함수 호출을 위한 설계 아이디어
 + [23부](23_Local_Variables/Readme.md): 지역 변수
 + [24부](24_Function_Params/Readme.md): 함수 매개변수
 + [25부](25_Function_Arguments/Readme.md): 함수 호출과 인자
 + [26부](26_Prototypes/Readme.md): 함수 프로토타입
 + [27부](27_Testing_Errors/Readme.md): 회귀 테스트와 즐거운 놀라움
 + [28부](28_Runtime_Flags/Readme.md): 런타임 플래그 추가
 + [29부](29_Refactoring/Readme.md): 리팩토링
 + [30부](30_Design_Composites/Readme.md): 구조체, 공용체, 열거형 설계
 + [31부](31_Struct_Declarations/Readme.md): 구조체 구현, 1부
 + [32부](32_Struct_Access_pt1/Readme.md): 구조체 멤버 접근
 + [33부](33_Unions/Readme.md): 공용체와 멤버 접근 구현
 + [34부](34_Enums_and_Typedefs/Readme.md): 열거형과 타입 정의
 + [35부](35_Preprocessor/Readme.md): C 전처리기
 + [36부](36_Break_Continue/Readme.md): `break`와 `continue`
 + [37부](37_Switch/Readme.md): switch 문
 + [38부](38_Dangling_Else/Readme.md): 매달린 else와 기타
 + [39부](39_Var_Initialisation_pt1/Readme.md): 변수 초기화, 1부
 + [40부](40_Var_Initialisation_pt2/Readme.md): 전역 변수 초기화
 + [41부](41_Local_Var_Init/Readme.md): 지역 변수 초기화
 + [42부](42_Casting/Readme.md): 타입 캐스팅과 NULL
 + [43부](43_More_Operators/Readme.md): 버그 수정과 추가 연산자
 + [44부](44_Fold_Optimisation/Readme.md): 상수 폴딩
 + [45부](45_Globals_Again/Readme.md): 전역 변수 선언 재검토
 + [46부](46_Void_Functions/Readme.md): void 함수 매개변수와 스캐닝 변경
 + [47부](47_Sizeof/Readme.md): `sizeof`의 부분 집합
 + [48부](48_Static/Readme.md): `static`의 부분 집합
 + [49부](49_Ternary/Readme.md): 삼항 연산자
 + [50부](50_Mop_up_pt1/Readme.md): 정리, 1부
 + [51부](51_Arrays_pt2/Readme.md): 배열, 2부
 + [52부](52_Pointers_pt2/Readme.md): 포인터, 2부
 + [53부](53_Mop_up_pt2/Readme.md): 정리, 2부
 + [54부](54_Reg_Spills/Readme.md): 레지스터 스필링
 + [55부](55_Lazy_Evaluation/Readme.md): 지연 평가
 + [56부](56_Local_Arrays/Readme.md): 지역 배열
 + [57부](57_Mop_up_pt3/Readme.md): 정리, 3부
 + [58부](58_Ptr_Increments/Readme.md): 포인터 증가/감소 수정
 + [59부](59_WDIW_pt1/Readme.md): 왜 동작하지 않는가, 1부
 + [60부](60_TripleTest/Readme.md): 삼중 테스트 통과
 + [61부](61_What_Next/Readme.md): 다음은 무엇인가
 + [62부](62_Cleanup/Readme.md): 코드 정리
 + [63부](63_QBE/Readme.md): QBE를 사용한 새로운 백엔드
 + [64부](64_6809_Target/Readme.md): 6809 CPU를 위한 백엔드

향후 부분에 대한 일정이나 시간표는 없다. 새로운 내용이 추가되었는지 수시로 확인하면 된다.

## 저작권

[SubC](http://www.t3x.org/subc/) 컴파일러의 작성자 Nils M Holm의 코드와 아이디어 중 일부를 차용했다. 그의 코드는 공용 도메인이다. 내 코드는 충분히 다르기 때문에 다른 라이센스를 적용할 수 있다고 판단했다.

별도로 명시하지 않는 한,

 + 모든 소스 코드와 스크립트는 GPL3 라이센스 하에 Warren Toomey가 저작권을 가진다.
 + 소스 코드가 아닌 모든 문서(영문 문서, 이미지 파일 등)는 크리에이티브 커먼즈 BY-NC-SA 4.0 라이센스 하에 Warren Toomey가 저작권을 가진다.
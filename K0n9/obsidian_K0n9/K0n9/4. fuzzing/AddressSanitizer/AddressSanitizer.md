### ASAN
AddressSanitizer는 Google이 개발한 도구로 사용 후 오류 및 메모리 누출과 같은 메모리 엑세스 오류를 탐지한다. GCC 버전 >= 4.8 에 내장되어 있으며 C, C++에서 모두 사용할 수 있다.
Address Sanitizer는 런타임 계측기를 사용하여 메모리 할당을 추적하므로 해당 기능을 사용하려면 Address Sanitizer로 코드를 구축해야한다.

이 tool은 아래와 같은 오류를 탐지할 수 있다.
- Use after free (dangling pointer dereference)
- Heap buffer overflow
- Stack buffer overflow
- Global buffer overflow
- Use after return : 이미 반환된 함수의 매개변수를 사용하려 한다.
- Use after scope : 다른 지역에서 선언된 변수를 사용하려 한다.
- Initialization order bugs : 객체나 변수의 초기화 순서 문제이다.
- Memory leaks


![](https://i.imgur.com/2f7f0DN.png)
## ASAN 분석
우선 ASAN은 위의 그림과 같은 로그를 보여준다.
여기서 우리가 알 수 있는것은 EXTRACT_16BITS에서 heap-buffer-overflow가 발생했다는 것이다.
\[ ] 가 가리키는 부분이 접근하여 터진 부분이다.

여기서 우리는 shadow byte에 대해서 알아야 한다.

#### shadow byte란?
가상 주소 공간에서 8바이트 마다 하나의 섀도 바이트를 사용하여 설명할 수 있다.

하나의 섀도 바이트는 다음과 같이 현재 액세스 할 수 있는 바이트 수를 설명한다.
- 0은 8바이트 모두를 의미한다.
- 1-7은 1~7바이트를 의미한다.
- 음수는 진단 보고에 사용할 런타임의 컨텍스트를 인코딩합니다.

여기서 섀도 바이트가 매핑되는 주소는 아래와 같이 구한다.
```C
//x86
Shadow = (Mem >> 3) + 0x20000000;

//x64
Shadow = (Mem >> 3) + 0x7fff8000;
```

그럼 (shadow_byte_value - 0x7fff8000) << 3 을 하면 매핑된 주소를 알 수 있다.
그래서 0x0c067fff8020 의 매핑된 주소를 구해보면 0x603000000100 임을 구할 수 있다.
(3shift를 하니까 shadow_byte_value에 1을 더하면 실제 주소는 8 씩 늘어난다. 즉 한 바이트로 8바이트를 표현 가능한 것이다.)

따라서 shadow byte는 프로그램 주소 지정 가능한 메모리의 상태를 설명하는 메타데이터이다.

#### 예제
```C
#include <stdio.h>
#include <string.h>

int main(){
        char a[20];
        printf("introduce : ");
        scanf("%s", a);
        printf("introduce : %s\n",a);

        return 0;
}
```
컴파일
```
gcc -fsanitize=address -static-libasan -g -o asan_test asan_test.c
```
-fsanitize=address : 컴파일러에 Address Sanitizer를 추가하도록 한다.
-static-libasan : OSC 시스템의 일부 환경 설정으로 인해 asan를 정적 링크를 한다. 
-g : 디버그 심볼과 같이 컴파일한다. 만약 디버그 심볼이 주어지면 line number가 출력된다.

해당 코드는 stack buffer overflow가 발생한다.
buffer overflow를 발생시켜보자.
![](https://i.imgur.com/2qLWt5H.png)
다음과 같은 로그가 뜬다.

하나하나 보자

![](https://i.imgur.com/fSsfpXI.png)
맨 위에 AddressSanitizer가 stack buffer overflow를 감지한 것을 알 수 있다.
그리고 쓰기가 발생한 위치의 backtrace를 표시한다.
scanf에서 터졌음을 알 수 있다.

![](https://i.imgur.com/TriLnaO.png)
다음으로는 쓰임으로 인해 터진 부분의 메모리 주소와 shadow byte를 보여준다.
(0x10007fff7c20 - 0x7fff8000) << 3 = 0x7FFFFFFFE100 
해당 주소를 구할 수 있다.

![](https://i.imgur.com/ETdQJ6f.png)
이는 실제 메모리의 상태를 설명하는 shadow byte 의 hexdump이다. 

전체 8바이트가 주소지정이 가능하다면 그에 해당하는 shadow byte는 00값을 가진다. 만약 부분적으로 주소지정이 가능하다면 01 ~ 07이 될 것이다. 이는 라인 내 주소지정이 가능한 바이트의 개수일 것이다.(02면 2byte 가능)

fa : Heap left [[redzone]], 이 값은 힙 할당 보호 영역이다.
fd : Fread heap region, fread로 생긴 힙 영역
f1 : Stack left redzone, stack의 왼쪽 보호 영역
f2 : Stack mid redzone, stack의 중간 보호 영역
f3 : Stack rigth redzone, stack의 오른쪽 보호 영역
등등

아무튼 만약 redzone을 hit 한거라면 건드려서는 안되는 부분을 건드렸다고 이해하면 된다.

20크기가 딱 맞춰져 있다.
![](https://i.imgur.com/qSs6bM9.png)
8 + 8 + 4 = 20

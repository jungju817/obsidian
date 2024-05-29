---
tistoryBlogName: k0n9
tistoryTitle: type confusion
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "38"
tistoryPostUrl: https://k0n9.tistory.com/38
---
### Type Confusion이란?
type confusion은 프로그램에서 사용하는 변수나 객체를 선언 혹은 초기화 되었을 때와 다른 타입으로 사용할 때 발생하는 취약점이다.
해당 취약점이 존재한다면 memory corruption이 유발되어 공격자가 프로그램을 공격하는 것이 가능해질 수 있다.

#### 예제
```c
#include <stdio.h>

int main(){ 
  int a; 

  scanf("%d", &a); 

  puts(a);
}
```
해당 코드는 정수를 입력받아 해당 값을 출력해주는 코드이다. 하지만 puts함수는 char * 형 포인터를 인자로 받는다. 그래서 type confusion이 발생하여 존재하지 않는 메모리를 참조해서 비정상 종료된다.

![](https://i.imgur.com/OZSk13o.png)

### Type Error 예시
- Out of Range : 데이터 유실
	  값이 너무 커져서 해당 자료형의 저장 가능한 범위보다 큰 값을 저장하려 하면 넘치는 부분은 버려진다.
	  예를 들어 4바이트 크기에 18!=0x16beecca730000 를 저장하려고 하면 하위 4바이트인 0xca730000 만 옮겨진다.
- Out of Range : 부호 반전과 값의 왜곡
	  정수형 변수 n에 -1를 입력한다. 그런데 이 n은 함수 a에서 사용하려는데 함수 a의 매개변수는 unsigned int 라면 -1을 4294967295 로 전달을 하게 된다.
- type overflow, underflow
	  변수의 값이 연산중에 자료형의 범위를 벗어나면 갑자기 크기가 커지거나 작아진다. 예를 들면 int형 최대값에서 1을 더한 순간 0xf0000000 가 되면서 음수 최솟값이 된다.

### 예제_제공파일
나중나중
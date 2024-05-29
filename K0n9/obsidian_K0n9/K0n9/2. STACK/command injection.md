command injection은 검증되지 않은 공격자의 입력을 셸 커맨드로 처리해 정상적인 흐름을 변경할 수 있는 취약점이다. 
공격자는 메타문자와 같은 특수한 문자를 활용해 임의 코드 실행까지 이어지게 할 수 있다. 아래는 메타문자에 대한 설명이다.

|   |   |   |
|---|---|---|
|`$`|쉘 환경변수|```$ echo $PWD/home/theori```|
|`&&`|이전 명령어 실행 후 다음 명령어 실행|```$ echo hello && echo theorihellotheori```|
|`;`|명령어 구분자|```$ echo hello ; echo theorihellotheori```|
|`\|`|명령어 파이핑|```$ echo id \| /bin/shuid=1001(theori) gid=1001(theori) groups=1001(theori)```|
|`*`|와일드 카드|```$ echo .*. ..```|
|`` ` ``|명령어 치환|```$ echo `echo hellotheori`hellotheori```|

아래의 예제는 사용자가 입력한 IP를 ping 명령어로 system 함수를 실행하는 코드이다.
```c
// gcc -o cmdi cmdi.c
#include <stdlib.h>
#include <stdio.h>
int main()
{
    char ip[36];
    char cmd[256] = "ping -c 2 ";
    printf("Alive Checker\n");
    printf("IP: ");
    read(0, ip, sizeof(ip)-1);
    printf("IP Check: %s",ip);
    strcat(cmd, ip);
    system(cmd);
    return 0;
}
```
해당 코드는 사용자의 입력에 검증이 없기 때문에 예상치 못한 결과가 발생할 수 있다.
만약 입력에 IP나 도메인이 아닌 특수 문자를 사용한다면 원하는 명령어를 삽입하여 하나 이상의 명령어를 실행할 수 있다.

![](https://i.imgur.com/GWWoeId.png)

![](https://i.imgur.com/59hAhcF.png)
참고로 -c 2 는 패킷 2번만 보내라는 의미이다.
---
tistoryBlogName: k0n9
tistoryTitle: race condition
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "37"
tistoryPostUrl: https://k0n9.tistory.com/37
---
### race condition이란?
둘 이상의 프로세스가 하나의 리소스에 사전 협의 없이 동시에 접근 가능할 때, 상호 간에 리소스의 현재 정보에 대해 동기화가 되지 않아 발생하는 취약점이다.

이 취약점은 임시파일이 사용되는 환경에서 발생하는데 만약 실행 중에만 중요한 파일을 임시로 생성후 사용이 끝난 후 삭제하는 프로그램에서 발생한다.

### 예제
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>

void setup_environment() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

int main(int argc, char *argv[])
{
	setup_environment();
    struct stat st;
    FILE *fp;
    char buf[0x30] = {0,};
  
    if (access("temp", F_OK | W_OK) == 0)
        remove("temp");

    printf("Input: ");
    read(0, buf, 0x30);
  
 
    fp = fopen("temp", "a");
  
 
    fprintf(fp, "%s\n", buf);
    fprintf(stderr, "Write Success!!!\n");
    fclose(fp);
  
    remove("temp");
    exit(EXIT_SUCCESS);
}
```

이 코드는 temp라는 파일에 값을 입력하는데 만약 기존에 temp라는 파일이 있으면 삭제한다.

만약 remove("temp"); 와 fp = fopen("tem", "a") 사이에 temp라는 심볼릭 링크 파일을 만들면 그 대상에 race condition!! 이 쓰일 것이다.

read로 buf를 입력받는 동안 
ln -s /etc/passwd temp
로 /etc/passwd를 가리키는 temp 심볼릭 링크 파일을 만든다.

그리고 race condition이라는 입력을 하면 /etc/passwd에 값이 쓰인다.

![](https://i.imgur.com/RFYfdRe.png)
![](https://i.imgur.com/7G6puVR.png)

![](https://i.imgur.com/sUk7lKT.png)

![](https://i.imgur.com/v30rOKf.png)

#### 해결방법
해당 파일이 심볼릭 링크로 연결이 되었는지 검사하는 문구를 추가한다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <seccomp.h>
#include <sys/ptrace.h>

typedef struct _functname{
}fuctname;

void check() __attribute__((constructor));

void detect(){
    if(ptrace(PTRACE_TRACEME, 0, 0, 0) == -1){
        printf("Debugging Detected\n");
        exit(-1);
    }
}

void sb(){
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_ALLOW);

    if(ctx=NULL){
        exit(0);
    }
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execveat), 0);
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(open), 0);
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(dup), 0);
    seccomp_load(ctx);
}

void initial(){
    setvbuf(stdin, 0, 2, 0);
    setvbuf(stdout, 0, 2, 0);
    setvbuf(stderr, 0, 2, 0);
    sb();
    detect();
}

int main(){
    initial();
    printf("blablalbablaa~~~\n");
    return 0;
}
```
detect 함수로 디버깅 중인지를 판단하여 동적 디버깅을 막을 수 있다.
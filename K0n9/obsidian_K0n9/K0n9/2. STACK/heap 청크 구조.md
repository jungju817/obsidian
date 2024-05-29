---
tistoryBlogName: k0n9
tistoryTitle: heap 청크 구조
tistoryVisibility: "3"
tistoryCategory: "1136868"
tistorySkipModal: true
tistoryPostId: "43"
tistoryPostUrl: https://k0n9.tistory.com/43
---
**Heap이란?** 

프로그램이 실행되는 도중 동적으로 할당하고 해제하여 사용하는 메모리 영역이다.

![](https://blog.kakaocdn.net/dn/Rn0t4/btsl1zItRJK/ViNxMcTpWMVc5241wLgfRK/img.png)

heap의 위치

현재 리눅스에서는 Memory Allocator로 ptmalloc2를 사용하고 있다. ptmalloc의 구현목표는 메모리의 효율적인 관리이다. 효율적인관리에는 크게 3가지 핵심 목표가 있다.

**1)** **메모리 낭비 방지**

메모리의 동적 할당과 해제는 매우 빈번하게 일어난다. 공간이 무한하지 않기 때문에 ptmalloc은 메모리 할당 요청이 발생하면 먼저 해제된 메모리 공간 중에 재사용이 가능한 공간이 있는지 탐색하고 해제된 메모리 공간중에 요청된 크기와 같은 크기의 메모리 공간이 있다면 이를 재사용한다. 또한 작은 크기를 요청할 경우 해제된 메모리 공간 중 매우 큰 메모리 공간이 있으면 그 영역을 나누어준다.

**2) 빠른 메모리 재사용**

해제된 메모리 공간을 빠르게 재사용하려면 해제된 메모리 공간의 주소를 기억하고 있어야 한다. 이를 위해 ptmalloc은 메모리 공간을 해제할 때, tcache또는 bin이라는 연결 리스트에 해제된 공간의 정보를 저장해둔다. tcache와 bin은 크기에 따라 나눠져있어서 관련된 저장소만 탐색하면 되므로 더욱 효율적으로 공간을 재사용할 수 있다.

**3) 메모리 단편화 방지**

메모리 단편화는 내부 단편화와 외부 단편화가 있다.

  **내부 단편화 :** 할당한 메모리 공간의 크기에 비해 실제 데이터의 크기가 적어서 남은 공간이 낭비된다.

  **외부 단편화 :** 할당한 메모리 공간들 사이에 공간이 많아서 충분한 크기가 있음에도 불연속적인 공간이여서 작업을 수행하지 못한다.

메모리 단편화를 줄이기 위해서 ptmalloc은 정렬과 병합, 분할을 사용한다. 

  **정렬 :** ptmalloc은 메모리 공간을 16바이트 단위로 정렬한다. 즉 1~16바이트를 요청하면 모두 16바이트로 할당하고 17~32바이트를 요청하면 모두 32바이트만큼 할당한다. 이는 내부 단편화가 일어나지만 외부 단편화는 감소시키는 효과가 있다. 이러한 방식을 사용하는 이유는 대부분 비슷한 크기의 요청이 발생할 확률이 높기 때문에 chunk의 재사용률을 높일 수 있기 때문이다.

  **병합 :** ptmalloc은 특정 조건을 만족하면 해제된 공간들을 합쳐서 큰 크기의 요청에 반환한다. 

  **분할 :** ptmalloc은 작은 크기의 요청에는 분할하여 재사용한다.

우분투에서 사용하는 memory allocator는 ptmalloc2이지만 기본 알고리즘은 dlmalloc이니 dlmalloc을 기준으로 설명하겠다. (참고로 dlmalloc에서 멀티 쓰레드 기능이 추가된게 ptmalloc이다)

heap은 특정 크기(0x21000)의 메모리 영역을 미리 할당한 뒤 이 영역을 사용하는 방식으로 free를 해도 이 영역안에 남아있다.

```cpp
#include <stdlib.h>

int main()
{
    unsigned long *p1;
    p1 = malloc(0x640);
    free(p1);
}
```

![](https://blog.kakaocdn.net/dn/GdbEe/btsmzKB99nH/NHKegKXc3f5FsS5FidjX5k/img.png)

malloc함수 실행 전과 실행 후 부분에 break를 건다.

![](https://blog.kakaocdn.net/dn/QBjtE/btsmBheU7y4/7yDp7Jql6AktmPgeTjpeXK/img.png)

malloc 실행 전

최초의 malloc이 실행되기 전에는 heap영역이 설정되어있지 않다.

![](https://blog.kakaocdn.net/dn/bgOLZa/btsmF55H6H4/pLsZFK90RGKRhYNTVZCaO1/img.png)

malloc 실행 후

malloc이 실행 된 후에 0x21000크기의 heap 영역이 설정된 것을 볼 수 있다.

malloc이 실행되지 않았다고 heap 영역이 아예 없는것은 아니다. initial heap이 기본적으로 설정되어 있으며 사용함에 따라 heap segment를 늘려가는 방식이다.

## chunk

chunk는 mem과 header로 나누어진다.

mem : malloc으로 할당받은 부분

header : prev_size + size

prev_size : 인접한 앞 쪽 chunk의 크기

size : 자신의 chunk의 크기

prev_size는 인접한 앞 쪽 chunk가 free되면 초기화가 된다. 

![](https://blog.kakaocdn.net/dn/bICFu4/btsmBg1sdLC/3mgW1kRuQwVr52lckjqBu0/img.png)

chunk의 기본 구조

참고로 chunk의 정렬은 mem의 크기가 정렬되고 정렬후에 header가 붙게 된다.

## chunk size

chunk의 크기는 메모리 단편화 방지를 위해 x86 에서는 8바이트, x64 에서는 16바이트 단위로 정렬된다. 그래서 chunk의 size는 x86에서는하위 3비트, x64에서는 하위 4비트가 사용되지 않아서 하위 3비트는 chunk의 3가지 속성을 나타낸다.(그래서 size에서 하위 3비트는 실제 크기에 포함되지 않는다) 

000 (1, 2, 3)

1) non_main_arena

thread arena에 속하면 1

thread arena에 속하지 않으면 0

2) is_mmapped

mmap으로 할당 되었으면 1

mmap으로 할당되지 않았으면 0

3) prev_inuse bit

인접한 이전 chunk가 할당되어있으면 1

인접한 이전 chunk가 해제되어있으면 0

arena란 heap영역을 전체를 관리하는 구조체이다. 

mmap이란 그냥 큰 메모리 요청이 들어왔을 때 malloc대신 호출되는 함수라고 보면 된다.

우리는 일단 prev_inuse bit만 외워두자.

(참고로 첫 번째 chunk는 이전의 chunk가 없으니 prev_inuse bit가 필요없지만 이전 영역이 bss영역이니 그냥 prev_inuse bit를 1로 고정한다.)

## chunk의 종류

```cpp
//예시 코드
#include <stdio.h>
int main(){
    char *p = malloc(0x30);
    *p = 'a';
    free(p);
    return 0;
}
```

Allocated chunk

malloc()을 호출했을 때 heap 영역에 생기는 chunk이다.

![](https://blog.kakaocdn.net/dn/dikGGJ/btsmDIxfFrl/7ReGMyaEPTZtKC1rjS5j6K/img.png)

chunk의 상태

malloc으로 0x30을 요청하니 size가 0x41로 되어있다. 이는 \[요청한 크기인 0x30] + \[header크기인 0x10] + \[prev_inuse bit 1] 이다.

data로 0x61 즉 'a'가 들어간 것을 알 수 있다.

![](https://blog.kakaocdn.net/dn/elcIYi/btsmA1YHpic/ouffoh1uWJnOKAD53xhGlK/img.png)

Freed chunk

말 그대로 free된 chunk 이다. 이때 free()를 했을 때 실제로 반환되는 것이 아니라 allocated chunk 구조에서 free chunk 구조로 변경이 되는 것이다. 변경되는 부분은 data가 있던 부분에 fd, bk가 생기게 된다. 이때 large bin 크기의 chunk이면 fd_nextsize, bk_nextsize가 추가로 생긴다.

![](https://blog.kakaocdn.net/dn/sL0Go/btsmFggnMiK/rAU44hnP9GpcDH7T85eaWk/img.png)

chunk의 상태

원래 data였던 부분에 fd와 bk가 쓰여졌다. 원래 있던 data는 신경쓰지 않고 그대로 overwrite한다.

기본적으로 free 과정에서 data 부분을 조작하거나 지우지는 않는다. 하지만 data 부분의 첫 8바이트와 다음 8바이트는 fd와 bk로 overwrite되고 그 이후의 data는 남아있게 된다. fd와 bk는 arena에 있는 주소를 가리키게 된다. fd와 bk는 binning에서 자세하게 다루겠다.

만약 동일한 크기의 chunk가 할당될때는 freed chunk가 allocated chunk로 변환되어 재사용된다.

![](https://blog.kakaocdn.net/dn/ZjtBU/btsmGKOFwE0/8CCSbkroGBFtQnmz7bTaUK/img.png)

Top chunk

힙 영역에 가장 마지막에 위치한다. 새롭게 할당되면 top chunk에서 분리해서 반환하고 top chunk에서 인접한 chunk가 free되면 병합한다. 

![](https://blog.kakaocdn.net/dn/bF6DMt/btsmFfIAGyu/YCSzKfMoZgSkptJShT1WLk/img.png)

top chunk의 위치

만약 heap에 충분한 공간이 없으면 top chunk를 확장하여 추가적인 메모리를 할당한다.

참고로 저 알수 없는 0x291은 tcache와 관련된 공간으로 tcache_perthread_struct구조체에 해당하며, libc 단에서 힙 영역을 초기화 할 때 할당하는 청크이다.

### Boundary Tag

chunk가 할당 될 때는 해당 chunk의 크기 정보가 해당 chunk의 size에 저장이 된다.

chunk가 해제 될 때는 해당 chunk의 크기 정보가 다음 chunk의 prev_size에 저장이 된다.

이러한 정보로 우리는 **인접한 앞/뒤 chunk의 주소 계산이 가능**하다.

#### 인접한 다음 chunk의 주소

![](https://blog.kakaocdn.net/dn/ZCKfz/btsmUHLdr0B/BJAF0rPI7I1YrybXUcHZwK/img.png)

allocated(A)와 allocated(B)

chunk(A)를 기준으로 **chunk_addr(A) + size(A) = chunk_addr(B)** 가 된다. 이를 통해 다음 chunk의 주소를 알 수 있다.

#### 인접한 앞의 chunk가 free된 경우의 freed chunk의 주소

![](https://blog.kakaocdn.net/dn/bR80xJ/btsm1Txjshf/R9R4jugAN29r4OCqKxHYf0/img.png)

인접한 앞의 chunk가 free된 경우

chunk(B)를 기준으로 chunk(A)가 해제되었다면 chunk(B)의 prev_size가 초기화 된다.

그래서 **chunk_addr(B) - prev_size(B) = chunk_addr(A)** 를 통해 chunk(A)의 주소를 구할 수 있다.

### binning

heap은 chunk를 재사용한다. 이때 free된 chunk를 크기 단위로 관리하는 것이 bin이다.

#### fast bin

- 최소 크기에서 8바이트(x86환경) 또는 16바이트(x64환경) 씩 정렬된다.

- 하나의 linked list에서는 size가 모두 같다.

- 10개(7개)의 bin으로 chunk를 single linked list 로 관리한다. FILO구조여서 fd만 사용한다.  

- size는 16 ~ 64바이트(x86환경) 또는 32 ~ 128바이트(x64환경) 이다.

- 실제로는 32 ~ 176 바이트(x64환경)의 크기를 사용해서 10개이지만 7개만 사용한다. 그래서 128바이트까지이다.

- 메모리 단편화보다는 **속도를 우선**한다.

- 해제되어도 다음 chunk의 prev_inuse 값이 변경되지 않는다. 

    => **인접한 chunk와 병합되지 않는다.**

#### small bin

- 최소 크기에서 8바이트(x86환경) 또는 16바이트(x64환경) 씩 정렬된다.

- 하나의 linked list에서 size가 모두 같다.

- 62개의 bin으로 chunk를 double linked list로 관리한다. FIFO구조이다.

- size는 16 ~ 512바이트(x86환경) 또는 32바이트 ~ 1024바이트(x64환경) 이다.

- 해제가 되면 뒤 chunk의 prev_inuse bit 가 clear되고 prev_size가 초기화된다.

    => **free할 때 인접한 freed chunk와 병합된다.**(특수한 경우에 병합된다. unsorted bin 참고)

#### large bin

- 특정 범위의 size가 다른 chunk를 63개의 bin으로 double linked list로 관리한다. FIFO구조이다.

    => 하나의 linked list에서 서로 다른 size도 포함된다.

- size는 512바이트(x86) 또는 1024바이트(x64) 이상의 chunk가 들어간다.

- 한 linked list 내에서 size가 큰 chunk가 제일 앞으로 간다.

    => 한 linked list 내에서 다른 작은 크기를 가진 chunk를 가리키는 포인터로 fd_nextsize, bk_nextsize가 존재한다.

- 메모리 할당과 반환의 속도가 가장 느리다.

- 128KB 이상의 공간이 할당되는 경우 mmap() 시스템 콜을 거쳐 별도의 공간을 할당한 뒤에 chunk가 생성된다. 이렇게 생성될 경우 bin에 속하지 않고 IS_MMAPPED 플래그를 체크하며 munmap()으로 해제한다.

- 해제가 되면 뒤 chunk의 prev_inuse bit가 clear되고 prev_size가 초기화된다.

    => **free할 때 인접한 freed chunk와 병합된다.**

- fd(next_size)로 갈수록 chunk의 size는 작아지고 bk(next_size)로 갈수록 chunk의 size는 커진다.

- 같은 size를 가진 chunk 무리에서 가장 앞의(큰 chunk에 가까운)chunk만 fd_nextsize, bk_nextsize를 가진다.

-제일 큰 chunk의 bk_next_size는 제일 작은 chunk의 가장 앞 chunk를 가리킨다.(반대도 같음)

- 새로운 chunk가 들어갈 때 기본적으로 가장 앞으로 들어가지만 nextsize를 가진 chunk의 앞으로 가지는 않는다.

![](https://blog.kakaocdn.net/dn/ckQ5vY/btsmRMM7HjI/srQ15uk3KQykZl54JQUM91/img.png)

large bin에서의 삽입 과정

#### unsorted bin

- size 제한이 없는 double linked list 이다. FIFO 구조이다.

- 1개의 bin이 있다.

- unsorted bin에 들어가는 경우

    1) free 했을 때 각 bin(small bin, large bin)으로 들어가기 전(fast bin 제외)

    2) fast bin 들이 병합되어 합쳐질 때

        => fast bin은 일반적으로 병합이 되지 않지만 큰 크기의 chunk의 요청이 있으면 인접한 fast bin의 chunk를 한꺼번에 병합한다. 이러한 것을 malloc_consolidate라고 한다.

    3) best fit으로 할당된 chunk의 남은 부분인 remainder

    4) 병합된 free chunk

- unsorted bin에 나오는 경우

    1) 사용자가 malloc을 호출하여 요청한 size와 동일한 chunk가 있을 때 해당 chunk가 allocated chunk로 바뀌면서 빠져나온다. 

    2) 사용자가 malloc을 호출했는데 요청한 chunk와 적합한 chunk가 없으면 unsorted bin에 있는 chunk를 각자 크기에 맞춰 bin으로 들어간다.

![](https://blog.kakaocdn.net/dn/uj7v8/btsmPHeGYGO/lCl3MnqaYLvHLupxI36QMK/img.png)

표

#### 병합

free하려는 chunk의 앞, 뒤 chunk가 이미 해제되어 있을 때 병합이 발생한다.

![](https://blog.kakaocdn.net/dn/q1BlI/btsm9BDlZUE/miFFlsODv8WWEcY5sDS6zk/img.png)

해제하려는 chunk의 인접한 이전 chunk가 해제되어있는 겨우

해제하려는 chunk의 인접한 이전 chunk가 해제되어있는 경우에는 다음과 같은 순서에 따라 병합된다.

1) chunk(B)를 해제하려고 한다.

2) 이때 prev_inuse bit 가 설정되어있는지 보고 clear되어있으면 이전 chunk가 해제되어있다고 판단을 한다. 그리고 이전 chunk가 해제되어있으니 prev_size는 초기화되어있다.

3) chunk_addr(B)에서 prev_size를 빼면 chunk_addr(A)가 나온다.

4) 병합하려는 두 chunk의 주소를 알았으니 병합을 진행한다.

![](https://blog.kakaocdn.net/dn/rQWpt/btsm4CKmczH/kXng50gxugiFm4mlpxgPgk/img.png)

해제하려는 chunk의 인접한 이후 chunk가 해제되어있는 경우

해제하려는 chunk의 인접한 이후 chunk가 해제되어있는 경우 다음과 같은 순서에 따라 병합된다.

1) chunk(A)를 해제하려고 한다.

2) chunk_addr(A)에 chunk_size(A)와 chunk_size(B)를 더한다. 그러면 chunk_addr(C)를 구할 수 있다.

3) chunk(C)의 prev_inuse bit가 clear되어있으면 이전 chunk(B)가 해제되어있다고 판단한다.

4) 두 chunk를 병합한다.

병합을 위해서는 binlist에 등록된 chunk를 제거해야 해서 freed chunk의 fd와 bk를 정리해줘야한다. 이를 **unlink**라고 한다.

### heap 영역이 할당되는 과정

기본적으로 malloc 등의 함수를 사용하지 않아도 132KB 크기의 initial heap이 존재한다. 그리고 할당 가능한 요구가 들어오면 heap segment를 확장한다. 여기서 확장하는 것을 sbrk()라고 한다.

start_brk : 프로그램 초기 heap 주소를 나타내는 변수

break location : heap의 끝 주소

brk() : 프로세스의 heap 끝 주소를 설정하는데 사용된다. 프로세스가 런타임 중에 메모리를 필요로 할 때마다 break location을 조정하여 힙의 크기를 증가시키거나 감소시킨다. 

sbrk() : brk와 비슷한 역할을 하는데 brk와 차이점으로는 인자값이 상대적인 값으로 받는다는 것이다. 

예를 들면

brk(주소 값) 은 주소 값까지 메모리를 할당하고

sbrk(주소 값) 은 주소 값 만큼 메모리를 추가 할당한다.

즉 start_brk 부터 break location 까지가 heap segment이며 brk()(sbrk())를 통해 heap segment를 확장시킨다.

![](https://blog.kakaocdn.net/dn/TYvqa/btsoxdbWgOS/zKYDlIVAOqtZkNsW6XvlI0/img.png)

heap 할당 과정

### arena

arena란 heap영역 전체를 관리하는 구조체이다. fastbin, smallbin, largebin 등의 정보를 모두 담고 있다. arena는 크게 main arena와 sub arena(main arena가 아닌 arena)로 나뉜다.

#### main arena

메인 쓰레드로 생성된 것이어서 main_arena이다. 이는 단일 스레드용 프로그램을 위해 존재한다. main_arena는 하나의 heap만 가질 수 있으며 heap_info 구조체를 가질 수 없다. main_arena는 malloc_state구조체를 사용하는 변수이며 이는 top chunk와 비슷하게 공간이 부족하면 더욱 커지고 비어있는 공간이 많으면 줄어들기도 한다. 구조체는 아래와 같다.

```cpp
static struct malloc_state main_arena =
{
  .mutex = _LIBC_LOCK_INITIALIZER,
  .next = &main_arena,
  .attached_threads = 1
};
```

```cpp
struct malloc_state
{
  /* Serialize access.  */
  __libc_lock_define (, mutex);

  /* Flags (formerly in max_fast).  */
  int flags;

  /* Set if the fastbin chunks contain recently inserted free blocks.  */
  /* Note this is a bool but not all targets support atomics on booleans.  */
  int have_fastchunks;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];

  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr top;

  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;

  /* Normal bins packed as described above */
  mchunkptr bins[NBINS * 2 - 2];

  /* Bitmap of bins */
  unsigned int binmap[BINMAPSIZE];

  /* Linked list */
  struct malloc_state *next;

  /* Linked list for free arenas.  Access to this field is serialized
     by free_list_lock in arena.c.  */
  struct malloc_state *next_free;

  /* Number of threads attached to this arena.  0 if the arena is on
     the free list.  Access to this field is serialized by
     free_list_lock in arena.c.  */
  INTERNAL_SIZE_T attached_threads;

  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};
```

![](https://blog.kakaocdn.net/dn/XTz54/btsoypijwHO/QY6AMQjhT8UwW4DTN0FVy1/img.png)

너무 길어서 중간은 잘랐다

![](https://blog.kakaocdn.net/dn/dTYscB/btsoGivC5Pv/AV3RCTH1GdiRo0TqZ00TvK/img.png)

여기서 bins의 첫 번째와 두 번째 는 unsorted bin이고 다음으로 small bin 그다음 large bin이다. 

그림으로 보면 편한다.

#### bins의 구조

기본적으로 bin이 비어있으면 이전 bin의 주소를 가리킨다.

![](https://blog.kakaocdn.net/dn/bSmsv3/btsoxHDYuoT/9QtEMZPYzOLU6XbSqoZVd1/img.png)

비어있는 bins

여기서 bin에 chunk가 들어가면 이렇게 된다.

![](https://blog.kakaocdn.net/dn/rU8cf/btsp3mxPew5/F1ERDMIKKmQRalCIj5bY4K/img.png)

진짜 엄청 열심히 그렸다

그림상으로는 fd와 bk가 다음 fd, bk를 가리키지만 실제로는 chunk의 주소를 가리키는것이다.

여기서 chunk가 들어올 때는 header의 fd를 참조하여 맨 왼쪽으로 들어오고 chunk가 나갈때는 header의 bk를 참조하여 맨 오른쪽 chunk가 나간다.

단일 연결 리스트인 fast bin은 구조가 살짝 다르다.

![](https://blog.kakaocdn.net/dn/bXXclB/btsoyXZ9t7D/amfXpfk5QYOMnCuJuwk261/img.png)

fastbinsY

마찬가지로 fd는 chunk의 주소를 가리키는 것이다.

chunk가 들어올 때도 왼쪽으로 들어오고 나갈때도 왼쪽에서 나간다.

#### sub arena

새로운 쓰레드가 생성될 때 다른 스레드를 기다리는 것을 줄이기 위해 새로운 arena를 생성하게 되는데 이를 sub arena라고 부른다. 이는 sbrk()를 사용하는 main arena와 달리 mmap()을 통해 새로운 힙 메모리를 할당받으며 mprotect()를 사용하여 확장한다. 또한 sub arena는 main_arena와 다르게 여러 개의 서브 힙과 heap_info 구조체를 가질 수 있다.

![](https://blog.kakaocdn.net/dn/odqWm/btsoyU969mU/jXhHUtrUMg3kUycgRBhDa0/img.png)

main arena와 sub arena

#### heap_info

heap_info 구조체는 힙의 첫 번째 구조체이다. 이는 **힙의 크기를 지정하고 arena의 데이터 구조에 대한 포인터를 제공**한다. 이는 멀티쓰레드 환경에서 필요한 것이기에 단일 스레드에서 사용되는 main_arena는 필요 없는 구조체이다.

### 멀티 쓰레드

일반적으로 단일 arena에서는 모든 스레드가 동일한 arena를 사용한다. 이를 single arena malloc이라 한다.

하지만 멀티 스레드 환경에서는 각 쓰레드마다 별도의 arena를 사용할 수 있다.(모든 스레드가 각자의 arena를 가지는것은 아니다.) 이를 multi arena malloc이라고 한다. multi arena malloc은 **각 쓰레드마다 자신만의 arena를 가지고 자신만의 top chunk, free 리스트, 사용 가능한 블록 리스트 등의 것**들을 가지고 있다. 

ptmalloc은 최대 64개의 arena를 생성할 수 있다. 그래서 과도한 멀티 쓰레드 환경은 병목현상이 발생한다. 그리고 이러한 멀티 쓰레드 환경에서 ptmalloc은 레이스 컨디션을 막기 위해 mutex를 적용한다.

#### 임계영역이란(critical section)?

함수 내에서 둘 이상의 쓰레드가 동시에 실행하면 문제를 일으키는 하나 이상의 코드 블록

#### Race condition

race condition은 두 개 이상의 쓰레드가 하나의 데이터를 공유할 때 데이터가 동기화되지 않는 상황을 말한다. 이렇게 될 경우 여러 문제가 발생한다.

#### Mutex(mutual exclusion)

임계영역에 진입할 때 사용하는 잠금장치이다.

작동방식 : mutex를 초기화한 후 임계영역에 mutex lock을 걸어준다. 현재 진입한 스레드 외에 다른 쓰레드는 lock을 건 부분에서 lock이 풀릴때 까지 블로킹 된다. 진입쓰레드가 적업을 마치고 락을 풀고 나가면 기다리던 쓰레드 중 하나가 다시 진입하고 lock을 걸게 된다.

lock() : lock을 걸고 사용권을 얻는다.

unlock() : lock을 해제하고 사용권을 반납한다.

mutex lock은 구현을 잘못하거나 스레드의 수가 과다하게 많아지면 병목현상을 일으킬 수 있다. 락으로 발생하는 대표적인 문제 중 하나가 데드락이다.

#### Dead-Lock

두 스레드가 서로 상대방의 작업이 끝나기를 계속 기다리는 상태이다.

ex) 쓰레드 1이 a, 쓰레드 2가 b를 lock 한 상태에서 쓰레드 1은 b를, 쓰레드 2는 a를 lock 하기 위해 서로 잠금 해제를 기다리는 상황이다.

#### Spinlock

임계 구역에 들어가는데 실패했을 때, 가능한 상태가 될 때까지 계속 돌면서 진입을 시도하는 방식으로 구현된 lock이다.

#### Semaphore

공유된 자원을 한정된 수의 스레드만 접근하도록 막는 것이다. 스레드가 임계영역에 접근할 때 해당 스레드는 semaphore의 count를 감소시키고 종료된 후에는 semaphore의 count를 원래대로 증가시킨다.
---
sticker: emoji//1f600
---
UAF를 이용해서 익스

보호기법 : no pie, no canary, Full Relro

---
일단 fast bin 만 가능하게 함

fastbin 병합시켜서 unsorted bin으로 넣어햐 할듯.

해제된 곳에 입력은 가능한데 출력은 못함
8byte memset

일반 review로 malloc을 large bin 크기만큼 해서 fast bin 병합시켜야함

weak review를 쓸 수 있음, weak를 정의하는거는 전역변수, 여기다 fastbin dup로 chunk할당을해서 weak review를 써서 하는거지

read 3번째 인자는 type confusion으로 하면 될듯

[[prob]]

exploit 시나리오

우선 1, 2 예약을 함 -> chunk1, chunk2 할당
1 예약을 취소함 ->  chunk1 해제
review를 쓰는데 크기를 1104로 함 -> lage bin 할당으로 chunk1이 small bin 이됨
다시 1 예약 -> small bin을 재할당
view로 봄 -> libc leak

그리고 fast bin dup 혹은 fast bin consolidate로 전역변수 weak에 할당하고 edit으로 1로 바꿔버림

그리고 type confusion으로 read 3번째 인자 조져서 익스 완

문제는 pie가 안걸려있으면 libc를 그냥 rop로 구해버릴 수 있

그냥 review는 그냥 weak으로 하고 임의 할당으로 environ에 chunk할당해서 stack leak, canary는 걸려있고, ret부분에 fastbin dup로 할당해서 one_gadget

weak를 일단 person_num으로 함. malloc 할때마다 1씩 늘어나고 60번 이상 malloc 하면 병원 끝났다 하고 종료

이게 person 구조체 위에 있어서 person_num을 0x31로 만들고 여기다 임의 할당, person 구조체 읽을수도 있고 overwrite도 가능 -> heap base leak, stack base leak

그래서 pie는 걸지 말자

언인텐 시잇팔
임의 쓰기가 무제한 가능이면 걍 hook에 one_gadget넣으면 끝임 

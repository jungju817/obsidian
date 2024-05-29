---
tistoryBlogName: k0n9
tistoryTitle: RELRO
tistoryVisibility: "3"
tistoryCategory: "0"
tistorySkipModal: true
tistoryPostId: "32"
tistoryPostUrl: https://k0n9.tistory.com/32
---
## RELRO란?
Relocation Read-Only

elf 바이너리 프로세스의 데이터 섹션의 메모리가 변경되는 것을 보호하는 기법이다.
크게 3가지가 있다.
NO RELRO, Partial RELRO, FULL RELRO

#### NO RELRO
ELF의 기본헤더, 코드 영역을 제외한 거의 모든 부분에 Read, Write 권한을 주는 것이다. 
거의 모든 부분을 읽고 쓰고 할 수 있으니 보안에 매우 취약하다. 

![](https://i.imgur.com/WLsoqyM.png)

파란선 아래로 쓰기가 가능하다. 
.fini_array overwrite, got overwrite 이 가능하다.

#### Partial RELRO
몇몇 부분에 쓰기권한이 없어졌다.

![](https://i.imgur.com/thG2unu.png)

파란선 아래로 쓰기가 가능하다.
.fini_array overwrite는 불가능하다.
got overwrite는 가능하다.
참고로 got overwrite에서 말하는 got영역은 got.plt 부분이다.
.got는 동적 라이브러리 링킹에 필요한 정보를 가진 부분이다.

#### Full RELRO
거의 모든 부분에 write 권한이 없다.

![](https://i.imgur.com/EIdrsuj.png)

파란선 아래로 쓰기가 가능하다.
got overwrite가 불가능하다.
got.plt가 없어진 것처럼 보이는데 .got랑 합쳐졌다. 실제로 까보면 그냥 있다.


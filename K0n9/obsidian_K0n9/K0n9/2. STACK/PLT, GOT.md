---
tistoryBlogName: k0n9
tistoryTitle: PLT, GOT
tistoryVisibility: "3"
tistoryCategory: "1117646"
tistorySkipModal: true
tistoryPostId: "26"
tistoryPostUrl: https://k0n9.tistory.com/26
---
## PLT

외부 프로시저를 연결해주는 테이블, PLT를 통해 다른 라이브러리에 있는 프로시저를 호출해 사용할 수 있다.

## GOT

PLT가 참조하는 테이블, 프로시저들의 주소가 들어있다.

![](https://i.imgur.com/VoHxA2Z.png)

### 처음 호출

함수 호출 -> PLT 이동 -> GOT 참조(null) -> dl_resolve 함수 실행 -> GOT에 함수 주소 쓰여짐 -> 해당 함수로 점프

### 처음 이후의 호출

함수 호출 -> PLT 이동 -> GOT 참조(주소 존재) -> 해당 함수로 점프

### debugging
![](https://i.imgur.com/F12Qbnz.png)

처음 printf를 실행한다.

![](https://i.imgur.com/LG5jhrn.png)
printf_got 로 jump 한다.

![](https://i.imgur.com/5T9Jv8T.png)
\_dl_runtime_resolve_xsavec 실행

![](https://i.imgur.com/cGVIgmw.png)

한 번 실행 후에는 got에 실제 주소가 쓰여 있다.
![](https://i.imgur.com/qma92as.png)


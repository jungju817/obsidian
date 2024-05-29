- target의 공식문서 잘 보기
- 다운 받은 폴더의 README나 INSTALL 같은거 있나 잘 보기 
![](https://i.imgur.com/OmxpTvY.png)
이런식으로 다 있으니 다 읽어보자

---
일단 configure 치면 아래와 같이 나옴

![](https://i.imgur.com/QiqnQ1d.png)

여기서 no인 거를 configure 할 때 다 --without 해주면 됨. workshop이랑 website는 안함. 왠지는 모름. 여기서 중요한거는 Cairo not found 이러면 그냥 --without-cairo 하면 되는데  png : no 인데 설명에 libpng not found이면 --without-libpng 이케 해야함. 그리고 openraw 같은것도 openraw library not found여서 --without-libopenraw 이케 해줘야 함. 근데 좆같은건 jasper은 설명에 jasper library not found인데 시발 --without-jasper 임 
시발 기준을 모르겠지만 아마도 아래와 같은 방법으로 하면 되는것 같음.

1. chat gpt에 ~~~ 설치하고 싶어 라고 질문
2. 그러면 계열에 따라 명령어를 알려주는데 거기서 Arch Linux 에 적힌것으로 --without 하면 됨

![](https://i.imgur.com/XYXi4q7.png)


![](https://i.imgur.com/pPRfFtU.png)

보면 둘다 lib이지만 libpng와 그냥 exiv2 이렇게 좆같이 나뉨. 근데 거의 높은 확률로 알맞는 걸로 나옴. 근데 간혹 버전 등의 문제로 조금씩 다를 수 있음

잘못된 패키지명으로 치면 이렇게 configure 하면 warning 뜸

![](https://i.imgur.com/x5hBsXh.png)


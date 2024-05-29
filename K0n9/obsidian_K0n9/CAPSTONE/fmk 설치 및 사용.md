fmk : firmware mod kit
firmware를 구성 요소로 추출한 다음 파일 시스템 이미지를 추출하는 툴이다.

패키지 설치

```bash
sudo apt-get install git build-essential zlib1g-dev liblzma-dev python3-magic autoconf python-is-python3
```

fmk 설치
```bash
git clone https://github.com/rampageX/firmware-mod-kit.git
cd firmware-mod-kit
```
이럼 끝(뭔가 복붙하면 안됨, 주소는 따로 복붙하자)

사용법
```
./firmware-mod-kit/extract-firmware.sh target_firmware.bin
```
이렇게 하면 fmk 라는 디렉토리가 생긴다.

[[좀 이상했던 점]]

![](https://i.imgur.com/jL8yXx2.png)

fmk를 보면 다음과 같음
![](https://i.imgur.com/JWR677B.png)

여기서 rootfs를 보면 파일시스템이 있다.
![](https://i.imgur.com/tOVksds.png)

https://ufo.stealien.com/2024-02-05/IoT-TechBlog-ko

참고자료


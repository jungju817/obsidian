우선 emacs 설치를 해야한다.

```
wget https://github.com/emacs-mirror/emacs/archive/refs/tags/emacs-28.1.tar.gz

tar -zxvf emacs-28.1.tar.gz

cd ./emacs_emacs-28.1

autoreconf -ivf


```
그리고 texinfo 를 설치해줘야한다.
```
sudo apt-get update

sudo apt-get install texinfo

makeinfo --version
```

그리고 환경변수와 configure로 prefix설정을 해준다.
```
export MAKEINFO=/usr/bin/makeinfo
./configure --prefix="/root/t/toor_emacs/install" --without-x --with-gnutls=ifavailable

make
make install
```
export MAKEINFO=/usr/bin/makeinfo : MAKEINFO라는 환경 변수를 /usr/bin/makeinfo 로 지정한다. makeinfo는 GNU Info 문서를 생성하는 데 사용되는 도구이다.

--without-x : X 윈도 시스템 지원을 끄도록 설정한다. X 윈도우 시스템은 그래픽 인터페이스를 지원하는 시스템으로 해당 옵션을 사용함으로서 그래픽이 아니라 텍스트 기반으로 동작하도록 설정한다.

--with-gnutls=ifavailable : GnuTLS라이브러리를 지원하도록 설정한다. ifavailable은 시스템에 GnuTLS라이브러리가 설치되어 있는 경우에만 이를 사용하도록 설정한다. GnuTLS는 네트워크 통신을 위한 암호화와 보안을 제공하는 라이브러리이다.



### 사용법

실행
```
./install/bin/emacs testfile
```

![](https://i.imgur.com/3AoQvwh.png)


저장하기
```
ctrl + x -> ctrl + s  , 저장 경로 확인
```

나가기
```
ctrl + x -> ctrl + c
```

저장하기와 나가기는 따로 실행한다.

화면 분리
```
(위아래로 분리) ctrl + x -> 2

(양 옆으로 분리) ctrl + x -> 3

(분리된 화면 없애기) ctrl + x -> 0

(분리된 화면간 커서이동) ctrl + x -> o

(현재 커서가 없는 화면에 파일 열기) ctrl + x -> 4f -> 파일이름+Enter
```

확대 or 축소
```
ctrl + x -> ctrl + (+/=) or (-)
```

커맨드 키의 반응이 살짝 느려서 ctrl + x를 하면 왼쪽 아래 C-x- 가 뜨면 다음 ctrl 를 한다.

[[Attack Surface]]


[[실행옵션]]
[[단축키]]

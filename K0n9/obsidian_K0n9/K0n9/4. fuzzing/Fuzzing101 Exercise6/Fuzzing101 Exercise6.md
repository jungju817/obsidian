### CVE-2016-4994
Use-after-free vulnerability in the xcf_load_image function in app/xcf/xcf-load.c in GIMP allows remote attackers to cause a denial of service (program crash) or possibly execute arbitrary code via a crafted XCF file.

GIMP(v2.8.16)의 xcf-load.c 의 xcf_load_image 함수에서 발생하는 use-after-free 취약점으로 원격 공격자는 조작된 XCF파일을 통해 서비스 거부를 발생 시키거나 코드를 실행할 수 있다.


#### Persistent mode
AFL Persistent mode 는 단일 프로세스를 사용하는 fuzzer, 즉 타겟 프로세스에 코드를 주입하고 메모리의 입력 값을 변경하는 프로세스 fuzzer를 기반으로 한다.

afl-fuzz는 단일 프로세스 fuzzing의 이점과 더 전통적인 다중 프로세스 도구인 Persistent Mode의 견고성을 결합한 작업모드를 지원한다.

Persistent mode에서 AFL++은  각 fuzzer 실행에 대해 새로운 프로세스를 fork하는 대신 단일 fork 프로세스에서 대상을 여러 번 fuzzing 한다. 이 모드는 fuzzing 속도를 최대 20배 향상시킬 수 있다.

대상의 기본 구조는 다음과 같다.

```c
  //Program initialization

  while (__AFL_LOOP(10000)) {

    /* Read input data. */
    /* Call library code to be fuzzed. */
    /* Reset state. */
  }
  
  //End of fuzzing
  
```

[여기](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md)에서 더 많은 AFL++ persistent mode 정보에 대해서 찾을 수 있다.[[여기요약]]

[[xcf]]
[[configure 오류]]


#### target build

종속 패키지를 설치한다.
```
sudo apt-get install build-essential libatk1.0-dev libfontconfig1-dev libcairo2-dev libgudev-1.0-0 libdbus-1-dev libdbus-glib-1-dev libexif-dev libxfixes-dev libgtk2.0-dev python2.7-dev libpango1.0-dev libglib2.0-dev zlib1g-dev intltool libbabl-dev
```
[[build-essential]]


GEGL 0.2(Generic Graphics Library)가 필요한데 우분투에는 이게 없다. 그래서 따로 build해야 한다.

```
wget https://download.gimp.org/pub/gegl/0.2/gegl-0.2.0.tar.bz2
tar xvf gegl-0.2.0.tar.bz2

wget https://mirror.klaus-uwe.me/gimp/pub/gimp/v2.8/gimp-2.8.16.tar.bz2
tar xvf gimp-2.8.16.tar.bz2
```
code를 살짝 수정한다.
```
cd gegl-0.2.0
sed -i 's/CODEC_CAP_TRUNCATED/AV_CODEC_CAP_TRUNCATED/g' ./operations/external/ff-load.c
sed -i 's/CODEC_FLAG_TRUNCATED/AV_CODEC_FLAG_TRUNCATED/g' ./operations/external/ff-load.c
```
sed는 스트림 치환명령이다.
s : 대체를 나타내는 명령이다
g : 전역을 나타내는 명령으로 한줄의 모든 스트림을 치환한다. (이걸 안하면 한줄에 처음 발견된 것만 치환됨)
-i : 원본 파일을 수정한다. 이를 안하면 치환되었을때를 출력하고 원본파일은 수정되지 않는다.

build
```
./configure --enable-debug --disable-glibtest  --without-vala --without-cairo --without-pango --without-pangocairo --without-gdk-pixbuf --without-lensfun --without-libjpeg --without-libpng --without-librsvg --without-openexr --without-sdl --without-libopenraw --without-jasper --without-graphviz --without-lua --without-libavformat --without-libv4l --without-libspiro --without-exiv2 --without-umfpack --prefix="$HOME/fuzzing_GIMP/gimp-2.8.16/install"

make -j$(nproc)
sudo make install
```
얘는 prefix 없어도 됌
##### configure 옵션
- `--enable-debug`: 이 옵션은 컴파일된 코드에 디버깅 정보를 포함시킵니다. 디버깅을 위한 추가 정보를 제공하며, 예를 들어 줄 번호와 변수 이름이 포함됩니다. 그러나 실행 파일 크기가 더 커지고 실행이 느려질 수 있습니다.
    
- `--disable-glibtest`: 이 옵션은 GLib에 대한 테스트를 비활성화합니다. GLib는 GNOME 데스크톱 환경에서 사용되는 유틸리티 라이브러리입니다.
    
- `--without-vala`: Vala는 GNOME 개발에서 자주 사용되는 프로그래밍 언어입니다. 이 옵션은 Vala 지원을 비활성화합니다.
    
- `--without-cairo`: Cairo는 2D 그래픽 라이브러리로, 이 옵션은 Cairo 지원을 비활성화합니다.
    
- `--without-pango`: Pango는 여러 언어를 지원하는 텍스트 레이아웃 및 렌더링을 위한 라이브러리입니다. 이 옵션은 Pango 지원을 비활성화합니다.
    
- `--without-pangocairo`: Pango와 Cairo를 함께 사용하는 경우에 해당하는 옵션입니다. Pango는 텍스트 렌더링에 사용되고, Cairo는 2D 그래픽 렌더링에 사용됩니다. 이 옵션은 Pango와 Cairo를 함께 사용하는 기능을 비활성화합니다.
    
- `--without-gdk-pixbuf`: GDK Pixbuf는 이미지 데이터를 다루는 라이브러리입니다. 이 옵션은 GDK Pixbuf 라이브러리 지원을 비활성화합니다.
    
- `--without-lensfun`: Lensfun은 사진 렌즈 왜곡을 보정하는 데 사용되는 라이브러리입니다. 이 옵션은 Lensfun 라이브러리를 비활성화합니다.
    
- `--without-libjpeg`, `--without-libpng`, `--without-librsvg`, 등: 각각 JPEG, PNG, RSVG 등의 이미지 형식에 대한 라이브러리 지원을 비활성화합니다. 예를 들어, `--without-libjpeg`는 JPEG 이미지 지원을 비활성화하는 옵션입니다.
    
- `--without-openexr`, `--without-sdl`, `--without-libopenraw`, 등: 각각 OpenEXR, SDL, LibOpenRAW 등의 라이브러리 지원을 비활성화합니다.
    
- `--without-jasper`: Jasper는 JPEG-2000 이미지 형식을 다루는 데 사용되는 라이브러리입니다. 이 옵션은 Jasper 라이브러리 지원을 비활성화합니다.
    
- `--without-graphviz`: Graphviz는 그래프 시각화 도구입니다. 이 옵션은 Graphviz 지원을 비활성화합니다.
    
- `--without-lua`: Lua는 스크립팅 언어로, 이 옵션은 Lua 지원을 비활성화합니다.
    
- `--without-libavformat`: libavformat은 오디오 및 비디오 형식 처리를 위한 라이브러리입니다. 이 옵션은 libavformat 지원을 비활성화합니다.
    
- `--without-libv4l`: libv4l은 비디오 관련 작업을 수행하는 데 사용되는 라이브러리입니다. 이 옵션은 libv4l 지원을 비활성화합니다.
    
- `--without-libspiro`: Libspiro는 글꼴 디자인 및 경로 조작을 위한 라이브러리입니다. 이 옵션은 Libspiro 지원을 비활성화합니다.
    
- `--without-exiv2`: Exiv2는 이미지 메타데이터를 다루는 데 사용되는 라이브러리입니다. 이 옵션은 Exiv2 지원을 비활성화합니다.
    
- `--without-umfpack`: UMFPACK은 희소 행렬을 다루는 라이브러리 중 하나입니다. 이 옵션은 UMFPACK 지원을 비활성화합니다.
###

```
cd gimp-2.8.16
CC=afl-clang-lto CXX=afl-clang-lto++ PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$HOME/fuzzing_GIMP/gegl-0.2.0/ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --disable-gtktest --disable-glibtest --disable-alsatest --disable-nls --without-libtiff --without-libjpeg --without-bzip2 --without-gs --without-libpng --without-libmng --without-libexif --without-aa --without-libxpm --without-webkit --without-librsvg --without-print --without-poppler --without-cairo-pdf --without-gvfs --without-libcurl --without-wmf --without-libjasper --without-alsa --without-gudev --disable-python --enable-gimp-console --without-mac-twain --without-script-fu --without-gudev --without-dbus --disable-mp --without-linux-input --without-xvfb-run --with-gif-compression=none --without-xmc --with-shm=none --enable-debug  --prefix="$HOME/fuzzing_GIMP/gimp-2.8.16/install"

make -j$(nproc)
make install
```
여기서 다른 패키지는 그냥 설치했지만 gegl은 따로 빌드해줬다. 그래서 
```
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$HOME/fuzzing_GIMP/gegl-0.2.0/
```
와 같이 환경변수를 추가하는 방법으로 라이브러리를 찾는 경로를 추가했다.

$(nproc)은 현재 시스템에서 사용 가능한 프로세서(코어)의 수를 반환하는 명령

##### configure 옵션
- `--disable-gtktest`, `--disable-glibtest`, `--disable-alsatest`: 각각 GTK, GLib, ALSA에 대한 테스트를 비활성화하는 옵션입니다. 테스트를 수행하지 않고 빌드할 때 사용됩니다.
    
- `--disable-nls`: Native Language Support(NLS)를 비활성화하는 옵션으로, 다국어 지원을 사용하지 않도록 설정합니다.
    
- `--without-libtiff`, `--without-libjpeg`, `--without-bzip2`, `--without-gs`, `--without-libpng`, `--without-libmng`, `--without-libexif`, `--without-aa`, `--without-libxpm`, `--without-webkit`, `--without-librsvg`, `--without-print`, `--without-poppler`, `--without-cairo-pdf`, `--without-gvfs`, `--without-libcurl`, `--without-wmf`, `--without-libjasper`, `--without-alsa`, `--without-gudev`: 각각의 라이브러리나 기능을 지원하지 않도록 설정하는 옵션입니다. 해당하는 기능을 사용하지 않을 때 사용됩니다.
    
- `--disable-python`: Python 지원을 비활성화하는 옵션으로, 프로그램이 Python 스크립트를 사용하지 않도록 설정합니다.
    
- `--enable-gimp-console`: GIMP의 콘솔(터미널) 모드를 활성화하는 옵션으로, 그래픽 사용자 인터페이스가 아닌 터미널에서 GIMP을 사용할 수 있도록 설정합니다.
    
- `--without-mac-twain`: macOS에서 TWAIN(스캐너 및 카메라와의 상호 작용을 위한 API)을 사용하지 않도록 설정하는 옵션입니다.
    
- `--without-script-fu`: Script-Fu를 사용하지 않도록 설정하는 옵션으로, GIMP의 스크립트 기능을 비활성화합니다.
    
- `--without-dbus`: D-Bus를 사용하지 않도록 설정하는 옵션으로, D-Bus 통신을 사용하지 않도록 합니다.
    
- `--disable-mp`: MP(마커 플롯)를 비활성화하는 옵션으로, 마커 플롯 지원을 사용하지 않도록 설정합니다.
    
- `--without-linux-input`: Linux Input을 사용하지 않도록 설정하는 옵션으로, Linux 입력 관련 기능을 지원하지 않도록 합니다.
    
- `--without-xvfb-run`: xvfb-run을 사용하지 않도록 설정하는 옵션으로, X Virtual Framebuffer를 사용하지 않도록 합니다.
    
- `--with-gif-compression=none`: GIF 이미지의 압축을 비활성화하는 옵션입니다. GIF 이미지를 압축하지 않도록 설정합니다.
    
- `--without-xmc`: X Motion Compensation을 사용하지 않도록 설정하는 옵션으로, XMC 지원을 비활성화합니다.
    
- `--with-shm=none`: 공유 메모리를 사용하지 않도록 설정하는 옵션으로, 공유 메모리를 사용하지 않도록 합니다.
    
- `--enable-debug`: 디버깅 정보를 포함하여 디버깅을 용이하게 하는 옵션입니다. 이는 컴파일된 코드에 추가 정보를 포함시키는 것을 의미합니다.
###

#### Persistent mode

두 가지 방법이 있다.

- app.c 파일을 수정하는 방법
- AFL_LOOP 매크로를 for 반복 루프 내에 포함시킨다.


![[Pasted image 20240114023333 1.png]]
파일을 load하는 것을 내포한 함수를 loop로 감싸서 반복


![[Pasted image 20240114023308 1.png]]

![[Pasted image 20240114023422 1.png]]
파일을 가져오고 일련의 작업을 하는 부분을 loop로 감쌈


#### seed corpus
예제로 준 것을 사용하자


#### Fuzzing

해당 취약점은 GIMP core에 영향을 준다. 필요없는 플러그인을 삭제해서 시작 시간을 줄여주자.
```
rm ./install/lib/gimp/2.0/plug-ins/*
```

```
ASAN_OPTIONS=detect_leaks=0,abort_on_error=1,symbolize=0 afl-fuzz -i './afl_in' -o './afl_out' -D -t 1000+ -- ./install/bin/gimp-console-2.8 --verbose -d -f @@
```

![[Pasted image 20240116110600.png]]

- gimp-console-2.8 버전은 GIMP의 콘솔 전용 버전이다.
- deterministic으로 해라 -D
- 이 코드에는 무한 루프 버그가 있어서 -t 옵션을 넣어준다.

![[Pasted image 20240115112117.png]]
이렇게 last new find 에 none yet (odd, check syntax!) 라고 뜰 경우 input이 프로그램에 들어가지 않는 상황으로 대부분 실행파일의 파라미터 (-d, -t 이런거) 문제일 확률이 크다. (존재하지 않는 파라미터, 잘못된 옵션)

## CVE-2016-9297
The TIFFFetchNormalTag function in LibTiff 4.0.6 allows remote attackers to cause a denial of service (out-of-bounds read) via crafted TIFF_SETGET_C16ASCII or TIFF_SETGET_C32_ASCII tag values.

해석하자면 LibTiff 4.0.6의 TIFFFetchNormalTag 함수에서 발생하는 취약점으로 out-of-bound read 가 발생한다.


빌드를 해보자...
우선 LibTIFF 를 설치한다.
```
wget https://github.com/vadz/libtiff/archive/refs/tags/Release-v4-0-6.tar.gz
tar -zxvf Release-v4-0-6.tar.gz
```

그리고 빌드를 한다.
```
cd $HOME/f/fuzzing_LibTIFF/libtiff-Release-v4-0-6/

cd tiff-4.0.4/
./configure --prefix="$HOME/f/fuzzing_tiff/install/" --disable-shared
make
make install
```

잘 되는지 한번 해 본다.
```
./install/bin/tiffinfo -D -j -c -r -s -w ./tiff-4.0.4/test/images/palette-1c-1b.tiff
```
![[Pasted image 20231026225501.png]]

옵션:
-D : data를 읽는다.
-j : JPEG table을 본다.
-c : grey/color response 혹은 colormap에 대한 표시를 한다.
-r : 디코드된 data 대신 raw image data로 표시한다.
-s : strip offset과 byte count를 표시한다.
-w : raw data를 bytes가 아닌 word로 표시한다.

-j -c -r -s -w 와 같은 옵션을 쓰는 이유는 code coverage를 높이고 bug를 찾는 가능성을 높이기 위해서이다.

### code coverage
code coverage는 코드의 각 줄이 실행된 횟수를 보여주는 것이다. code coverage를 사용함으로써 우리는 코드의 어느 부분에 fuzzer가 도달했는지 알 수 있고 퍼징 과정을 시각화할 수 있다.

우선은 lcov를 설치해야한다.
```
sudo apt install lcov
```

이제는 --coverage 옵션과 함께 다시 rebuild를 한다.
```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
  
CFLAGS="--coverage" LDFLAGS="--coverage" ./configure --prefix="$HOME/f/fuzzing_LibTIFF/install/" --disable-shared
make
make install
```
[[CFLAGS]] [[LDFLAGS]]

그래서 우리는 다음과 같은 작업으로 code coverage data를 얻을 수 있다.
```
cd $HOME/fuzzing_tiff/tiff-4.0.4/
lcov --zerocounters --directory ./
lcov --capture --initial --directory ./ --output-file app.info

$HOME/f/fuzzing_LibTIFF/install/bin/tiffinfo -D -j -c -r -s -w $HOME/f/fuzzing_LibTIFF/tiff-4.0.4/test/images/palette-1c-1b.tiff

lcov --no-checksum --directory ./ --capture --output-file app2.info
```

lcov --zerocounters --directory ./ : 이전 counters를 reset 한다.

lcov --capture --initial --directory ./ --output-file app.info : 모든 instrumented line의 zero coverage가 기록된 초기 code coverage data file을 생성한다.

$HOME/f/fuzzing_LibTIFF/install/bin/tiffinfo -D -j -c -r -s -w $HOME/f/fuzzing_LibTIFF/tiff-4.0.4/test/images/palette-1c-1b.tiff : 분석하려는 application을 실행한다. 동시에 다양한 inputs을 실행시킬 수 있다.

lcov --no-checksum --directory ./ --capture --output-file app2.info : 최근 coverage를 app2.info file 에 저장한다.

마지막으로 우리는 HTML output을 만들어낸다.
```
genhtml --highlight --legend -output-directory ./html-coverage/ ./app2.info
```
code coverage report는 html-coverage folder에 만들어졌다. ./html-coverage/index.html 을 열고 아래와 같이 볼 수 있다. 

[[html 열기]]

![[Pasted image 20231027003212.png]]

### fuzzing
이제 libtiff 를 ASAN으로 compile 하자.

```
rm -rf ./install
cd ./tiff-4.0.4
make clean
export LLVM_CONFIG="llvm-config-15" ; CC=afl-clang-lto ./configure --prefix="$HOME/f/fuzzing_libtiff/install/" --disable-shared

AFL_USE_ASAN=1 make -j4
AFL_USE_ASAN=1 make install
```
여기서 make 명령어의 옵션으로 병렬 빌드를 수행하도록 한다. -j 뒤에 오는 숫자만큼 동시에 작업을 실행한다. 여기서는 동시에 4개의 작업을 수행한다.

fuzzing start
```
afl-fuzz -m none -i $HOME/f/fuzzing_LibTIFF/tiff-4.0.4/test/images/ -o $HOME/f/fuzzing_LibTIFF/out/ -s 123 -- $HOME/f/fuzzing_LibTIFF/install/bin/tiffinfo -D -j -c -r -s -w @@
```
![[Pasted image 20231105061049.png]]


![](https://i.imgur.com/2Jdl8uV.png)

0x6070000001b1 is located 0 bytes to the right of 65-byte region \[0x607000000170,0x6070000001b1)
이것을 보건데 
\[heap left redzone] + \[사용 가능 공간 65] + \[heap left redzone]
heap chunk 아마도 65크기의 chunk가 있을 것이다.

0x0c0e7fff802e - 0x7fff8000 = 0xc0e0000002e
0xc0e0000002e << 3 = 0x607000000170

[[TIFF란]]

[[K0n9/4. fuzzing/Fuzzing101 Exercise4/취약점 분석.canvas|취약점 분석]]

### 패치
4.0.7 버전을 설치해서 crash를 넣어보았다.
![[Pasted image 20231105060201.png]]
Tag에서 정상적인 길이로 출력된다.


![[Pasted image 20231105060117.png]]
![[Pasted image 20231105060254.png]]
TIFFFetchNormalTag 함수에서 추가적인 패치가 이루어졌다.
기능을 요약하자면 마지막 값이 null이 아니면 ASCII 값을 가지는 tag가 끝 byte가 null이 아니라는 경고문과 함께 마지막 byte를 null로 바꿔버린다.

요약을 하자면 
1. TIFFFetchNormalTag에서 꼼꼼한 길이 검사를 하지 않음
2. 그 상태로 TIFFPrintDirectory가 실행됨
3. 그리고 \_TIFFPrintField를 실행하고 oob read가 발생한다.

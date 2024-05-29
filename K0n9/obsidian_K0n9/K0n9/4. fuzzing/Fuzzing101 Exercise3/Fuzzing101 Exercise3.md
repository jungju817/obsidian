---
sticker: lucide//folder-closed
---
## CVE-2017-13028

The BOOTP parser in tcpdump before 4.9.2 has a buffer over-read in print-bootp.c:bootp_print().

4.9.2 버전 미만, 즉 4.9.1 까지 버전에서 발생한 취약점.



우선 target이 될 tcpdump를 설치한다.

```
wget https://github.com/the-tcpdump-group/tcpdump/archive/refs/tags/tcpdump-4.8.1.tar.gz

tar -zxvf tcpdump-4.8.1.tar.gz
```

그리고 libpcap도 설치한다. 이는 TCPdump에 빌요한 cross-platform library이다.
```
wget https://github.com/the-tcpdump-group/libpcap/archive/refs/tags/libpcap-1.8.0.tar.gz

tar -zxvf libpcap-1.8.0.tar.gz
```

그리고 libpcap-libpcap-1.8.0을 libpcap-1.8.0 으로 바꾼다. 그렇지 않으면 tcpdump가 libpcap을 찾지 못한다.
```
mv libpcap-libpcap-1.8.0 libpcap-1.8.0
```

그리고 build하고 install한다.
```
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
./configure --enable-shared=no
make
```
make install을 안하는 이유는 tcpdump를 install할때 같이 install한다.

그리고 tcpdump를 install한다.
```
cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.8.1/
./configure --prefix="$HOME/fuzzing_tcpdump/install/"
make
make install
```

한번 잘 되는지 시험해본다.
```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -h
```

![](https://i.imgur.com/sEztNV4.png)

help가 뜨면 잘 된거다.

```
./install/sbin/tcpdump -vvvvXX -ee -nn -r [.pcap file]
```
-vvvv : 패킷정보를 자세하게 출력한다. -v를 많이 사용하면 더 자세한 정보가 표시된다. 
-XX : 패킷 데이터를 16진수와 ASCII문자로 함께 출력한다.
-ee : 이더넷 헤더를 자세하게 출력한다. 이더넷 헤더는 네트워크 프레임의 기본적인 정보를 포함하고 있다.
-nn : 이옵션을 사용하면 호스트 및 포트 번호를 DNS이름으로 해석하지 않고 숫자 형태로 표시한다. 이것은 성능을 향상시키고 불필요한 DNS 조회를 피하는데 도움이 된다.
-r './crash1' : tcpdump가 읽어오는 파일을 crash1 으로 한다. 

위와 같이 tcpdump를 실행할 수 있다. .pcap file은 tcpdump-tcpdump-4.9.2/tests/ 에 많이 있다.
![](https://i.imgur.com/mxf5dwi.png)

### ASAN
AddressSanitizer이란 C, C++에서 빠른 메모리 에러 탐지기이다.
이는 컴파일러 계측모듈과 런 타임 라이브러리로 구성되어있다. 이 tool은 heap, stack, 전역 객체에서 out of bounds를 탐지하고, use after free, double free, memory leak bug등을 탐지할 수 있다.
AddressSanitizer는 오픈소스로 버전 3.1 부터 LLVM 컴파일러 툴체인과 통합되어 있으며, 원래 LLVM을 위한 프로젝트로 개발되었으나 GCC로 포팅되어 GCC >= 4.8에 포함되어있다.

### Build with ASan enabled
이제 tcpdump를 asan이 가능하도록 build를 한다.

일단 이전에 컴파일 한것들을 모두 지우자

```
rm -r $HOME/fuzzing_tcpdump/install
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
make clean

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.1/
make clean
```

이제 make 하기 전에 AFL_USE_ASAN=1 을 set한다.

```
cd $HOME/f/fuzzing_tcpdump/libpcap-1.8.0/
export LLVM_CONFIG="llvm-config-15"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/f/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make

cd $HOME/f/fuzAFL_USE_ASAN=1 makezing_tcpdump/tcpdump-tcpdump-4.8.1/
AFL_USE_ASAN=1 CC=afl-clang-lto ./configure --prefix="$HOME/f/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
```


```
afl-fuzz -m none -i $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.8.1/tests/ -o $HOME/fuzzing_tcpdump/out/ -D -s 123 -- $HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r @@
```
-m none : 자식 프로세스의 메모리 제한
ASAN은 많은 가상 메모리를 잡아먹기 때문에 정확한 크래시를 보기 위해선 제한을 비활성화 해주어야 한다.


![](https://i.imgur.com/uTeabv4.png)
fuzzing start!

crash를 넣고 돌려본다.

```
gdb --args ./install/sbin/tcpdump -vvvvXX -ee -nn -r './crash1'
```

![[Pasted image 20231227235650.png]]

## ASAN 분석
우선 ASAN은 위의 그림과 같은 로그를 보여준다.
여기서 우리가 알 수 있는것은 EXTRACT_16BITS에서 heap-buffer-overflow가 발생했다는 것이다. 여기서 우리는 shadow byte에 대해서 알아야 한다.

#### shadow byte란?
가상 주소 공간에서 8바이트 마다 하나의 섀도 바이트를 사용하여 설명할 수 있따.

하나의 섀도 바이트는 다음과 같이 현재 액세스 할 수 있는 바이트 수를 설명한다.
- 0은 8바이트 모두를 의미한다.
- 1-7은 1~7바이트를 의미한다.
- 음수는 진단 보고에 사용할 런타임의 컨텍스트를 인코딩합니다.

여기서 섀도 바이트가 매핑되는 주소는 아래와 같이 구한다.
```C
//x86
char shadow_byte_value = *((Your_Address >> 3) + 0x30000000)

//x64
char shadow_byte_value = *((Your_Address >> 3) + _asan_runtime_assigned_offset)
```
arm64 offset : 0x1000000000 

그럼 (shadow_byte_value - offset) << 3 을 하면 매핑된 주소를 알 수 있다.



[[K0n9/4. fuzzing/Fuzzing101 Exercise3/취약점 분석.canvas|취약점 분석]]

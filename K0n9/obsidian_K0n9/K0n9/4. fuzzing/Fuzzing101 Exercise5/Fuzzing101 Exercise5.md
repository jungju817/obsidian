## CVE-2017-9048 
libxml2 20904-GITv2.9.4-16-g0741801 is vulnerable to a stack-based buffer overflow. The function xmlSnprintfElementContent in valid.c is supposed to recursively dump the element content definition into a char buffer 'buf' of size 'size'. At the end of the routine, the function may strcat two more characters without checking whether the current strlen(buf) + 2 < size. This vulnerability causes programs that use libxml2, such as PHP, to crash.

해석하자면 libxml2 20904-GITv2.9.4-16-g07418101 버전에서 발견된 취약점으로 stack-based buffer overflow 가 발생했다. valid.c 의 xmlSnprintfElementContent함수는 요소의 정의를 재귀적으로 char 버퍼 buf에 size 크기만큼 덤프하는 역할을 한다.
이 루틴이 끝나고 함수는 현재의 strlen(buf) + 2 가 size 보다 작은지를 확인하지 않고 두개의 문자를 strcat으로 연겨한다. 이 취약점으로 인해 libxml2를 사용하는 프로그램들(예:PHP)이 크래시하는 문제가 발생할 수 있다.

### Dictionaries

복잡한 text 기반 파일을 fuzzing하려면 basic syntax token의 list를 dictionary에 포함하여 미리 제공하는 것이 fuzzer에 유용하다. 
AFL의 경우 dictionary는 단순히 AFL이 현재 인메모리 파일에 변경사항을 적용하기 위해 간단한 단어 혹은 값으로 이루어져 있다. 구체적으로 AFL은 사전에 제공된 값으로 다음과 같은 변경을 수행한다.

override : 특정 위치를 n바이트로 대체한다. 여기서 n은 사전 항목의 길이이다.
insert : 현재 파일 위치에 dictionary entry 삽입하여 모든 문자를 강제로 아래로 이동시키고 파일 크기를 늘린다.

아래 링크에서 좋은 예를 찾을 수 있다.
https://github.com/AFLplusplus/AFLplusplus/tree/stable/dictionaries

### Paralellization
멀티 코어 시스템이 있는 경우 fuzzing 작업을 병렬화하여 CPU 리소스를 최대한 활용하는것이 좋다.

#### Independent instances
이는 가장 간단한 병렬화 전략이다. 이 모드는 afl-fuzz의 인스턴스를 완전히 분리하여 실행한다.
AFL은 non-deterministic 테스트 알고리즘을 사용한다는 것을 기억해야 한다. 따라서 여러 AFL 인스턴스를 실행하면 성공 확률이 높아진다.

이를 위해서는 여러 개의 터미널 창에서 여러 개의 afl-fuzz 인스턴스를 실행하기만 하면 되며, 각 인스턴스에 대해 다른 출력 폴더를 설정한다. 간단하게 생각하면 시스템에 코어가 있는 만큼 많은 fuzzing 작업을 실행하는 것이다.

#### Shared instances
병렬 퍼징은 공유 인스턴스를 사용하는 것이 더 나은 접근방식이다. 이 경우 각 fuzzer 인스턴스는 다른 fuzzer에 의해 발견된 모든 test case를 수집한다.

일반적으로 한 번에 하나의 마스터 인스턴스만 있다.

```
./afl-fuzz -i afl_in -o afl_out -M Master -- ./program @@
```

그리고 N-1 개의 slaves instance 가 있다.

```
./afl-fuzz -i afl_in -o afl_out -S Slave1 -- ./program @@
./afl-fuzz -i afl_in -o afl_out -S Slave2 -- ./program @@
...
./afl-fuzz -i afl_in -o afl_out -S SlaveN -- ./program @@
```

[[xml]]

### build
```
wget http://xmlsoft.org/download/libxml2-2.9.4.tar.gz
tar -zxvf libxml2-2.9.4.tar.gz && cd libxml2-2.9.4/
```

```
sudo apt-get install python-dev
CC=afl-clang-lto CXX=afl-clang-lto++ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --prefix="$HOME/f/fuzzing_libxml2/libxml2-2.9.4/install" --disable-shared --without-debug --without-ftp --without-http --without-legacy --without-python LIBS='-ldl'
make -j$(nproc)
make install
```

#### 빌드옵션
1. `CC=afl-clang-lto`: C 언어 컴파일러를 `afl-clang-lto`로 설정합니다. 
2. `CXX=afl-clang-lto++`: C++ 언어 컴파일러를 `afl-clang-lto++`로 설정합니다.
3. `CFLAGS="-fsanitize=address"`: C 컴파일러 플래그를 설정합니다. 여기서는 AddressSanitizer를 사용하여 메모리 오류를 검출하도록 설정합니다.
4. `CXXFLAGS="-fsanitize=address"`: C++ 컴파일러 플래그를 설정합니다. 마찬가지로 AddressSanitizer를 사용하여 메모리 오류를 검출하도록 설정합니다.
5. `LDFLAGS="-fsanitize=address"`: 링커 플래그를 설정합니다. 여기서도 AddressSanitizer를 사용하여 메모리 오류를 검출하도록 설정합니다.
6. `./configure --prefix="$HOME/Fuzzing_libxml2/libxml2-2.9.4/install" --disable-shared --without-debug --without-ftp --without-http --without-legacy --without-python LIBS='-ldl'`: libxml2를 빌드하기 위한 `configure` 스크립트를 실행합니다. 여러 옵션들이 설정되어 있습니다.
    
    - `--prefix="$HOME/Fuzzing_libxml2/libxml2-2.9.4/install"`: 빌드된 파일들을 지정된 디렉토리에 설치할 경로를 지정합니다.
    - `--disable-shared`: 공유 라이브러리를 생성하지 않도록 설정합니다.
    - `--without-debug`: 디버깅 정보를 포함하지 않도록 설정합니다.
    - `--without-ftp`, `--without-http`, `--without-legacy`: 각각 FTP, HTTP, 레거시 관련 기능을 빌드에서 제외하도록 설정합니다.
    - `--without-python`: Python 바인딩을 빌드에서 제외하도록 설정합니다.
    - `LIBS='-ldl'`: `-ldl` 라이브러리를 링크합니다. 이것은 다이나믹 로딩을 지원하는 라이브러리로 보입니다.


#### 실행
잘 되는지 확인해보자.

```
./xmllint --memory ./test/wml.xml
```
![[Pasted image 20240104175122.png]]

#### Seed corpus
XML sample 파일이 필요하다

```
mkdir afl_in && cd afl_in
wget https://raw.githubusercontent.com/antonio-morales/Fuzzing101/main/Exercise%205/SampleInput.xml
cd ../
```

#### Custom dictionary
```
mkdir dictionaries && cd dictionaries
wget https://raw.githubusercontent.com/AFLplusplus/AFLplusplus/stable/dictionaries/xml.dict
cd ../
```


#### fuzzing start
해당 버그를 찾기 위해서는 반드시 --valid 매개변수를 활성화 해야 한다. 또한 dictionary 경로를 -x flag 로 설정하고 deterministic mutations을 -D flag 로 활성화한다.(master fuzzer를 위해)

```
afl-fuzz -m none -i ./afl_in -o afl_out -s 123 -x ./dictionaries/xml.dict -D -M master -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
```
또 다른 slave instance를 실행할 수 있다
```
afl-fuzz -m none -i ./afl_in -o afl_out -s 234 -S slave1 -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
```

###### 옵션
        --memory : parse from memory
        --noenc : ignore any encoding specified inside the document
        --nocdata : replace cdata section with text nodes
        --dtdattr : loaddtd + populate the tree with inherited attributes
        --loaddtd : fetch external DTD
        --valid : validate the document in addition to std well-formed check
        --xinclude : do XInclude processing





![[Pasted image 20240108121932.png]]

![[Pasted image 20240108121944.png]]

![[Pasted image 20240109120119.png]]

[[K0n9/4. fuzzing/Fuzzing101 Exercise5/취약점 분석.canvas|취약점 분석]]

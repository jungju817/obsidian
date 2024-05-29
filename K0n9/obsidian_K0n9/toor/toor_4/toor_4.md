상용 퍼저 분석 보고서 작성
필수 내용 : 분석 대상 퍼저, 퍼징 대상, 퍼징 방식, 사용 방법
AFL 퍼저 선택 금지

Radamsa는 fuzzer가 아니다!!
각 fuzzer마다 mutate하는 방법에 대해서 알아야한다.
다시 할것!

[[보고서]]

Radamsa

Radamsa는 Oulu University Secure Programming Group이 개발한 퍼즈 테스팅 도구이다. 기존에 출시되어있던 유사한 프로그램들은 설치 자체가 굉장히 복잡하고 개별 프로젝트에 적용하기 위해서 수많은 추가작업이 필요하다는 단점이 있었다. 이러한 문제점에 착안하여 새롭게 개발한 도구가 바로 Radamsa이다. 

Radamsa는 커맨드라인 기반으로 작동할 수 있으며, 샘플 파일을 주입하면 자동으로 mutated된 결과물을 생산해준다. 그러므로 이 도구는 mutation-based fuzzer라고 구분할 수 있겠다.

그리고 Radamsa는 타겟 프로그램의 구조를 모르는 Black-Box Fuzzer 이다.

또한 TCP Client 또는 서버로써 사용할 수도 있다. 즉 TCP 연결을 수립한 상태에서 서버 또는 클라이언트의 입장에서 mutation된 입력값을 송수신하게 만드는 것이다.

Radamsa는 하나의 패키지 안에 포함된 다수의 fuzzer를 통칭한다. 단순히 bit를 임의로 몇 번 바꾸는 것에 그치지 않고, 샘플 파일의 구조를 분석하여 보다 색다른 퍼징 기법에 적용할 방안을 찾는다.

Radamsa는 무료로 사용할 수 있으며 개조하여 사용하는 것 또한 허용된다.

---
##### README
Radamsa는 강건함 테스트를 위한 테스트 케이스 생성기 또는 퍼저로 알려진 도구이다. Radamsa는 유효한 데이터를 포함하는 샘플 파일을 읽고 그것들에 흥미로운 다양한 출력을 생성함으로써 작동한다. Radamsa의 주요 장점은 이미 중요한 프로그램에서 여러 버그를 찾아냈으며 스크립트 작성이 쉽고 빠르게 시작할 수 있다는 것이다.

해당 작업은 docker에서 하자

#### 사용법
Radamsa는 Windows 및 linux 용으로 prebuilt된 바이너리 형태가 제공되며, 자신의 환경에 알맞도록 직접 컴파일하여 사용할 수 있도록 소스코드 방식으로도 지원된다. 설치는 아래와 같다.

```
sudo apt-get install gcc make git wget
git clone https://gitlab.com/akihe/radamsa.git
cd radamsa/
make
sudo make install
```

[https://gitlab.com/akihe/radamsa/blob/master/README.md](https://gitlab.com/akihe/radamsa/blob/master/README.md)
대략적인 사용법이 적혀있는 help 문서가 있다.

![](https://i.imgur.com/0Ji3Grg.png)
위와 같이 어러번 재수행할 때마다 새로운 방식으로 mutate되는 것을 알 수 있다. 기본적으로 radamsa는 특정한 무작위 상태가 주어지지 않은 경우 /dev/urandom에서 무작위 시드를 가져온다.
사용할 무작위 상태는 -s 메개변수와 숫자가 뒤따른다. 동일한 무작위 상태를 사용하면 동일한 데이터가 생성된다.
![](https://i.imgur.com/67fFEd1.png)

아래의 예제는 radamsa가 텍스트 형태의 숫자를 mutate 하는 것이다. 
-n 매개변수를 사용하여 다음과 같이 여러 개의 출력을 생성할 수 있다.
![](https://i.imgur.com/73T4629.png)

우리는 이렇게 지금까지 얻은 결과물을 표준 입력에서 입력을 읽는 프로그램을 테스트하는 데 사용될 수 있다.
![](https://i.imgur.com/l5Viovh.png)
(bc가 없음)

또는 Radamsa를 컴파일하는데 사용된 컴파일러에 대해서도 사용할 수 있다.
```
echo '((lambda (x) (+ x 1)) #x124214214)' | radamsa -n 10000 | ./bin/ol
```
![](https://i.imgur.com/2FAf5qD.png)

또는 압축 해제를 하는데 사용할 수 있다.
```
gzip -c /bin/bash | radamsa -n 1000 | gzip -d > /dev/null
```
![](https://i.imgur.com/nq7JMFS.png)

-c : 압축
-d : 해제

각 출력에 대해 프로그램을 별도로 실행하길 원할 수 있다. 기본적인 셸 스크립트를 사용하면 이를 쉽게 할 수 있다. 보통은 테스트 스크립트가 계속해서 실행되도록 하고자 하니 여기서는 무한 루프를 사용한다.

```sh
gzip -c /bin/bash > sample.gz
while true; do radamsa sample.gz | gzip -d > /dev/null; done
```
![](https://i.imgur.com/lRmqW6D.png)

여기서는 Radamsa를 파이프로 실행하는 대신 샘플을 파일로 제공하고 있음에 유의하세요. Radamsa는 기본적으로 출력을 stdout에 작성하지만 cat과 달리 하나 이상의 파일이 주어지면 보통 하나 또는 몇 개의 파일만 사용하여 하나의 출력을 생성합니다. 이 테스트는 gzip에 대한 퍼징된 데이터를 전달하지만 그 후에 결과가 무엇이든 상관하지 않습니다. (간단한 단일 스레드) 프로그램에 뭔가 나쁜 일이 발생했는지 확인하는 간단한 방법은 종료 값이 127보다 큰지 확인하는 것입니다. 이는 다음과 같이 수행될 수 있습니다:
```sh
gzip -c /bin/bash > sample.gz
while true
do
  radamsa sample.gz > fuzzed.gz
  gzip -dc fuzzed.gz > /dev/null
  test $? -gt 127 && break
done
```
이는 gzip이 충돌할 때까지 계속 실행될 것이며, 아마 그럴일은 없을것이다.
스크립트가 중지되면 fuzzed.gz를 확인하는데 사용할 수 있다. 

주의할 점은 대부분의 출력이 주어진 샘플의 데이터를 기반으로 하기 때문에 좋은 샘플을 찾아내는게 중요하다.
더 현실적인 스크립트는 보통 radamsa가 수십 또는 수천 개의 샘플을 기반으로 한 번에 여러 출력을 생성하는데 사용되며, 출력의 결과는 주로 병렬로 테스트되며 각 출력을 명령 줄에서 대상 프로그램에 전달하여 확인한다.

아래는 명령줄에서 파일을 허용하는 bc를 위한 간단한 스크립트이다. -o 플래그를 사용하여 radamsa가 표준 출력 대신 출력을 작성할 파일 이름을 지정할 수 있다. 여러 출력이 생성되면 경로에는 %n이 있어야하며 이는 출력 번호로 확장된다.

```sh
echo "1 + 2" > sample-1
echo "(124 % 7) ^ 1*2" > sample-2
echo "sqrt((1 + length(10^4)) * 5)" > sample-3
bc sample-* < /dev/null
3
10
5
while true
do
  radamsa -o fuzz-%n -n 100 sample-*
  bc fuzz-* < /dev/null
  test $? -gt 127 && break
done
```
아니 근데 왜 bc 가 없냐...

#### Output Options
아래의 예제는 모두 표준 출력 또는 파일로 작성했다. 특별한 매개변수를 -o에 사용하여 radamsa에게 TCP클라이언트 또는 서버 역할을 수행하도록 할 수도 있다. output 패턴은 다음과 같다.

|-o argument|meaning|example|
|---|---|---|
|:port|act as a TCP server in given port|# radamsa -o :80 -n inf samples/*.http-resp|
|ip:port|connect as TCP client to port of ip|$ radamsa -o 127.0.0.1:80 -n inf samples/*.http-req|
|-|write to stdout|$ radamsa -o - samples/*.vt100|
|path|write to files, %n is testcase # and %s the first suffix|$ radamsa -o test-%n.%s -n 100 samples/*.foo|

기억해라, 예를 들면 tcpflow를 사용하여 TCP 트래픽을 파일로 기록할 수 있으며, 이 파일은 나중에 radamsa의 샘플로 사용될 수 있다.

#### Radamsa의 장점
셸 스크립트로 간단히 퍼징을 돌릴 수 있다는게 좋은 듯 하다.

|   |   |
|---|---|
|-o  |--output \<arg>, 출력 패턴|
|-n  |--count \<arg>, 생성할 출력 수|
|-s  |--seed \<arg>, random seed (number, default random)|
|-m  |--mutations \<arg>, 사용할 변이 \[ft=2,fo=2,fn,num=5,td,tr2,ts1,tr,ts2,ld,lds,lr2,li,ls,lp,lr,lis,lrs,sr,sd,bd,bf,bi,br,bp,bei,bed,ber,uw,ui=2,xp=9,ab]|
|-p  |--patterns \<arg>, 사용할 변이 패턴 \[od,nd=2,bu]|
|-g  |--generators \<arg>, 사용할 데이터 생성기 \[random,file=1000,jump=200,stdin=100000]|
|-M  |--meta \<arg>, 생성된 파일에 대한 메타 데이터를 이 파일에 저장|
|-r  |--recursive, 하위 디렉토리의 파일 포함|
|-S  |--seek \<arg>, 주어진 테스트 케이스에서 시작하라.|
|-d  |--delay \<arg>, 출력 간 n 밀리 초 동안 대기|
|-l  |--list, mutation, 패턴 및 생성기 목록|
|-C  |--checksums \<arg>, 고유성 필터의 최대 체크섬 수 (0은 사용 불가능) \[10000]|
|-v  |--verbose, 만드는 동안 진행상황 표시|


### 셸 스크립트
```
radamsa hello.pdf > crash.pdf
```
mutation 진행

```
  ./install/bin/pdftotext crash.pdf output > /dev/null
```
target program 실행 , /dev/null은 표준 출력을 버리는 특수 파일로 변환된 텍스트를 화면에 출력하지 않고 버린다는 의미이다.
```
  test $? -gt 127 && break
```
명령어의 종료 코드가 127보다 큰지를 확인한다.
만약 크다면 break

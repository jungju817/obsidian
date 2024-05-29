과제

1-day 분석 실습

1. CVE
2. CWE
3. 분석 환경 (타겟 버전, OS 버전, 빌드 옵션, 빌드 명령어)
4. 루트 커즈 파악
5. PoC 재현
6. 참고자료

### CVE
CVE-2022-48337

![[Pasted image 20231102190356.png]]
CVE-2022-48337 은 lib-src/etags.c에서 발생하는 취약점으로 소스 코드 파일의 이름으로 셸 메타문자를 통해 명령을 실행할 수 있는 취약점이다. 

### CWE
CWE-78

![[Pasted image 20231102190819.png]]
CWE-78 은 OS Command Injection

### 분석 환경

타겟 버전 : 28.1
![[Pasted image 20231102191115.png]]

OS 버전
Linux version 5.15.90.1
![[Pasted image 20231103020801.png]]

빌드 옵션, 명령어

우선 emacs 설치를 해야한다.
```
wget https://github.com/emacs-mirror/emacs/archive/refs/tags/emacs-28.1.tar.gz

tar -zxvf emacs-28.1.tar.gz

cd ./emacs-emacs-28.1

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
여기서 --without-x 는 gui를 지원하지 않도록 하는것이고 --with-gnutls=ifavialable은 GnuTLS라이브러리가 존재한다면 사용하겠다는것이다.

### 루트 커즈 파악
우선 emacs의 etags라는 실행파일에서 터지는 취약점이다.
[[etags]]

[[CVE-2022-48337 루트 커즈.canvas|CVE-2022-48337 루트 커즈]]
원래의 목적은 etags를 적용하기 위해 .zip파일을 해제하는 과정인 것 같다. 

### PoC 재현
![[Pasted image 20231102191846.png]]
기본 .c 파일이 있다

![[Pasted image 20231102191937.png]]
이를 "';sh;'test.z" 로 압축한다.

![[Pasted image 20231102192238.png]]
그러면 위와 같은 이름의 zip 파일이 생성된다.

![[Pasted image 20231102192315.png]]
이를 etags로 실행하면 셸이 실행된다.

.zip은 안되고 .z 로만 되더라

-------------
두 번째 방법으로는 tmp_name에 shell metacharater를 포함시키는것이다.
tmp_name 은 TMPDIR 환경변수이다. TMPDIR 임시 파일을 위한 디렉토리를 지정하는 환경변수로 프로그램이 종료되면 삭제된다. 해당 프로그램은 zip파일을 압축해제 한 후 그 파일을 tmpdir 에 저장한다.

결과적으로 system ("gzip -d -c 'filename' > tmpdir")  가 실행되니 tmpdir에 shell metacharacter가 포함되면 exploit이 가능하다.

![[Pasted image 20231103014923.png]]
기본 .c 파일이 있다

![[Pasted image 20231103015012.png]]
그냥 etags.z 라는 이름으로 압축한다.

![[Pasted image 20231103015210.png]]
tmpdir 변수에 "/tmp/;sh;/" 를 저장한다

![[Pasted image 20231103020215.png]]
그리고 실제로 /tmp에 ';sh;' 라는 디렉토리를 만든다.

![[Pasted image 20231103020350.png]]

결과적으로 보면 system("gzip -d -c 'etags.z' > /tmp/;sh;/etpCV8Va") 가 실행되어서 셸이 실행된다.

#### patch
29.1 버전으로 해보자.
![](https://i.imgur.com/Y2gJkMk.png)

안되는 것을 볼 수 있다.

![](https://i.imgur.com/tJxlhFk.png)
복잡한 과정이 있는데 잘 보면 escape_shell_arg_string으로 파일 이름들을 처리하는 부분이 있다. 이부분을 거치고 나면 파일들은 다음과 같이 바뀐다.
```C
      char *new_real_name = escape_shell_arg_string (real_name);
      char *new_tmp_name = escape_shell_arg_string (tmp_name);
```

![](https://i.imgur.com/h1aUJNK.png)

![](https://i.imgur.com/c3XdvBY.png)
"';sh;'test.z" 에서 "'''';sh;'''test.z'" 으로 바뀐 것을 볼 수 있다. 
여기서 escape_shell_arg_string 함수의 작동원리는 우선 전체 문자열을 '로 감싸고 '를 만나면 '를 '로 둘러싸서 '''로 만들어버린다.
### 참고자료
https://www.cvedetails.com/cve/CVE-2022-48337/
https://ubuntu.com/security/CVE-2022-48337
https://debbugs.gnu.org/cgi/bugreport.cgi?bug=59817

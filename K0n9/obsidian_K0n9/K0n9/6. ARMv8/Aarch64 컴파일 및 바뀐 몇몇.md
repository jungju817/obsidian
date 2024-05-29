```
sudo apt-get install qemu-user gcc-aarch64-linux-gnu
```

```c
#include <stdio.h>

int main()
{
        printf("Hello World\n");
        return 0;
}
```

```
aarch64-linux-gnu-gcc -static -o test test.c
```

![](https://i.imgur.com/EKLeXy0.png)

```
qemu-aarch64 ./test
```
![](https://i.imgur.com/b5IkfcB.png)

```
aarch64-linux-gnu-gcc -o test test.c
```
-static 을 빼면 다음과 같이 실행 한다
```
qemu-aarch64 -L /usr/aarch64-linux-gnu ./test2
```
![](https://i.imgur.com/LjII35C.png)

디버깅 하는 법
우선 gdb-multiarch 설치(여러 아키텍처 디버깅 가능)
```
apt-get install gdb-multiarch
```

포트는 아무렇게나
```
qemu-aarch64-static -L /usr/aarch64-linux-gnu/ -g 8888 ./test2
```
새 터미널 열고
```
gdb-multiarch
set arc aarch64
target remote localhost:8888
file ~/Desktop/[filename]         or  file ./[filename]     가끔 둘 중 하나 안됨
```

![](https://i.imgur.com/DTCIkYV.png)

![](https://i.imgur.com/gL1FW4I.png)

디버깅 중 입력을 받을때 입력은 바이너리를 실행한 창에서 입력해야한다. 디버깅 창 말고!!

gef로 하는법
gef는 좀 다름
```
gdb-multiarch -q -ex 'init-gef' -ex 'set architecture aarch64' -ex 'set solib-absolute-prefix /usr/arm-linux-gueabihf/'
gef-remote --qemu-user --qemu-binary test localhost 8888
file ./test
```


![](https://i.imgur.com/rmZcR12.png)
![](https://i.imgur.com/CA8ljmA.png)

---
### objdump

그냥 objdump 하면 안되고 `aarch64-linux-gnu-objdump` 로 해야 한다.
![](https://i.imgur.com/iNsRE1Z.png)

---
python으로 익스 짤 때 
`p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "./chain"])`
이렇게 하면 된다.
만약 pause와 attach가 필요하면
`p = process(["qemu-aarch64", "-L", "/usr/aarch64-linux-gnu/", "-g", "8888", "./chain"])`
이렇게 하면 된다. pause걸고 set arc aarch64 하고 targer remote localhost:8888 
하던것 처럼 하면 된다.

qemu 설치
```
sudo apt-get install qemu-user-static
```

```
sudo apt-get install gcc-multilib-mips-linux-gnu
```

컴파일
```
mips-linux-gnu-gcc -o test test.c
```
실행
```
qemu-mips -L /usr/mips-linux-gnu/ ./test
qemu-mips-static -L /usr/mips-linux-gnu/ ./test
```

리모트 디버깅
```
qemu-mips -g (포트) -L /usr/mips-linux-gnu ./실행파일
```


우선 mips32를 타겟으로 하였다.

### QEMU설치
```
apt-get install qemu
```
qemu를 설치하면 mips64도 같이 설치된다.

### QEMU squeeze/wheezy mips 이미지 다운로드

아래 사이트에서 다운받을 수 있다.
https://people.debian.org/~aurel32/qemu/
mips디렉토리로 들어가서 debian_~.qcow2 중 하나를 다운받고 vmlinux~malta 중 하나를 다운받아 총 2개의 파일을 다운받아야 한다. 제일 정확한 방법은 하단의 To use this image, you need to install QEMU 1.1.0(or later). Start QEMU 이 부분을 참고하는 것이다.

mip32 환경이니 
vmlinux-3.2.0-4-4kc-malta / debian_wheezy_mips_standard.qcow2
를 wget으로 다운받아 준다.


```
qemu-system-mips -M malta -kernel vmlinux-3.2.0-4-4kc-malta -hda debian_wheezy_mips_standard.qcow2 -append "root=/dev/sda1 console=tty0"
```
나올때는
`ctrl+alt+2` 이후 `quit` 치면 됨

근데 이제 ssh 접속 등 동적 분선을 편리하게 하기 위해 몇 가지 옵션을 추가하면 다음과 같이 된다.


```
qemu-system-mips \
-m 256 
-M malta -kernel vmlinux-3.2.0-4-4kc-malta \
-hda debian_wheezy_mips_standard.qcow2 \
-append "root=/dev/sda1 console=tty0" \
-net user,hostfwd=tcp::2222-:22,hostfwd=tcp::5555-:1234 \
-net nic,model=e1000
```

```
- m : 램 크기를 설정하는 부분이다. (32-bit MIPS에서는 기본 128m, 최대 256 인식)
- net : 포트 포워딩을 설정하는 부분이다. ip는 로컬 호스트로 설정하고 포트의 경우 2222 -> 22 (ssh)로, 5555 -> 1234(gdbserver)로 설정해준다. 
-redir 옵션은 mips32에서는 사용되지 않는다. 
```

아니 이게 이상한 방법이라니...


### /etc/inittab
이는 리눅스 시작 시 부팅 방법을 설정하는 파일이며 셸을 획득한 후 파일을 검토하여 실행되는 스크립트 또는 바이너리를 확인해야 한다.

```
::sysinit:/etc/init.d/rcS

::respawn:-/bin/sh

# Stuff to do when restarting the init process
::restart:/sbin/init

# Stuff to do before rebooting
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

```
:: 는 초기화 파일에서 사용되는 특별한 문구이다. 이는 inittab 파일에서 각 줄이 특정 
```

### 시작 스크립트 분석
#### init 스크립트
해당 스크립트를 분석하면 어떤 서비스가 활성화되고 임베디드 기기가 어떻게 실행되는지 파악할 수 있다. 대부분의 경우 /etc/inittab에 설명된 스크립트를 실행하여 다양한 기능을 활성화한다.

#### 활성화 서비스 분석
설정된 기기에서 사용되는 서비스에 대한 분석이 우선되어야 한다.
다양한 스크립트는 init 스크립트 실행을 통해 작동하며, 보통 실행되는 마지막 바이너리는 임베디드 기기의 서비스를 활성화하는 역할을 한다.

따라서 netstat, lsof, ps와 같은 명령어를 사용하여 바이너리 참조가 어떤 파일에서 실행되는지 분석한 후 실행 중인 프로세스를 결정해야 한다. 바이너리의 리버스는 활성화된 서비스를 분석하기 위해 필요에 따라 수행되어야 한다.

#### mount 된 장비 확인
서비스 활성화 분석과 더불어 임베디드 기기에서 장착된 기기를 확인하는 것이 중요하다. 다음과 같은 정보를 확인하며 읽기 전용으로 장착된 기기와 읽기 및 쓰기가 가능한 기기를 식별할 수 있다.

```
~ # mount
rootfs on / type rootfs (rw)
none on /dev type devtmpfs (rw,relatime,size=316408k,nr_inodes=79102,mode=755)
proc on /proc type proc (rw,relatime)
/dev/mtdblock3 on /mnt type squashfs (ro,relatime)
/dev/loop0 on / type squashfs (ro,relatime)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
tmpfs on /root type tmpfs (rw,relatime)
none on /dev type devtmpfs (rw,relatime,size=316408k,nr_inodes=79102,mode=755)
devpts on /dev/pts type devpts (rw,relatime,mode=600)
none on /sys/kernel/debug type debugfs (rw,relatime)
/dev/mtdblock8 on /mnt/ext_usr type squashfs (ro,relatime)
/dev/mtdblock5 on /mnt/web type squashfs (ro,relatime)
/dev/mtdblock4 on /mnt/custom type squashfs (ro,relatime)
/dev/mtdblock6 on /mnt/logo type cramfs (ro,relatime)
/dev/mem on /var type ramfs (rw,relatime)
/dev/ubi0_0 on /mnt/mtd type ubifs (rw,relatime)
```
`/dev/mtdblock` 장치를 mount함에 있어 플래시 메모리와 하드디스크 데이터를 획득하기 위해 설정과 관련된 파일을 자주 위치시킨다. 폴더에는 기본 설정, 암호키, 로그 등이 남아있을 가능성이 높다. 

따라서 스크립트 분석 시 마운트되는 장치 및 폴더 경로를 자세히 조사하면 임베디드 파일 시스템을 분석하는데 큰 도움이 될 것이다.

### watchdog 분석

워치독이란 임베디드 장치에서 특정 시간 간격으로 지정된 작업의 발생을 모니터링하는 기능이다.
일반적으로 워치독은 서비스를 수행하는 바이너리와 상호 작용하여 /dev/watchdog 라는 장치로 주기적으로 신호를 전송한다. 이러한 신호는 장치가 정상적으로 작동하고 있음을 나타낸다. 그러나 이 신호가 일정 시간 동안 수신되지 않으면 장치는 문제가 발생한 것으로 간주하고 다시 시작하도록 강제된다.

```
~~~~~~~~~~~~pid (492) will exit!
~~~~~~~~~~~~FN:[fn_master_status]
~~~~~~~~~~~~iRet [-88][Session Handle Err!!]
[165249.722559] [HKBSP][hik_wdt hik_wdt.1]hik-wdt:hikwdt_isr. I'm so Sorry (>_<)…
[165249.730063] [HKBSP][hik_wdt hik_wdt.1]hik-wdt:hikwdt_isr. last_feedwdt:4311458247(jiffies64:4311461694,timeout:25)
```
따라서 취약점 분석을 위해 바이너리를 디버그할 경우 감시 장치에 신호가 적시에 전송되지 않으면 일정 시간이 지나면 장치가 자동으로 재시작될 수 있다.

#### watchdog 분석
/dev/watchdog 장치는 워치독의 핵심이며, 활성화된 바이너리의 PID를 확인한 후 `/proc/<PID>/fd` 폴더에서 참조되는지 확인하여 워치독과 상호작용 여부를 확인할 수 있다.

```
[root@dvrdvs fd] # pwd
/proc/421/fd
[root@dvrdvs fd] # ls -l
lrwx------    1 root    root    64 Nov 19 14:54 36 -> socket:[3139]
lrwx------    1 root    root    64 Nov 19 14:54 37 -> socket:[3140]
lrwx------    1 root    root    64 Nov 19 14:54 38 -> socket:[3141]
lrwx------    1 root    root    64 Nov 19 14:54 39 -> /monitorMsg
l-wx------    1 root    root    64 Nov 19 14:54 4 -> /dev/watchdog
lr-x------    1 root    root    64 Nov 19 14:54 40 -> /home/app/exec/ptzCfg.bin
lrwx------    1 root    root    64 Nov 19 14:54 41 -> /ptz-mq
```
이 정보를 바탕으로 디버깅을 진행하면 워치독이 재부팅을 일으킬 수 있다. 따라서 스크립트에서 분석할 때 바이너리 패치나 워치독 feeding을 통해 워치독이 어떻게 실행되는지 확인하고 재부팅을 방지해야 한다.

일반적으로 워치독은 라이브러리나 커널 모듈에서 타임아웃을 설정하여 실행하고 ioctl 함수를 통해 설정한다. 따라서 타임아웃을 조정하거나 셸스크립트를 통해 워치독 feeding을 수행하여 재부팅을 방지하기 위해 라이브러리와 커널 모듈을 패치할 필요가 있다.

```
#!/bin/sh
while true; do
        echo -n "C" > /dev/watchdog
        sleep 1
done
```


## CVE-2019-14776
Description
A heap-based buffer over-read exists in DemuxInit() in demux/asf/asf.c in VideoLAN VLC media player 3.0.7.1 via a crafted .mkv file.

VideoLAN VLC media player 3.0.7.1의 demux/asf/asf.c 의 DemuxInit()에 .mkv 파일을 통해 heap-base buffer over-read 가 존재한다는 것이다. 


### Partial Instrumentation
진화적 커버리지 퍼저를 사용하는 것의 장점 중 하나는 새로운 실행경로를 스스로 찾을 수 있다는 것이다.하지만 이 또한 단점이 될 수 있다. 특히 VLC 미디어 플레이어의 경우와 같이 각 모듈이 특정 작업을 수행하는 고도로 모듈화된 아키텍처의 소프트웨어를 마주할 때 더욱 그렇다.

우리가 fuzzer에 유한 MKV 파일을 넣었다고 가정해보자. 하지만 입력 파일을 몇 번 mutate 한 후에 magic byte 파일이 바뀌었고, 이제 우리 프로그램은 입력 파일을 AVI 파일로 알아본다. 그리고 이 mutate된 MKV파일은 이제 AVI Demux에 의해 처리된다. 얼마 후에 매직 바이트 파일은 다시 한 번 바뀌고 이제 그 파일은 MPEG 파일로 보여진다. 두 경우 모두, 이 새로운 파일에는 유효한 구문 구조가 없기 때문에 새로 변형된 파일이 우리의 코드 coverage를 증가시킬 가능성은 매우 낮다.

즉, code coverage에 제약을 두지 않으면 fuzzer가 잘못된 경로를 쉽게 선택할 수 있으므로 fuzzing 프로세스의 효율성이 떨어진다.


이 문제를 해결하기 위해 AFL++에는 계측을 사용하거나 사용하지 않고 컴파일 해야 하는, 기능/파일을 지정할 수 있는 `부분 계측 기능`이 포함되어 있다. 이를 통해 fuzzer는 프로그램의 중요한 부분에 집중하고 흥미롭지 않은 코드 경로를 사용하여 원하지 않는 노이즈와 방해를 피할 수 있다.

이를 사용하기 위해서 컴파일 시 환경변수 AFL_LLVM_ALLOWLIST 를 설정한다. 이 환경 변수는 계측해야할 모든 함수/파일명이 포함된 파일을 가리켜야 한다.

AFL++ 부분 계측에 대한 자세한 내용은 [여기](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.instrument_list.md)에서 확인할 수 있다. [[여기정리]]

일단 target을 download한다.
```
wget https://get.videolan.org/vlc/3.0.7.1/vlc-3.0.7.1.tar.xz
tar xvf vlc-3.0.7.1.tar.xz
cd vlc-3.0.7.1
```

종속 패키지 설치
```
sudo apt install libxcb1 libxcb1-dev libxcb-keysyms1 libxcb-keysyms1-dev libxcb-image0 libxcb-image0-dev \
libxcb-shm0 libxcb-shm0-dev libxcb-icccm4 libxcb-icccm4-dev libxcb-sync1 libxcb-sync-dev libxcb-render-util0 \
libxcb-render-util0-dev libxcb-xfixes0-dev libxcb-shape0-dev libxcb-randr0-dev libxcb-glx0-dev libxcb-xinerama0-dev \
libxcb-xkb1 libxcb-xkb-dev libxkbcommon-x11-0 libxkbcommon-x11-dev

sudo apt install libxcb-composite0 libxcb-composite0-dev
export PKG_CONFIG_PATH=/usr/lib/pkgconfig

sudo apt install libxcb-xv0 libxcb-xv0-dev
```

빌드
```
CC=afl-clang-lto CXX=afl-clang-lto++ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --disable-lua --disable-avcodec --disable-swscale --disable-a52 --disable-alsa --enable-run-as-root --enable-debug --prefix="$HOME/vlc/vlc-3.0.7.1/install"
make -j$(nproc)
make install
```
---

solution
타겟 설치
```
wget https://download.videolan.org/pub/videolan/vlc/3.0.7.1/vlc-3.0.7.1.tar.xz
tar -xvf vlc-3.0.7.1.tar.xz && cd vlc-3.0.7.1/
```

```
 ./configure --prefix="$HOME/fuzzing_vlc/vlc-3.0.7.1/install" --disable-a52 --disable-lua --disable-qt --disable-avcodec --disable-swscale --disable-alsa --enable-run-as-root

make -j$(nproc)
```

```
./bin/vlc-static --help
```

![[Pasted image 20240126044126.png]]

잘된다.

#### fuzzing harness

harness란?
일부 프로그램은 프로그램에 입력을 받기 위한 구체적인 방법을 요구한다. fuzzer 가 모든 종류의 프로그램을 수용하기는 불가능하다. 그래서 fuzzer 가 대상 프로그램과 더 쉽게 상호작용 하기 위해서 harness가 필요하다. 이는 입력에 대한 작동을 결정할 수 있게 한다.
간단히 말하면 필요없는 부분을 없애고 fuzzing하려는 함수를 실행하도록 하는 프로그램이다.

vlc-static binary를 직접 퍼징하려고 하면 AFL이 초당 몇 번만 실행된다는 것을 알 수 있다. VLC를 시작하려면 시간이 많이 걸리기 때문입니다. 그래서 VLC를 퍼징하기 위한 맞춤 퍼징 harness를 만드는 것이 좋습니다.  
  
퍼징 하네스를 포함하도록 ./test/vlc-demux-run.c 파일을 수정한다. 이런 식으로 하네스를 컴파일하는 방법은 다음과 같습니다:  

```
cd test  
make vlc-demux-run -j$(nproc) LDFLAGS="-fsanitize=address"  
cd..
```
버그가 ASF demuxing에 존재하기 때문에 vlc_demux_process_memory 함수를 호출한다. 이 함수는 메모리에 이전에 저장되어 있던 데이터 버퍼를 demuxing하려고 한다. 코드의 변경사항은 다음과 같다.

```c
/**
 * @file vlc-demux-test.c
 */
/*****************************************************************************
 * Copyright © 2016 Rémi Denis-Courmont
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation; either version 2 of the License, or
 * (at your option) any later version.
 *
 * Rémi Denis-Courmont reserves the right to redistribute this file under
 * the terms of the GNU Lesser General Public License as published by the
 * the Free Software Foundation; either version 2.1 or the License, or
 * (at his option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, write to the Free Software Foundation,
 * Inc., 51 Franklin Street, Fifth Floor, Boston MA 02110-1301, USA.
 *****************************************************************************/
  
#ifdef HAVE_CONFIG_H
# include "config.h"
#endif
  
#include <stdio.h>
#include "src/input/demux-run.h"
  
int main2(int argc, char *argv[])
{
    const char *filename;
    struct vlc_run_args args;
    vlc_run_args_init(&args);
  
    switch (argc)
    {
        case 2:
            filename = argv[argc - 1];
            break;
        default:
            fprintf(stderr, "Usage: [VLC_TARGET=demux] %s <filename>\n", argv[0]);
            return 1;
    }
  
    return -vlc_demux_process_path(&args, filename);
}
  
//#include <fstream>
#include <errno.h>
#include <stdlib.h>
#include <fcntl.h>
  
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
  
#include <inttypes.h>
  
int main(int argc, char **argv){
  
    if (argc != 2) {    // input file 을 인자로 넣지 않았을 경우
        fprintf(stderr, "Usage %s <input file> \n", argv[0]);
        return -1;
    }
    struct vlc_run_args args;
    libvlc_instance_t *vlc;
  
    vlc_run_args_init(&args);  // initialize
    vlc = libvlc_create(&args);  // initialize
  
    int len;
    unsigned char *buf;
  
    //string filename(argv[1]);
  
#ifdef __AFL_COMPILER

 
    while (__AFL_LOOP(1000)) {  // persistent mode
  
#endif
  
        int fd = open(argv[1], O_RDONLY);  // file open
  
        if(fd < 0){  // file open error
            printf("Error opening file \n");
            printf("Errno: %i\n", errno);
            return -1;
        }
  
        struct stat st;
        stat(argv[1], &st);  // file의 속성을 가져와 buf에 저장
        len = st.st_size;
  
        buf = (unsigned char *)malloc(len);
  
        read(fd, buf, len);
  
//      libvlc_demux_process_memory(vlc, &args, buf, len);
  
        vlc_demux_process_memory(&args, buf, len);//취약점이 발생하는 (실행하길 원하는) 함수 호출
  
#ifdef __AFL_COMPILER
  
    }
  
#endif
  
}
```

#### Partial instrumentation
ASF demuxing과 관련된 파일 이름만 포함하는 방법은 안된다고 한다.  
  
파일 이름에 대한 일치가 항상 가능한 것은 아니여서 함수 일치와 파일 이름 일치를 포함한 혼합 접근 방식을 사용한다.

partial instrumentation file 은 다음과 같다.
```
demux/asf/asf.c
demux/asf/asfpacket.c
demux/asf/libasf.c
vlc.c

#fun: Demux
fun: WaitKeyframe
fun: SeekPercent
fun: SeekIndex
fun: SeekPrepare
#fun: Control
fun: Packet_SetAR
fun: Packet_SetSendTime
fun: Packet_UpdateTime
fun: Packet_GetTrackInfo
fun: Packet_DoSkip
fun: Packet_Enqueue
fun: Block_Dequeue
fun: ASF_fillup_es_priorities_ex
fun: ASF_fillup_es_bitrate_priorities_ex
fun: DemuxInit
fun: FlushQueue
fun: FlushQueues
fun: DemuxEnd


fun: AsfObjectHelperHave
fun: AsfObjectHelperSkip
fun: AsfObjectHelperReadString
fun: ASF_ReadObjectCommon
fun: ASF_NextObject
fun: ASF_FreeObject_Null
fun: ASF_ReadObject_Header
fun: ASF_ReadObject_Data
fun: ASF_ReadObject_Index
fun: ASF_FreeObject_Index
fun: ASF_ReadObject_file_properties
fun: ASF_FreeObject_metadata
fun: ASF_ReadObject_metadata
fun: ASF_ReadObject_header_extension
fun: ASF_FreeObject_header_extension
fun: ASF_ReadObject_stream_properties
fun: ASF_FreeObject_stream_properties
fun: ASF_FreeObject_codec_list
fun: ASF_ReadObject_codec_list
fun: ASF_ReadObject_content_description
fun: ASF_FreeObject_content_description
fun: ASF_ReadObject_language_list
fun: ASF_FreeObject_language_list
fun: ASF_ReadObject_stream_bitrate_properties
fun: ASF_FreeObject_stream_bitrate_properties
fun: ASF_FreeObject_extended_stream_properties
fun: ASF_ReadObject_extended_stream_properties
fun: ASF_ReadObject_advanced_mutual_exclusion
fun: ASF_FreeObject_advanced_mutual_exclusion
fun: ASF_ReadObject_stream_prioritization
fun: ASF_FreeObject_stream_prioritization
fun: ASF_ReadObject_bitrate_mutual_exclusion
fun: ASF_FreeObject_bitrate_mutual_exclusion
fun: ASF_FreeObject_extended_content_description
fun: ASF_ReadObject_marker
fun: ASF_FreeObject_marker
fun: ASF_ReadObject_Raw
fun: ASF_ParentObject
fun: ASF_GetObject_Function
fun: ASF_ReadObject
fun: ASF_FreeObject
fun: ASF_ObjectDumpDebug
fun: ASF_ReadObjectRoot
fun: ASF_FreeObjectRoot
fun: ASF_CountObject
fun: ASF_FindObject


fun: DemuxSubPayload
fun: ParsePayloadExtensions
fun: DemuxPayload
fun: DemuxASFPacket

```

#### fuzzing

build를 한다.
```
CC="afl-clang-fast" CXX="afl-clang-fast++" ./configure --prefix="$HOME/vlc/vlc-3.0.7.1/install"  --disable-a52 --disable-lua --disable-qt --disable-avcodec --disable-swscale --disable-alsa --enable-run-as-root --with-sanitizer=address
AFL_LLVM_ALLOWLIST=$HOME/vlc/vlc-3.0.7.1/Partial_instrumentation make -j$(nproc) LDFLAGS="-fsanitize=address"
```

```
cd test
make vlc-demux-run -j$(nproc) LDFLAGS="-fsanitize=address"
cd ..  
```

```
AFL_IGNORE_PROBLEMS=1 afl-fuzz -t 100 -m none -i './afl_in' -o './afl_out' -x asf_dictionary.dict -D -M master -- ./test/vlc-demux-run @@
```
![[Pasted image 20240126105737.png]]

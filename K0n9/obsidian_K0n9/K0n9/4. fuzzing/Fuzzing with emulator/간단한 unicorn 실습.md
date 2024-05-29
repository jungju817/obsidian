[참고자료](https://www.unicorn-engine.org/docs/tutorial.html)

```python
#!/usr/bin/python

# 유니콘 모듈을 가져온다. 이 샘플은 x86 레지스터 상수도 사용하므로 unicorn.x86_const도 필요하다.
from __future__ import print_function
from unicorn import *
from unicorn.x86_const import *
  
# code to be emulated, 에뮬레이트 될 코드
X86_CODE32 = b"\x41\x41\x41\x41\x41\x4a" # INC ecx; EC edx
  
# memory address where emulation starts, 에뮬레이션이 시작될 가상 메모리 주소 설정
ADDRESS = 0x1000000
  
print("Emulate i386 code")
try:
    # Initialize emulator in X86-32bit mode    
    # Unicorn을 클래스 Uc로 초기화 한다. 이 클래스는 하드웨어 아키텍처와 하드웨어 모드의 두가지 인자를 받는다. 이 샘플에서는 x86 아키텍처를 위한 32비트 코드를 에뮬레이션 한다. 변수 mu를 사용한다.
    mu = Uc(UC_ARCH_X86, UC_MODE_32)
  
    # map 2MB memory for this emulation
    # 에뮬레이션 2MB의 메모리를 매핑한다. 이 프로세스 동안의 모든 CPU 작업은 이 메모리에만 접근해야 한다. 이 메모리는 기본 권한인 READ, WRITE 및 EXECUTE가 매핑된다.
    mu.mem_map(ADDRESS, 2 * 1024 * 1024)
  
    # write machine code to be emulated to memor
    # 에뮬레이션 할 기계어를 매핑된 메모리에 입력한다. 인자로는 사용할 주소와 사용할 코드가 된다.
    mu.mem_write(ADDRESS, X86_CODE32)
  
    # initialize machine registers, 에뮬레이션 환경의 레지스터를 초기화 한다.
    mu.reg_write(UC_X86_REG_ECX, 0x1234)
    mu.reg_write(UC_X86_REG_EDX, 0x7890)
  
    # emulate code in infinite time & unlimited instructions
    # 기계어 코드를 끝까지 에뮬레이션을 시작한다. 해당 함수는 에뮬레이션된 코드의 주소, 에뮬레이션이 중지되는 주소, 에뮬레이션 할 시간, 에뮬레이션 할 명령어의 수 등 4가지 인자를 사용한다. 다음과 같이 두 인자를 무시하면 무한한 시간과 무제한의 명령어로 코드를 에뮬레이션 할 것이다.
    mu.emu_start(ADDRESS, ADDRESS + len(X86_CODE32))
  
    # now print out some registers
    print("Emulation done. Below is the CPU context")
  
    r_ecx = mu.reg_read(UC_X86_REG_ECX) # 에뮬레이션 이후 ECX, EDX 레지스터의 값들을 가져온다.
    r_edx = mu.reg_read(UC_X86_REG_EDX)
    print(">>> ECX = 0x%x" %r_ecx)
    print(">>> EDX = 0x%x" %r_edx)
  
except UcError as e:
    print("ERROR: %s" % e)
```

![](https://i.imgur.com/x1bglfS.png)

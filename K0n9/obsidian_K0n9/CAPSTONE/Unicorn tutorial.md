사이트 번역할 겸 정리

#### unicorn engine이란?
emulator 이다.

#### 어디에 쓰이나?
process에 피해 없이 malware에서 흥미로운 함수를 호출할 수 있다
CTF풀때
**Fuzzing**
gdb 플러그인으로 나올 것임
난독한 코드 에뮬레이팅

이 튜토리얼을 하려면 python의 unicorn engine, disassembler가 필요하다.

#### Task1
2017 hxp CTF를 풀어보자.
해당 binary를 실행시키면
![](https://i.imgur.com/qtDDpWn.png)

이렇게 flag가 출력이 되는데 점점 느려진다. 이는 프로그램을 최적화할 필요가 있음을 알려준다. 우선 IDA로 슈도 코드를 보자. 

```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  void *v3; // rbp@1
  int v4; // ebx@1
  signed __int64 v5; // r8@2
  char v6; // r9@3
  __int64 v7; // r8@3
  char v8; // cl@3
  __int64 v9; // r9@5
  int a2a; // [sp+Ch] [bp-1Ch]@3

  v3 = &encrypted_flag;
  v4 = 0;
  setbuf(stdout, 0LL);
  printf("The flag is: ", 0LL);
  while ( 1 )
  {
    LODWORD(v5) = 0;
    do
    {
      a2a = 0;
      fibonacci(v4 + v5, &a2a);
      v8 = v7;
      v5 = v7 + 1;
    }
    while ( v5 != 8 );
    v4 += 8;
    if ( (unsigned __int8)(a2a << v8) == v6 )
      break;
    v3 = (char *)v3 + 1;
    _IO_putc((char)(v6 ^ ((_BYTE)a2a << v8)), stdout);
    v9 = *((char *)v3 - 1);
  }
  _IO_putc(10, stdout);
  return 0LL;
}
```

```c
unsigned int __fastcall fibonacci(int i, _DWORD *a2)
{
  _DWORD *v2; // rbp@1
  unsigned int v3; // er12@3
  unsigned int result; // eax@3
  unsigned int v5; // edx@3
  unsigned int v6; // esi@3
  unsigned int v7; // edx@4

  v2 = a2;
  if ( i )
  {
    if ( i == 1 )
    {
      result = fibonacci(0, a2);
      v5 = result - ((result >> 1) & 0x55555555);
      v6 = ((result - ((result >> 1) & 0x55555555)) >> 2) & 0x33333333;
    }
    else
    {
      v3 = fibonacci(i - 2, a2);
      result = v3 + fibonacci(i - 1, a2);
      v5 = result - ((result >> 1) & 0x55555555);
      v6 = ((result - ((result >> 1) & 0x55555555)) >> 2) & 0x33333333;
    }
    v7 = v6 + (v5 & 0x33333333) + ((v6 + (v5 & 0x33333333)) >> 4);
    *v2 ^= ((BYTE1(v7) & 0xF) + (v7 & 0xF) + (unsigned __int8)((((v7 >> 8) & 0xF0F0F) + (v7 & 0xF0F0F0F)) >> 16)) & 1;
  }
  else
  {
    *a2 ^= 1u;
    result = 1;
  }
  return result;
}
```
이 문제를 해결하는 데에는 여러 가지 가능한 방법이 있습니다. 예를 들어, 프로그래밍 언어 중 하나로 코드를 재구성한 다음 최적화를 적용할 수 있습니다. 코드를 재구성하는 과정은 쉽지 않으며 당연히 버그와 오류를 도입할 수 있습니다. 실수를 발견하기 위해 코드를 응시하는 것은 전혀 재미있지 않습니다. 유니콘 엔진으로 이 문제를 해결하면 코드를 재구성하는 과정을 건너뛰고 위에서 언급한 것과 같은 문제를 피할 수 있습니다. 예를 들어, gdb 스크립트를 작성하거나 프리다를 사용하는 등 몇 가지 다른 방법으로 코드를 다시 작성하는 것을 건너뛸 수 있습니다.  
  
최적화를 적용하기 전에 먼저 유니콘 엔진에서 최적화 없이 일반 프로그램을 에뮬레이트할 것입니다. 성공 후에는 최적화할 것입니다.

#### Part 1 : 프로그램을 emulate 하자

```python
from unicorn import *            # 1
from unicorn.x86_const import *  # 2
```

첫 번째 줄은 메인 binary 및 기본 유니콘 상수를 로드합니다. 두 번째 줄은 x86 및 x86-64 아키텍처에 특정한 상수를 로드합니다.  
  
다음 행을 추가합니다:

```python
from pwn import *
 
def read(name):
    with open(name, "rb") as f:
        return f.read()
```

read는 전체 파일의 내용을 반환한다

아키텍처용 unicorn engine 클래스 x86-64 를 초기화 해 보자:

```python
mu = Uc (UC_ARCH_X86, UC_MODE_64)
```
우리는 다음과 같은 인수로 함수 Uc를 호출한다.
1. 주 아키텍처 분기, 상수는 UC_ARCH_ 로 시작한다.
2. 추가 아키텍처 사양, 상수는 UC_MODE_ 로 시작한다.
[[여기]]에서 아키텍처 상수의 전체 목록을 찾을 수 있다.

유니콘 엔진을 사용하려면 가상 메모리를 수동으로 초기화해야 한다. 
이 바이너리를 위해서는 어딘가에 코드를 작성하고 스택도 할당해야 한다.

binary의 base는 0x400000이다. 스택이 주소 0x0에서 시작하여 크기가 1024 * 1024 = 0x100000 라고 가정해보자. 이렇게 많이는 필요없지만 해가 될건 없다.

mem_map 메서드를 호출하여 메모리를 매핑할 수 있다.

```python
BASE = 0x400000
STACK_ADDR = 0x0
STACK_SIZE = 1024*1024

mu.mem_map(BASE, 1024*1024)
mu.mem_map(STACK_ADDR, STACK_SIZE)
```

이제 로더처럼 base 주소에 binary를 로드해야한다. 그런 다음 RSP를 스택의 끝을 가리키도록 설정해야한다.

```python
mu.mem_write(BASE, read("./fibonacci"))
mu.reg_write(UC_X86_REG_RSP, STACK_ADDR + STACK_SIZE - 1)
```

에뮬레이션을 시작하고 코드를 실행할 수 있지만 시작 주소가 무엇인지, 에뮬레이터가 어디에서 멈춰야 하는지 알아야 한다.

메인의 첫 번째 주소인 0x4004e0 에서 코드 에뮬레이트를 시작할 수 있다. 끝은 0x400575 가 될 수 있다. 이것은 우리의 전체 플래그가 출력된 후에 호출되는 `puts("\n")`이다. 
![](https://i.imgur.com/vegWhoZ.png)

![](https://i.imgur.com/FIApptu.png)

다음과 같이 시작할 수 있다.
```python
mu.emu_start(0x00000000004004E0, 0x0000000000400575)
```

![](https://i.imgur.com/OFgyFvJ.png)

뭔가 잘못됐다. emu_start에 다음을 추가해주자

```python
def hook_code(mu, address, size, user_data):  
    print('>>> Tracing instruction at 0x%x, instruction size = 0x%x' %(address, size)) 

mu.hook_add(UC_HOOK_CODE, hook_code)
```

이 코드는 후크(hook)를 추가합니다. 우리는 각 명령어를 에뮬레이션하기 전에 hook_code라 함수를 정의합니다. 이것은 다음과 같은 인수를 필요로 합니다:  
  
우리의 Uc 인스턴스  
명령어의 주소  
명령어의 크기  
사용자 데이터(hook_add ()의 선택적 인수로 이 값을 전달할 수 있습니다.)  

실행하면 다음을 확인할 수 있습니다:

![](https://i.imgur.com/Jzk7GiT.png)

이는 다음 명령어 실행하는 동안 스크립트가 실패한다는 것을 의미합니다:

![](https://i.imgur.com/wM0MZad.png)
이 명령어는 주소 0x601038에서 메모리를 읽는다. 
이것은 .bss 섹션이며 우리가 할당하지 않았다. 이 문제에 대한 해결책은 문제가 잇는 모든 명령어를 건너뛰는 것이다.

아래에는 다음과 같은 명령어가 있다.

![](https://i.imgur.com/bDXkBf2.png)

가상 메모리에 glibc가 로드되어 있지 않기 때문에 glibc 함수를 호출할 수 없다. 어차피 이 함수를 호출할 필요가 없기 때문에 생략할 수도 있다.

다음은 건너뛸 수 있는 전체 명령 목록이다.

![](https://i.imgur.com/ui10Dts.png)
![](https://i.imgur.com/ATQuMLn.png)
![](https://i.imgur.com/sjyD1O6.png)

아래와 같은 방법으로 다음 instruction의 레지스터 rip 주소에 기록하여 instruction을 건너뛸 수 있다.

```python
mu.reg_write(UC_X86_REG_RIP, address+size)
```

hook_code 는 다음과 같아 진다.
```python
instructions_skip_list = [0x00000000004004EF, 0x00000000004004F6, 0x0000000000400502, 0x000000000040054F]

def hook_code(mu, address, size, user_data):  
    print('>>> Tracing instruction at 0x%x, instruction size = 0x%x' %(address, size))
    
    if address in instructions_skip_list:
        mu.reg_write(UC_X86_REG_RIP, address+size)
```

우리는 또한 flag를 바이트 단위로 출력하는 명령어에 대해 무언가를 해야 한다.

![](https://i.imgur.com/NfsA63r.png)

레지스터 RDI에서 값을 읽고 출력하여 이 명령어를 건너뛸 수 있습니다. hook_code function은 다음과 같이 된다.

mu: 엔진 객체, 
address: 현재 실행중인 명령어 주소, 
size : 현재 실행중인 명령어 크기 -> 바이트 수, 
user_data : 사용자 정의 데이터 -> hook_add 함수를 호출할 때 제공된 사용자 정의 데이터이며 선택적으로 쓰인다.
hook_add에서 알잘딱으로 들어간다.

```python
instructions_skip_list = [0x00000000004004EF, 0x00000000004004F6, 0x0000000000400502, 0x000000000040054F]

def hook_code(mu, address, size, user_data):  
    #print('>>> Tracing instruction at 0x%x, instruction size = 0x%x' %(address, size))
    
    if address in instructions_skip_list:
        mu.reg_write(UC_X86_REG_RIP, address+size)
    
    elif address == 0x400560: #that instruction writes a byte of the flag
        c = mu.reg_read(UC_X86_REG_RDI)
        print(chr(c))
        mu.reg_write(UC_X86_REG_RIP, address+size)
```

이제 실행시키면 다음과 같이 된다.

![](https://i.imgur.com/Ym7vvtI.png)

#### Part 2 : Improve the speed!
속도에 대해 생각해보자. 이 프로그램은 왜 이렇게 느릴까

디컴파일된 코드를 보면 main에서 fibonacci()를 여러 번 호출하고 fibonacci()는 재귀 함수임을 알 수 있습니다.

해당 함수를 보면 우리는 2개의 인수가 필요하고 2개의 값을 반환한다는 것을 알 수 있습니다. 첫 번째 반환 값은 RAX 레지스터를 통해 전달되고, 두 번째 반환 값은 두 번째 인수를 통해 참조됩니다. main()과 fibonacci()를 자세히 살펴보면 두 번째 인수는 0 또는 1의 값만 취할 수 있다는 것을 알 수 있습니다. 

이 함수를 최적화하려면 동적 프로그래밍을 사용하여 주어진 인수에 대한 반환 값을 기억할 수 있습니다. 두 번째 인수는 2개의 값만 가지므로 `2*MAX_OF_FIRST_ARGMENT` 쌍만 기억하면 됩니다.  (첫 인수의 최대 값)
  
RIP가 피보나치 함수의 시작을 가리키면 함수 인수를 얻을 수 있습니다. 우리는 함수를 나갈 때 함수 반환 값을 알고 있습니다. 우리는 두 가지를 한 번에 모두 모르기 때문에 나갈 때 두 가지를 모두 얻는 데 도움이 되는 스택을 사용해야 합니다 - 피보나치 항목에서 우리는 인자를 stack에 push하고 마지막에 인자를 스택에서 pop해야 합니다. 쌍을 기억하기 위해 우리는 dictionary을 사용할 수 있습니다.  
  
쌍의 값을 유지하는 방법은 무엇입니까?  
  
함수를 시작할 때 반환 값이 이러한 인수에 대해 dictionary에 기억되어 있는지 확인할 수 있습니다  
만약 그것들이 있다면, 우리는 이 쌍을 반환할 수 있습니다. 우리는 단지 reference와 RAX에 반환 값을 적습니다. 우리는 또한 함수를 종료하기 위해 일부 RET 명령의 주소로 RIP를 설정했습니다. 우리는 이 명령이 hook에 있기 때문에 피보나치 함수의 RET으로 점프할 수 없습니다. 그것이 우리가 메인의 RET로 점프하는 이유입니다.  
사전에 없으면 인수를 스택에 추가합니다.  
함수를 나가는 동안 우리는 반환 값을 저장할 수 있습니다. 우리는 스택 구조에서 인수와 참조 포인터를 읽을 수 있기 때문에 인수와 참조 포인터를 알고 있습니다.  
코드는 아래에 나와 있습니다:
```python
FIBONACCI_ENTRY = 0x0000000000400670
FIBONACCI_END = [0x00000000004006F1, 0x0000000000400709]

stack = []  # Stack for storing the arguments
d = {}      # Dictionary that holds return values for given function arguments 

def hook_code(mu, address, size, user_data):  
    #print('>>> Tracing instruction at 0x%x, instruction size = 0x%x' %(address, size))
    
    if address in instructions_skip_list:
        mu.reg_write(UC_X86_REG_RIP, address+size)
    
    elif address == 0x400560:     # That instruction writes a byte of the flag
        c = mu.reg_read(UC_X86_REG_RDI)
        print(chr(c))
        mu.reg_write(UC_X86_REG_RIP, address+size)
    
    elif address == FIBONACCI_ENTRY:   # Are we at the beginning of fibonacci function?
        arg0 = mu.reg_read(UC_X86_REG_RDI)       # Read the first argument. Tt is passed via RDI
        r_rsi = mu.reg_read(UC_X86_REG_RSI)      # Read the second argument which is a reference
        arg1 = u32(mu.mem_read(r_rsi, 4))        # Read the second argument from reference
        
        if (arg0,arg1) in d:   # Check whether return values for this function are already saved.
            (ret_rax, ret_ref) = d[(arg0,arg1)]
            mu.reg_write(UC_X86_REG_RAX, ret_rax)   # Set return value in RAX register
            mu.mem_write(r_rsi, p32(ret_ref))       # Set retun value through reference
            mu.reg_write(UC_X86_REG_RIP, 0x400582)  # Set RIP to point at RET instruction. We want to return from fibonacci function
            
        else:
            stack.append((arg0,arg1,r_rsi))         # If return values are not saved for these arguments, add them to stack.
        
    elif address in FIBONACCI_END:
        (arg0, arg1, r_rsi) = stack.pop()           # We know arguments when exiting the function
        
        ret_rax = mu.reg_read(UC_X86_REG_RAX)       # Read the return value that is stored in RAX
        ret_ref = u32(mu.mem_read(r_rsi,4))         # Read the return value that is passed reference
        d[(arg0, arg1)]=(ret_rax, ret_ref)          # Remember the return values for this argument pair
```

그니까 정리하자면 일단 fibonacci실행하고 인자를 받아둠. 종료되면 인자에대한 결과값을 d에 쌍으로 저장.
만약 fibonacci를 실행했을때(재귀) 그 인자가 d에 있는 인자면 우린 이 결과를 d에 저장해놨으니 바로 리턴 값을 알 수 있음. 바로 리턴. 그리고 ret으로 점프


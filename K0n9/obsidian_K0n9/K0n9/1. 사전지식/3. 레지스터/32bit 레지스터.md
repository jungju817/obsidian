# 범용 레지스터(32bit x 8개)

이름 그대로 여러 목적으로 쓰이는 레지스터이다. 32비트의 이전 버전은 16비트 그 전은 8비트이다. 상위, 하위 8비트가 모여 16비트가 되고 16비트가 확장되어 32비트가 된다.

### EAX - Accumulator
- 더하기, 빼기 등 산술/논리 연산을 수행하며 함수의 return값이 저장된다. 
- 시스템콜 함수를 사용하려면 RAX에 함수의 syscall 번호를 넣어준다. 
- 함수의 syscall 번호를 가짐 (ex, exit() = 0x2000001)

### EBX - Base
- 메모리 주소를 저장하기 위한 용도로 사용

### ECX - Count
- 반복문에서 카운터로 사용되는 레지스터, 
- 고급언어 for문의 i와 같은 역할 
- ECX는 미리 반복 값을 정해두고 명령어를 사용할 때마다 값이 하나씩 줄어든다는 점이 다르다. 
- syscall을 호출했던 사용자 프로그램의 return 주소를 가진다. 
- return 주소를 rcx 값으로 채움

### EDX - data
- 다른 레지스터를 서포트하는 여분의 레지스터. 
- 큰 수의 곱셈이나 나눗셈 연산에서 EAX와 함께 사용된다.

### ESI - source index
- 데이터를 옮길 때 원본을 가르키는 포인터

### EDI - destination index 
데이터를 옮길 때 목적지를 가르키는 포인터

### ESP - Stack pointer
- 스택프레임에서 스택의 끝 지점 주소(현재 스택 주소)가 저장된다. 
- 데이터가 계속 쌓일 때 스택의 가장 높은 곳을 가리킨다. 
- push, pop 명령을 통해 RSP값이 위아래로 8바이트씩 이동하면서 스택프레임의 크기를 변경하게 된다.

### EBP - Base pointer
- 함수가 호출되면 스택프레임이 형성 되는데 이 스택프레임의 시작 지점 주소(스택 복귀 주소)가 저장된다.

![[Pasted image 20230918141043.png]]
이런 구조를 가진다.

# 명령어 포인터 레지스터
### EIP
- 다음 명령어가 저장된 메모리의 주소를 저장

# [[세그먼트]] 레지스터(16bit x 6개)
- 메모리의 세그먼트의 한 영역에 대한 주소를 지정한다.
- 코드 : CS 
- 데이터 : DS 
- 스택 : SS 
- extra : ES, FS, GS 추가 레지스터
이런 세그먼트 레지스터로 우리가 원하는 세그먼트 안의 특정 데이터, 명령어를 가져올 수 있다.

# 플래그 레지스터(32bit)
프로그램의 현재 상태나 조건 등을 검사하는데 사용되는 플래그들이 있는 레지스터이다. 시스템이 초기화되면 이 레지스터는 0x00000002의 값을 가지고 1,3,5,16,22~31번 비트는 예약되어 있어서 소프트웨어로 조작을 할 수 없다.

#### Status flags (상태 플래그) 
- CF(carry flag) : 연산 도중 carry(자리올림) 혹은 borrow(빌림수) 가 발생하면 1이 된다. 
- PF(Parity flag) : 연산 결과 최하위 바이트중 1비트가 짝수일 경우 1이된다. 
- AF(Adjust flag) : 연산 결과 carry 혹은 borrow가 3bit 이상 발생할 경우 1이된다. 
- ZF(Zero flag) : 결과가 0이면 1이된다. 여기서 0은 조건문이 만족일 때도 됨.(0은 참 1은 거짓) 
- SF(Sign flag) : 연산결과가 양수이면 0, 음수이면 1 
- OF(Overflow flag) : 정수형 결과값이 너무 크거나 너무 작은 음수여서 데이터 타입에 모두 들어가지 않으면 1이된다.

#### Control flag (컨트롤 플래그) 
- DF(Direction flag) : 1일 경우 문자열 처리 instruction이 감소, 0일 경우 증가 문자열을 이동하면 ESI, EDI가 증가하는데 DF가 0이면 증가하고 1이면 감소한다. 
#### System flags (시스템 플래그) 
- IF(Interrupt enable flag) : 프로세스로부터 인터럽트가 발생하면 인터럽트를 처리할 것인지 제어, 1일 경우 인터럽트 신호가 들어왔을 때 인터럽트를 처리한다. 
- TF(Trap flag) : 디버깅 할 때 Single step mode을 하려면 1 
- IOPL(I/O privilege level field) : 현재 수행중인 프로세스 혹은 task의 권한 레벨을 가르킴 
- NT(Nested task flag) : 인터럽트의 chain을 제어, 1이되면 이전 task와 현재 task가 연결되어 있음을 나타냄 
- RF(Resume flag) : 예외 디버그를 하기 위해 프로세서의 응답을 제어, 1이면 디버그 fault를 무시하고 다음 명령어 수행 
- VF(Virtual-8086 mode flag) : Virtual-8086 모드를 사용하려면 1을 준다 
- AC(Alignment check flag) : 메모리 참조시 정렬 기능을 활성화 
- VIF(Virtual interrupt flag) : IF flag의 가상 이미지이다. VIP flag와 결합시켜 사용한다. 
- VIP(Virtual interrpt pending flag) : 인터럽트가 경쟁 상태(인터럽트 2개이상 동시발생)가 되었음을 가리킨다. 1이면 발생 
- ID(Identification flag) : CPUID instruction을 지원하는 CPU인지를 나타낸다.

중요한 것만 보자, 너무 어렵게 쓰여있는거는 뺐다.
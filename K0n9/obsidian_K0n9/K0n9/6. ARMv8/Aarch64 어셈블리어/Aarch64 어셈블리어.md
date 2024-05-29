새로운 어셈

#### 조건 분기

B cond : 조건 분기
CBNZ : 비교 후 nonzero이면 분기
CBZ : 비교후 zero이면 분기

#### 무조건 분기

B : 무조건 분기
BL : link와 함께 분기
BL 명령은 반환을 위해(RET 참조) 순차적으로 다음 명령의 주소를 범용 레지스터인 X30에 기록한다. (서브루틴 혹은 함수를 호출할 때)

##### 무조건 분기 (register)

BLR : 레지스터에 저장된 주소로 분기하고 분기전에 다음 명령어의 주소를 X30에 저장한다
BR : 레지스터에 저장된 주소로 무조건 분기한다.
RET : 서브루틴 또는 함수 호출 이후에 호출한 곳으로 프로그램의 흐름을 반환한다. 보통 함수의 끝에서 사용되며 BL명령어로 호출된 함수 내에서 복귀할 때 사용된다.

#### Exception Generating

HMC (Hypervisor Management Call) : 익셉션 레벨 2 기능을 호출하기 위해 사용됨
SMC (Secure Monitor Call) : 익셉션 레벨 3 기능을 호출하기 위해 사용됨
SVC (Supervisor Call) : 익셉션 레벨 1 기능을 호출하기 위해 사용됨

#### 기타

WFE(Wait For Event) : 실행중인 프로세스 코어가 이벤트를 기다리는 동안 대기하도록 한다.
WFI(Wait For Interrupt) : 실행중인 프로세스 코어가 인터럽트를 기다리는 동안 대기하도록 한다.
SEV(Send Event) : 다른 프로세스 코어에게 이벤트를 보낸다. 이벤트를 받은 코어는 대기중인 WFE 명령어에 의해 깨어나게 된다.
SEVL (Send Event Local) : 로컬 코어에게 이벤트를 보낸다. 이벤트를 받은 코어는 대기중인 WFE 명령어에 의해 깨어나게 된다. 

#### Load/Store addressing modes

Base plus offst
주소 지정은 주소가 64비트 기본 레지스터에 값에 오프셋을 더한 값을 의미한다.
exam : `ldrsw x0, [x29, 76]`
X0의 워드를 X29 + 76 주소에 저장한다.

Pre-indexed
주소 지정은 주소가 64비트 기본 레지스터에 값과 오프셋을 더한 값이다. 주소와 오프셋 합이 base 레지스터에 다시 기록된다.
exam : `stp x29, x30, [sp, -80]!`
연속된 레지스터 쌍의 데이터를 메모리에 저장한다. x29, x30 순서

Post-indexed
주소 지정은 주소가 64비트 기본 레지스터에 값이며 주소와 오프셋의 합이 base 레지스터에 다시 기록된다. 주소와 오프셋 합이 base 레지스터에 다시 기록된다.
exam : `ldp x29, x30, [sp, 80]`
메모리의 값을 레지스터 쌍에 저장한다. x29, x30 순서

자주 쓰이는 레지스터 함 정리해야할듯

#### Single Register
메모리의 값을 레지스터에 저장한다.
exam : `ldr <Xt>, [add]`
a
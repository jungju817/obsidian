익셉션 레벨은 EL이라 하고 EL은 0~3까지 있다.
숫자가 커질 수록 privilege가 높으며 higher exception level이라고 한다. 반대로는 lower exception level이라고 한다.
#### EL0
User 모드이다.
유저 애플리케이션이 돌고 있는 모드이다.(카카오톡 등)

#### EL1
Supervisor 모드이며 커널 모드라고 부른다.
리눅스 커널 코드가 실행 중인 모드이다.

#### EL2
Hypervisor 모드이다.
Hypervisor 코드가 실행되는 모드로 EL2에서 구동 중인 Hypervisor는 Guest OS를 제어하는 역할을 수행한다.

Hypervisor란 운영체제에서 상이한 운영체제를 구동시키기 위한 소프트웨어 아키텍처라고 볼 수 있다.
예를 들어 윈도우에서 Vmware를 통해 리눅스를 사용한다고 하면 윈도우에서 vm을 구동하는 소프트웨어가 Hypervisor이고 리눅스를 Guest OS 라고 볼 수 있다.
그래서 Guest OS를 제어할 수 있는 모드를 Hypervisor 모드 라고 한다.

#### EL3
Secure monitor 모드이다.
TrustZone이 실행된다. 
이는 시스템 보안의 근간을 이루며 가장 신뢰할 수 있는 코드로 간주된다.


EL의 특징
```
1> EL0 -> EL1 -> EL2 -> EL3로 갈수록 execution privilege가 증가한다. 즉 볼 수 있는 코드나 파일에 대한 Permission이 더 있다는 거다. 

2> EL0는 유일한 unprivileged 특성을 가진다.

3> EL2는 Non-secure 모드에서 가상화를 구현하기 위해서 사용되곤 하는데 자주 쓰지는 않는다.

4> EL3는 secure 와 Non-secure 모드 전환을 위해서 사용된다.

5> ARMv8에서 EL0, EL1은 필수 구현 사항이며 나머지는 Option이다.

즉 ARMv8을 탑재한 SoC 업체나 벤더에서 특정 요구 사항에 따라 구현을 안할 수도 있다는 거다.

만약에 TrustZone을 안 쓰는 자동차 텔레메틱스 용 임베디드 장비를 개발할 때는 구지 EL3까지 구현할 필요는 없다.
```

EL의 이동

```
1> EL1: Supervisor Call(SVC)로 EL0 -> EL1로 이동하며 System Call 개념과 동일하다. ARM32 아키텍쳐의 Supervisor Mode와 비슷하다고 보면 되는데요, 보통 리눅스 커널이 구동될 때의 모드죠.   

2> EL2:  Hypervisor Call(HVC)로  EL1 -> EL2이 이동한다.

3> EL3: Secure Monitor Call(SMC)로 레벨 이동을 할 수 있다.  EL1 -> EL3, EL2 -> EL3 방향으로 이동 가능하다.
```

중요한 점은 낮은 EL에서 높은 EL로 이동하고 싶을 때는 반드시 Exception을 트리거 해줘야 한다.
그러면 Aarch64는 Exception을 발생시키면 스택을 푸쉬하는 등 백업을 위한 추가 동작을 해줘야 하고 Exception 발생 시 preempt_count를 보고 스케쥴을 할 수 있기 때문에 시스템 설계 시 고려해야 할 사항이 늘어난다.
여기서 preempt_count 란 preemption 관리에 대한 것으로 preemption이란 프로세스나 스레드가 실행중일 때 더 높은 우선순위를 가진 다른 테스크가 이를 중단시키고 CPU를 점유할 수 있게 하는 기능이다. preempt_count는 이 값이 0이면 preemption이 가능한 상태, 0보다 크면 preemption이 금지된 상태를 나타낸다.
preempt_count는 1보다 커질 수 있는데 예를 들어 커널이 하나의 크리티컬 섹션에 있을 때 preempt_count가 1 증가하고 또 다른 크리티컬 섹션에 진입하면 preempt_count가 다시 증가하여 중첩이 될 수 있다.

반대로 높은 EL은 execution privilege이 낮은 EL보다 높기 때문에 굳이 Exception을 트리거 할 필요는 없다.

각 Exception Level 사용 예를 들면 아래와 같다.
```
EL0 : Applications
EL1 : OS Kernel
EL2 : Hypervisor
EL3 : Secure monitor
```


## AFL란?

American Fuzzy Lop의 약자로, mutational 방식의 coverage guided fuzzer이다.

브르투포스로 입력을 받고 coverage를 넓혀가며 프로그램 제어 흐름에 대한 변경사항을 기록하고,이를 로깅하여 유니크한 crash를 발경해 낼 수 있다.

## AFL의 특징

coverage 기반 fuzzer이기 때문에 매우 효율적인 fuzzing이 가능하다.

code coverage 측을 위한 코드를 컴파일 타임에 삽입한다.

QEMU를 이용해서 컴파일 타임이 아닌, 런타임시에 코드삽입도 가능하다. 여기서 말하는 코드는 코드가 어디가 실행됬고, 어디가 실행 안됬는지를 측정해주는 코드를 뜻한다.

- Coverage Guided Feedback : AFL의 커버리지 피드백 기법은 하이브리드 기법을 사용한다. 우선 한번 실행될 때 대상 edge가 얼마나 많이 실행되는지 횟수를 측정한다. 이를 통해 path explosion을 방지한다. 이 횟수를 측정하기 위한 edge bucket은 일명 hitcounts라고 하는데, bitmap 방식으로 카운트되며 각각의 바이트들이 하나의 edge를 표현한다. 이 map 자료구조의 크기는 제한적이기 때문에 collision이 발생할 수밖에 없다. 이를 나름대로 최적화하기 위해서 AFL 은 weighted minimum set을 관리하기 위해 favored testcased 라는 이름으로 speed, size 및 weight 사이에서 효율을 도모한다. coverage feedback 을 할 때에 test case의 크기를 줄이고 실행 속도를 향상하기 위해 trimming 이라는 작업을 수행하기도 한다.
- Mutations : AFL은 deterministic 과 havoc(non-deterministic) 뮤테이션이 있고, 두개의 test case를 하나로 병합한 후 havoc을 적용하는 방식도 있다(splicing stage).
- Forkserver : execve() 시스템 콜을 호출할 때 발생하는 부하를 줄이기 위하여 forserver라는 개념을 사용하는데, IPC 방식으로 대상 프로그램을 관리하는 것이다. AFL 이 대상 프로그램에 test case를 주입하려할 때 입력 값을 지정한 후 target이 스스로 fork 하도록 한다. 그럼 자식 프로세스가 해당 tc를 실행하게 되고 부모 프로세스가 이를 기다리게 된다. 이렇게 되면 fuzzer는 매번 프로그램을 재수행할 때 초기화 과정을 반복할 필요가 없어지므로 시간을 절약할 수 있다.
- Persistent Mode : fork()를 지속해서 사용하는 것 역시 bottleneck이 될 수 있다. 이를 개선하기 위해 각각의 test case마다 fork를 수행하지 않고, 반복문을 사용해 한 회 수행시 그 내용을 패치하는 방식으로 대체할 수 있다.

## AFL 전체 로직

![](https://i.imgur.com/AhAhJ4a.png)

1. afl-fuzz에서 우리가 퍼징하려는 프로그램을 실행한다.
2. 해당 프로그램이 처음으로 실행되면서 afl-fuzz와 pipe로 통신을 하면서 fork server를 만들고, 새로운 타겟 인스턴스를 fork call()로 실행한다.
3. 표준 입력 or file로 들어온 입력이 fuzz 대상 프로그램으로 전달된다.
4. 실행된 결과를 공유 메모리에 기록하여 실행을 완료한다.
5. afl-fuzz는 공유 메모리에서 fuzz 대상이 남긴 기록을 읽고 이전 항목을 변경하여 새로운 입력을 만든다.
6. 새롭게 만든 입력은 다시 프로그램에 들어가서 실행된다

공유메모리

공유메모리를 통해 새로운 입력을 만든다고 했는데 이 부분이 바로 code coverage를 높이기 위한 로직이다.

공유메모리를 이전 실행과 비교하여 새로운 path 발견시 해당 input data를 저장하고 이를 다음 루틴의 입력으로 사용한다.

## AFL++

AFL의 원 제작자인 Michal Zalewski가 추가적인 관리를 하고 있지 않는 것으로 보이며, 2019년 7월 Google의 공식 오픈소스 도구가 되긴했지만 functional update는 크게 이루어지지 않고 있다.

그래서 기존에 제안된 많은 afl 추가 기법들을 구현한 확장형 AFL인 AFL++이 나왔다.
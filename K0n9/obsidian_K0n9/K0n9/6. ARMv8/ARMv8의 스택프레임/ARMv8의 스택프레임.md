스택 구현은 완전히 내림차순이다. (높은 곳에서 낮은곳으로)
푸시에서 스택 포인터가 감소합니다. 즉, 스택이 낮은 주소로 증가합니다.  
또 다른 특징은 스택이 쿼드워드(16byte)로 정렬되어야 한다는 것입니다. SP mod 16 = 0.  (aarch64에서 word는 4byte)
A64 명령어는 제한된 수의 경우에만 스택 포인터를 사용할 수 있다:  
	- Load/Store 명령은 현재 스택 포인터를 기본 주소로 사용한다. 시스템 소프트웨어에서 스택 정렬 검사를 활성화하고 기본 레지스터가 SP인 경우, 현재 스택 포인터는 처음에 쿼드워드 정렬, 즉 16바이트로 정렬해야 합니다. 정렬 오류는 Stack Alignment fault를 발생시킨다.  
	- 즉시 레지스터와 확장 레지스터 양식에서 데이터 처리 명령을 추가 및 차감하고 현재 스택 포인터를 소스 레지스터 또는 대상 레지스터 또는 둘 다 사용합니다.즉시 형식의 논리적 데이터 처리 명령은 현재 스택 포인터를 대상 레지스터로 사용합니다.

스택 프레임
Aarch64의 스택 프레임은 amd 64와는 반대이다.

함수가 시작될때 다음과 같은 명령어가 이루어진다.
![](https://i.imgur.com/2PnmgHr.png)

그러면 스택 구조는 다음과 같다.
(x29 == FP, x30 == LR) (stp후에는 SP 업데이트)

![](https://i.imgur.com/Uc9k56U.png)



만약 여기서 함수 func가 호출된다고 하자.
우선 함수를 call하면 함수 이후에 나올 명령어의 주소를 x30에 저장하고 해당 함수로 진입한다.(추측)

그리고 다시 아래와 같은 명령이 이루어진다.
![](https://i.imgur.com/loyXe2I.png)


![](https://i.imgur.com/UEy5ldD.png)

여기서 func의 함수가 종료될 때 다음과 같은 명령어가 실행된다.
![](https://i.imgur.com/Npgwyso.png)
`ldp x29, x30, [sp], #32` 이후에 스택은 다음과 같이 된다.

![](https://i.imgur.com/loMLZH5.png)
그러면 x29, x30, sp 가 다시 복원이 된다.(ldp 이후 sp 업데이트)

그리고 ret을 하면 x30의 값이 PC로 넘어가서 다음 명령어 주소가 실행된다.(추측)

ldp나 stp 를 해도 pop과 다르게 stack에 값은 그대로 남아있는다.(추측)

### canary
참고로 canary는 stack의 가장 아래(높은 주소)에 쌓인다.
```
saved FP(func)
saved LR(func)
stack (func)
canary (func)
saved FP(main)
saved LR(main)
stack (func)
```
main은 overflow가 발생해도 별 영향이 없어서 그런지 canary가 없다.
cloud, edge computing 이 해킹을 당한다면?

Trusted Execution Environments
TEEs
하드웨어적으로 안전한 공간을 만들고, 민감한 연산들을 전부 거기서 해버림

TrustZone
ARM으로 TEEs를 구현한거임,
normal world, secure world로 나눠서 민감한 연산을 전부 secure world로 던져서 처리해버림

만약 normal world를 해킹해서 셸을 탈취했다고 침, 그래서 OS까지 탈취를 함, 하지만 TEE 패러다임은 그래도 이를 방어할 수 있음. 왜냐먼 secure world에서 하는건 볼 수도 없고 걍 존나 제한적임, 하드웨어적으로 까는거는 ARM회사 밖에 못한다고 봐도 됨. 그래서 민감한 정보는 탈취당하지 않음

Trusted Application
TAs
TEE에서 뭔가 작업을하는 application
뭐 예를 들어 비트코인을 판다고 침, 그러면 client쪽에서 뭐 기타 등등을 다 적음. 그러면 그에 해당하는 서명을 TEEs에서 함. 그러면 엄청 안전함

Client Application
CAs는 TAs와 커뮤니케이션 하는 normal wolrd의 application임 
command ID가 필요하고 TA commands 로 design된 매개변수가 필요함 그러면 이제 결과를 리턴받음
근데 이제 만약에 TA에서 터졌다고 했을때 부검을 못하니까 좀 쉽지 않음

*그러면 TrustZone은 정말 안전한가?*

생각보다 취약점이 많음.
오히려 TA가 공격벡터가 되기까지 함
그래서 user가 TA를 이용해서 kernel에 쓰기를 하는 공격이 가능함

validations bugs가 가장 많았다고 함(if 잘못씀)
그리고 TAs에서가 가장 많음


*Fuzzing*

challenges in TA Fuzzing
trust zone은 뭐 싯팔 아무것도 알 수 없음, 걍 black box 임

이전의 작업
symbolic execution을 도입함, 
이전 버전의 취약점을 찾는 논문임.

그다음으로는 전제 자체가 trusted operation을 마음대로 할 수 있다는게 전제임. 
그 다음에는 걍 전체 다 hardware까지 에뮬레이션 함. 근데 해당 논문이 오픈소스가 아님
마지막으로 TEEzz 임, black box로 함, 

발표자의 접근 방법은 TA를 user world로 끄집어 와서 fuzzing 하는 거임
물론 그냥 되는거는 아니고 TA 개발자가 해당 기능을 코드에 넣어놔야 하는거임
그래서 CAs를 , 복사한 TAs로 통신을 시켜서 fuzzing을 함
그리고 만약 syscall이 들어올 경우 터짐. user world에 있으니까

syscall은 어케 해결할까
코드 패치는 불가능, 그대로 가져온거여서 바꾸면 다 깨짐
뭐 어케 해결했다는데 이해 못하겠음

그래서 아무튼 문제 잘 해결했고 퍼징 잘 돌아감


conclusion
최근 turstzone fuzzing 은 많은 연구가 필요함
해당 발표자는 위에 설명한 방법으로 잘함
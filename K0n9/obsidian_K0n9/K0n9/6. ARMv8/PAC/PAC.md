---
sticker: lucide//folder-closed
---
#### PAC 란?
Pointer Authentication Code 의 약자로 포인터들을 인증하는 미티게이션이다.
ARMv8.3-A 부터 지원한다.

#### PAC 동작원리
먼저 pac로 시작하는 명령어(ex, paciasp, pacia)를 통해 context와 key 그리고 해당 포인터를 이용해 sign을 한다. 이 때 key는 128비트 값이며 시스템 레지스터에 저장되어 있는데, 일단 유저 권한에서는 접근할 수 없다. context는 64비트 값을 사용한다. paciasp를 사용하여 PAC를 생성할 경우, 스택 포인터를 사용한다.

그리고 이 세 개의 값은 특정 알고리즘으로 계산되어 PAC를 생성할 수 있는데, 기본적으로 [QARMA](https://eprint.iacr.org/2016/444.pdf)를 사용하고 CPU 제조사마다 커스텀이 가능합니다. 이렇게 생성된 `PAC`는 sign하려고 한 포인터의 상위 주소에 저장됩니다.

Sign한 뒤, 해당 포인터를 사용할 때에는 먼저 aut로 시작하는 명령어를 사용하여 해당 포인터가 변조되었는지를 확인합니다. 똑같이 포인터 값과 context 그리고 key 값을 사용해 해당 포인터가 정상인지를 검증합니다. 따라서 userspace에서는 `PAC`생성 알고리즘에서 사용하는 key와 context뿐만 아니라 `PAC`를 생성하는 알고리즘을 알고 있어야만 알고리즘에 부합하는 값을 얻어낼 수 있습니다.

#### PAC Forgery
`PAC Forgery`는 key와 알고리즘 없이 이를 우회하는 기법입니다.


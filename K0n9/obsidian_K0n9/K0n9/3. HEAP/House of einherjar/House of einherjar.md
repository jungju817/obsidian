https://guyinatuxedo.github.io/42-house_of_einherjar/house_einherjar_exp/index.html
맛있는듯

`_int_free`가 chunk를 top chunk에 등록하는 과정을 악용하는 기법

**조건**
 메모리에 fake chunk를 작성할 수 있을 때
 in-use chunk의 헤더를 변경할 수 있을 때
 libc 2.31 에서 터짐 (ubuntu 20)


#### 과정
1. fake free chunk를 stack(fake chunk할 곳)에 작성하고, fast bin에 해당하지 않는 크기의 메모리를 2개 할당한다.
2. 마지막에 할당된 chunk의 헤더의 값들을 변경한다.
	1. size가 가지고 있는 값에서는 PREV_IUSE flag를 제거한다.
	2. 해당 chunk의 헤더 주소에서 fake chunk의 주소를 뺀 값을 prev_size에 저장한다. 
3. Fake chunk는 다음과 같은 값들을 가지고 있어야 한다.
	1. 마지막에 할당된 chunk와 같은 크기를 prev_size에 저장한다.
	2. 마지막에 할당된 chunk의 헤더 주소에서 fake chunk의 주소를 뺀 값을 size에 저장하고 fake chunk의 주소를 fd,bk에 저장한다.
4. 마지막에 할당된 chunk를 해제하면, fake chunk의 주소가 arena->top에 저장된다.
5. 메모리 할당을 요청하면 fake chunk의 영역의 포인터를 반환한다.

살짝 어떤 느낌이냐면 

chunk1
chunk2
top chunk

fake chunk

 이런 구조로 되어있는데 fake chunk를 chunk1과 chunk2 사이에 넣어져있는 구조로 만드는거임
 그러면 
 chunk1
 fake chunk
 chunk2
 top chunk
 이런식이겠지. (실제로는 아니고)
fake chunk는 free 되었다는 설정이고 여기서 chunk2를 해제할 경우 우선 chunk2와 fake chunk 병합을 함. 다음 chunk가 top chunk이니 다시 topchunk와 병합을 함. top chunk와 병합을 하는거는 해당 chunk의 size에 top chunk의 size를 더하고 해당 chunk를 arena의 top chunk로 업데이트 함.
그러면 결국 fake chunk가 top chunk가 되며 다시 할당을 요청하면 fake chunk가 할당된다.
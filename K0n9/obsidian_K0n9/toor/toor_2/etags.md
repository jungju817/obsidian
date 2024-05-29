여기서 etags란 emacs와 함께 제공되는 ctags utility이다. 이것의 기능은 소스파일에 있는 항목들을 쉽고 빠르게 찾을 수 있도록 tag 파일을 생성한다.
예를 들어서 아래와 같은 코드가 있다.
```C
#include <stdio.h>

int func1(){
        printf("this is func1");
}

int func2(){
        printf("this is func2");
}

int main(){
        int a;
        printf("inputs: ");
        scanf("%d", &a);
        printf("a : %d", a);
}
```
위 코드에 etags를 적용하면 TAGS란 파일이 생긴다.

![](https://i.imgur.com/6HMqD6N.png)

TAGS는 아래와 같은 내용이다.
![](https://i.imgur.com/cWW2YYZ.png)

---
tistoryBlogName: k0n9
tistoryTitle: UAF
tistoryVisibility: "3"
tistoryCategory: "1136867"
tistorySkipModal: true
tistoryPostId: "44"
tistoryPostUrl: https://k0n9.tistory.com/44
---
## UAF란?

Use After Free

해제된 메모리에 접근할 수 있을때 발생하는 취약점

## 공격조건

- 메모리 참조에 사용한 포인터를 메모리 해제 후에 적절히 초기화하지 않는경우
    
- 해제한 메모리를 초기화하지 않고 다음 청크에 재할당
    
- glibc 2.27 이하에서 가능
    

### Dagling Pointer란?

유효하지 않은 메모리 영역을 가리키는 포인터

## 취약한 부분

malloc함수는 할당한 메모리의 주소를 반환하여 선언된 포인터에 저장된다. 그리고 free를 하면 chunk를 반환하는데 이때 **chunk의 주소를 담고 있던 포인터를 초기화하지 않는다.** 따라서 free이후에 프로그래머가 포인터를 초기화하지 않으면 포인터는 해제된 chunk를 가리키는 Dangling Pointer가 된다.

따라서 공격자에게 공격 수단으로 활용될 수도 있다.

## 예1

```c
// Name: dangling_ptr.c
// Compile: gcc -o dangling_ptr dangling_ptr.c
#include <stdio.h>
#include <stdlib.h>
int main() {
	char *ptr = NULL;
	int idx;
	while (1) {
		printf("> ");
		scanf("%d", &idx);
		switch (idx) {
			case 1:
				if (ptr) {
					printf("Already allocated\\n");
					break;
				}
				ptr = malloc(256);
				break;
			case 2:
				if (!ptr) {
					printf("Empty\\n");
				}
				free(ptr);
				break;
			default:
				break;
		}
	}
}
```

![](https://i.imgur.com/PeqFho1.png)


free를 한 후에도 주소가 남아있어서 double free 에러가 뜬다.

---

새롭게 할당한 chunk를 프로그래머가 명시적으로 초기화하지 않으면 메모리에 남아있던 데이터가 유출되거나 사용될 수 있다.

## 예2

```c
// Name: uaf.c
// Compile: gcc -o uaf uaf.c -no-pie
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
struct NameTag {
	char team_name[16];
	char name[32];
	void (*func)();
};
struct Secret {
	char secret_name[16];
	char secret_info[32];
	long code;
};
int main() {
	int idx;
	struct NameTag *nametag;
	struct Secret *secret;
	secret = malloc(sizeof(struct Secret));
	strcpy(secret->secret_name, "ADMIN PASSWORD");
	strcpy(secret->secret_info, "P@ssw0rd!@#");
	secret->code = 0x1337;
	free(secret);
	secret = NULL;
	nametag = malloc(sizeof(struct NameTag));
	strcpy(nametag->team_name, "security team");
	memcpy(nametag->name, "S", 1);
	printf("Team Name: %s\\n", nametag->team_name);
	printf("Name: %s\\n", nametag->name);
	if (nametag->func) {
		printf("Nametag function: %p\\n", nametag->func);
		nametag->func();
	}
}
```

![](https://i.imgur.com/rYmoRV1.png)


입력한적이 없는 Name와 Nametag function이 출력된다.

![](https://i.imgur.com/Xwjsxng.png)

secret_name은 fd와 bk로 덮어씌여졌지만 secret_info와 code는 남아있는것을 확인할 수 있다.

이렇게 초기화되지 않은 메모리의 값을 읽어내거나, 새로운 객체가 악의적인 값을 사용하도록 유도하여 프로그램의 정상적인 실행을 방해할 수 있다.

### 예제_DreamHack_cpp_smart_pointer_1

checksec
![](https://i.imgur.com/4iYPHh8.png)


```C++
// g++ -o pwn-smart-poiner-1 pwn-smart-pointer-1.cpp -no-pie -std=c++11
  
#include <iostream>
#include <memory>
#include <csignal>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <cstdio>
#include <cstdlib>
  
char* guest_book = "guestbook\x00";
  
void alarm_handler(int trash)
{
    std::cout << "TIME OUT" << std::endl;
    exit(-1);
}
  
void initialize()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
  
    signal(SIGALRM, alarm_handler);
    alarm(30);
}
  
void print_menu(){
    std::cout << "smart pointer system!" << std::endl;
    std::cout << "1. change smart pointer" << std::endl;
    std::cout << "2. delete smart pointer" << std::endl;
    std::cout << "3. test smart pointer" << std::endl;
    std::cout << "4. write guest book" << std::endl;
    std::cout << "5. view guest book" << std::endl;
    std::cout << "6. exit system" << std::endl;
    std::cout << "[*] select : ";
}
  
void write_guestbook(){
    std::string data;
    std::cout << "write guestbook : ";
    std::cin >> data;
    guest_book = (char *)malloc(data.length() + 1);
    strcpy(guest_book, data.c_str());
}
  
void view_guestbook(){
    std::cout << "guestbook data: ";
    std::cout << guest_book << std::endl;
}
  
void apple(){
    std::cout << "Hi im apple!" << std::endl;
}
  
void banana(){
    std::cout << "Hi im banana!" << std::endl;
}
  
void mango(){
    std::cout << "Hi im mango!" << std::endl;
}
  
void getshell(){
    std::cout << "Hi im shell!" << std::endl;
    std::cout << "what? shell?" << std::endl;
    system("/bin/sh");
}
  
class Smart{
public:
    Smart(){
        fp = apple;
    }
    Smart(const Smart&){
    }
  
    void change_function(int select){
        if(select == 1){
            fp = apple;
        } else if(select == 2){
            fp = banana;
        } else if(select == 3){
            fp = mango;
        } else {
            fp = apple;
        }
    }
    void (*fp)(void);
};
  
void change_pointer(std::shared_ptr<Smart> first){
    int selector = 0;
    std::cout << "1. apple\n2. banana\n3. mango" << std::endl;
    std::cout << "select function for smart pointer: ";
    std::cin >> selector;
    (*first).change_function(selector);
    std::cout << std::endl;
}
  
int main(){
    initialize();
    int selector = 0;
    Smart *smart = new Smart();
    std::shared_ptr<Smart> src_ptr(smart);
    std::shared_ptr<Smart> new_ptr(smart);
    while(1){
        print_menu();
        std::cin >> selector;
        switch(selector){
            case 1:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    change_pointer(src_ptr);
               } else if(selector == 2){

                   change_pointer(new_ptr);
                }
                break;
           case 2:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    src_ptr.reset();
                } else if(selector == 2){
                    new_ptr.reset();
                }
                break;
            case 3:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    (*src_ptr).fp();
                } else if(selector == 2){
                    (*new_ptr).fp();
                }
                break;
            case 4:
                write_guestbook();
                break;
            case 5:
                view_guestbook();
                break;
            case 6:
                return 0;
                break;
            default:
                break;
        }
    }
}
```
여기서 smart pointer라는 개념이 나온다. src_ptr과 new_ptr은 같은 공간을 공유하는 포인터이다. 여기서 둘 new_ptr을 free하면 원본인 smart 포인터도 free된다. 여기서 write_geustbook을 하면 방금 free했던 new_ptr이 할당되고 또 write_geustbook을 하면 smart 포인터가 재할당 된다. 그래서 2번째 write_guestbook에서 get shell의 주소를 입력하고 3번을 눌러 1을 선택하면 셸이 따진다.

![](https://i.imgur.com/CPbzTWk.png)

Python-CLE의 CLE는 Common Language Execution 의 약자로 CLE는 바이너리와 관련 라이브러리를 로드하고, import를 해결하며, OS의 로더 같은 방식으로 프로세스 메모리의 추상화를 제공한다.

이는 `angr` 프로젝트의 일부로 개발되었으며, 다양한 바이너리 포맷을 로드하고, 분석하기 위한 플랫폼을 제공합니다. `CLE`는 바이너리와 그 의존성을 추상화하여, 사용자가 바이너리 분석을 보다 쉽게 할 수 있게 도와줍니다.

```shell
pip install cle
```
설치

~~아니 pie 걸려있는 애들은 뭔가 이상하게 됨~~

```python
import cle
ld = cle.Loader("test")
# 그니까 pie걸린 바이너리가 0x401060 이렇게 뜨는 이유가
# 여기서 보여주는건 메모리에 로드된 주소가 런타임이후 재배치 되서 그런거인듯
print("1-----")
print(ld.all_objects)           # 모든 load된 object 출력, 그니까 내가 봤을 때 메모리에 로드 된거랑 실제 

# # print("2-----")
# # print(ld.main_object)          # 프로젝트를 load할때 직접 지정한 main 개체

# # print("3-----")
# # print(ld.shared_objects)       # shared object 이름에서 헤딩 객체로의 매핑된 딕셔너리이다.

# # print("4-----")
# # print(ld.all_elf_objects)       # ELF 파일에서 로드된 모든 개체

# # print("5-----")
# # print(ld.extern_object)         # 해결되지 않은 import 에 대한 주소를 제공하는데 사용되는 extrens개체

# # print("6-----")
# # print(ld.kernel_object)         # 에뮬레이트된 시스템 호출에 대한 주소를 제공하는데 사용된다.

# # print("7-----")                 # 해당 주소가 어디에 포함되어있는지 
# # print(ld.find_object_containing(0x600000)) # 

obj = ld.main_object

# # print("8-----")                 # object의 entry point
# # print(hex(obj.entry))

# # print("9-----")                 # object의 최대, 최소 주소
# # print(hex(obj.min_addr))          
# # print(hex(obj.max_addr))

# # print("10-----")                 # ELF의 segmnets와 sections을 받는다.
# # print(obj.segments)
# # print(obj.sections)

# # print("11-----")                 # 해당 주소가 포함된 segment or section을 받는다.
# # print(obj.find_segment_containing(obj.entry))
# # print(obj.find_section_containing(obj.entry))


print("12-----")                   # 심볼에 대한 plt 주소 가져오기
addr = obj.plt['puts']
print(hex(addr))
# # print(obj.reverse_plt[addr])

# # print("13-----")   # CLE에 의해 객체의 사전에 연결된 base와 실제로 메모리에 매핑된 위치를 표시한다
# # print(hex(obj.linked_base))  # pie 걸리면 0으로 뜸
# # print(hex(obj.mapped_base))

puts = ld.find_symbol('puts')         # symbol개체 반환
print(puts)

# # # symbol의 주소를 보고하는데 세 가지 방법이 있다.
# # # 1. .rebased_addr는 글로벌 주소 공간에 있는 주소이다. 
# # # 2. .linked_addr은 바이너리의 사전에 link된 베이스에 상대적인 주소이다.
# # # 3. .relative_addr는 객체 베이스에 대한 상대적인 주소이다. 이것은 RVA로 알려져있다.

# # print(puts.name)
# # print(puts.owner)

# # print(hex(puts.rebased_addr))
# # print(hex(puts.linked_addr))
# # print(hex(puts.relative_addr))

# print(puts.is_export)
# print(puts.is_import)


# main_strcmp = ld.main_object.get_symbol('puts')
# print(main_strcmp)

# print(main_strcmp.is_export)
# print(main_strcmp.is_import)
# print(main_strcmp.resolvedby)

# print(ld.shared_objects['libc.so.6'].imports)


# from pwn import *

# e = ELF("./test")
# e1 = e.libc

# put_plt = e.symbols['plt.puts']

# print(hex(put_plt))
# print(hex(e.address))
# print(hex(e1.symbols['puts']))
```

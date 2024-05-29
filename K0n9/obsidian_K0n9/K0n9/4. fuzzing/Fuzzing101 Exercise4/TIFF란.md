Tagged Image File Format(.tiff)

래스터 그래픽과 이미지 정보를 저장하는 데 사용되는 컴퓨터 파일이다.

이는 세가지 섹션으로 구성된다.

IFH (Image File Header)
IFD (Image File Directory)
Image data (생략 가능 - 이미지 데이터가 없는 파일 존재 가능)

IFD와 Bitmap Data는 이미지의 갯수에 따라 여러개가 될 수 있다.

TIFF의 가장 큰 특징은 이미지에 대한 정보를 태그의 형식으로 제공한다. 이미지 정보를 위해 메이커, 컴퓨터 종류 혹은 생성 날짜 및 시간, 사용 소프트웨어 등을 위한 상당한 많은 태그들을 정의하고 있다.

TIFF는 하나의 파일에 여러 개의 이미지가 함께 저장되는 것이 가능하다. 각 이미지는 하나의 IFD와 하나의 Image data로 구성되며 이를 TIFF 서브파일이라 부른다. 

IFH는 파일의 처음 8 byte 라는 고정 위치를 갖는다.
![](https://i.imgur.com/PwVTNrx.png)

아무튼 여기서는 TIFF는 tag로 이미지의 정보를 가지고 있다고 생각해두자.

6버전 이후 즉, 현재 버전은 tag가 아니라 field로 이름이 바뀌었다.

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426584-f700c3df-935e-406d-a918-540a42c7635b.png)


메인 함수 부터 간략하게 살펴보자면,

Benchmark 클래스에서 클래스 변수 filterpolicy를 선언하고

생성자 Benchmark()에서 해당 변수에 Bloomfilterpolicy 값을 채웠다.

이 다음엔 해당 클래스 변수를 바탕으로 Run() 클래스 함수를 실행하게 된다.

<br/>
<br/>

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426611-de6ca39a-c5a3-4023-a8a1-eb495d5cb9b5.png)

Run() 클래스 함수 역시 크게 3가지 부분으로 나뉘는데,

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426642-fe0298bb-797c-4453-8baf-877f042e7468.png)


PrintHeader()는 db_bench를 돌리는 환경이나 데이터의 정보를 터미널에 출력해주는 함수이다.

<br/>
<br/>

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426672-0d944475-79c4-432c-ba36-14df703c97f6.png)


그 다음 Open 함수의 경우

Options이란 클래스를 새롭게 생성하여

filterpolicy를 포함한 캐시나 버퍼 사이즈 등의 변수들을 저장하고

이를 인자로 DB::Open 함수를 실행한다.

<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426707-58f23dad-5837-46e5-aa3d-a652ff239d58.png)
 

이 다음 DB::Open는 데이터베이스를 여는 함수로, 

인자로 받은 options의 값들을 impl이란 변수에 넣고

이를 인자로 MaybeScheduleCompaction() 함수를 수행한다.

<br/>
<br/>
 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426744-152b6145-f791-45f1-b45e-3e8c5139930f.png)


 다음 흐름은 위와 같은데 Compaction을 진행하고

필터 블록을 포함하여 SSTable을 생성한다.

이 과정에서 filter_policy는 이전 함수에서 다음 함수로 값을 전달 받기만 하므로 자세한 설명은 생략한다.

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426775-4a824072-59c7-41dc-bb55-70ff53007dc4.png)

참고로 블룸필터는 SSTable에 생성되므로,

전체 데이터 크기가 적어 compaction이 발생하지 않는다면

BGWork()가 아닌 NeedsCompaction()이 수행되어 블룸 필터가 생성되지 않는다.

 <br/>
<br/>

 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426808-21842d9f-e759-45ce-8a3c-1228726f1928.png)



이후 마지막 GenerateFilter 함수에서,

지금까지 해온 filterpolicy의 createFilter 함수를 호출하여 write를 진행한다.

<br/>
<br/>
<br/>
<br/>

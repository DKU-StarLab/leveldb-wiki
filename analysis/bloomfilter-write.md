
# Bloom Filter Write

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426584-f700c3df-935e-406d-a918-540a42c7635b.png)


db_bench의 메인함수 실행할 때,

Benchmark 클래스에서 filterpolicy와 같은 클래스 변수들을 선언하고

생성자 Benchmark()에서 해당 변수들의 값을 할당한 다음

해당 클래스 변수들을 바탕으로 Run() 클래스 함수를 실행하게 된다.

<br/>
<br/>
<br/>


 ![image](https://user-images.githubusercontent.com/101636590/189701906-715a3bca-2840-444f-a2a6-65d9e488a611.png)
 
Run() 클래스 함수는 크게 3가지 부분으로 나뉘게 되는데,

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426642-fe0298bb-797c-4453-8baf-877f042e7468.png)


PrintHeader()는 db_bench를 돌리는 환경이나 데이터의 정보를 터미널에 출력해주는 함수,

그리고 Open()은 write 과정을, RunBenchmark()는 Read 과정을 처리하는 함수이다. 

<br/>
<br/>

 
![image](https://user-images.githubusercontent.com/101636590/189703746-c4dbafce-b352-46d5-8404-ac614a7536ff.png)

그 중 Open() 함수에선

filterpolicy를 포함한 클래스 변수값들을 options이란 새로운 Struct에 담아 DB::Open() 함수로 전달,

DB::Open() 함수도 인자로 받은 options의 값들을 impl이란 클래스에 넣고

이를 인자로 MaybeScheduleCompaction() 함수를 수행하는 등,

filterpolicy를 포함한 변수값들을 다음 함수로 계속 전달하는 과정을 반복한다. 

<br/>
<br/>
 <br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426744-152b6145-f791-45f1-b45e-3e8c5139930f.png)


그 다음 코드 흐름은 위와 같은데, 

위 코드 흐름은 Compaction을 진행하고 FilterBlock을 포함한 SSTable들을 생성하는 과정으로

해당 과정에서 filterpolicy는 이전 함수에서 다음 함수로 값을 전달 받기만 하므로 자세한 설명은 생략한다.

<br/>
<br/>
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426775-4a824072-59c7-41dc-bb55-70ff53007dc4.png)

참고로 블룸필터는 SSTable에만 생성되므로,

전체 데이터 크기가 적어 compaction이 발생하지 않는다면

MabueScheduleCompaction()이 BGWork()가 아닌 NeedsCompaction()를 호출하게 되어 블룸 필터가 생성되지 않는다.

 <br/>
<br/>
<br/>
<br/>

 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426808-21842d9f-e759-45ce-8a3c-1228726f1928.png)



이후 마지막 GenerateFilter() 함수에서

지금까지 전달해온 filterpolicy의 createFilter 클래스 함수를 호출하게 되는데,

<br/>
<br/>
<br/>
<br/>

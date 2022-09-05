# #1 블룸 필터란 무엇인가<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424363-05494e10-e230-45b1-9a2a-18f413748970.png)

블룸 필터는 데이터 블록에 특정 key의 데이터가 존재하는지 확인할 수 있는 확률적 자료 구조이다.

블룸 필터는 데이터를 Write를 할 때에 각각의 key에 k개의 해시 함수를 수행하여 각 key 마다 k개의 해시 값을 얻고,

해당 값에 해당하는 블룸 필터 배열 칸들의 값을 0에서 1로 수정하여 해당 key가 데이터 블록에 존재함을 나타낸다.

그럴경우 반대로 Read를 할 때엔 데이터 블록을 하나하나 전부 읽는 대신

읽으려는 데이터에 동일한 k개의 해시 함수를 수행하여 k개의 해시 값을 얻고

해당 배열 칸의 값이 전부 1인 데이터 블록만을 선별하여 읽는 것으로 Read 성능을 향상시키는 것이 가능하다.
<br><br><br><br>
   

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424577-84d02a8f-c306-431f-b7b9-df92d3c2027b.png)

하나의 sstable엔 n개의 데이터 블록과 1개의 필터 블록, 1개의 메타 인덱스 블록이 존재하며

필터 블록엔 n개의 블룸 필터 배열이, 메타 인덱스 블록은 각 블룸 필터 배열이 어느 데이터 블록의 필터인지 저장한다.<br>
  
  <br><br><br><br>
   
   
  
# #2 False Positive


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424609-121954ba-8a0d-486e-83f5-c4102a669c8e.png)

블룸 필터의 장점은 True Negative가 절대로 발생하지 않는단 점이다.

True Negative는 데이터베이스에 존재하는 데이터를 존재하지 않는다 판단하는 것으로,

이는 필터의 신뢰성과 직결된 문제이므로 True Negative가 발생하지 않는단 점은 큰 장점이다.

<br>

다만 반대로 존재하지 않는 데이터를 존재한다 판단하는 False Positive의 경우엔 종종 발생할 수 있는데,

이는 True Negative보단 상대적으로 덜 중요한 문제이지만

False Positive로 인해 읽을 필요 없는 데이터 블록까지 읽어 성능이 떨어지게 되므로

False Positive를 줄이는 것 역시 블룸 필터의 중요한 과제이다.
  
   
 
<br/>
<br/>
<br/>
<br/>



![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424697-ef93e101-a865-47a3-9e14-2046590dd9d9.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424752-f19bab99-09cd-4c83-9686-c0de4776c88d.png)

LevelDB가 제공하는 벤치마킹 도구인 db_bench를 통해 측정을 진행할 때,

우리는 key 하나당 사용할 비트의 개수(Bloom_bits)나 쓰거나 읽을 데이터의 개수(Num) 등을 조절할 수 있다.

이때 생성되는 블룸 필터 배열의 크기는 (Bloom_bits) * (Num) bits가 된다.

이는 블룸필터의 핵심 코드 파일인 bloom.cc의 CreateFilter 클래스 함수에서 확인할 수 있다.

 
 <br>

db_bench에선 블룸 필터를 사용하지 않는 것이 디폴트이기에 Bloom_bits 값을 지정하여 블룸 필터를 켜주어야 하며,

만약 블룸 필터를 사용한다면 LevelDB에서 이상적으로 생각하는 Bloom_bits 값은 10이다.

(이는 쓰기 성능을 크게 떨어뜨리지 않으면서, 읽기 성능을 향상시킬 수 있는 마지노선의 수치이다.)

 

<br/>
<br/>
<br/>
<br/>
 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424934-cea90e5b-ca24-449a-90f4-53dc6452f539.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424941-231cf55b-6188-42a0-88ec-8b7c46d644f0.png)

또한 해시 함수의 개수 K의 경우 K = ln2 * (M/N) = ln2 * B이란 수식을 사용한다.

이것은 수학적으로 증명된, 블룸 필터의 false positive 발생률을 최소한으로 줄일 수 있는 값이다.

이에 관한 더 자세한 증명은 https://d2.naver.com/helloworld/749531 해당 글을 참조.

이 역시 bloom.cc 파일의 코드에서 구현되어 있는 것을 확인할 수 있다.

<br/>
<br/>
<br/>
<br/>
 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424987-d8a58b47-0e68-4f13-b3d1-bc7ea413406a.png)


False positive가 발생할 확률을 수학적으로 정리하면 위와 같으며,

이를 e의 정의를 통해 정리하면 오른쪽과 같은 식이 나온다.

여기서 k의 값이 ln2 * (m/n)이라면 해당 식은 (1/2)^k의 꼴로 정리되므로,

해당 값이 false positive 발생 확률을 가장 최소화 시킬 수 있는 최적의 수치이며

또한 k의 값이 클수록 false positive가 발생할 확률이 작아짐을 알 수 있다.

(그리고 k = ln2 * b = 0.69 b 이므로 k의 값은 bloom_bits의 값과 비례한다.)

<br/>
<br/>
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425038-94b8594d-a3c9-4325-8ddc-61f6c90c89b9.png)

이를 db_bench를 통해 실험해 본 결과 bloom_bits 값이 커질수록

false positive가 적게 발생하여 평균보다 성능이 빠른 경우가 더 많이 발생함을 알 수 있다.

(해당 경우는 노란색으로 표시)

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425050-e20cbdb2-f988-45aa-bb71-f423c77236f8.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425059-75c0aa55-ee7a-4b32-9905-e97275d20966.png)



다만 bloom bits의 값이 너무 커질경우 오히려 false positive가 더 많이 발생하는 경향을 보였는데,

이는 bloom.cc 코드에서 해시 함수의 최대 개수를 30개로 제한하고 있기에

k = ln2 * b 라는 공식이 지켜지지 않아 되려 false positive가 증가한 것으로 추정된다.
<br/>
 <br/>
<br/>
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425778-ecfbef99-a13b-4efb-82ad-c666be9dc22e.png)


그렇기에 해시 함수 개수를 제한하는 코드 부분을 제거하고 다시 db_bench를  본 결과,

모든 ReadRandom의 측정값이 평균과 비슷한 값이 나옴을 알 수 있는데

이는 false positive의 발생 확률이 대폭 낮아져서 false positive가 거의 발생하지 않았기 때문으로 추정된다.

허나 이 경우엔 한번의 read에 무려 690번의 해시를 처리해야 하므로 (전자의 경우엔 30번)

false positive 자체는 덜 발생하였으나 이를 위한 더 많은 해시 함수를 처리하는데 필요한 overhead가 더 컸기에

되려 성능이 떨어진 것을 볼 수 있다.

 

 <br/>
<br/>
<br/>
<br/>

# #3 블룸 필터의 코드 분석

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425832-0ca0a6f6-a8d4-471b-8765-44a3ccad1904.png)


LevelDB의 전체 코드엔 약 100개 가량의 메인 함수가 존재한다.

여기서 db_bench의 makefile을 확인해 보면 db_bench의 메인 함수는

db_bench.cc 파일에 존재함을 알 수 있다.

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425880-7bcff039-5faa-4e42-9ad9-763c12f6c614.png)


db_bench.cc 파일의 메인 함수는 크게 2개의 파트로 구성되는데,

첫번째는 db_bench를 실행할때 bloom_bits, num과 같은 파라미터를 읽어들이는 sscanf 파트,

그리고 benchmark.Run()이란 클래스 함수 두 파트로 구성된다.

  <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425930-e7017bb8-d77e-428a-a4c1-fa9e1b5288ca.png)

benchmark.Run()도 크게 3가지 파트로 나눌 수 있는데,

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425988-2d5638f4-ba97-4a54-9f5e-ddc3529dbe7a.png)

벤치마크 클래스에서 filter policy라는 클래스 변수가 선언되며

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425998-87fd0957-3bc3-4c7f-9239-ebf597053cf7.png)

이후 해당 클래스의 생성자에서 filter policy 변수의 값을 지정한다.

메인함수에서 sscanf로 읽은 bloom_bits 값이 0 이상이면 

블룸 필터를 사용한다는 것이니 NewBloomFilterPolicy 함수를 호출하고,

아닐 경우엔 블룸 필터를 사용하지 않는다는 것이니 Null을 리턴한다.

(bloom_bits의 디폴트 값은 -1로 따로 설정하지 않으면 블룸 필터를 사용하지 않으며,

0으로 지정한다면 최소한의 크기의 블룸 필터를 사용하게 된다.)

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426008-acd92e3d-9adf-4d7b-8986-2c248e6705b1.png)

NewBloomFilterPolicy()는 "FilterPolicy" 클래스를 오버라이드한 "BloomFilterPolicy" 클래스를 생성하여 리턴하는 함수인데,

  <br/>
<br/>



![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426122-bd927d23-6f3e-45f1-a2d4-056b08589f74.png)


FilterPolicy 클래스의 경우

필터의 이름을 리턴하는 Name() 함수

Write할 때 블룸 필터 배열을 생성하는 CreateFilter() 함수 

그리고 Read할 때 배열에 특정 key 값이 존재하는지 확인하는 KeyMayMatch()

3개의 클래스 함수로 구성되어있다.

이를 오버라이드한 BloomFilterPolicy 클래스의 경우

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426214-c4c7fdf2-881d-46b7-80a4-01c4988f520b.png)



해시 함수의 개수를 정하고 최대 개수를 제한하는 코드 부분과

특정 이름을 리턴하는 Name() 함수

<br/>
<br/>

 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426303-d9c4be90-2d6b-4025-8035-5483070159f5.png)

그리고 CreateFilter() 함수는 bloom_bits(=bits_per_key)값과 num(=n)값을 곱해

블룸 필터 배열의 크기(=bits)를 정하는 파트와

<br/>
<br/>

 ![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426325-b0586409-deeb-4856-b1f2-8611ce6b46e9.png)

"더블 해싱"을 처리하는 파트로 나뉘게 된다.

  <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426381-db205ffc-c946-49af-a75d-2b0869145737.png)
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426385-eb7ead05-1359-4802-9bc6-0f2d892756bb.png)


https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf

해당 함수의 주석에 작성되어 있는 논문을 참고하면,

기존의 k개의 서로 다른 해시 함수를 사용하는 방식 대신

2개의 해시 함수와 그 중에서 하나를 중복 사용하는 "더블 해싱" 방식을 통해 

해시 함수로 인한 부하와 연산을 대폭 줄이면서 기존의 성능을 유지할 수 있음을 수학적으로 증명하고있다.

<br/>
<br/>

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426427-2a7c8496-b118-42aa-b6da-112205d9b581.png)

LevelDB에서 첫번째 해시 함수는 BloomHash()란 함수로,

두번째 해시 함수는 ( h>>17 ) | ( h<<15 ) 란 비트 연산으로 구현되어있다.

 
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426431-9ec3e7cb-ba41-48da-b90a-280456b4ceb7.png)
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426436-93fc5dbf-b93a-4b9d-8abc-9b2a41aba6cc.png)


BloomHash는 key 값을 인자로 특정 값을 리턴하는 전형적인 해시함수의 형태를 취하고 있으며,

 
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426489-099b0b6f-8a1c-4263-a575-60407a8b8c65.png)


쉬프트 연산은 앞의 15자리와 뒤의 17자리를 바꾸는 형태로 구성되어있다.

(참고로 해시값은 uint32_t라는 32비트 자료형을 사용한다.)

이러한 쉬프트 연산은 해시 함수의 성질을 만족하면서도

연산에 필요한 overhead가 적어 사용되는 것으로 추정된다.

  <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426517-b9b43ff8-3541-474a-8549-2132edf178f5.png)



그리고 특이사항으론 hash 하나당 하나의 bit를 1로 바꾸는데,

블룸 필터 배열은 char 배열로 한 칸의 크기가 1 byte = 8 bits 이므로

배열의 한 칸을 8칸으로 쪼개서 사용하고 있음을 알 수 있다.

 <br/>
<br/>
<br/>
<br/>



![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426561-9998ffc6-d182-497e-8cc2-8a92f6a5611e.png)


데이터를 Read할때 사용하는 KeyMayMatch 함수의 경우

CreateFilter와 거의 동일한 연산을 사용하나

or  대신 and 연산을 사용하여 필터 내부에 특정 key 값이 존재하는지 확인하는 것을 볼 수 있다.

   <br/>
<br/>
<br/>
<br/>


 # #4 Write를 할 때의 코드 흐름

 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426584-f700c3df-935e-406d-a918-540a42c7635b.png)


다시 코드 흐름을 살펴보자면,

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

# #5 Read를 할 때의 코드 흐름

![img](https://user-images.githubusercontent.com/101636590/187571334-4c1d3c8d-77e1-4824-8338-45dcf735d4c1.png)
db_bench는 db_bench.cc 파일의 main 함수로 부터 시작되며,

파라미터를 scanf로 읽어들인 뒤 benchmark.Run() 클래스 함수를 실행한다.

해당 함수는 Write 과정을 처리하는 Open() 함수와 Read 과정을 처리하는 RunBenchmark()를 순서대로 실행한다.

Open() 함수를 필두로 한 Write 과정을 설명하였으니

이제 RunBenchmark() 함술를 필두로 한 Read 과정에 대해 살펴보자면,

<br/>
<br/>
<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571403-87a6b37d-79d7-4316-8fcb-0be921793b05.png)

해당 함수는 num_threads, name, method 3개의 파라미터를 필요로 하는데,

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571432-9cd667da-2c20-4e5e-ab32-524c60a0f005.png)

num_threads는 스레드의 갯수로 db_bench를 돌릴때 파라미터로 지정할 수 있다.

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571446-87ebe70f-8ff6-4f80-b139-bc8e4992bc8c.png)

Name은 벤치마크의 종류를 나타내는 문자열 변수이고,

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571476-2412be67-b288-4a32-9dda-09c1aaa5463e.png)

Method는 name에 해당하는 벤치마크 함수의 주소값을 저장하는 포인터 변수이다.

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571491-fc28e122-7213-43b5-9a9b-18d37b2aebce.png)

입력한 인자중 method 변수의 주소값은 arg[] 배열에 들어가게 되고,

ThreadBody() 함수와 arg[] 배열을 인자로 StartThread 함수를 실행하게 되는데

해당 함수는 arg[] 배열을 인자로 ThreadBody() 함수를 실행시키는 함수라 보면 편하다.


<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571527-24c226d1-7472-4f77-8d3e-a3d5d63dd9d5.png)

이후 ThreadBody() 함수를 통해 할당된 스레드로 벤치마크를 실행한다.

<br/>
<br/>
<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571539-e04da925-24a4-45ab-a18a-a3df6905e80c.png)

벤치마크의 실행 과정은 다음과 같은데,

Bencmark::ReadRandom()과 같은 벤치마크 함수를 시작으로 여러번의 Get() 함수를 통해

데이터베이스에서 sstable로, 그리고 각 레벨, 테이블, 필터 블록, 블룸 필터 순으로 점차 탐색 범위를 좁혀간다.

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571560-8df9e556-8de2-4417-8406-276b15299050.png)

벤치마크 함수에선 각 벤치마크에 할당된 기능을 수행한 다음 db->Get() 함수를 사용한다.

이때 db는 write 과정에서 데이터베이스를 열때 DB::open() 함수에서 사용한 데이터 베이스의 주소값 이다.

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571596-8689132d-30a6-46c7-9573-6ed38a1afb8e.png)

해당 DBImpl::Get() 함수는 memtable, immemtable, sstable을 순서대로 탐색하는데

이때 sstable을 탐색할때 사용하는 함수는 Version::Get() 함수이다

(블룸 필터는 sstable에서만 사용된다.)

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571611-12c3dc7f-8354-41c1-a755-cb0c615a7f79.png)


Version::Get() 함수는 Match() 함수를 인자로 ForEachOverlapping() 함수를 실행시키는 함수이고,


<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571632-506f8716-7407-4241-a4b3-2adbbbc165d1.png)

ForEachOverlapping() 함수는 sstable의 level 0를 먼저 탐색한 다음 다른 레벨을 탐색하는 함수이며,

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571645-e9646045-da9d-4c6a-8ec5-648d7bb4952a.png)


해당 탐색 과정에 사용되는 함수가 Match() 함수이다.

해당 함수는 TableCache::Get() 함수를 통해 테이블에 특정 key가 존재하는지 확인한 다음

존재 여부에 따라 bool 값을 리턴한다.

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571728-d8a35e58-9fca-4da4-8cbc-ff2c0efca467.png)

TableCache::Get() 함수는 Table::InternalGet() 함수를 호출하여 각 테이블을 탐색하고,

<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571735-41dbff23-72b8-4dbc-b991-d202481bb541.png)

InternalGet() 함수는 FilterBlockReader::KeyMayMatch() 함수를 통해 필터 블록을 탐색한 다음

특정 key값이 존재하면 해당 블룸 필터가 가리키는 데이터 블록을 읽게된다

<br/>
<br/>


![img](https://user-images.githubusercontent.com/101636590/187571767-41e151fb-626a-4fde-a91c-29e2982f2ed2.png)

마지막으로 FilterBlockReader::KeyMayMatch()는 BloomFilterPolicy::KeyMayMatch()를 통해 

필터 블록 안의 블룸 필터를 탐색하는데,

<br/>
<br/>


![img](https://user-images.githubusercontent.com/101636590/187571797-2acbf595-64cb-475d-a080-b3d74276ef80.png)


해당 함수는 앞서 미리 설명하였던 블룸 필터의 핵심 코드 bloom.cc 파일의 함수로,

Write를 할 때 사용했던 hash 연산을 똑같이 처리한 다음

해당 hash 값이 필터 블록에 존재하는지 확인하게 된다.


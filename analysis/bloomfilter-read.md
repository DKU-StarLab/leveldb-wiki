

![img](https://user-images.githubusercontent.com/101636590/187571334-4c1d3c8d-77e1-4824-8338-45dcf735d4c1.png)
db_bench는 db_bench.cc 파일의 main 함수로 부터 시작되며,

파라미터를 scanf로 읽어들인 뒤 benchmark.Run() 클래스 함수를 실행한다.

해당 함수는 Write 과정을 처리하는 Open() 함수와 Read 과정을 처리하는 RunBenchmark()를 순서대로 실행한다.


<br/>
<br/>
<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571403-87a6b37d-79d7-4316-8fcb-0be921793b05.png)

이때 RunBenchmark() 함수는 num_threads, name, method 3개의 파라미터를 필요로 하는데,

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

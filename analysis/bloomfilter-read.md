# Bloom Filter - Read

![img](https://user-images.githubusercontent.com/101636590/187571334-4c1d3c8d-77e1-4824-8338-45dcf735d4c1.png)
db_bench는 db_bench.cc 파일의 main 함수로 부터 시작되며,

파라미터를 scanf로 읽어들인 뒤 benchmark.Run() 클래스 함수를 실행한다.

해당 함수는 Write 과정을 처리하는 Open() 함수와 Read 과정을 처리하는 RunBenchmark()를 순서대로 실행한다.


<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189677379-32e48a2e-b087-47e3-9b8c-5771cc7a783c.png)

이때 RunBenchmark() 함수는 num_threads, name, method 3개의 파라미터를 필요로 하는데,

그 중 num_threads는 스레드의 갯수를 나타내며 db_bench를 시작할때 파라미터 값으로 지정할 수 있다.

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189677993-8bdbcf82-3b03-43c1-8c76-2b55927d5d93.png)


그 다음 Name은 벤치마크의 종류를 나타내는 문자열 변수로

파라미터에서 입력한 값들을 쉼표로 나누어 저장하며,

Method는 Name에 해당하는 벤치마크 함수의 주소값을 저장하는 포인터 변수이다.

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189682019-4ded8020-839b-48f1-bcdb-7ad3a886a9b7.png)

이후 Runbenchmark() 함수의 method 주소값은 arg[] 배열에 저장,

해당 값과 ThreadBody() 함수를 인자로 StartThread 함수를 실행하게 되며

ThreadBody() 함수에선 할당된 스레드로 주어진 벤치마크를 수행한다.

<br/>
<br/>
<br/>
<br/>

![img](https://user-images.githubusercontent.com/101636590/187571539-e04da925-24a4-45ab-a18a-a3df6905e80c.png)

그 다음 벤치마크의 코드 흐름은 다음과 같은데,

ReadRandom()이나 ReadHot() 같은 벤치마크 함수를 시작으로 여러번의 Get() 함수를 통해

데이터베이스에서 sstable로, 그리고 각 레벨, 테이블, 필터 블록, 블룸 필터 순으로 점차 탐색 범위를 좁혀간다.

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189691492-2cfdce0a-d5d8-4eeb-b5d5-80627986b82d.png)

벤치마크 함수에선 각 벤치마크에 할당된 기능을 수행한 다음 db->Get() 함수를 사용한다.

이때 db는 write 과정에서 데이터베이스를 열때 DB::open() 함수에서 사용한 데이터 베이스의 주소값 이다.

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189692806-b438bfc1-8ea4-4aa5-a65d-a134b6223b82.png)

해당 DBImpl::Get() 함수는 memtable, immemtable, sstable을 순서대로 탐색하는데

이때 sstable을 탐색할때 사용하는 함수가 Version::Get() 함수이며 

(참고로 블룸 필터는 sstable에서만 사용된다.)

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189693687-c932cc19-9c1b-456e-8201-b87d3525df0c.png)


Version::Get() 함수는 Match() 함수를 인자로 ForEachOverlapping() 함수를 실행시키는 함수이다.

ForEachOverlapping() 함수는 sstable의 level 0를 가장 먼저 탐색한 다음 다른 레벨들을 탐색하는 함수,

그리고 해당 탐색 과정에 사용되는 Match() 함수는 

테이블에 특정 key가 존재하는지 확인한 다음 존재 여부에 따른 bool 값을 리턴하는 함수이다.

<br/>
<br/>
<br/>
<br/>

![image](https://user-images.githubusercontent.com/101636590/189698016-8058db18-1d10-4b3b-b57c-363699681d02.png)

이 Match() 함수가 호출한 TableCache::Get() 함수는 테이블을 탐색,

그 다음 FilterBlockReader::KeyMayMatch()는 필터 블록을,

BloomFilterPolicy::KeyMatMatch()는 블룸 필터를 탐색하여 

최종적으로 특정 키가 존재 여부를 확인하게된다.

그리고 중간의 InternalGet() 함수는 찾고있는 키가 존재할 경우 Read를 진행하는 함수이다.



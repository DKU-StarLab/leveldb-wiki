# SSTable 
LevelDB는 LSM트리(Log Structured Merge Tree)를 기반으로 만들어졌으며, 이 때문에 write를 할 때 데이터를 디스크에 직접 쓰는게 아니라 Log에 처음 쓰고 MemTable에 쓰게 된다. MemTable은 데이터가 꽉 차면 Immutable MemTable로 데이터를 보내고, Immutable MemTable에선 storage(즉 disk)로 `flush`를 하게 된다.

이렇게 메모리의 데이터가 storage로 `flush`될때 levelDB는 데이터를 SSTable(Sorted String Table)이라는 자료구조에 담아 보관하며, 이 글에선 이 SSTable에 대해 다룬다.  
<br/>  

## SSTable format  
SSTable은 물리적으론 4KB짜리의 블록들로 나뉘며, 각 블록들은 데이터를 저장하는 필드 외에 압축 유형(블록에 저장된 데이터가 압축됐는지, 압축됐다면 어떤 알고리즘으로 압축했는지를 나타냄)과 CRC 확인(데이터에 오류가 있는지)을 위한 필드가 있다.  

논리적인 관점에선 SSTable을 다음과 같은 구조로 볼 수 있다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/182532427-47d356d8-c3a7-4d72-b8df-f3adcb75bcbe.png"></p>  

<br/>  

### Data Block  
Key-Value Pair들이 저장되는 블록이다. 구조는 다음과 같다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187910209-7d931fb7-6870-45e2-ade4-f0316cb25c72.png"></p>  

SSTable은 key들을 정렬된 형태로 가지고 있기 때문에 각각의 key들은 서로 겹치는 부분들을 많이 가질 수 있게 된다. LevelDB는 이런 문제점으로부터 공간상의 효율을 위해 key를 저장할 때 전체 Key값을 저장하는 게 아니라 이전 레코드의 key와 겹치지 않는 부분(공유하지 않는 부분이라고 표현하며, 정확히 말하자면 이전 레코드의 key와 공유하는 접두사를 제외한 뒷부분)만 저장하도록 설계했다.  

이러한 방식은 Key-Value Pair를 저장할 때 key를 저장하기 위해 쓰이는 부분이 줄어들기 때문에 메모리 절약이 가능하다는 장점이 있다. 그러나 SSTable이 내부적으론 key들을 정렬된 형태로 가지고 있어 원래대로라면 Binary Search를 통한 탐색이 가능함에도 불구하고, 이런 방식은 각 Entry들이 key값을 전체 값이 아닌 일부 값만 가지기 때문에 Binary Search를 제대로 사용할 수 없어 읽기 성능이 안 좋아진다는 단점이 있다.  

이런 문제점을 해결하고자 LevelDB는 `Restart Point`라는 개념을 도입했다. Data Block의 모든 Entry가 key값을 일부만 저장하도록 하는게 아니라 k개(디폴트는 16)의 Entry마다 전체 key값을 저장하도록 한 것이며, 이 때 전체 key값이 저장되는 Entry들을 `Restart Point`라고 한다. 위 그림에서 볼 수 있듯 Data Block은 내부적으로 각각의 `Restart Point`들의 위치를 별도로 저장한다.  

이렇게 `Restart Point`를 도입하여 k개의 레코드마다 전체 key값을 다시 쓰게 해줌으로써, Data Block 내부의 Entry들은 다음과 같이 `Restart Point`를 기준으로 일종의 구역들을 형성하게 된다.

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187920060-32e4a102-a0e7-4929-8d59-6494f9f09ee8.png"></p> 

이 때 SSTable이 내부적으로 key들을 정렬된 형태로 보관하므로, 각각의 `Restart Point`에 해당하는 Entry들도 서로 정렬된 형태를 가진다. 따라서 내가 찾고 있는 key가 있을 만한 구역을 `Restart Point`를 이용해 Binary Search로 찾아갈 수 있고 이를 이용해 read성능을 높일 수 있다.  
<br/>  

### Filter Block  
Bloom Filter들을 가지는 블록이다. 구조는 다음과 같다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187955758-acd5f9e8-8d00-4ea4-898b-9773e5071b9b.png"></p> 

Filter는 Bloom Filter들을 말하며, Filter offset은 각 Filter들의 시작 offset정보를 가진다. Filter offset\`s offset은 Filter 1 offset의 offset정보를 가진다. 즉 Bloom Filter를 읽을 땐 Filter offset\`s offset을 먼저 읽고, 이를 토대로 원하는 Filter의 offset을 읽은 후 해당하는 Filter로 찾아가 읽게 된다.  
<br/>  

### Meta Index Block  
Filter Block의 Index정보를 갖는 블록이다. 이와 관련된 하나의 레코드만 저장한다.  
<br/>  

### Index Block  
각 Data Block들의 Index정보를 갖는 블록이다. 구조는 다음과 같다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187952948-860a8f3d-1b91-4ce0-b127-3b7a56b64008.png"></p>  

각 Entry들은 해당하는 Data Block의 Index정보뿐만 아니라 Data Block의 size와 해당 Data Block에서 가장 큰 key값도 가진다.  
<br/>  

### Footer  
48Byte의 고정된 크기를 가지며, Meta Index Block의 index정보와 Index Block의 Index정보를 갖는 블록이다. 구조는 다음과 같다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187959918-6f36b165-b598-4aed-8c88-72716c7d0179.png"></p>  
<br/>  

### 참고 - Block Entry 구조
Data Block, Index Block, Meta Index Block은 BlockBuilder라는 동일한 객체를 통해 만들기 때문에 이 Block들의 구조는 사실상 같다. 즉 Data Block뿐만 아니라 Index Block과 Meta Index Block도 `Restart Point`를 가지고 있다. Data Block이 Key-Value Pair를 저장하듯이 Index Block과 Meta Index Block도 일종의 Key-Vaue Pair를 저장하는 것이며 Index Block을 예로 들면 각 Entry의 Max key항목이 key역할을, offset과 length항목이 value역할을 한다.  

이 Block들을 구성하는 Entry의 구조는 다음과 같다.  

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187917146-dcf4bd36-30b6-4406-ab15-ac461bb48f64.png"></p>  

- Shared key length : 이전 레코드의 key와 겹치는 부분의 길이
- Unshared key length : 이전 레코드의 key와 겹치지 않는 부분의 길이
- Value length : value의 길이
- Unshared key content : 이전 레코드의 key와 겹치지 않는 부분
- Value : value값  

#### ex)
```
- restart_interval = 3
- entry 1: key = abc, value = v1
- entry 2: key = abe, value = v2
- entry 3: key = abg, value = v3
- entry 4: key = chesh, value = v4
- entry 5: key = chosh, value = v5
- entry 6: key = chush, value = v6
```  
<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187918354-89d666b2-8a44-401f-97e4-fbf63f5fde0d.png"></p>  

`restart_interval`을 3으로 지정하여 3개의 레코드마다 `Restart Point`가 찍히는 것을 볼 수 있고, 이 `Restart Point`에 해당하는 Entry들은 다른 레코드들과는 달리 전체 key값을 저장하는 모습을 볼 수 있다.  

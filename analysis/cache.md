# LevelDB Cache Analysis

### <center> Contents 
  - [1. What is Cache](#1-what-is-cache)
  - [2. What is LRU](#2-what-is-lru)
  - [3. LRU Cache Structure in LevelDB](#3-lru-cache-structure-in-leveldb)
  - [4. Overrall Cache Flow](#4-overrall-cache-flow)
  - [5. Cache Source Code Analysis](#5-cache-source-code-analysis)
---
## 1. What is Cache

 캐시는 하드웨어의 한 장치로서 메인 메모리나 디스크에 접근을 하기 이전에 거치는 메모리 저장 장치이다. 
  
캐시의 장점은 먼저 접근하는 속도가 굉장히 느린 메인 메모리에 비해서 접근 속도가 굉장히 빨라서 메인 메모리의 값을 미리 저장을 해두어서 하웨어의 처리 속도를 굉장히 효율적으로 줄이는게 가능하다. 
  
하지만 캐시의 가장 큰 단점은 빠른 접근 속도로 인한 하드웨어의 비싼 가격이다. 비싼 가격으로 인해 큰 용량을 캐시에서는 사용하지 못하는 단점도 발생한다.
  
하지만 빠른 처리 속도로 인해 꼭 필요한 장치이다.

<img width="754" alt="스크린샷 2022-08-27 오후 2 15 50" src="https://user-images.githubusercontent.com/74447373/187015861-02659a06-9a5a-4951-8487-97f5d17ee9ea.png">


---
## 2. What is LRU
  LRU캐시란 말 그대로 Least Recently Used 알고리즘을 칭하는 말로 LRU캐시는 이 알고리즘을 사용하여 캐시를 처리하는 캐시를 말한다.

<img width="661" alt="스크린샷 2022-08-27 오후 2 23 03" src="https://user-images.githubusercontent.com/74447373/187016130-35ee76ac-7b3e-4430-aa4e-80b1ec5a0506.png">

 LRU캐시의 장점으로는 먼저 빠른 액세스 타임과 그리고 빠른 업데이트이다 탐색과 업데이트에서 각각 최악의 경우에서는 캐시를 탐색하는 O(n)의 시간복잡도를 필요로 함으로 빠름 탁색과 업데이트가 가능하다. 
  
 
  LUR 캐시의 단점은 위 그림과 같은 방식으로 캐시가 처리가 되며 가장 오래 참조하지 않는 캐시의 위치에 새로운 값을 삽입하는 방식으로 알고리즘을 처리한다. 이 방식의 단점은 먼저 어떤 캐시의 위치가 가장 오래 참조를 하지 않았는지 표시를하는 방식이 필요로 한다. 
  
---
## 3. LRU Cache Structure in LevelDB

내용작성

글

그림


---
## 4. Overrall Cache Flow

**_<span style="background-color:#808080">LevelDB에서 Cache의 메커니즘_**

LevelDB에서 Cache는 읽기 작업과 함께 Cache에 대한 메커니즘이 확인이 가능하다.

![화면 캡처 2022-08-30 201901](https://user-images.githubusercontent.com/84978165/187461770-b324bdf4-c633-4c48-a2bf-b38944f33912.jpg)

읽기 작업이 수행될 때 수행되는 함수 ReadOption이 먼저 확인 가능하다.

<img width="191" alt="image" src="https://user-images.githubusercontent.com/84978165/187462225-a978f054-a8af-4e7d-9f94-46b1e90e684f.png">

ReadOption이 호출된 후 DBimpl::Get이 호출되며 이 함수에서 구체적인 읽기 작업이 시작된다.


DBimpl:Get의 부분 소스코드:

      <1> MemTable* mem = mem_;
      <2> MemTable* imm = imm_;
      <3> Version* current = versions_->current();

      if (mem->Get(lkey, value, &s)) { <4>
      } 
      else if (imm != nullptr && imm->Get(lkey, value, &s)) { 
        <5>
      } 
      else { <6>
        s = current->Get(options, lkey, value, &stats);
        have_stat_update = true;
      }

<span style="color:red"><1></span> memtable에서 table을 찾기 위해 memtable mem_ 선언

<span style="color:red"><2></span>  memtable에 없다면 immutable memtable에서 table을 찾기 위해 memtable  imm_ 선언

<span style="color:red"><3></span>  둘 다 발견하지 못 할 경우 sstable에서 찾기 위해 현재 version을 가져오기 위하여 version current 선언

<span style="color:red"><4></span>  memtable에서 디스크 파일 검색을 수행하는 Memtable::Get호출

<span style="color:red"><5></span>  마찬가지로 immutable memtable에서 디스크 파일 검색을 수행하기 위하여 Memtable::Get호출

<span style="color:red"><6></span>  sstable에서 디스크 파일 검색을 수행하기 위하여 Version::Get 호출

이후 TableCache::Get 호출 -> **_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

**_<span style="background-color:#808080">LevelDB에서 Cache의 생성_**

TableCache는 ./db_bench의 option중 하나인 cache_size의 크기를 받아 TableCache::TableCacheSize함수를 통해 크기를 설정한 뒤 TableCache::TableCache를 통해 생성

TableCache's BackTrace:

    leveldb::Benchmark::Run
    leveldb::Benchmark::Open 
    leveldb::DB::Open
    leveldb::DBImpl::DBImpl
    leveldb::TableCache::TableCache

**_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

TableCache::Get, 본격적인 Cache를 이용한 읽기 작업을 수행하는 부분이다.

TableCache::Get:

    Cache::Handle* handle = nullptr;
    <1> Status s = FindTable(file_number, file_size, &handle);
    if (s.ok()) {
      <2> Table* t = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;
      s = t->InternalGet(options, k, arg, handle_result);

      cache_->Release(handle);
    }
    return s;

+LevelDB에는 TableCache와 BlockCache 두 가지의 다른 캐시가 도입되어 있다.

<1> 파일에 해당하는 테이블을 캐시에서 찾는다. 발견되지 않을 경우 파일을 먼저 오픈한 후 파일에 해당하는 테이블을 생성하여 캐시에 추가한다.

아래는 TableCache의 구조이다.

![화면 캡처 2022-08-30 232237](https://user-images.githubusercontent.com/84978165/187462453-17094db7-7466-45bf-978e-b556f496b1c2.jpg)

<2> InternalGet 함수는 SSTable에서 Key를 찾는 논리를 구현하며 BlockReader를 호출한다.

BlockReader의 Cache 사용 알고리즘:

1. 블록 캐시에서 직접 블록가져오기를 시도함

2. 블록캐시에 없으면 ReadBlock을 호출하여 파일에서 읽음

3. 읽기에 성공하면 블록 캐시에 블록을 추가함

아래는 BlockCache의 구조와 그에 대한 설명이다.

![화면 캡처 2022-08-30 232324](https://user-images.githubusercontent.com/84978165/187462821-adb03d15-b05e-462f-888e-9d3737b340bc.jpg)
  
![화면 캡처 2022-08-30 232339](https://user-images.githubusercontent.com/84978165/187462857-6c96fa56-7f2d-4d31-ad91-33aca4249dcb.jpg)


---

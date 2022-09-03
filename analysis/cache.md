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

<p align="center">
<img width="700" alt="스크린샷 2022-08-27 오후 2 15 50" src="https://user-images.githubusercontent.com/74447373/187015861-02659a06-9a5a-4951-8487-97f5d17ee9ea.png">
</p>


---
## 2. What is LRU
  LRU캐시란 말 그대로 Least Recently Used 알고리즘을 칭하는 말로 LRU캐시는 이 알고리즘을 사용하여 캐시를 처리하는 캐시를 말한다.

  LevelDB는 LRU 캐시를 구현의 전형적인 사례이다.
  
  아래는 LRU 캐시 구조이며 아래 그림의 노드 제거 프로세스와 같은 전략을 LevelDB의 캐시가 가지고 있다.

<p align="center">
<img width="700" alt="스크린샷 2022-08-27 오후 2 23 03" src="https://user-images.githubusercontent.com/74447373/187016130-35ee76ac-7b3e-4430-aa4e-80b1ec5a0506.png">
</p>

읽는 순서는 A, B, C, D, E, D, F이다.

1. 괄호 안의 숫자는 정렬을 나타내며 숫자가 작을수록 제일 왼쪽 즉, 나중에 나타난다.

2. 읽는 순서에 따라 E를 읽는 순서가 다가왔을 때 캐시가 가득 찬 것으로 확인되므로 가장 빠른(Least Recently) A가 제거된다.

3. 계속해서 D, F를 읽으며 정렬을 갱신하며 방문 횟수가 가장 적은 노드가 제거 및 대체가 반복되는 모습을 확인할 수 있다.

LevelDB는 LRU 캐시를 구현하기 위해서 두가지의 다른 데이터 자료 구조를 도입하여 사용하고 있다.

#### 1. HashTable(HandleTable) : 조회 구현에 사용
#### 2. Double linked list : 오래된 데이터를 제거하는데 사용되는 정렬
  
---

## 3. LRU Cache Structure in LevelDB

LevelDB의 LRU Cache Structure를 정확하게 파악하기 위하여 git clone을 통해 LevelDB소스코드를 받아와 분석하였다.

(source code download)
  
    git clone --recurse-submodules https://github.com/google/leveldb.git leveldb_release

LevelDB에서 Cache는 LRU Cache의 형태를 띄고있다.

아래는 전체 LRU Cache Structure의 개략도이다.

[정확한 코드의 분석은 leveldb_release/build/util/cache.cc 참고]

<p align="center">
<img width="800" alt="image" src="https://user-images.githubusercontent.com/84978165/188290272-482a3314-72e1-4ed9-b5d8-357276512bd7.png">
</p>

<br />

#### <span style="color:red">(1)</span> ShardedLRUCache : LevelDB내에서 ShardedLRUCache는 내부에 LRU Cache 16개를 구성하고 있으며(위 그림에서 LRU Cache는 원래 16개여야 한다) 외부 LRUCache로 정의한다.

### +Sharded란?
용량 이슈의 이유로 DB Level에서 한 데이터베이스의 데이터들을 다수의 데이터베이스에 분산 저장하는 기법

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/188273992-8eaf9d3f-378e-4f7e-bf4f-f43e8ac92950.png">
</p>

<br />

#### <span style="color:red">(2)</span> LRUCache : LRUCache의 각 Item들은 LRUHandle이며 이는 <span style="color:red">(3)</span> LRUHandle 에서 후술한다. 
#### 또한 LRUCache는 1개의 HandleTable(HashTable)과 두개의 dobble linked list로 관리되며 LRUHandle들은 이중 연결 목록과 해시 테이블 HandleTable에서 모두 관리된다.

<br/>

아래 3 요소는 LRUCache를 이루는 두 Double Linked List와 HashTable이다.


    LRUHandle lru_; // (5)에서 후술

    LRUHandle in_use_; // (6)에서 후술

    HandleTable table_; // (4)에서 후술

<br />
<br />

#### <span style="color:red">(3)</span> LRUHandle : LRUCache의 객체이다.

+HandleTable(HashTable)은 double linked list과 결합된다.
HandleTable은 빠른 검색을 구현하고 double linked list는 빠른 추가 및 삭제를 구현한다.

<br />

LRUHandle의 구성요소:

    struct LRUHandle {
    void* value;
    void (*deleter)(const Slice&, void* value); // 키, 값 공간을 해제하기 위한 사용자 콜백
    LRUHandle* next_hash; // 해시 충돌을 처리하고 배열에 속하는 객체(LRUHandle)을 유지
    LRUHandle* next; // 이중 연결 목록에서 LRU 순서를 유지하는 데 사용
    LRUHandle* prev; // 이중 연결 목록에서 이전 노드를 가르킴
    size_t charge;  
    size_t key_length; // 키 값의 길이
    bool in_cache; // 핸들이 캐시 테이블에 있는지 여부 
    uint32_t refs; // 핸들이 참조된 횟수     
    uint32_t hash; // 샤딩 및 빠른 비교를 결정하는 데 사용되는 키의 해시 값    
    char key_data[1]; // 키의 시작

<br/>

#### <span style="color:red">(4)</span> HandleTable : 구현은 매우 간단한 해시 테이블로 되어있다.

<p align="center">
<img width="300" alt="image" src="https://user-images.githubusercontent.com/84978165/188291092-8b674772-0cb1-4fb8-abcc-e6b75164b9bc.png">
</p>

코드상에서 list_라는 이중 포인터를 선언하여 LRUHandle 객체를 관리한다.

    uint32_t length_; // LRUHandle* 배열의 길이이다.
    uint32_t elems_; // 테이블의 노드 수
    LRUHandle** list_;

아래는 LRUHandle** list_에 대한 LRUHandle* list_를 그림으로 나타낸 것이다.

<p align="center">
<img width="300" alt="image" src="https://user-images.githubusercontent.com/84978165/188284416-8d24b765-0996-4af6-ac16-97fcea285223.png">
</p>

#### *위의 LRUHandle과 같은 그림이다.

아래는 클래스 HandleTable에 대한 LRUHandle 메소드들의 정의이다.

    LRUHandle** FindPointer(const Slice& key, uint32_t hash) 
    // next_hash를 사용하여 key에 해당하는 항목을 찾을 때까지 double linked list들을 탐색
    // 일치하는 항목이 없다면 next_hash는 list_의 끝을 가르키고 next_hash의 값은 nullptr임
    // 일치하는 항목을 찾았다면 next_hash가 가르키는 이중 포인터(LRUHandle)을 반환
    // 위의 특성에 의해 Lookup, Insert, Remove의 첫줄에서 쓰인다. 

    LRUHandle* Lookup(const Slice& key, uint32_t hash)
    // FindPointer를 호출하여 포인터가 가르키는 값(LRUHandle)을 직접 읽음

    LRUHandle* Insert(LRUHandle* h)
    // FindPointer를 호출하여 포인터(next_hash)가 가르키는 위치에 LRUHandle을 집어넣음

    LRUHandle* Remove(const Slice& key, uint32_t hash)
    // FindPointer를 호출하여 포인터(next_hash)가 가르키는 항목을 직접 삭제함

    void Resize() 
    // 요소 즉, LRUHandle이 증가함에 따라 배열을 동적으로 확장함

<br/>
<br/>

<span style="color:red">(5)</span>, <span style="color:red">(6)</span> 각각은 LRU Cache를 구성하는 자료구조인 double linked list이며 hot과 cold의 분리를 보장하고 내부적으로 HashTable을 유지한다.

  +여기서 hot은 자주 접근되는 데이터, cold는 접근되지 않는 데이터를 칭한다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/188290373-5c1348b6-5bc7-4eea-895e-3b53b68c7b2f.png">
</p>

#### <span style="color:red">(5)</span> in_use_ : hot linked list이며 사용을 위해 호출자에게 반환되는 캐시의 캐시 객체를 유지 관리한다.

#### <span style="color:red">(6)</span> lru_ : cold linked list이며 캐시에서 캐시된 개체의 인기도를 유지한다. 

#### 데이터에 액세스할 때마다 이 list의 최신 위치에 삽입된다.(중요, 데이터는 항상 lru_ list에 먼저 삽입)

#### lru_->next는 가장 오래된 데이터를 가리키고 lru_->prev는 최신 데이터를 가리킨다. 캐시가 차지하는 메모리 제한을 초과할 경우 lru_->next부터 데이터가 정리되며 LRU정책을 확인할 수 있다.


---
## 4. Overrall Cache Flow

**_<span style="background-color:#808080">LevelDB에서 Cache의 메커니즘_**

LevelDB에서 Cache는 읽기 작업과 함께 Cache에 대한 메커니즘이 확인이 가능하다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/188290416-f01c0e2a-ddb1-4eae-b10f-8055b7df8d5c.png">
</p>

<br/>

* 데이터를 읽을때 호출되는 함수 Get을 통해 ReadOptions()이 호출된다.

      leveldb::Status s = db -> Get(leveldb::ReadOptions(),key,&value);

<br/>

* DBimpl:Get의 부분 소스코드:

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

#### <1> memtable에서 table을 찾기 위해 memtable mem_ 선언

#### <2>  memtable에 없다면 immutable memtable에서 table을 찾기 위해 memtable  imm_ 선언

#### <3>  둘 다 발견하지 못 할 경우 sstable에서 찾기 위해 현재 version을 가져오기 위하여 version current 선언

#### <4> memtable에서 디스크 파일 검색을 수행하는 Memtable::Get호출

#### <5> 마찬가지로 immutable memtable에서 디스크 파일 검색을 수행하기 위하여 Memtable::Get호출

#### <6> sstable에서 디스크 파일 검색을 수행하기 위하여 Version::Get 호출

<br/>
<br/>

이후 TableCache::Get 호출 -> **_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

---

**_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

* TableCache::Get, 본격적인 Cache를 이용한 읽기 작업을 수행하는 부분이다.

TableCache::Get:

    Cache::Handle* handle = nullptr;

    <1> Status s = FindTable(file_number, file_size, &handle);
    
    if (s.ok()) {
      <2> 
      Table* t = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;

      s = t->InternalGet(options, k, arg, handle_result);

      cache_->Release(handle);
    }

    return s;

<br>

+LevelDB에는 TableCache와 BlockCache 두 가지의 다른 캐시가 도입되어 있다.


#### TableCache 이용(FindTable) : 파일에 해당하는 테이블을 캐시에서 찾는다. 발견되지 않을 경우 파일을 먼저 오픈한 후 파일에 해당하는 테이블을 생성하여 캐시에 추가한다.

아래는 TableCache의 구조이다.


<p align="center">
<img width="896" alt="image" src="https://user-images.githubusercontent.com/84978165/188290917-4f120d0d-0132-46f5-ad36-3c37b4c62769.png">
</p>

<br>
<br>

부가 설명


<br>
<br>

#### BlockCache 이용(InternalGet -> BlockReader) : SSTable에서 Key를 찾는 논리를 구현하는 InternalGet 함수는 BlockReader를 호출한다. 
#### 블록 캐시에서 블록을 찾고 발견되지 않을 경우 파일을 오픈한 후 읽어들인 뒤 블록 캐시에 블록을 추가한다.

* BlockReader의 Cache 사용 알고리즘:

  1. 블록 캐시에서 직접 블록가져오기를 시도함

  2. 블록캐시에 없으면 ReadBlock을 호출하여 파일에서 읽음

  3. 읽기에 성공하면 블록 캐시에 블록을 추가함


아래는 BlockCache의 구조이다.

<p align="center">
<img width="694" alt="image" src="https://user-images.githubusercontent.com/84978165/188290995-f6730c28-1a67-4aaa-b7b0-667a6c5025e8.png">
</p>

<br>
<br>

부가 설명

<br>
<br>


---

**_<span style="background-color:#808080">LevelDB에서 Cache의 생성_**

TableCache는 ./db_bench의 option중 하나인 cache_size의 크기를 받아 TableCache::TableCacheSize함수를 통해 크기를 설정한 뒤 TableCache::TableCache를 통해 생성한다.

TableCache's BackTrace:

    leveldb::Benchmark::Run
    leveldb::Benchmark::Open 
    leveldb::DB::Open
    leveldb::DBImpl::DBImpl
    leveldb::TableCache::TableCache

---

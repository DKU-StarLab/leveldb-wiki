# LevelDB Cache Analysis

### <center> Contents 
  - [1. What is Cache](#1-what-is-cache)
  - [2. What is LRU](#2-what-is-lru)
  - [3. LRU Cache Structure in LevelDB](#3-lru-cache-structure-in-leveldb)
  - [4. Overrall Cache Flow](#4-overrall-cache-flow)
  - [5. Cache Source Code Analysis](#5-cache-source-code-analysis)
---
## 1. What is Cache

 캐시는 컴퓨팅 환경에서 일시적으로 무언가 데이터를 저장하는데 사용되는 하드웨어 또는 소프트웨어이며 최근 또는 자주 액세스하는 데이터의 성능을 향상시키는데 더 빠르게 사용되는 더 빠르고 더 비싼 소량의 메모리이다.

캐시는 여러가지 종류를 가지고 있다.

(1) CPU 캐시 : CPU 옆에 탑재한 메모리

(2) 디스크 캐시 : 하드 디스크가 디스크 제어와 외부와의 인터페이스를 위해 내장한 컴퓨터

(3) 기타 캐시 : 우리가 LevelDB의 캐시에서 파악해야 할 캐시이다. 위의 두 캐시 밖의 다른 캐시들은 대개 소프트웨어적으로 관리되며 예로는 운영체제의 메인 메모리를 하드 디스크에 복사해 놓는 페이지 캐시등이 있다.


---
## 2. What is LRU
  LRU(Least Recently Used) 정책이란 페이지 교체 알고리즘으로 불린다.

  약자를 직역해 보았을 때 의미는 가장 최근에 사용하지 않은 것이며 말 그대로 페이지에서 데이터를 제거할 때 가장 오랫동안 사용하지 않은 것을 제거하겠다는 알고리즘이다.

  이 알고리즘의 기본 가설은 가장 오랫동안 사용하지 않았던 데이터는 앞으로도 사용할 가능성이 적다는 것이다.

그렇다면 LRU Cache는 캐시에 공간이 부족할 때 가장 오랫동안 사용하지 않은 항목을 제거하고 새로운 노드를 배치하는 형식으로 동작됨을 알 수 있다.

LRU Cache의 구현은 Double Linked List를 통해 이루어질 수 있다.

아래는 LRU Cache를 설명하는 그림이다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189130049-634c1d81-5b49-49b5-bfea-3621a82c6687.png">
</p>

기본 전제는 Head(맨 오른쪽 노드)에 가까운 데이터일 수록 최근에 사용한 데이터이고 Tail(맨 왼쪽 노드)에 가까울 수록 오랫동안 사용하지 않은 데이터로 간주한다.

그리고 만약 캐시에 적재된 데이터를 사용하는 경우 해당 데이터를 Head(맨 오른쪽 노드)로 옮겨 가장 최근에 사용된 값임을 명시하며 삭제 우선순위에서 멀어지게 한다.

읽는 데이터의 순서는 A, B, C, D, E, D, F이다.

1. 괄호 안의 숫자는 정렬을 나타내며 숫자가 작을수록 맨 왼쪽 즉, Tail에 데이터가 담긴다.

2. 읽는 순서에 따라 E를 읽는 순서가 다가왔을 때 캐시가 가득 찬 것으로 확인되므로 가장 오랫동안 사용하지 않은 데이터인 T즉, Least Recently Used인 A가 제거된다.

3. 계속해서 D, F를 읽으며 정렬을 갱신하며 방문 횟수가 가장 적은 노드가 제거 및 대체가 반복되는 모습을 확인할 수 있다.

<br>
<br>

LevelDB는 LRU 캐시를 구현하기 위해서 두가지의 다른 데이터 자료 구조를 도입하여 사용하고 있다.

#### 1. HashTable(HandleTable) : 조회 구현에 사용
#### 2. Double linked list : 오래된 데이터를 제거하는데 사용되는 정렬


  
---

## 3. LRU Cache Structure in LevelDB

LevelDB의 LRU Cache 구조를 정확하게 파악하기 위하여 git clone을 통해 LevelDB소스코드를 받아와 분석하였다.

(source code download)
  
    git clone --recurse-submodules https://github.com/google/leveldb.git leveldb_release


<br>
<br>
<br>

아래는 전체 LRU Cache 구조의 개략도이다.

[정확한 코드의 분석은 leveldb_release/build/util/cache.cc 참고]



<p align="center">
<img width="800" alt="image" src="https://user-images.githubusercontent.com/84978165/189133099-0140c4df-884a-4591-870d-a5e92be39df5.png">

<br />

#### <span style="color:red">(1)</span> ShardedLRUCache : LevelDB내에서 ShardedLRUCache는 내부에 LRU Cache 16개를 구성하고 있으며 외부 LRUCache로 정의한다.

<br>

### +Sharding
#### 먼저 Shard란 조각, 파편이라는 의미를 담고 있는 단어이며 Sharding은 데이터를 조각으로 나누어 관리하는 일종의 로드밸런싱 기술이다. 

#### 로드 밸런싱이란 작업 요청을 여러 개의 서버나 처리 장치에 분산하여 나누는 것을 의미한다.하나의 큰 데이터베이스나 네트워크 시스템을 여러 개의 작은 조각으로 나누어 분산 저장하여 관리하는 것을 말한다.


아래는 테이블의 각 행을 다른 테이블에 분산시켜 데이터를 분할 즉, 데이터베이스를 Sharding 시키는 예를 그린 그림이다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189147668-2f5907b4-7529-427a-8176-7d4d9a55223d.png">
</p>

<br />

LevelDB에서 사용하는 LRU Cache는 Shard기법을 통해 ShardedLRUCache가 내부적으로 LRU Cache 배열을 유지하고 있다. 

*ShardedLRUCache에서 LRU Cache들을 호출할 때 각각에 대해서 상호 배제 잠금(Lock)을 수행하는데 이는 동일한 LRU Cache 개체에 대한 여러 스레드의 경합을 줄여 다중 스레드 조회 속도를 높이기 위함이다.*

<br />

#### <span style="color:red">(2)</span> LRUCache : LRUCache의 각 Item들은 LRUHandle이며 이는 *(3) LRUHandle* 에서 후술한다. 

#### 또한 LRUCache는 1개의 HandleTable(HashTable)과 두개의 dobble linked list로 관리되며 LRUHandle들은 이중 연결 목록과 해시 테이블 HandleTable에서 모두 관리된다.

<br/>

아래 3 요소는 Cache를 이루는 두 Double Linked List와 HashTable이다.


    LRUHandle lru_; // will be described (5).

    LRUHandle in_use_; // will be described (6).

    HandleTable table_; // will be described (4).

<br />
<br />

#### <span style="color:red">(3)</span> LRUHandle : LRUCache의 객체이다.

+HandleTable(HashTable)은 double linked list과 결합된다.
HandleTable은 빠른 검색을 구현하고 double linked list는 빠른 추가 및 삭제를 구현한다.

<br />

LRUHandle의 구성요소:

    struct LRUHandle {
    void* value;
    void (*deleter)(const Slice&, void* value); // to free the key and value space
    LRUHandle* next_hash; // handle hash collisions and keep objects(LRUHandle) belonging to the array
    LRUHandle* next; // used to maintain LRU order in doubly linked list
    LRUHandle* prev; // points to the previous node in a doubly linked list
    size_t charge;  
    size_t key_length; // length of key value
    bool in_cache; // whether the handle is in the cache table
    uint32_t refs; // the number of times the handle was referenced    
    uint32_t hash; // hash value of key used to determine sharding and fast comparison    
    char key_data[1]; // start of key

<br/>

#### <span style="color:red">(4)</span> HandleTable : 구현은 매우 간단한 해시 테이블로 되어있다.

<p align="center">
<img width="300" alt="image" src="https://user-images.githubusercontent.com/84978165/189152590-843477ba-efbd-41e6-a00e-6e0ed2e28062.png">
</p>

코드상에서 list_라는 이중 포인터를 선언하여 LRUHandle 객체를 관리한다.

    uint32_t length_; // length of LRUHandle* array
    uint32_t elems_; // number of nodes in the table
    LRUHandle** list_;

아래는 LRUHandle** list_에 대한 LRUHandle* list_를 그림으로 나타낸 것이다.

<p align="center">
<img width="300" alt="image" src="https://user-images.githubusercontent.com/84978165/189152971-8c8aabfd-4bea-4a50-a3f1-60e0e028aa96.png">
</p>

#### *위의 LRUHandle과 같은 그림이다.

아래는 클래스 HandleTable에 대한 LRUHandle 메소드들의 정의이다.

    LRUHandle** FindPointer(const Slice& key, uint32_t hash) 
    // use next_hash to traverse the double linked lists until find the item corresponding to the key.
    // if no match is found, next_hash points to the end of list_ and the value of next_hash is nullptr.
    // if a match is found, a double pointer (LRUHandle) pointed to by next_hash is returned.
    // due to the above characteristics, it is used in the first line of Lookup, Insert, and Remove Funtion.

    LRUHandle* Lookup(const Slice& key, uint32_t hash)
    // directly read the value pointed to by the pointer (LRUHandle) by calling FindPointer

    LRUHandle* Insert(LRUHandle* h)
    // call FindPointer to put LRUHandle at the location pointed to by the pointer (next_hash).

    LRUHandle* Remove(const Slice& key, uint32_t hash)
    // directly deletes the item pointed to by the pointer (next_hash) by calling FindPointer

    void Resize() 
    // dynamically expand array as LRUHandle grows

<br/>
<br/>

<span style="color:red">(5)</span>, <span style="color:red">(6)</span> 각각은 LRU Cache를 구성하는 자료구조인 double linked list이며 hot과 cold의 분리를 보장하고 내부적으로 HashTable을 유지한다.

  +여기서 hot은 자주 접근되는 데이터, cold는 접근되지 않는 데이터를 칭한다.

<p align="center">
<img width="400" alt="image" src="https://user-images.githubusercontent.com/84978165/189153724-0f2ab665-8ffb-4cc1-a986-e0425560178e.png">
</p>

#### <span style="color:red">(5)</span> in_use_ : hot linked list이며 사용을 위해 호출자에게 반환되는 캐시의 캐시 객체를 유지 관리한다.

#### <span style="color:red">(6)</span> lru_ : cold linked list이며 캐시에서 캐시된 개체의 인기도를 유지한다.

#### 위 두 double linked list는 refs라는 변수에 의해 관리되며 외부 사용자에 의해 참조되고 있는 캐시를 다루는 in_use_는 refs가 2이며 참조되지 않는(사용되지 않는) lru_는 refs가 1이다.

*여기서 lru_, in_use_는 꼭두각시 노드이며 데이터 자체를 저장하지 않는다.*

#### 데이터에 액세스할 때마다 이 list의 최신 위치에 삽입된다.(중요, 데이터는 항상 lru_ list에 먼저 삽입)

#### lru_->next(다음 포인터)는 가장 오래된 데이터를 가리키고 lru_->prev(이전 포인터)는 최신 데이터를 가리킨다. 새로운 Handle이 삽입될 때마다 Double linked list의 끝에 삽입되며 lru_->prev(이전 포인터)는 새로 삽입된 Handle을 가리키도록 변경된다.

#### 사용자에 의하여 호출되는 캐시는 lru_에서 next_hash를 통해 in_use_로 넘어가며 refs가 1에서 2로 증가한다.


---
## 4. Overrall Cache Flow

**_<span style="background-color:#808080">LevelDB에서 Cache의 메커니즘_**

LevelDB에서 Cache는 읽기 작업과 함께 Cache에 대한 메커니즘이 확인이 가능하며 아래는 그에 대한 개략도이다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189153950-24b00d7d-a39b-44ac-b2f6-4defe5a08299.png">
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

Version::Get함수에서 Table을 읽기 위해 TableCache::Get을 호출 -> **_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

---

**_<span style="background-color:#808080">LevelDB에서 Cache의 이용_**

* TableCache::Get, 본격적인 Cache를 이용한 읽기 작업을 수행하는 부분이다.

TableCache::Get 소스코드:

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


#### <1> TableCache 이용(FindTable) : 파일에 해당하는 테이블을 캐시에서 찾는다. 발견되지 않을 경우 파일을 먼저 오픈한 후 파일에 해당하는 테이블을 생성하여 캐시에 추가한다.

아래는 TableCache의 구조이다.


<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189172319-30682e1a-6f1b-4b1d-8b15-5e6dce460180.png">
</p>

*key*값은 file_number로 SSTable의 파일 이름이다.
 
*value*값은 두 섹션으로 나뉘어진다.

*RandomAccessFile**: 디스크에서 열린 SSTable에 대한 포인터이다.

*Table**: 메모리에 있는 SSTable 파일에 해당하는 Table 구조에 대한 포인터로, 메모리에 있는 테이블 구조로 SSTable의 인덱스 내용과 블록 캐시를 나타내는 cache_id를 저장한다.


<br>
<br>

#### <2> BlockCache 이용(InternalGet -> BlockReader) : SSTable에서 Key를 찾는 논리를 구현하는 InternalGet 함수는 BlockReader를 호출한다. 
#### 블록 캐시에서 블록을 찾고 발견되지 않을 경우 파일을 오픈한 후 읽어들인 뒤 블록 캐시에 블록을 추가한다.

* BlockReader의 Cache 사용 알고리즘:

  1. 블록 캐시에서 직접 블록가져오기를 시도함

  2. 블록캐시에 없으면 ReadBlock을 호출하여 파일에서 읽음

  3. 읽기에 성공하면 블록 캐시에 블록을 추가함


아래는 BlockCache의 구조이다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189172561-b52b24a4-dfb5-4bb4-b0d4-69ca0963380e.png">
</p>

*key*값은 각 SSTable 파일의 offset에 고유한 cache_id를 조합하여 구성되어 있다.

이는 다른 각각의 SSTable 파일의 block_offset이 동일할 수 있으므로 구별하기 위함이다.

<br>

*value*값은 열린 SSTable 파일의 Data block들로 구성되어 있다.

<br>
<br>
<br>
<br>

BlockReader함수를 통해 Block Cache에 Lookup하고 Insert하는 작업이 끝나고 나면 읽기 작업을 통해 Cache를 이용하는 작업이 끝나게 된다. 

아래의 Cache 이용 개략도를 통해 위에서 쭉 설명한 LevelDB에서 Cache 이용의 흐름을 다시 파악할 수 있다.

<p align="center">
<img width="700" alt="image" src="https://user-images.githubusercontent.com/84978165/189153950-24b00d7d-a39b-44ac-b2f6-4defe5a08299.png">
</p>

---

**_<span style="background-color:#808080">LevelDB에서 Cache의 생성_**

TableCache는 ./db_bench의 option중 하나인 cache_size의 크기를 받아 TableCache::TableCacheSize함수를 통해 크기를 설정한 뒤 TableCache::TableCache를 통해 생성한다.

TableCache::TableCache의 BackTrace:

    leveldb::Benchmark::Run
    leveldb::Benchmark::Open 
    leveldb::DB::Open
    leveldb::DBImpl::DBImpl
    leveldb::TableCache::TableCache

---

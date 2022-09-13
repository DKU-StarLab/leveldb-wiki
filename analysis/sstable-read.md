# SSTable - Read
LevelDB에서 Get Operation을 통해 원하는 key의 value를 찾을 때, 어떤 과정을 거쳐서 key를 찾는지, 그 중에서 storage에 저장되는 SSTable에서 어떤 과정을 거쳐 찾는지에 대해 Top-Down 방식으로 알아본다.  
<br/>

## LevelDB Get Operation
LevelDB에선 다음과 같은 순서로 원하는 key를 찾는다.  

1. MemTable에서 탐색
2. 없다면, Immutable MemTable에서 탐색
3. 없다면, storage(disk)에서 탐색  
<br/>

`DBImpl::Get`에서 다음과 같이 탐색해나가는 것을 볼 수 있다.  

```cpp
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                   std::string* value) {
  // ...생략

  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  
  // ...생략
  {
    mutex_.Unlock();
    LookupKey lkey(key, snapshot);
    // 1. MemTable에서 탐색
    if (mem->Get(lkey, value, &s)) {
    // 2. 없다면, Immutable MemTable에서 탐색
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {
    // 3. 없다면, storage(disk)에서 탐색
    } else {
      s = current->Get(options, lkey, value, &stats);
      have_stat_update = true;
    }
    mutex_.Lock();
  }
  // ...생략
}
```  
<br/>

## Storage에서 target key를 찾아가는 과정
즉 storage에서 target key를 찾는 과정은 `Version::Get`으로부터 시작하며, 다음과 같은 과정으로 target key를 찾아간다.  

1. 각 Level에서 target key가 있을 만한 SSTable들을 골라낸다
2. 골라낸 SSTable로부터 target key를 찾는다.  
<br/>

`Version::Get`에서 다음과 같이 탐색해나가는 것을 볼 수 있다.

```cpp
Status Version::Get(const ReadOptions& options, const LookupKey& k,
                    std::string* value, GetStats* stats) {
  // ...생략

  struct State {
    // ...생략

    static bool Match(void* arg, int level, FileMetaData* f) {
      // ...생략
      // 2. 골라낸 SSTable로부터 target key를 찾는다.
      state->s = state->vset->table_cache_->Get(*state->options, f->number,
                                                f->file_size, state->ikey,
                                                &state->saver, SaveValue);
      // ...생략
    }
  };

  // ...생략
  // 1. 각 Level에서 target key가 있을 만한 SSTable들을 골라낸다
  ForEachOverlapping(state.saver.user_key, state.ikey, &state, &State::Match);

  return state.found ? state.s : Status::NotFound(Slice());
}
```

> *`ForEachOverlapping`을 호출할 때 `Match`를 인자로 넘겨주고, `ForEachOverlapping`에서는 골라낸 SSTable들에 대해 인자로 받은 `Match`를 수행해줌으로써 골라낸 SSTable로부터 target key를 찾는 과정을 수행하게 된다.*

### Version::ForEachOverlapping  
> *각 Level에서 target key가 있을 만한 SSTable들을 골라낸다*  

- Level 0 : Level 0의 SSTable들은 key range가 겹칠 수 있다. 따라서 Linear Search로 각 SSTable들을 하나하나 판단한다
- Ohter Levels : Level 0 이외의 각 Level에선 SSTable의 key range가 분리되어 있다. 따라서 Binary Search로 target key가 있을 만한 SSTable을 빠르게 검색한다  
<br/>

```cpp
void Version::ForEachOverlapping(Slice user_key, Slice internal_key, void* arg,
                                 bool (*func)(void*, int, FileMetaData*)) {

  const Comparator* ucmp = vset_->icmp_.user_comparator();
  std::vector<FileMetaData*> tmp;
  tmp.reserve(files_[0].size());
  // Level 0: Linear Search로 SSTable들을 골라낸다
  for (uint32_t i = 0; i < files_[0].size(); i++) {
    FileMetaData* f = files_[0][i];
    if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
        ucmp->Compare(user_key, f->largest.user_key()) <= 0) {
      tmp.push_back(f);
    }
  }
  if (!tmp.empty()) {
    std::sort(tmp.begin(), tmp.end(), NewestFirst);
    for (uint32_t i = 0; i < tmp.size(); i++) {
      // 파라미터로 받은 함수를 수행
      if (!(*func)(arg, 0, tmp[i])) return;
    }
  }

  // Level 0 이외의 Level에서의 탐색
  for (int level = 1; level < config::kNumLevels; level++) {
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // FindFile : Binary search로 target key가 있을 만한 SSTable의 인덱스를 얻어옴
    uint32_t index = FindFile(vset_->icmp_, files_[level], internal_key);
    if (index < num_files) {
      FileMetaData* f = files_[level][index];
      if (ucmp->Compare(user_key, f->smallest.user_key()) < 0) {

      } else {
        // 파라미터로 받은 함수를 수행
        if (!(*func)(arg, level, f)) return;
      }
    }
  }
}
```  
<br/>
   
## 골라낸 SSTable로부터 target key를 찾는 과정  
`TableCache::Get`으로부터 시작하며, 다음과 같은 과정을 거쳐 target key를 찾는다.

1. 해당 SSTable 개체가 기존에 이미 캐싱됐는지 살피고, 캐싱되지 않았다면 해당 SSTable 개체를 캐싱한다.
2. 해당 SSTable 내부를 탐색해 target key를 찾는다.  
<br/>

`TableCache::Get`에서 다음과 같이 탐색해나가는 것을 볼 수 있다.  

```cpp
Status TableCache::Get(const ReadOptions& options, uint64_t file_number,
                       uint64_t file_size, const Slice& k, void* arg,
                       void (*handle_result)(void*, const Slice&,
                                             const Slice&)) {
  Cache::Handle* handle = nullptr;
  // 1. 해당 SSTable 개체가 기존에 이미 캐싱됐는지 살피고, 캐싱되지 않았다면 해당 SSTable 개체를 캐싱한다.
  Status s = FindTable(file_number, file_size, &handle);
  if (s.ok()) {
    Table* t = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;
    // 2. 해당 SSTable 내부를 탐색해 target key를 찾는다.
    s = t->InternalGet(options, k, arg, handle_result);
    cache_->Release(handle);
  }
  return s;
}
```  
> `TableCache::FindTable`이 수행되면서 `Table::Open`이란 메소드가 수행되는데, 이로 인해 해당 SSTable의 Index Block과 Filter Block이 메모리로 로드된다.  
 
<br/>

## SSTable 내부를 탐색해 target key를 찾는 과정  
`Table::InternalGet`으로부터 시작하며, 다음과 같은 과정을 거쳐 target key를 찾는다.  

1. Index Block를 탐색해 target key가 있을 만한 Data Block을 추려낸다
2. 블룸필터를 사용할 경우, 추려낸 Data Block에 target key가 있는지 블룸필터로 조사한다
3. target key가 있다고 판단되면, 추려낸 Data Block에 대한 Iterator를 만든다
4. 만든 Iterator를 활용해 Data Block를 탐색한다
5. target key를 찾았다면 그 value를 저장한다  
<br/>

   
`Table::InternalGet`에서 다음과 같이 탐색해나가는 것을 볼 수 있다.  

```cpp
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg,
                          void (*handle_result)(void*, const Slice&,
                                                const Slice&)) {
  Status s;
  // Index Block에 대한 Iterator를 만든다
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  // 1. Index Block를 탐색해 target key가 있을 만한 Data Block을 추려낸다
  iiter->Seek(k);
  if (iiter->Valid()) {
    // ...생략

    // 2. 블룸필터를 사용할 경우, 추려낸 Data Block에 target key가 있는지 블룸필터로 조사한다
    if (filter != nullptr && handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {
      // Not found
    } else {
      // 3. target key가 있다고 판단되면, 추려낸 Data Block에 대한 Iterator를 만든다
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      // 4. 만든 Iterator를 활용해 Data Block를 탐색한다
      block_iter->Seek(k);
      // 5. target key를 찾았다면 그 value를 저장한다
      if (block_iter->Valid()) {
        (*handle_result)(arg, block_iter->key(), block_iter->value());
      }
      // ...생략
    }
  }
  // ...생략
}
```  
<br/>  

### Table::BlockReader  
> *Index Block Iterator가 가리키는 entry가 참조하는 Data Block에 대한 Iterator를 만들어 반환한다*  

1. `Lookup`메소드를 통해 해당하는 Data Block이 기존에 이미 캐싱됐는지 본다
2. 캐싱되지 않았다면 해당하는 Data Block을 캐싱한다  
   1) `ReadBlock`으로 해당하는 Data Block의 내용을 읽는다  
   2) 읽은 내용을 담아 새 Block객체를 만든다(즉 해당하는 Data Block을 메모리로 로드하는 것)
   3) 로드한 Data Block을 Cache에 넣는다
3. 그 Data Block에 대한 Iterator를 만든다  
   
> *만약 Cache를 쓰지 않는다면 `ReadBlock`으로 해당하는 Data Block의 내용을 읽고 이를 메모리로 로드만 한다*  
<br/>

`Table::BlockReader`에서 다음과 같이 수행되는 것을 볼 수 있다.  

```cpp
Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                             const Slice& index_value) {
  // ...생략

  if (s.ok()) {
    BlockContents contents;
    if (block_cache != nullptr) {
      // ...생략

      // 1. Lookup메소드를 통해 해당하는 Data Block이 기존에 이미 캐싱됐는지 본다
      cache_handle = block_cache->Lookup(key);
      if (cache_handle != nullptr) {
        block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
      } else {
        // 2. 캐싱되지 않았다면 해당하는 Data Block을 캐싱한다
        // 2-1. ReadBlock으로 해당하는 Data Block의 내용을 읽는다
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          // 2-2. 읽은 내용을 담아 새 Block객체를 만든다(즉 해당하는 Data Block을 메모리로 로드하는 것)
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {
            // 2-3. 로드한 Data Block을 Cache에 넣는다
            cache_handle = block_cache->Insert(key, block, block->size(),
                                               &DeleteCachedBlock);
          }
        }
      }
    } else {
      // 만약 Cache를 쓰지 않는다면 해당하는 Data Block의 내용을 메모리로 로드만 한다
      s = ReadBlock(table->rep_->file, options, handle, &contents);
      if (s.ok()) {
        block = new Block(contents);
      }
    }
  }

  // 3. 그 Data Block에 대한 Iterator를 만든다
  Iterator* iter;
  if (block != nullptr) {
    iter = block->NewIterator(table->rep_->options.comparator);

    // ...생략
  } else {
    iter = NewErrorIterator(s);
  }
  return iter;
}
```  
   

### Block::Iter::Seek
> *Block내에서 인자로 받은 target을 찾는다*

1. target이 있을 만한 구역을 Binary Search로 찾는다
2. Linear Search로 찾은 구역 내에서 target을 찾는다  
<br/>  

`Block::Iter::Seek`에서 다음과 같이 탐색하는 것을 볼 수 있다. 

```cpp
void Seek(const Slice& target) override {
    // ...생략

    // 1. target이 있을 만한 구역을 Binary Search로 찾는다
    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      
      // ...생략
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // "mid"의 key값이 "target"보다 작은 경우
        left = mid;
      } else {
        // "mid"의 key값이 "target"보다 크거나 같을 경우
        right = mid - 1;
      }
    }

    // ...생략

    // 2. Linear Search로 찾은 구역 내에서 target을 찾는다 
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }
```  
<br/>

기술한 내용을 토대로 SSTable내부를 탐색해 target key를 찾는 과정을 좀 더 구체적으로 다시 설명하자면 다음과 같다.  

<p align="center">
   <img src = "https://user-images.githubusercontent.com/65762283/187970494-255dbac9-d76f-46a0-8ff9-3061506ae5a9.png">
</p>  

1. `NewIterator` : Index Block에 대한 Iterator를 만든다
2. `Seek` : 생성한 Iterator를 이용해 Index Block내부를 뒤져 target key가 존재할 만한 Data Block를 파악한다
3. `KeyMayMatch` : (만약 블룸필터를 쓴다면)블룸필터를 이용해 target key가 해당 Data Block에 있는지 조사한다
4. `BlockReader` : 있다면, 해당 Data Block에 대한 Iterator를 만든다
5. `Seek` : 생성한 Iterator를 이용해 해당 Data Block내부를 뒤져 target key를 찾는다.
6. `SaveValue` : target key를 찾았다면 value를 저장한다.  
<br/>

## Summary - Storage에서 target key를 찾아가는 과정 요약  

<p align="center">
   <img src = "https://user-images.githubusercontent.com/65762283/187468693-a1819b3e-8c09-4cff-828d-3e9f6707b340.png">
</p>  

1. 각 Level에서 target key가 있을 만한 SSTable들을 고른다
2. 고른 각각의 SSTable 개체들이 이미 캐싱됐는지 보고 안 됐다면 캐싱해준다
   - 이 과정에서 해당 SSTable의 Index Block과 Filter Block이 메모리로 로드된다
3. Index Block을 뒤져 target key가 있을 만한 Data Block을 파악한다
4. Filter Block의 블룸필터를 이용해 해당 Data Block에 target key가 있는지 조사한다
5. 있다면, 해당 Data Block이 이미 캐싱됐는지 보고 안 됐다면 캐싱해준다
   - 이 과정에서 해당 Data Block이 메모리로 로드된다
6. 해당 Data Block을 뒤져 target key를 찾는다

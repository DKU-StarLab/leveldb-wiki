# SSTable - Write
SSTable은 다음과 같은 상황에서 만들어진다.  

1. MemTable로부터 `Flush(Minor Compaction)`가 일어날 때
2. Storage에서 `Compaction`이 일어날 때  

이 중 MemTable로부터 `Flush`가 일어날 때에 초점을 맞춰 SSTable에 어떻게 만들어지는지 다룬다.  
<br>  

### MemTable로부터 Flush가 일어날 때  
이 때 MemTable로부터 `Flush`가 일어나는 과정은 다음과 같다.

<p align="center"><img src="https://user-images.githubusercontent.com/65762283/187967954-c9b740dc-18c6-4def-8170-bb29d3bb8809.png"></p>  

`CompactMemTable`이 호출되고, 이로 인해 `WriteLevel0Table`이 호출되면서 SSTable이 만들어지는데 이 때 `BuildTable`이 실질적으로 SSTable을 만든다.  

`BuildTable`의 흐름은 다음과 같다.

#### 전체 과정
![Sequence 01](https://user-images.githubusercontent.com/65762283/183480267-03e4a024-06b0-4798-83d0-74c50a5d0f1a.gif) 

#### 순서
1. `TableBuilder` 인스턴스를 만든다  
2. `TableBuilder`의 `Add`메소드를 통해 MemTable의 key-value pair들을 하나하나 추가한다.
3. `TableBuilder`의 `Finish`메소드를 통해 SSTable을 만드는 과정을 마무리한다.
4. `WritableFile`에 있는 내용들을 storage에 쓴다
5. storage에 저장한 SSTable을 cache에 올려서 사용가능한지 확인해본다.

```cpp
Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
  Status s;
  meta->file_size = 0;

  // 이터레이터가 첫 요소를 가리키도록 함
  iter->SeekToFirst();

  std::string fname = TableFileName(dbname, meta->number);
  if (iter->Valid()) {
    WritableFile* file;
    s = env->NewWritableFile(fname, &file);
    if (!s.ok()) {
      return s;
    }

    // 1. TableBuilder 인스턴스를 만든다
    TableBuilder* builder = new TableBuilder(options, file);
    meta->smallest.DecodeFrom(iter->key());
    Slice key;

    // 2. TableBuilder의 Add메소드를 통해 MemTable의 key-value pair들을 하나하나 추가한다.
    for (; iter->Valid(); iter->Next()) {
      key = iter->key();
      builder->Add(key, iter->value());
    }
    if (!key.empty()) {
      meta->largest.DecodeFrom(key);
    }

    // 3. TableBuilder의 Finish메소드를 통해 SSTable을 만드는 과정을 마무리한다.
    s = builder->Finish();
    if (s.ok()) {
      meta->file_size = builder->FileSize();
      assert(meta->file_size > 0);
    }
    delete builder;

    // 4. WritableFile에 있는 내용들을 storage에 쓴다
    if (s.ok()) {
      s = file->Sync();
    }
    if (s.ok()) {
      s = file->Close();
    }
    delete file;
    file = nullptr;

    if (s.ok()) {
      // 5. storage에 저장한 SSTable을 cache에 올려서 사용가능한지 확인해본다.
      Iterator* it = table_cache->NewIterator(ReadOptions(), meta->number,
                                              meta->file_size);
      s = it->status();
      delete it;
    }
  }

  // 이터레이터 관련 오류 확인
  if (!iter->status().ok()) {
    s = iter->status();
  }

  if (s.ok() && meta->file_size > 0) {
    // Keep it
  } else {
    env->RemoveFile(fname);
  }
  return s;
}
```

#### TableBuilder::Add  
> *TableBuilder안에 있는 각각의 BlockBuilder들에게 Iterator가 현재 참조하고 있는 key-value pair를 전달하는 역할을 한다.*  

1. 현재 `BlockBuilder`로 만드는 Data Block이 비어있다면, 즉 새로운 Data Block을 구성하기 시작했다면 Index Block에 새 Entry를 추가한다. 이 때 추가되는 Entry는 현재 새로 만들기 시작한 Data Block에 대한 것이 아니라 바로 이전까지 만들던 Data Block에 대한 Entry이다.
2. Bloom Filter를 사용하는 경우 Filter Block도 업데이트한다.
3. Data Block에 데이터를 추가한다.
4. 만약 작성 중인 Data Block이 꽉 찼다면(option으로 지정한 block size이상이 된 경우) `Flush`를 호출한다.  
   
```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  
  // ...생략

  // 1. 현재 BlockBuilder로 만드는 Data Block이 비어있다면, Index Block에 새 Entry를 추가한다.
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  // 2. Bloom Filter를 사용하는 경우 Filter Block도 업데이트한다.
  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  // 3. Data Block에 데이터를 추가한다.
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  // 4. 만약 작성 중인 Data Block이 꽉 찼다면 Flush를 호출한다.
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```
#### TableBuilder::Finish
> *MemTable의 모든 key-valur pair들에 대해 `Add`가 끝났을 때 호출되며, 작성중인 SSTable을 마무리하는 역할을 한다.*

1. `Flush`를 호출한다.
2. Bloom Filter를 사용하는 경우 `WritableFile`에 FilterBlockBuilder로 Filter Block을 추가한다.
3. `WritableFile`에 Meta Index Block을 추가한다.
4. `WritableFile`에 `BlockBuilder`로 만들고 있던 Index Block을 추가한다.
5. `WritableFile`에 Footer를 추가한다.  

```cpp
Status TableBuilder::Finish() {
  Rep* r = rep_;
  // 1. Flush를 호출한다.
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // 2. Bloom Filter를 사용하는 경우 WritableFile에 Filter Block을 추가한다.
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // 3. WritableFile에 Meta Index Block을 추가한다.
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // 4. WritableFile에 Index Block을 추가한다.
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // 5. WritableFile에 Footer를 추가한다.
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

#### TableBuilder::Flush
> *BlockBuilder로 만들고 있는 Data Block을 storage에 쓰는 역할을 한다.*

1. `BlockBuilder`로 만들고 있는 Data Block의 contents를 `WritableFile`에 추가한다.
2. `WritableFile`에 쓴 내용을 storage에 쓴다.
3. Bloom Filter를 사용할 경우 새 Bloom Filter를 만든다.  

```cpp
void TableBuilder::Flush() {
  Rep* r = rep_;
  
  // ...생략

  // 1. BlockBuilder로 만들고 있는 Data Block의 contents를 WritableFile에 추가한다.
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    r->pending_index_entry = true;
    // 2. WritableFile에 쓴 내용을 storage에 쓴다.
    r->status = r->file->Flush();
  }
  // 3. Bloom Filter를 사용할 경우 새 Bloom Filter를 만든다. 
  if (r->filter_block != nullptr) {
    r->filter_block->StartBlock(r->offset);
  }
}
```
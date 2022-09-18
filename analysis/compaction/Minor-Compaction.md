## Minor Compaction
  `Minor Compaction` 
  
   -  `flush` 라 부르며 메모리에 있는 데이터(imm memtable)을 디스크(level 0)로 내려주는 작업을 합니다. 


      
## Overall Minor Compaction Code Flow
> Minor compaction 작업만 설명합니다.

> Major compaction 은 [여기에서](./analysis/compaction/Major-Compaction.md) 확인 부탁드립니다.
                                              
          
![image](https://user-images.githubusercontent.com/106041072/188577384-fca24121-ef6d-40b7-aa82-020faf6cc965.png)  
##### 전체적인 과정          
1. `MaybeScheduleCompaction`에서는  Minor Compaction 작업이 필요한지 체크하고 필요하면 `BackgroundCall`을 호출합니다.
2. `BackgroundCall`에서  `BackgroundCompaction`을 호출하며  imm memtable이 있는지 확인합니다.
3. `CompactMemTable`는 `WriteLevel0Table` 함수를 호출하여 `Minor Comapction` 작업을 실행합니다.


###  MaybeScheduleCompaction & BackgroundCompaction 
> 이두 함수 부분은 [Major compaction](./analysis/compaction/Major-Compaction.md)에서 설명하여 여기서는 생략하도록 하겠습니다.

### CompactMemTable
> 본격적인  Minor Compaction 이 진행됩니다.

```cpp
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
   //(TeamCompaction)1.Main point : call the `WriteLevel0Table` function for mior compaction
  Status s = WriteLevel0Table(imm_, &edit, base); 
  base->Unref();

   //(TeamCompaction)2. Check if WriteLevel0Table was executed successfully
  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) { 
    s = Status::IOError("Deleting DB during memtable compaction");
  }
  
  //(TeamCompaction)3.If mior compaction is successful
  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {  
    // Commit to the new state
    imm_->Unref();
    imm_ = nullptr;  
    has_imm_.store(false, std::memory_order_release);
    RemoveObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
}
```
1. `Minor Compaction`을 실행하기 위해 `WriteLevel0Table`함수를 호출합니다.인자로 현재 `Minor Compaction`을 실행할 imm memtable 과 현재 version을 줍니다.
2. 만약 `WriteLevel0Table`함수 실행 중에 오류가 발생 하였으면 Error 메시지 전송합니다.
3. 성공적으로 `Minor Compaction`을 실행하였다면 imm memtbale을 rellease을 해줍니다.

### WriteLevel0Table
> imm memtable 을 디스크로 내려주는 작업을 해줍니다. 

```cpp
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  //(TeamCompaction)1. Organizes imm memtable's meta information.
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    //(TeamCompaction)2. Change imm memtable to SST.
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  //(TeamCompaction)3. update the version
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```
1. 첫번째로 imm memtable의 정보를 정리합니다.
2. `BuildTable`함수로 imm memtbale 정보로 SST를 만듭니다.
3. 성공적으로 SST를 생성하게 되었다면 Version에 이 정보를 업데이트 해줍니다.

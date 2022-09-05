#  Minor Compaction
> Minor Compaction은  memtable이 Full 상태가 되면서 Immmutable mematble로 변하고 해당 Immmutable memtable을 메모리단에서 디스크의 level 0에 내리는 중요한 작업을 합니다.


## Mior Compaction 실행
Minor Compaction의 기능을 수행하는 함수는 **DBImpl::WriteLevel0Table** 입니다.

그전에 해당 함수를 호출하는 **DBImpl::CompactMemTable**을 분석해 보도록 하겠습니다.

- ### DBImpl::CompactMemTable()
***DBImpl::CompactMemTable()*** 은 ***DBImpl::BackgroundCompaction()*** 에 의해 호출 됩니다. 

***DBImpl::BackgroundCompaction()*** 에서는 ***Minor Compaction*** 과 ***Major Compaction***의 호출을 담당합니다.

간단하게 호출 부분만 보게되면 참 간단 합니다.

```c++
void DBImpl::BackgroundCompaction() {
...

//if  Immmutable mematble not empty
  if (imm_ != nullptr) { 
    CompactMemTable();    
    return;
  }

...

```

```c++
  
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  // call function WriteLevel0Table
  Status s = WriteLevel0Table(imm_, &edit, base); 
  base->Unref();

  // check error 
  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) { 
    s = Status::IOError("Deleting DB during memtable compaction");
  }


  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  // if mior compaction is successful
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

- ### DBImpl::WriteLevel0Table

```c++

Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  // memtable - iter
  Iterator* iter = mem->NewIterator(); 
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    // make sstable
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta); 
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);


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
  //save in stats
  stats_[level].Add(stats);  
  return s;
}

```

> compaction에 관한 code부분만  주석을 달았습니다.

- ### MiNor Compaction 
![image](https://user-images.githubusercontent.com/86946575/188358601-12514a26-0894-4ea7-bb99-de2c6028a82f.png)

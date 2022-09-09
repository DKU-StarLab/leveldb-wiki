## Major Compaction
  `Major Compaction` 이란?  
      - 우리가 흔히 `Compaction` 이라 부르는 실질적인 부분입니다.  
      - 메모리에 있는 데이터를 디스크에 옮기는 `Minor Compaction`와 다르게 디스크 내부에 있는 데이터를 병합하는 일을 합니다.  
      - 각 데이터를 병합하여 한 단계 낮은 레벨로 옮깁니다.  
      
  `Major Compaction`을 왜 사용할까?  
      - `Major Compaction`을 사용하는 가장 분명한 이점은 중복 데이터를 정리하는 것입니다.  
      - 다른 레벨의 sst 파일에 동일한 키가 있는 경우 더 낮은 레벨의 데이터(오래된 데이터)를 삭제할 수 있습니다.  
      - 전에 썼던 데이터가 필요할 수 있기 때문에 삭제할 데이터는 순차적으로 기록하고 최신 데이터를 갱신하여 디스크 공간을 절약합니다.   
      - 레벨0은 데이터 파일 간에 순서가 맞지 않을 수 있기 때문에 병합하여 레벨1에 병합되면 데이터가 정렬되고 파일을 쉽게 찾을 수 있어 읽기 효율성을 향상시킵니다.
      
  `Major Compaction`은 언제 발생할까?  
      - 각 레벨의 파일이 쌓이다가 임계치에 다다르고 이때 immutable이 존재하지 않으면 발생합니다.
      
## Overall Major Compaction Code Flow
  > 전체적인 Major Compaction Code Flow를 살펴보겠습니다.

![image](https://user-images.githubusercontent.com/106041072/188577384-fca24121-ef6d-40b7-aa82-020faf6cc965.png)  
각 레벨의 파일이 쌓여 임계치에 다다르면 `MaybeScheduleCompaction`이 합병을 해야하는지 판단합니다. 합병이 필요하면 `immutable`의 존재를 확인하고 없으면 `PickCompaction` 호출되고 합병할 정보를 갖고 `DoCompactionWork`에서 Major Compaction 이 진행됩니다.  

Major compaction이 발생하는 과정을 소스 코드를 통해 세부적으로 알아보겠습니다.    


##### 전체적인 과정  
  1. 어떤 레벨의 파일 수가 임계치에 다다릅니다. 
  2. `MaybeScheduleCompaction`에서 합병이 필요한지 판단합니다.  
  3. `BackgroundCompaction`에서 immutable 있는지 판단하여 없으면 Major Compaction을 위한 함수를 호출합니다.  
  4. `PickCompaction`에서 합병이 필요한 정보를 저장합니다. 
  5. 저장한 정보를 갖고 `DoCompactionWork`에서 실질적인 합병이 진행됩니다.  
  6. `DoCompactionWork`에서 `OpenCompactionOutputFile`,`FinishCompactionOutputFile`,`InstallCompactionResults` 가 실행되어 sst 파일을 만들고 합병한 레코드를 넣고 오류를 점검하며 해당 레벨로 파일을 넣습니다.  
  7. 그 후 버전을 업데이트하고 마지막으로 `CleanupCompaction`을 호출하여 불필요한 sst 파일을 삭제하여 Major Compaction을 마무리합니다.  

### MaybeScheduleCompaction
> 합병이 필요한지를 판단합니다.
```cpp
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  //(TeamCompaction)If a merger is already in progress, DB is being deleted, or if there is an error, nothing will happen.
  if (background_compaction_scheduled_) {
  } else if (shutting_down_.load(std::memory_order_acquire)) {
  } else if (!bg_error_.ok()) {
  //(TeamCompaction)If immutable does not exist, manual compaction do not exist, and mergers at each level are not required, nothing will happen.
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
  //(TeamCompaction)In other cases, since a merger occurs, call 'BGWork' to proceed with the merger.
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}
```  
1. 이미 합병이 진행중이거나 DB가 삭제되어지는 중이거나 에러가 있으면 아무것도 발생하지 않습니다.  
2. immutable이 존재하지 않고 수동 합병이 존재하지 않고 각 레벨의 합병이 필요하지 않으면(`NeedCompaction`) 아무것도 발생하지 않습니다.(수동 합병은 BackgroundCompaction 부분에서 설명하겠습니다.)  
3. 그 외에는 합병이 필요하기 때문에 `BGWork`을 호출하여 합병을 진행합니다.  

### BackgroundCompaction
> immutable의 존재여부를 통해 Major Compaction, Minor Compaction을 결정합니다. 또한 사용자가 수동 합병을 원하면 여기서 진행합니다.  

```cpp
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

//(TeamCompaction)If imutable exists, proceed with Minor Compact through the CompactMemtable function.
  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }

  Compaction* c;
  //(TeamCompaction)If you want to merge manually, proceed with the manual merger with true (most of them are automatically merged, so they are not used well)
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level, (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } //(TeamCompaction)Store information that needs to be merged if immutable does not exist
  else {
    c = versions_->PickCompaction();
  }
  
//...생략

Status status;
  //(TeamCompaction)Nothing happens if there is no information to merge
  if (c == nullptr) {
  } else if (!is_manual && c->IsTrivialMove()) {
  
    //...생략 (TeamCompaciton) Manual Compaction progression part
    
  } //(TeamCompaction)With information to merge, proceed with Major Compaction.
  else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    //(TeamCompaction)If there is no problem after completing the merger completely, remove the sst files (files collected after the merger) that are now unnecessary
    CleanupCompaction(compact);
    //(TeamCompaction)Remove the sst files that were saved for the merger
    c->ReleaseInputs();
    RemoveObsoleteFiles();
  }
  
//...생략

```  

1. immutable이 존재하면 `CompactMemtable`을 통해 Minor Compaciton(Flush)가 진행됩니다.  
2. immutable이 존재하지 않다면 수동합병을 원치 않는다면 `PickCompaction`을 통해 합병할 정보를 저장합니다.  
3. 합병할 정보가 없으면 아무것도 일어나지 않고 있다면 그 정보를 토대로 `DoCompactionWork`에서 본격적인 Major Compaction을 진행합니다.  
4. `DoCompactionWork`에서 합병이 끝나고 그 상태가 온전하다면 `CleanupCompaction`을 통해 합병전 파일이였던 불필요한 sst파일을 제거하여 마무리합니다.  

##### (부가 설명)  
> Manual Compaction vs Automatic Compaction  

Manual Compaction(수동합병)  
  - 자주 사용되지 않고 주로 디버깅할 때 사용합니다.  
  - 사용방법은 benchmark를 key range을 설정하여 실행하면 수동합병 여부의 bool 값이 true가 되어 수동합병을 실행합니다.  
  
Automatic Compaction(자동합병)  
  - 주로 저희가 사용하는 대부분의 합병은 자동합병으로 진행됩니다.  

### DoCompactionWork
> 본격적인 Major Compaction이 진행됩니다.  

```cpp
Status DBImpl::DoCompactionWork(CompactionState* compact) {

//...생략

  //(TeamCompaction)Create an iter for the index block and data block of each sst file from level 0, 1 to N, and use it to find each key. 
  //(TeamCompaction)After that, arrange the created iters so that they can be listed in the order of level 0, 1 to N, and store them in input (see Compaction-Iter.md for details)
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  
// Release mutex while we're actually doing the compaction work
  mutex_.Unlock();
  //(TeamCompaction)Position the pointer position of the created iter first
  input->SeekToFirst();
  Status status;
  //(TeamCompaction)Parse the internal key and divide it into user key, sequence number, and type
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  //(TeamCompaction)Set the latest key to the highest value
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  //(TeamCompaction)The process of repeatedly finding and processing the key/value that needs to be merged through iter stored in the input
  while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
  
   //...생략
   
    //(TeamCompaction)Obtain the key of the current corresponding sst file
    Slice key = input->key();
    //(TeamCompaction)Check if you need an sst file to put the key in, and if there is an sst file to put in, the merger is completed
    if (compact->compaction->ShouldStopBefore(key) && 
        compact->builder != NULL) {
      status = FinishCompactionOutputFile(compact, input);
    }
   
    //(TeamCompaction)Compare different sequences for the same key to obtain the latest user key and delete records from different user keys that were the same
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) !=
              0) {
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }

      if (last_sequence_for_key <= compact->smallest_snapshot) {
        // Hidden by an newer entry for same user key
        drop = true;  // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }
    
  //...생략

    if (!drop) {
      //(TeamCompaction)Create new sst file if necessary
      if (compact->builder == nullptr) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      //(TeamCompaction)Add records from the latest user key to the sst file
      compact->builder->Add(key, input->value());

      //(TeamCompaction)If data accumulates in the sst file and exceeds the maximum file size, the merger of the sst file is completed
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }
    //(TeamCompaction)Among the sst files arranged in order of level 0, 1 to N in the input variable, the next sst file is moved on
    input->Next();
  }
  
  if (status.ok() && shutting_down_.load(std::memory_order_acquire)) {
    status = Status::IOError("Deleting DB during compaction");
  }
  //(TeamCompaction)If the sst file does not have an error and the sst file exists, the merger is completed
  if (status.ok() && compact->builder != nullptr) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  
  //...생략
  
  //(TeamCompaction)Move the merged sst file to its level
  if (status.ok()) {
    status = InstallCompactionResults(compact);
  }
  if (!status.ok()) {
    RecordBackgroundError(status);
  }
  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log, "compacted to: %s", versions_->LevelSummary(&tmp));
  return status;
}

```  
1. 레벨0, 1-N까지 각 sst 파일의 `index block` 과 `data block`에 대한 iter을 만들어 각 key을 찾을 수 있게 하고 다시 iter을 정렬하여 레벨0, 1-N 순으로 iter들이 나열될 수 있게 만들어 `input` 저장합니다.      
2. iter의 포인터 위치를 첫번째로 위치시키고 `internalkey`를 파싱하여 user key, sequence, type으로 나눕니다.  
3. `key()`를 통해 현재 해당하는 sst 파일의 key 획득합니다.  
4. 동일한 키에 대한 서로 다른 sequence를 비교하여 제일 큰 sequence를 갖는 최신 user key의 레코드를 얻고 동일했던 다른 user key의 레코드를 삭제합니다.  
4. sst 파일이 필요하다면 새로 만들어 최신 user key의 레코드를 넣습니다.  
5. sst 파일이 최대 파일 사이즈를 넘게 되면 합병을 마무리하고 다음 sst 파일로 넘어갑니다.  
6. 2~5 과정을 반복하여 모든 sst 파일의 합병을 마무리한 후 전체적인 상태를 체크 후 에러가 없다면 합병이 완료된 sst 파일을 해당 레벨로 옮깁니다.  

### OpenCompactionOutputFile
> 합병된 user key 레코드를 넣을 새로운 sst 파일을 만듭니다.  

```cpp
Status DBImpl::OpenCompactionOutputFile(CompactionState* compact) {

 //...생략
 
  //(TeamCompaction)Set the number of the newly created sst file
  std::string fname = TableFileName(dbname_, file_number);
  //(TeamCompaction)Numbered and temporarily merged records in writablefile
  Status s = env_->NewWritableFile(fname, &compact->outfile);
  //(TeamCompaction)If the record is in good condition, create a sst file and add it to the builder
  if (s.ok()) {
    compact->builder = new TableBuilder(options_, compact->outfile);
  }
  return s;
}
```  
1. 새로 만들어질 sst 파일의 번호를 매깁니다.  
2. 번호가 매겨지고 임시로 `WriteableFile`에 합병된 레코드를 넣습니다.  
3. 레코드의 상태가 괜찮으면 sst 파일을 만들고 `WriteableFile`에 넣었던 레코드를 다시 넣어 
에 추가합니다.  

### FinishCompactionOutputFile
> 합병된 iter과 sst 파일에 대한 에러를 체크합니다.  

```cpp
Status DBImpl::FinishCompactionOutputFile(CompactionState* compact,
                                          Iterator* input) {
  //...생략

  //(TeamCompaction)Check for errors for iter
  Status s = input->status();
  const uint64_t current_entries = compact->builder->NumEntries();
  if (s.ok()) {
    s = compact->builder->Finish();
  } else {
    compact->builder->Abandon();
  }
  
  //...생략

  //(TeamCompaction)Check and finalize errors for the sst file itself
  if (s.ok()) {
    s = compact->outfile->Sync();
  }
  if (s.ok()) {
    s = compact->outfile->Close();
  }
  delete compact->outfile;
  compact->outfile = nullptr;
  
  //...생략
  
}
```  
1. iter에 대한 에러를 체크합니다.  
2. sst 파일 자체에 대한 에러를 체크하고 마무리합니다.  

### InstallCompactionResults
> 합병이 완료된 sst 파일을 해당 레벨로 옮기고 버전을 업데이트합니다.  

```cpp
Status DBImpl::InstallCompactionResults(CompactionState* compact) {

  //...생략
  
  //(TeamCompaction) Moved merged sst files to that level
  compact->compaction->AddInputDeletions(compact->compaction->edit());
  const int level = compact->compaction->level();
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    compact->compaction->edit()->AddFile(level + 1, out.number, out.file_size,
                                         out.smallest, out.largest);
  }
  //(TeamCompaction)Update version
  return versions_->LogAndApply(compact->compaction->edit(), &mutex_);
}
```  

1. 합병된 sst 파일을 해당 레벨에 옮깁니다.
2. 버전을 업데이트하여 마무리합니다.  

### CleanupCompaction
>합병이 마무리된 후 이제 불필요한 sst 파일을(합병 전에 있던 sst 파일) 제거합니다.

```cpp
void DBImpl::CleanupCompaction(CompactionState* compact) {
  mutex_.AssertHeld();
  //(TeamCompaction)Remove files if they exist in the builder that contains the sst files that need to be merged
  if (compact->builder != nullptr) {
    // May happen if we get a shutdown call in the middle of compaction
    compact->builder->Abandon();
    delete compact->builder;
  } else {
    assert(compact->outfile == nullptr);
  }
  delete compact->outfile;
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    pending_outputs_.erase(out.number);
  }
  delete compact;
}
```  
1. 최신 sst 파일로 업데이트했기 떄문에 합병해야하는 sst 파일을 모아둔 `builder`에 파일이 존재하면 이제 사용하지 않기 때문에 제거합니다.  
2. Major Compaction을 최종적으로 마무리합니다.






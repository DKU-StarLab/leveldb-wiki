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
      - 메모리에 immutable 존재하지 않거나 각 레벨의 파일 수가 쌓이다가 임계치에 다다르면 발생합니다.
      
## Overall Major Compaction Code Flow
  > 전체적인 Major Compaction Code Flow를 살펴보겠습니다.

![image](https://user-images.githubusercontent.com/106041072/188577384-fca24121-ef6d-40b7-aa82-020faf6cc965.png)  
`immutable`이 존재하지 않으면 `PickCompaction` 호출되고 합병할 정보를 갖고 `DoCompactionWork`에서 Major Compaction 이 진행됩니다.  

Major compaction이 발생하는 과정을 소스 코드를 통해 세부적으로 알아보겠습니다.    


##### 전체적인 과정  
  1. `MaybeScheduleCompaction`에서 합병이 필요한지 판단한다.  
  2. `BackgroundCompaction`에서 immutable 있는지 판단하여 없으면 Major Compaction을 위한 함수를 호출합니다.  
  3. `PickCompaction`에서 합병이 필요한 정보를 저장합니다. 
  4. 저장한 정보를 갖고 `DoCompactionWork`에서 실질적인 합병이 진행됩니다.  
  5. `DoCompactionWork`에서 `OpenCompactionOutputFile`,`FinishCompactionOutputFile`,`InstallCompactionResults` 가 실행되어 sst 파일을 만들고 합병한 레코드를 넣고 오류를 점검하며 해당 레벨로 파일을 넣습니다.  
  6. 그 후 버전을 업데이트하고 마지막으로 `CleanupCompaction`을 호출하여 불필요한 sst 파일을 삭제하여 Major Compaction을 마무리합니다.  

### MaybeScheduleCompaction
> 합병이 필요한지를 판단합니다.
```cpp
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  //이미 합병이 진행중이거나 DB가 삭제되어지는 중이거나 에러가 있으면 아무것도 발생하지 않음
  if (background_compaction_scheduled_) {
  } else if (shutting_down_.load(std::memory_order_acquire)) {
  } else if (!bg_error_.ok()) {
  //immutable이 존재하지 않고 수동 합병이 존재하지 않고 각 레벨의 합병이 필요하지 않으면 아무것도 발생하지 않음
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
  //그 외에는 합병이 일어난 경우이므로 `BGWork`을 호출하여 합병 진행           
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

//immutable이 존재한다면 CompactionMemtable 함수를 통해 Minor Compaction 진행
  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }

  Compaction* c;
  //수동으로 합병하길 원하면 true로 하여 수동 합병 진행(대부분 자동으로 합병하기 때문에 사용 잘 안함)
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
  } //immutable이 존재하지 않는다면 합병할 정보 저장
  else {
    c = versions_->PickCompaction();
  }
//...생략
Status status;
  //합병할 정보가 없다면 아무것도 일어나지 않음
  if (c == nullptr) {
  } else if (!is_manual && c->IsTrivialMove()) {
    //...생략 (수동합병 진행부분)
  } //합병할 정보를 갖고 본격적인 Major Compaction 진행
  else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    //합병을 마친 후 이상이 이상이 없으면 불필한 sst 파일(합병전 파일) 제거
    CleanupCompaction(compact);
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

  //레벨0, 1~N까지 각 sst 파일의 index block 과 data block에 대한 iter을 만들어 각 key을 찾을 수 있게 하고 
  다시 iter을 정렬하여 레벨0, 1~N 순으로 iter들이 나열될 수 있게 만들어 input에 저장(자세한 설명은 Compaction-Iter.md 참고)
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  
// Release mutex while we're actually doing the compaction work
  mutex_.Unlock();
  //만들어진 iter의 포인터 위치를 첫번째로 위치시킴
  input->SeekToFirst();
  Status status;
  //interalkey를 파싱하여 user key, sequence number, type으로 나눔
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  //input에 저장되어 있는 iter을 계속 반복하여 합병할 key/value를 찾고 처리
  while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
  
   //...생략
   
    //현재 해당하는 sst 파일의 key 획득
    Slice key = input->key();
    //해당 key를 넣을 sst 파일이 필요한지 확인하고 만약 넣을 sst 파일이 있으면 합병을 마무리함
    if (compact->compaction->ShouldStopBefore(key) &&   //是否需要停止Compaction
        compact->builder != NULL) {
      status = FinishCompactionOutputFile(compact, input);
    }
   
    // 동일한 키에 대한 서로 다른 sequence를 비교하여 최신 user key를 얻고 동일했던 다른 user key의 레코드를 삭제합니다.
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
      //새로운 sst 파일이 필요하면 생성
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
      //최신 user key의 레코드를 sst 파일에 기록
      compact->builder->Add(key, input->value());

      // sst 파일에 데이터가 쌓이다가 최대 파일 사이즈를 넘게 되면 해당 sst 파일의 합병을 마무리
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }
    //input 변수 안에 레벨0, 1~N 순으로 정렬되어 있는 sst 파일들에 대해 다음 sst 파일로 넘어감
    input->Next();
  }
  
  if (status.ok() && shutting_down_.load(std::memory_order_acquire)) {
    status = Status::IOError("Deleting DB during compaction");
  }
  //sst 파일이 에러가 없고 sst 파일이 존재한다면 합병을 마무리함
  if (status.ok() && compact->builder != nullptr) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  
  //...생략
  
  //합병이 완료된 sst 파일을 해당 레벨로 옮김
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
 
  // 새로 만들어질 sst 파일의 번호 설정.
  std::string fname = TableFileName(dbname_, file_number);
  // 번호가 매겨지고 writeablefile에 임시로 합병된 레코드를 넣음
  Status s = env_->NewWritableFile(fname, &compact->outfile);
  //레코드의 상태가 괜찮으면 sst 파일을 만들고 builder에 추가
  if (s.ok()) {
    compact->builder = new TableBuilder(options_, compact->outfile);
  }
  return s;
}
```  
1. 새로 만들어질 sst 파일의 번호를 매깁니다.  
2. 번호가 매겨지고 `WriteableFile`에 임시로 합병된 레코드를 넣습니다.  
3. 레코드의 상태가 괜찮으면 sst 파일을 만들고 builder에 추가합니다.  

### FinishCompactionOutputFile
> 합병된 iter과 sst 파일에 대한 에러를 체크합니다.  

```cpp
Status DBImpl::FinishCompactionOutputFile(CompactionState* compact,
                                          Iterator* input) {
  //...생략

  // iter에 대한 에러를 체크
  Status s = input->status();
  const uint64_t current_entries = compact->builder->NumEntries();
  if (s.ok()) {
    s = compact->builder->Finish();
  } else {
    compact->builder->Abandon();
  }
  
  //...생략

  // sst 파일 자체에 대한 에러를 체크하고 마무리
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
  
  // 합병된 sst 파일을 해당 레벨에 옮김
  compact->compaction->AddInputDeletions(compact->compaction->edit());
  const int level = compact->compaction->level();
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    compact->compaction->edit()->AddFile(level + 1, out.number, out.file_size,
                                         out.smallest, out.largest);
  }
  //버전 업데이트
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
  //합병해야하는 sst 파일을 모아둔 builder에 파일이 존재하면 제거
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






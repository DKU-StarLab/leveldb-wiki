#  Iterator
Iterator는 levelDB 뿐만 아니라  다양한 데이터 구조에 대한 통합 액세스 메커니즘을 제공하는 인터페이스로, iterator 인터페이스가 배포되는 한 모든 데이터 구조에서 순회 작업을 할수 있습니다.

LevelDB에서도 Iterator을 다양한 방면에서 사용을 합니다.

Compaction에서도 Iterator을 사용하기도 합니다.
***TwoLevelIterator*** 와 ***MergingIterator*** 입니다.이에대해 분석해보도록 하겠습니다.

## 1.  TwoLevelIterator 
            
                                                sst의 구조를 안다는 전제하에 설명을 하겠습니다.
                                                     그림으로 간단하게 그려 보았습니다.
            
TwoLevelIterator는 sstable에서 사용되는  iterator 이며 이름대로 2중으로 되어있습니다. 
- index_iter
  - 해당 index_iter는 index block에서 찾는 key가 포함되어있는  data block을 찾는데 사용됩니다. 
- data_iter
  - data_iter는 data block에는 여러 entry fild가 있으며 그중 찾고있는 키를 하나하나 검색합니다.

![image](https://user-images.githubusercontent.com/86946575/188078001-56ba72ba-0308-4cb1-b966-cbe65263e5cd.png)


***TwoLevelIterator*** 의 코드 부분을 보게 되면 일반  ***Iterator*** 을 상속받아서 사용하고 있고 4개의 인자를 받습니다.

1. index_iter는 index block을 가리키는 iter

2.  block_function은 Table::BlockReader, 즉 블록을 읽는 것

3. arg는 SST을 가리키는 것

4.  Options 읽기 옵션


```c++
class TwoLevelIterator : public Iterator {
 public:
  TwoLevelIterator(Iterator* index_iter, BlockFunction block_function,
                   void* arg, const ReadOptions& options);
```

## 2.  MergingIterator
MergingIterator는 이름에서 알수있 듯이 Iterator들의 Merge 입니다.

이 part에서는 MergingIterator에 들어가는 Iterator는 3가지로 나눌수 있습니다.
- memtable(Imm)Iterator
- Level 0 TwoLevelIterator
- Level 1 ~ N TwoLevelIterator


***MergingIterator*** 는 code 상 Iterator type의 vector을 생성하여 ***memtable ,Level 0 ,level 1 ~ N***  순선대로 넣어서 사용합니다.
![image](https://user-images.githubusercontent.com/86946575/188079477-709ffe0d-4437-4d08-aec6-612ec53031fe.png)
![image](https://user-images.githubusercontent.com/86946575/188080151-3ff12581-988d-4435-9a49-6bb2bf563da8.png)

### Iterator의  기능적 흐름도
![image](https://user-images.githubusercontent.com/86946575/188079392-ac5486f3-3dfd-4b32-b87c-7622c2c96835.png)

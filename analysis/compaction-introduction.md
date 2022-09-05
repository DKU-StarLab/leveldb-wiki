# Compaction 
 > Compaction은  db에서 가장 복잡한 프로세스 중 하나이며 db의 성능에 큰 영향을 주기도 합니다.

 > 내부 데이터 중첩 및 통합 메커니즘이며 읽기 및 쓰기 속도의 균형을 맞추는 효과적인 수단이기도 합니다.

- compaction은  leveldb의 핵심 기능이며 백그라운드 스레드에 의해 실행되며 백그라운드 스레드 중  BackgroundCompaction() 함수는 두 가지 작업 주요 작업을 수행합니다. 
   1. imm_ 즉 immutable memtable 이 비어 있지 않으면 imm_을 디스크에 기록하여 레벨 0에서 새 sstable 파일을 생성합니다. (***Minor Compaction***)
   2. 특정 기준에 따라 level n과 level n+1을 선택을 하고  중첩 되는 key가 들어가있는 sstable 파일을 선택하고  merge & sort를 하여 새로운 sstable을 생성하여 level n+1에 생성을 하게됩니다. (***Major Compaction***)


## Compaction의 필요성
compaction이 없으면  level 1 ~ N 까지(level 0 제외)의 key 원자성을 보장 못하므로 동일한 key가 한 level에 여러개 존재 할 가능성이 높습니다.읽기(get,seek)할시에 높은 overhead가 발생합니다.

## Compaction Type
### 1. Minor Compaction


![image](https://user-images.githubusercontent.com/86946575/181177580-415e1214-edfc-4180-b072-b36b8827ca1f.png)

- Trivial move  

![image](https://user-images.githubusercontent.com/86946575/181178432-ba39014c-a4a7-4d2e-ad15-5333109bdb22.png)

### 2. Major Compaction


![image](https://user-images.githubusercontent.com/86946575/181183259-327818ac-1a2d-4e0e-91c9-24bc99cd1c3b.png)


## Compaction Code Flow

![image](https://user-images.githubusercontent.com/86946575/188067273-44e604d2-3f74-49fc-90a1-da5959d06dac.png)


# WAL/MANIFEST
이 문서는 WAL/MANIFEST 에 대한 문서이다.
## Index
- [WAL](#WAL): WAL 에 대한 개요를 설명한다.
- [Functions](#Functions): WAL/MANIFEST  관련 함수들에 대한 설명이다.
> MANIFEST 와 WAL 에서 사용하는 함수들이 기본적으로 다 공통되는 부분이 많기 때문에, 이를 따로 구분하지 않고 WAL 문서에 추가했음
- [ETC](#ETC): 추가적으로 알게된 연구 결과에 대한 내용을 설명한다.

## WAL
### Introduction
> LevelDB 에는 WAL 을 사용할지에 대한 여부가 존재하지 않는다. 만일 WAL 을 활성화하거나 비활성화 하기 위해서는 RocksDB 를 사용하거나, 기존의 LevelDB 의 소스코드를 수정해야한다. 

Key-Value Store 인 LevelDB에 데이터가 기록될 때 WAL 을 사용한다. WAL 은 Write Ahead Log 의 약자로, LevelDB 에서 일어나는 모든 transaction 의 log 를 기록한다. WAL 을 사용하는 이유 중 하나는 translaction 을 잃어버리지 않기 위함이다.

LevelDB 에서 데이터가 저장되기 위해서 임시적으로 Memtable 에 기록이 된다. 하지만 Memtable 은 실제적으로 디스크에 저장이 되지 않고, memory 상에 저장된다. 따라서 만일 LevelDB 의 동작이 정상적으로 이루어지지 않고, 예상치 못한 종료 및 에러가 발생한다면, 기존의 Memtable 에 작성된 데이터는 잃어버린다. 

> 위의 데이터를 잃어버리는 경우에 대한 설명과 이에 대한 분석은 [ETC](#ETC) 에 추가적으로 정리했다.

데이터를 잃어버리지 않기 위해서 LevelDB 에서 일어나는 일들을 log 로 별도로 저장하는 방식을 WAL 이라고 할 수 있다.

### Format
LevelDB 의 경우 `.log` 의 파일 형식으로 파일을 저장한다.

이는 binary 파일로 저장이 되는데, WAL 은 다음의 두가지를 저장한다.
- 해당 Transaction 의 header
- 실제 Transaction 의 데이터 (payload)

#### Header
WAL 에서 header 를 저장하는 형식은 다음과 같다.

<img src="https://user-images.githubusercontent.com/49092508/190959632-08d86d79-d24c-4d50-bbf6-55fccb829556.png" width="700"/>

순차적으로
1. CRC Checksum 4byte
2. 데이터의 사이즈 2byte
3. Record Type 1byte 

를 사용한다.

#### Payload
WAL 의 header 이후 payload 는 다음의 형식으로 저장한다.
> 다음의 예시는 {"A": "Hello world!", "B": "Good bye world!", "C": "I am hungry"} 이렇게 3쌍의 Key-value 쌍을 PUT 했을 때 작성되는 WAL 파일의 예시이다.

<img src="https://user-images.githubusercontent.com/49092508/190959530-2832a72d-8f65-4207-9ca1-2fa227d17acb.png" width="700"/>

## Functions
해당 내용은 WAL/MANIFEST 에 관련하여 공부를 하며 알게된 함수들과, 해당 함수들에 대한 설명을 포함한다.

### `log::reader::ReadRecord`
```log::reader``` 파일에는  

```Reader``` 클래스와 몇 가지 함수가 있다.  

<img src="https://drive.google.com/u/1/uc?id=1VQ_Q20Z1ZtrY39Lead8EDtCSbr9z-BIS&export=download" width="700"/>  

나는 이 파일의 흐름을 살펴보고자 노력했다.  

```ReadRecord()``` 함수가 실행되면 이니셜 블록의 위치를 찾은 후 ```while```문을 돌면서 읽고자 하는 데이터를 읽는다.  

```ReadRecord()``` 함수의 매개변수로는 ```record```와 ```scratch```를 받는데,  

읽고자 하는 데이터가 여러 블록으로 나뉘어져 있을 경우 ```scratch```에 각 블럭의 데이터를 ```append```해주었다가 마지막 블럭을 만나면 한번에 ```record```에 준다.  

<img src="https://drive.google.com/u/1/uc?id=1hWZOM3mO0TeymflMS6knKOzybNslOmhE&export=download" width="700"/>  

읽으려는 데이터가 몇 개의 블럭에 걸쳐 있는지 알기 위해서 ```switch```문에서는 각 블럭의 ```type```을 읽고 블럭을 더 읽을 것인지 판단하며  

에러가 있으면 이 또한 처리하는 과정이 있다.  

```kFullType```의 경우 데이터를 읽어 바로 ```record```에 준다음 ```true```를 반환하여 함수를 빠져나온다.  

```kFirstType```의 경우 ```scratch```에 데이터를 ```append```한 후  

```in_fragmented_record``` 변수를 ```true```로 설정하여 뒤에 블럭이 더 있음을 알리는 역할을 한다.  

<img src="https://drive.google.com/u/1/uc?id=1hpQesw4cq7697jc_H0pcu_sP-1igFiiK&export=download" width="700"/>  

```kMiddleType```의 경우 ```scratch```에 데이터를 ```append``` 해준다.  

```kLastType```의 경우 ```scratch```에 데이터를 ```append``` 해준 후  

여태 추가했던 데이터가 담겨있는 ```scratch```를 ```Slice``` 객체로 바꿔서 ```record```에 준다.  

```kEof```의 경우는 더이상 읽을 블럭이 없을 때이다.  

<img src="https://drive.google.com/u/1/uc?id=1nPL8OqHRJ03CzOKO6cWtbdU1i-T_fCiC&export=download" width="700"/>  

 ```kBadRecord```의 경우 ```checksum```이 맞지 않거나 레코드의 길이가 0이거나 등  

 오류가 있어 물리 레코드에서 읽어오지 못한 경우 처리된다.

 ```모든  keyType에  해당되지  않는  경우```는 오류를 알린다.  

<img src="https://drive.google.com/u/1/uc?id=1wa81Pks8xtTKhS7hz0gK579JJK1gAKMg&export=download" width="700"/>  

### `log::Writer:AddRecord`

![image](https://user-images.githubusercontent.com/49092508/190640640-38b59509-b6e9-4d54-8e00-34f6fe403b43.png)

`slice` 타입의 데이터를 받은 이후, slice 를 실제적으로 Record 로 남긴다. 

#### 동작
- 이를 기록으로 남기기 위해 `log::Writer::EmitPhysicalRecord` 를 사용한다.

### `log::Writer::EmitPhysicalRecord`

![image](https://user-images.githubusercontent.com/49092508/190629855-b7ff4227-11b4-484a-a97d-d21073bee682.png)

실제적으로 `slice` 타입의 데이터를 받고, header 를 생성하고 파일을 작성하기 시작하는 함수이다.

#### 동작
- `slice` 타입의 데이터를 받은 이후, 해당 내용을 WAL 의 header 형식에 맞추어서 header 를 추가한다.
- 이후 해당 데이터를 `PosixWritableFile::Append()`를 통해서 실제적인 `.log` 파일로 작성한다

### `PosixWritableFile::Append()`

![image](https://user-images.githubusercontent.com/49092508/190641273-f542d83d-e79c-4af5-b2bd-10dfb399c4ab.png)

POSIX 환경에서 `WritableFile` 을 implmentation 한 내용이다. `WritableFile` 의 경우 buffer 를 가지고 있고, 해당 buffer 가 다 찰 때까지 데이터를 추가한다. WAL  파일 또한 `PosixWritableFile` 객체이므로, 데이터가 이를 통해서 작성된다.

#### 동작
- `slice` 타입의 데이터를 가지고 Buffer 에 최대한 적을 수 있는 만큼 작성한다.
- `slice` 타입의 데이터의 전부를 작성할때까지 반복한다.

## ETC
해당 내용은 WAL 의 여부에 따른 데이터 손실에 대한 간단한 실험에 대한 내용을 포함한다.

> WAL 에 대한 옵션은 RocksDB 에서만 허용된다. 따라서 해당 내용은 LevelDB 가 아닌 RocksDB 로 구현되었다는 점을 인지해야한다. 

### Summary
다음의 상황에 대한 실험이다.
- 실제적으로 RocksDB 를 통해 PUT 을 사용하여 데이터를 작성한다.
- 작성 도중 임의로 RocksDB 가 사용되는 프로세스를 terminate 한다 (`ctrl + c` 를 사용하여 `SIGINT` 를 통해 프로세스를 강제로 종료함)
- 종료 이전의 데이터가 얼마나 손실되고, 얼마나 저장되었는지 확인한다.

### Design
실험에 사용된 코드는 [여기](https://github.com/gooday2die/WAL-Testing)에서 자세하게 확인할 수 있다.

### Scenarios
다음 4가지 시나리오를 생각하여 실험을 진행한다.

||WAL Enabled|Manual Flush|
|--|--|--|
|Scenario 1|X|X|
|Scenario 2|X|O|
|Scenario 3|O|X|
|Scenario 4|O|O|

### Results
해당 실험의 경우 다음의 결과를 확인할 수 있었다.
|Scenario|Data Integrity|
|--|--|
|Scenario 1|Flush 이전까지 Memtable 에 있는 모든 데이터를 다 잃음|
|Scenario 2|Manual Flush 이전까지 Memtable 의 일부 데이터를 잃음|
|Scenario 3|모든 데이터를 다 보존|
|Scenario 4|모든 데이터를 다 보존|

추가적으로 진행한 실험에서 다음의 조건으로 실험을 더 진행했다.
- 총 1,000,000 개의 PUT 진행.
- [0] 의 경우 1,000 개의 entry 마다 flush, [1] 의 경우 10,000 entry 마다 flush.

|Type|Speed|Data Integrity|
|--|--|--|
|WAL Disabled|1.843s|Flush 이전까지 Memtable 에 있는 모든 데이터를 다 잃음|
|WAL Enabled|3.604s|모든 데이터 보존|
|Manual Flush [0] & WAL Disabled|39.128s|Manual Flush 이전까지 Memtable 의 일부 데이터를 잃음|
|Manual Flush [1] & WAL Disabled|5.611s|Manual Flush 이전까지 Memtable 의 일부 데이터를 잃음|

### Bottom line
- WAL 을 활성화하면 데이터를 보존한다.
- WAL 을 비활성화하면 데이터의 일부를 잃어버릴 가능성이 충분하다.
- WAL 을 비활성화하고 수동적으로 Flush 를 진행하면 일부 데이터를 보존할 수 있다. 

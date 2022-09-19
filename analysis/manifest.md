## MANIFEST
MANIFEST 는 `VersionSet` 과 `VersionEdit` 를 파일로 정리해놓는 기능을 한다. 이는 `MANIFEST-000000` 와 같은 이름으로 저장이 된다. 

MANIFEST 파일은 LevelDB 를 활용하며 이름이 바뀌는데, 이번 실행에서 LevelDB가 사용할 MANIFEST 파일을 지칭하는 파일은 `CURRENT` 파일에 적혀있다.

### VersionSet, VersionEdit
`VersionSet` 과 `VersionEdit` 의 경우 둘다 version 에 관한 내용을 정리한다. 
> `VersionSet` 과 `VersionEdit` 의 경우 서로 Friend class 관계이다.

각 class 에서 MANIFEST 파일을 작성하며 사용하는 함수는 각각 하나씩 존재한다.

- `VersionSet::LogAndApply` : 현재 실행하는 버전에 대한 데이터를 MANIFEST 에 저장할 준비를 하고, 최종적으로 데이터를 `VersionEdit::EncodeTo` 를 활용하여 작성하고 MANIFEST 파일에 저장한다.

![image](https://user-images.githubusercontent.com/49092508/190644040-e60217d7-ca32-4419-8a32-b8034aa9909d.png)

- `VersionEdit::EncodeTo` : 실제적으로 작성해야하는 데이터를 binary 형식으로 작성한다. 

![image](https://user-images.githubusercontent.com/49092508/190644269-23463bc4-27e8-4e4c-955c-abfa4df476cd.png)

### DB 를 생성할 때
![image](https://user-images.githubusercontent.com/49092508/190642300-5b4b12a4-7c4a-4ec0-af74-a339f5a87124.png)

초기에 DB 를 생성할 때, MANIFEST 파일은 위의 절차를 통해서 생성이 된다. 초기에는 다음의 값을 가지고 `VersionEdit` 을 생성하고, 이를 `MANIFEST-000001` 파일에 작성한다.

- `comparator` : `"leveldb.BytewiseComparator`
- `log_number` : `0`
- `next_file_number` : 2
- `last_sequence` : 0

이후 `log::Writer` 함수를 통해 해당 내용을 기록한다.

### DB 를 종료할 때
![image](https://user-images.githubusercontent.com/49092508/190645030-4865a546-b26b-4702-8bd3-06d1d31f41a0.png)

LevelDB 를 종료할 때 `~DBImpl` 이 호출된다. 이는 기존에 가지고 있던 `VersionSet` 을 `delete` 하며 기존의 버전에 대한 데이터를 MANIFEST 파일에 작성한다. 이때 작성된 `MANIFEST` 파일을 통해, 다음에 LevelDB 를 시작할 때 기존의 버전을 복구할 수 있게 된다.

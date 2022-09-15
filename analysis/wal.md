# WAL
Key-Value Store 에서는 데이터베이스에 데이터가 기록될 때 WAL 옵션이 켜져있다면 Memtable과 WAL에 동시에 저장되고 그렇지 않다면 Memtabel에만 저장된다. (이는 WAL의 오버헤드 때문인데 performance와 reliability의 trade-off를 고려해서 결정한다.) WAL은 모든 데이터베이스 작업의 실행기록을 유지함으로써 MemTable의 백업 역할을 한다. 다시 시작하는 경우 또는 시스템 오류나 전원 차단과 같이 정상적으로 종료되지 않았을 경우에  WAL에서의 작업(log)을 재생하여 MemTable을 완전히 복구할 수 있다. 

MemTable이 가득 차서 SSTable로 flush되면 새 WAL을 위한 공간을 만들기 위해 디스크에서 WAL이 지워진다.   

<p align="center"><img src="https://github.com/Student5421/Image/blob/main/Image/WAL_Image1.png"></p>

<br>

# WAL Structure
RocksDB는 WAL의 레코드를 블록 형식으로 저장한다. 각 블록은 32KB이며 최대 하나의 레코드를 포함하며 블록보다 큰 레코드는 여러 블록으로 나뉜다. 각 블록의 시작 부분에는 무결성을 확인하기 위한 블록 내용의 checksum이 있다.

<p align="center"><img src="https://github.com/Student5421/Image/blob/main/Image/WAL_Image2.png" width=70%></p>

<br>

# Image soure
Image1: https://github-wiki-see.page/m/facebook/rocksdb/wiki/RocksDB-Overview
<br>
Image2: https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format#record-format

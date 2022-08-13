# WAL

## Option - Disable_WAL
LevelDB 의 경우 `db_bench` 를 실행하며 WAL 을 사용할지 말지에 대한 옵션이 존재하지 않는다. 반면 LevelDB 를 기반으로 만든 [RocksDB](https://github.com/facebook/rocksdb) 의 경우 `disable-wal` 옵션이 존재하여 이를 통해 WAL을 사용할지 말지에 대한 옵션을 선택할 수 있다.

이 게시글에서 사용하는 성능(Throughput, Latency) 비교의 경우, LevelDB가 아닌 RocksDB를 통해서 측정한 것을 미리 알린다.
### Hypothesis
- WAL 을 활성화하는 경우 latency 및 throughput 의 측면에서 부정적인 영향을 끼칠 것이다.

### Design
- RocksDB 의 `db_bench` 를 통해 다음의 benchmark 를 실행한다.
1. `fillseq`
2. `fillrandom`
3. `readrandom`

### Experiment Enviornment
#### CPU
Model: Intel(R) Core(TM) i9-7940X CPU @ 3.10GHz
Spec: 
```
Caches (sum of all):
  L1d:                   448 KiB (14 instances)
  L1i:                   448 KiB (14 instances)
  L2:                    14 MiB (14 instances)
  L3:                    19.3 MiB (1 instance)
```
#### OS & HW
- OS: Ubuntu 22.04 LTS (Not VM)
- Storage: Samsung SSD 860 2TB

### Result
RocksDB 에서 사용되는 `disable-wal` 옵션은 WAL 의 활성화 여부를 결정한다. 이를 통해서 Throughput, SAF, WAF, Latench (Average) 의 성능을 측정하였다. 다음의 내용은 RocksDB 의 `db_bench` 를 10회 실행 후 모든 결과의 평균에 대한 자료이다.

![wal_performance](https://user-images.githubusercontent.com/49092508/184468454-28357711-46af-400d-bb15-c9ad97bf9fd8.png)

위의 그래프를 표로 나타내면 다음과 같다.

|WAL|Throughput (MB/s)|SAF|WAF|Latency (Average)|Benchmark Type|
|--|--|--|--|--|--|
|Disabled|180.22|2.667|2|0.61416|`fillseq`|
|Disabled|77.33|1.625|1.7|1.43117|`fillrandom`|
|Enabled|51.73|2.06667|2|2.13972|`fillseq`|
|Enabled|36.17|1.625|1.7|3.06011|`fillrandom`|

### Discussion
RocksDB 의 경우 WAL을 작성할 시 `rocksdb::DBImpl::WriteToWAL` 을 호출한다. 해당 Member function 의 경우 호출될 시 다음의 절차가 발생한다.
![writetowal](https://user-images.githubusercontent.com/49092508/184468676-8c0ebc8f-2ab5-4350-87dc-94808d4d3d71.png)
> 위의 그림은 완벽한 UML class diagram 이 아닌 전반적인 흐름을 보여주기 위한 Diagram 이다.

해당 Diagram 을 통해 우리는 `rocksdb::DBImpl::WriteToWAL` 가 호출되었을 시 `std::write` 또는 `pwrite` 를 사용한다는 것을 알 수 있다. 따라서 WAL을 허용할 시 그만큼 더 많은 IO 작업을 요하고, 이런 IO 작업들이 많이 사용되기에 Latency, Throughput 의 Performance 에 큰 영향을 미치는 것을 알 수 있다.

다음의 그래프는 WAL 을 작성하는 시간을 측정한 그래프이다.

![wal3](https://user-images.githubusercontent.com/49092508/184468818-e39c1f08-ad7c-4239-8fdb-7df621b19744.png)

그래프의 내용과 같이, WAL 은 IO를 사용하기에 다소 Overhead 가 큰 작업임을 알 수 있다. 

#### Summary
1. WAL 의 장점
- 비정상적인 종료가 발생할 시 데이터를 잃지 않을 확률이 높아진다
2. WAL 의 단점
- IO 를 사용하기에 상당히 Overhead 가 크고 그렇기에 Latency 와 Throughput 의 측면에서 Performance 에 부정적인 쪽으로 영향을 준다.

## Option - Max_total_wal_size

### Hypothesis

### Design

### Experiment Enviornment

### Result

### Discussion

## Option - Manual_wal_flush

### Hypothesis

### Design

### Experiment Enviornment

### Result

### Discussion

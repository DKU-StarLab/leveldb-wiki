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

`DB::put`이 호출되면 데이터는 memtable에 쓰이고 WAL이 켜져 있다면 WAL에도 쓰인다. 
애플리케이션 메모리 버퍼에 먼저 기록되고 버퍼는 fwrite syscall을 호출하여 OS 버퍼로 플러시 한다. 
(OS 버퍼는 나중에 영구적인 저장장치로 동기화 된다.)
이런 경우 crush가 발생하면, RocksDB는 memtable에 있는 데이터만 복구가 가능하다.
기본적으로 RocksDB는 매번 put이 호출될 때마다 자동으로 WAL을 메모리에서 OS buffer로 flush 한다. 
그러나 `manual_wal_flush` 옵션을 끄면 `DB::FlushWAL` 이 호출 될 때만 flush가 되도록 수동적이게 설정할 수 있고 WAL 버퍼가 가득차거나 FlushWAL API가 호출될 때만 플러시 된다.
매번 put이 호출될 때마다 fwrite syscall을 하지 않는 것은 일반적으로 reliability와 latency에서 tradeoff가 있다.
데이터의 보존이 중요한 경우는 `manual_wal_flush` 옵션을 True로 설정해야 하고 데이터 손실을 감수하더라도 성능을 높이는 경우는 False로 설정을 하면 된다.

### Hypothesis

flush가 적게 될 수록 성능(latency)에는 유리하다. 그럼 어느 정도의 차이가 있는지 확인해보고 이를 고려해서 tradeoff를 따져야하기 때문에 직접 실험을 해서 확인해보았다.
manual_wal_flush 옵션을 true와 false로 각각 설정하여 throughput과 latency를 측정해보았다.
Benchmark는 fillseq과 fillrandom 일 때 모두 실험해봤는데 이 옵션은 WAL과 관련이 없기 때문에 기존의 fillseq과 fillrandom의 실험결과와 동일할 것이다.
추가적으로 sync/async(동기/비동기)인 경우에도 비교해보았다.

### Design
>Independent Variable : `manual_wal_flush` (true / false)  
Dependent Variable : Latency, Throughput  
Controlled Variable : Benchmark (fillseq, fillrandom), Sync mode (Sync / Async)  


### Experiment Enviornment
<p align="center"> <img src = ./img/wal_environment.png></p>


### Result
아래의 실험 결과는 각 실험을 5번씩 진행한 결과의 평균값이다.
<br/><br/>

<p align="center"> <img src = ./img/wal_table2.png></p> 

<br/><br/><br/>

- Async 
<p align="center"> <img src = ./img/wal_result4.png></p>  
비동기 모드에서의 Latency 와 Throughput 
  

<br/><br/>
- Latency 
<p align="center"> <img src = ./img/wal_result5.png></p>
비동기, 동기 모드에서의 Latency <br/>
<br/><br/>

### Discussion
>
    
    - manual_wal_flush 옵션에 따른 latency와 throughput  
      true일 때가 false일 떄에 비해 성능이 나쁘다. 

    - Benchmark (fillseq, fillrandom)   
      예상했던 것처럼 fillseq의 경우가 fillrandom에 비해 성능이 좋고 각  
      manual_wal_flush의 차이는 1000정도로 benchmark에 영향을 받지 않는 것을 볼 수 있다.

    - sync와 async에서의 performance
      일반적인 경우 manual_wal_flush가 true 일 때가 false 일 때 보다 성능이 낮아야  
      하는데 동기 모드에서의 latency는 3000 micros/op으로 압도적으로 높은 수치를  
      관찰할 수 있고 true인 경우가 false일 때 보다 성능이 좋게 나오기도 한다. 이는  
      manual_wal_flush 옵션이 sync모드에서는 영향을 주지 못하는 것을 알 수 있다.
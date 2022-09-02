# WAL

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

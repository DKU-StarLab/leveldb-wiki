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

column family들은 각각의 ssTable을 갖지만 WAL은 공유한다.   
하나의 column family가 flush 될 때마다 새 WAL이 생성된다.    
그리고 모든 column families에 대한 쓰기는 새 WAL로 이동한다.    
한편 WAL의 모든 데이터가 ssTable로 들어가야만 WAL이 삭제될 수 있으므로 정기적으로 flush 되도록 해야한다.   
만약 wal size의 제약이 없다면 wal이 삭제되는 속도가 느려지고 flush가 자주 일어나지 않을 것이다.    
그래서 max_total_wal_size 옵션을 사용하여 해당 옵션의 값만큼 크기를 초과하면 trigger되어 가장 오래된 라이브 WAL 파일을 삭제하는데, 만약 라이브 데이터가 있다면 강제로 flush한다.     
그런데 wal의 size를 작게하면 자주 flush가 일어나서 성능이 안좋아질까?     
   
### Design   
```   
Independent Variable: --max_total_wal_size=[int value]   
Dependent Variable: SAF, WAF, Latency, Throughput   

./db_bench --benchmarks="fillseq" --max_total_wal_size=[0,1,10,100,1000,10000,100000,1000000,10000000, 100000000] --num=10000000   
```   


### Experiment Enviornment   
*Server Spec*     
    -OS: macOS Monterey   
    -Processor: 2.3GHz 8Core Intel Core i9   
    -SSD: APPLE SSD AP1024N 1TB   


### Result    
![r1-1](https://drive.google.com/u/1/uc?id=1PYd69aCFcH0GaBVUIAKNit_4TPkdfqqI&export=download)   
![r1-2](https://drive.google.com/u/1/uc?id=12zU7vK_JZjnUA1FbO19ciofSQZPC4SkV&export=download)   
위의 환경에서 실험했을 때는 위의 결과 사진과 같이 유의미한 결과는 없었다.   

사실 이 ```max_total_wal_size```옵션은 기본 값(default)인 0이 실제 값 0이 아니라    

```[sum of all write_buffer_size * max_write_buffer_number] * 4```로 계산이 된다.   

다음은 ```options.h```파일의 일부분이다.   
![c-1](https://drive.google.com/u/1/uc?id=1VnoWgvvAfbxkFBnGIPu5Mc-oDHc8n8LQ&export=download)   

또한 이 옵션은 2개 이상의 column family가 존재해야 효과가 있다.   

그래서 10개, 15개의 column family를 가지고 다시 실험해 보았다.    
일단 0(default)값을 계산했다.   
```
10개의 column families,   
write_buffer_size = 64 MB(Default),   
max_write_buffer_number = 2(Default),   
max_total_wal_size = [10*64MB*2]*4 = 5.12GB   
```
```
15개의 column families,   
write_buffer_size = 64 MB(Default),   
max_write_buffer_number = 2(Default),   
max_total_wal_size = [15*64MB*2]*4 = 7.68GB   
```

다음은 실험 결과이다.    
참고로 옵션을 ```./db_bench --benchmarks="fillseq" --num_column_families= --max_total_wal_size=``` 이렇게 했다.   
   
![r2-1](https://drive.google.com/u/1/uc?id=1TLp8_9TqoELmEilbImaxPKwBFWTQHRAF&export=download)   
![r2-2](https://drive.google.com/u/1/uc?id=1Pcu91FwX1fdXh_-s9kArDSE83M9OPfkO&export=download)   
   
사실 이건 default column family를 분석한 결과만 반영된 것 같다.   
하지만 ```--num_column_families=``` 옵션을 주지 않았을 때와 결과가 다르기 때문에 10개, 15개의 패밀리가 모두 사용된 것 같다.   
   
max_total_wal_size의 범위를 특이하게 잡았는데,   
실험 시 범위를 너무 작게 잡으면 ```Too many open file```이라는 코멘트와 함께 벤치마크에 실패했다.   
또한 default값보다 커졌을 때의 변화도 궁금해서 1000GB까지 늘려봤는데 잘 모르겠다.  
   
### Discussion     
   
내 생각에는 default의 max_total_wal_size값에서 어느 정도 max_total_wal_size값이 작아질수록 성능이 안좋아지는 것 같다.   

또한 max_total_wal_size를 500MB로 고정하고 coulumn falimy의 개수를 10, 15, 20, 25, 30으로 테스트 해봤는데       
column family의 개수가 많아지면 wal을 공유하는데 있어 성능이 안좋아지는 것 같고    
성능이 안좋아지는 max_total_wal_size 값의 기준도 커지는 것 같다.   
   
기회가 된다면 여러 column family를 직접 세팅하고(각 column family의 특성을 바꿔서 다양한 변수를 가지고 실험할 수 있도록) 플러시의 횟수를 확인하는 등 좀 더 자세한 benchmarking을 시도해보고 싶다.   



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

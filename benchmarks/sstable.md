# SSTable

SSTable은 key-value pair들을 담는 Data Block, 각 Data Block들에 대한 블룸필터를 갖고 있는 Filter Block 등으로 구성된다. 이 때 [LevelDB 핸드북](https://leveldb-handbook.readthedocs.io/zh/latest/sstable.html)에는 Filter Block과 관련해 다음과 같이 적혀있다. 

<br/>

>  *사용자가 필터를 사용하도록 leveldb를 지정하지 않으면 leveldb는 이 블록에 내용을 저장하지 않습니다*  

<br/>  

즉 블룸필터를 사용하도록 지정하지 않으면 Filter Block에 데이터를 저장하지 않는다는 것이므로, *"그렇다면 블룸필터를 사용하도록 하지 않으면 Filter Block에 데이터를 쓰지 않는 거니까 쓰기(write)를 할 때 성능이 좀 더 좋아지지 않을까?"* 라는 생각이 들어서 이에 관한 실험을 하기로 했다.  
<br/>
실험을 진행하기 전, `uftrace`를 이용해 블룸필터를 적용했을 때만 `leveldb::FilterBlockBuilder::Addkey`함수가 호출되는 것을 보고 블룸필터를 쓰지 않으면 Filter Block에 데이터를 쓰지 않는다는 것이 사실임을 확인하고 실험을 진행했다.

## Hypothesis
블룸 필터를 적용하지 않으면 필터 블록에 데이터를 저장하지 않으므로, 쓰기 작업을 할 때 성능이 좀 더 향상될 것이다.  
`Latency`는 감소하고, `Throughput`은 증가할 것이다.  

## Design  
- Controlled Variables
  - `--value_size` : 2,000
  - `--use_existing_db` : 0
  - `--compression_ratio` : 1
  - `--benchmarks` : "fillseq"
- Independent Variables
  - `--bloom_bits` : 블룸필터를 적용하지 않을 땐 지정하지 않고, 블룸필터를 적용할 땐 64로 지정
- Dependent Variables
  - `Throughput`
  - `Latency` 
  
## Experiment Environment
- CPU : 40*Intel® Xeon® Silver 4210R CPU @2.40GHz
- CPUCache : 14080KB
## Result
블룸필터를 적용했을 때와 적용하지 않았을 때 각 10번씩 측정했다    
`Latency` : micros/op  
`Throughput` : MB/s


  <br/>  

- 블룸필터를 적용했을 때  
  
||1|2|3|4|5|6|7|8|9|10|
|-------|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|Latency|11.194|11.314|11.178|11.176|11.023|11.374|11.204|11.047|11.117|11.206|
|Throughput|171.8|169.9|172.0|172.0|174.4|169.0|171.6|174.0|172.9|171.6|  
> *Average Latency = 11.183micros/op*  
> *Average Throughput = 172.92MB/s*

 <br/>


- 블룸필터를 적용하지 않았을 때  
   

||1|2|3|4|5|6|7|8|9|10|
|-------|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|Latency|10.543|11.258|10.280|10.674|10.976|10.537|10.724|10.353|10.619|10.785|
|Throughput|182.4|170.8|187.0|180.1|175.2|182.5|179.3|185.7|181.1|178.3|  
> *Average Latency = 10.674micros/op*  
> *Average Throughput = 180.24MB/s* 

<br/>  

## Discussion  
블룸필터를 적용했을 때와 적용하지 않았을 때의 `Latency`와 `Throughput`을 비교하면 다음과 같다.  
<br/>  

- Latency
<p align="center"><img src="https://user-images.githubusercontent.com/65762283/181172862-7d4635c1-9263-4926-9f49-eb68fbf1fa23.png"></p><br/>

- Throughput  
<p align="center"><img src="https://user-images.githubusercontent.com/65762283/181174429-a12f8a94-176d-466a-b9b7-eb895f58295a.png"></p><br/>  

실험 전엔 블룸필터를 쓰지 않으면 쓰기를 할 때 `Latency`가 좀 더 낮아지고 `Throughput`이 좀 더 높아질 것이라 생각했는데, 실제로 블룸필터를 적용하지 않을 때가 적용했을 때에 비해 `Latency`가 낮게 나오고 `Throughput`은 높게 나오는걸 볼 수 있었다.<br/>  
그러나 실험 전에는 쓰기 작업을 할 때 블룸 필터를 적용하는 경우와 적용하지 않는 경우 간에 좀 더 큰 차이가 날 것이라 생각했는데, 실제로는 `Latency`의 경우 하나의 key-value pair를 넣는데 있어서 0.5micros정도밖에 차이나지 않았다. 이 차이가 그리 크게 느껴지지 않아서, 왜 큰 차이가 나지 않았던 걸까에 대해 생각해봤다.<br/>  
  

1. db_bench의 output  
db_bench를 실행해 출력된 결과는 엄밀히 말하면 하나의 SSTable을 만드는데 걸린 시간이 아니라, 하나의 key-value pair를 처리하는데 걸린 시간을 의미한다. 따라서 db_behcn의 출력결과는 SSTable을 만드는데 걸린 시간에 대한 지표가 아니라 전체적인 쓰기 과정에 대한 지표로 봐야 한다. 그러나 하나의 key-value pair를 넣을 때마다 하나의 SSTable이 만들어지는게 아니므로, SSTable이 써지는데 걸리는 시간차이는 전체적인 쓰기 과정에 큰 영향을 주지 않을 수도 있다. 때문에 하나의 SSTable을 처리하는데 걸린 시간차이는 눈에 띄게 났을지 몰라도 전체적인 쓰기 과정에선 그 차이가 크게 나지 않은 것일 수도 있다고 생각한다.
1. fillseq  
   이 실험을 진행할 때 `fillrandom`이 아니라 `fillseq`를 사용한 이유는 `fillrandom`으로 실행을 진행할 경우 블룸필터를 적용할 때와 적용하지 않을 때에 대해 동일한 순서로 key를 넣어준다는 보장이 없었기 때문이었다. `fillseq`로 수행할 때는 `compaction`이 일어나지 않는다는 특징이 있는데, 이로 인해 `fillseq`로 수행할 때는 `flush`로만 만들어지는 SSTable들에 대해서만 성능 차이가 나타나게 된다. 즉 `flush`와 `compaction`이 둘 다 일어나는 환경에선 SSTable들이 만들어지는 횟수가 더 많으므로 더 많은 성능차이가 나타날 수 있지만 본 실험은 `flush`만 일어나는 상황에서 진행됐기 때문에 SSTable이 만들어지는 횟수 자체가 적었고, 이로 인해 성능차이가 작게 나타난 것이라 생각한다.


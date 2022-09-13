# WAL

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




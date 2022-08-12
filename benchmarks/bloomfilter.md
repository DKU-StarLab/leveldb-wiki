#가설
>LevelDB가 이상적으로 생각하는 Bloom bits 개수는 10개이다.<n>
>key당 비트수가 커질수록 BloomFilter로 인한 성능이 향상되나
>BloomFilter를 처리하는데 필요한 overhead도 증가하기에
>최적의 Bloom bits 값은 10일 것이다.

  
  
## 1. fillrandom 측정
  |Bits per key |Throughput |Latency |
|---|---|---|
|0|29.26|3.78 |
|1|28.91|3.82 |
|5|28.92|3.82 |
|10|29.56|3.74 |
|30|26.72|4.15 |
|50|25.49|4.35 |
|100|25.46|4.36 |
|1000|19.11|5.78
  
  
  ![그림1](https://user-images.githubusercontent.com/101636590/183421596-8429999d-5ef0-4049-8893-61b1f4abadb2.png)
  
  ![그림2](https://user-images.githubusercontent.com/101636590/183421987-2de46c4f-a3d5-48a0-9120-2717196cd1c9.png)

  >Key 당 비트수가 커질수록 지연 시간이 늘어나고 처리율이 떨어지는 모습을 볼 수 있다.
  
  
  ## 2. readrandom 측정
  |Bits per key |Latency |
|---|---|
|0|3.29 |
|1|3.22 |
|5|2.73 |
|10|2.68 |
|30|2.68 |
|50|2.82 |
|100|3.05 |
|1000|3.80 |
  
 ![그림3](https://user-images.githubusercontent.com/101636590/183422548-271a9131-f466-45ea-b50a-f2cc49b406a2.png)
  
  >Key당 비트수가 늘어나자 지연시간이 감소했으나 이후 다시 증가했다.
  이는 초반엔 BloomFilter로 인해 성능이 향상되었으나, 이후 bloom bits가 너무 많아져 이를 처리하는데 너무 많은 연산을
  필요로 하기에 오히려 처리율이 떨어진 것으로 추정된다.
  
  
  ![그림4](https://user-images.githubusercontent.com/101636590/183422850-3879d47a-209e-4c00-9462-96f1cd8cd5b1.png)
  >또한 10이란 숫자는 쓰기 성능을 떨어뜨리지 않으면서 읽기 성능을 향상시키는 마지노선이라 
  LevelDB는 key당 10bits를 사용하는 것으로 추정된다.
  
  ![image](https://user-images.githubusercontent.com/101636590/183423221-a635e656-f46e-4fb3-b292-f71ac3f1c0ae.png)
  >그리고 key당 bit 수가 늘어날수록 false positive가 발생할 확률이 줄어들어,
  평균보다 성능이 뛰어난 경우가 더 자주 발생한다.

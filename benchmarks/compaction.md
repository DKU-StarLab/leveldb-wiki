# 실험 환경
> Key와 Value의 크기가 Compaction에 영향을 주게 된다 생각되어 실험을 하게되었습니다.


|Various **Key** Size |Various **Value** Size |
|---|---|
|16 byte|256 byte|
|32 byte|1 kb|
|64 byte|4 kb|
|128 byte|16 kb|

> benchmarks로는 Compaction이 발생하는 fillrandom과 readrandom만을 사용하게 되었습니다. 

|Experiment Setup | Benchmarks |
|---|---|
|version 1.23|fillrandom|
|CPU Intel Xeon(R)|readrandom|
||~~fillseq~~|
||~~readseq~~|

## 1.  가변적인 key size에서의 'fillrandom'
> 가변적인 key size에서의 fillrandom에서는 저희 예상으로 WAF laytency는 점점 높아지는것으로 예상하였으나 WAF는 정 반대로 key size가 작아지면서 점점 낮아지는 것을 확인하였습니다.

![image](https://user-images.githubusercontent.com/86946575/181190994-d47ec06f-3ca6-4589-9c33-b9b075662053.png)

## 2.  가변적인 value size에서의 'fillrandom'
> 가변 value size에서는 크기를 4배 씩 늘렸으니 성능 측면에서도 4배 씩 차이난 다고 생각하였으나 

>**Compaction latency**는 5.18배

>**latency**에서는  5.33배 차이 났습니다.
-  이유로는 value 값이 커지면서 Compaction이 자주 발생하여 이러한 결과가 나온거 같습니다.

![image](https://user-images.githubusercontent.com/86946575/181191389-3b6f2350-37fc-4d1c-9d3f-0c3f1925a744.png)

## 3. 가변적인 key/value size에서의 'readrandom'
> Read random에서의 가변 적인 value size에서는  4kb 랑  16kb에서 100배이상의 latency가 발생하였습니다.

![image](https://user-images.githubusercontent.com/86946575/181191670-7da09cdf-5b99-41e8-a8fa-6650fb9567e1.png)

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

## 예상치 못한 결과값에 의한 재실험
> fillrandom benchmark 실험에서 그래프의 변화율이 적절하지 못한 것으로 판단하여 재실험을 진행했습니다.  
> 동일한 실험을 3번 반복한 값의 평균을 구하여 전 실험과 비교해보았습니다.

## 재실험-1  가변적인 key size에서의 'fillrandom'
> 세번 반복한 값의 평균으로 측정하였습니다.  

![image](https://user-images.githubusercontent.com/106041072/188067454-ab99fc82-ee58-45c2-93d8-5e03cd8285a6.png)

> 그 결과 전실험과 동일하게 **WAF**가 계속 줄어들었습니다.  
> **WAF**가 예상과 다르게 왜 계속 줄어드는 지는 추후 따로 연구를 해보았습니다.

![image](https://user-images.githubusercontent.com/106041072/188067476-fd1f0d75-e041-45d7-ab62-395132b6d600.png)

## 재실험-2  가변적인 value size에서의 'fillrandom'
> 세번 반복한 값의 평균으로 측정하였습니다.  
> 또한 더 정확한 측정을 위해 구간을 좀 더 세분화하였습니다.

![image](https://user-images.githubusercontent.com/106041072/188064488-c6a8640d-c2f2-4d24-bca7-14b12cdbfba9.png)

> 그 결과 **latency**가 5.33배에서 2.5배로 좀 더 정밀한 결과를 얻을 수 있었습니다.

![image](https://user-images.githubusercontent.com/106041072/188066351-8090ecc3-813f-4afb-b960-ed4ae737f1c8.png)


> 위와 동일하게 3번 반복한 값의 평균을 구하고 구간을 더 세분화하여 비교해보았습니다.

![image](https://user-images.githubusercontent.com/106041072/188066684-fe2aa797-0280-4714-ab78-f52e89675a8a.png)

> 그 결과 **compaction latency**가  5.18배에서 3.5배로 좀 더 정밀한 결과를 얻을 수 있었습니다.

![image](https://user-images.githubusercontent.com/106041072/188066725-a0430329-c16c-4e85-a69b-8837ba8613b4.png)

## 결론 및 토의

> key/value size가 각각 compaction 에 영향을 준다는 가설을 토대로 실험을 진행하였습니다.  
> fillrandom의 경우, value size가 증가함에 따라 compaction 해야할 value가 많아지기 때문에 throughput은 감소하고 WAF, latency, compaction latency는 점점 증가하는 것을 알 수 있었습니다.  
> readrandom의 경우도 한번에 읽어야하는 value 가 많아지기 때문에 latency가 점점 증가하는 것을 알 수 있었습니다.  
> fillrandom의 경우에서 key size가 증가함에 따라 나오는 결과가 특이했습니다. key size 가 증가하면서 그 크기 안에 들어가는 key의 수는 동일한 데 size가 커졌기 때문에 key range는 더 커집니다(아래그림 참조). 따라서 compaction trigger 될때 메모리에 올리고 merge sort 후 다시 디스크에 내리는 크기가 늘어나기 때문에 latency, compaction latency 가 증가한 것으로 판단되고, key range 가 커졌지만 그 range 안에 들어가 있는 key의 수는 동일하기 때문에 더 빠르게 쓰기를 처리할 수 있어 WAF 감소하고, throughput는 증가하는 것으로 판단했습니다.

![image](https://user-images.githubusercontent.com/106041072/188073166-66f5514e-cc06-4131-bc36-67a923b129da.png)

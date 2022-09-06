# Bloom Filter

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424363-05494e10-e230-45b1-9a2a-18f413748970.png)
(출처:https://en.wikipedia.org/wiki/Bloom_filter) 

<br>

블룸 필터는 데이터 블록에 특정 key의 데이터가 존재하는지 빠르게 확인할 수 있는 확률적 자료 구조이다.

db_bench에선 bloom_bits의 값을 조절하여 자신이 원하는 크기의 블룸 필터를 만들어 사용할 수 있다.

이번 실험에서는 이 bloom_bits의 값에 따라 어떠한 차이가 발생하는지와 그 이유에 대해 알아보고자 한다.


<br/>
<br/>
<br/>
<br/>

# Hypothesis 1

블룸 필터의 크기가 커지면 false positive가 적게 발생하여 성능이 향상되나,

블룸 필터를 읽고 쓰는데 필요한 overhead도 같이 증가하는 tradeoff가 발생한다.

이 trade off를 고려하였을때, 최적의 bloom_bits 값은 LevelDB가 디폴트로 사용하는 값인 10 bits일 것이다.

<br/>
<br/>

# Fillrandom 성능 측정
  |Bits per key |Throughput |Latency |
|---|---|---|
|-1|28.21|4.327
|0|28.13|4.389 |
|1|28.01|4.472 |
|5|27.92|4.475 |
|10|27.56|4.486 |
|30|26.72|4.729|
|50|25.49|5.292 |
|100|25.46|5.382 |
|1000|19.11|7.781 |
  
![image](https://user-images.githubusercontent.com/101636590/188536205-ebae47e8-9f0e-46be-8c13-d98c026a67e2.png)
  
![image](https://user-images.githubusercontent.com/101636590/188535916-10cb4bd4-d76c-42d5-90bf-62369d71cbb3.png)

  Key 당 bits 수가 커질수록 지연 시간이 늘어나고 처리율이 떨어지는 모습을 볼 수 있다.
  
  <br/>
<br/>
  
# Readrandom 성능 측정
  |Bits per key |Latency |
|---|---|
|-1|3.20|
|0|2.84 |
|1|2.81 |
|5|2.78 |
|10|2.41 |
|30|2.97 |
|50|3.40 |
|100|3.56 |
|1000|5.69 |
  
![image](https://user-images.githubusercontent.com/101636590/188536768-d836e937-2788-44c4-b292-e89d8c09f535.png)
  
  Key당 bits 수가 늘어나자 지연시간이 감소했으나 이후 다시 증가했다.
  
  이는 초반엔 블룸 필터로 인한 성능 향상이 overhead보다 컸으나, 
  
  이후엔 블룸 필터로 인한 overhead가 너무 커져 오히려 처리율이 떨어진 것으로 추정된다.
  
  
<br/>
<br/>
  
  
  ![그림4](https://user-images.githubusercontent.com/101636590/183422850-3879d47a-209e-4c00-9462-96f1cd8cd5b1.png)
  <br>
  
  또한 10이란 숫자는 쓰기 성능을 크게 떨어뜨리지 않으면서 읽기 성능을 향상시키는 마지노선의 수치로 
  
  LevelDB는 해당 값인 key당 10bits를 디폴트 값으로 사용하는 것을 알 수 있다.
  
  <br/>
 <br/>
<br/>
<br/>
 <br/>
<br/>
<br/>

# Hypothesis 2

블룸 필터의 크기는 key당 bits수와 데이터의 갯수에 따라 결정된다.

그렇다면 해당 값들을 고정시키고 데이터의 크기만 변경하였을때

블룸 필터에 의한 overhead는 변하지 않을 것이다.


<br/>
<br/>

# Result

![image](https://user-images.githubusercontent.com/101636590/188540907-795ac9bf-348f-463a-a79d-c82a4fcb68dd.png)

 ![image](https://user-images.githubusercontent.com/101636590/188540987-7aa69da2-f7ec-4e7e-81cb-93f7f23eeda3.png)
 
 value_size가 128 Bytes일때, 블룸필터를 사용하자 쓰기 성능은 7% 감소, 읽기 성능은 7% 증가했다.
 
 이후 value_size를 1280 Bytes로 10배로 늘리자, 블룸 필터로 인한 쓰기 성능은 8% 감소, 읽기 성능은 9.5% 증가했다.
 
 <br>
 
 가설과 달리 value_size도 블룸 필터로 인한 성능 차이에 영향을 미치는 것으로 보여지는데,
 
 이는 블룸 필터의 전체 크기는 동일하나 각각의 블룸 필터들은 더 작 여러개로 나뉘었기에
 
 이를 읽고 쓰는데 추가적인 overhead를 필요로 한 것으로 추정된다.
 
 
   <br/>
 <br/>
<br/>
<br/>
 <br/>
<br/>
<br/>

# Hypothesis 3

블룸 필터의 key당 bits 수가 정해졌을때, 

수학적으로 증명된 최적의 hash 개수는 ln2 * bloom_bits ≈ 0.69 * bloom_bits 이다.

다만 hash의 개수는 소수가 될 수 없으므로

LevelDB의 디폴트 값인 10 bits를 사용한다면 hash의 개수는 6.9개가 아닌 6개가 된다.

그렇다면 이론상 최적의 값과 더 가까운 7개의 hash 함수를 사용한다면 성능의 개선이 있을 것이다.



<br/>
<br/>

# Result

![image](https://user-images.githubusercontent.com/101636590/188542665-f40355bf-2e3e-4926-913f-8deaabadcf42.png)

<br> 

측정 결과 거의 동일한 수치가 나왔으나 하나의 해시 함수를 추가로 수행해야 하는 만큼 

7개의 해시 함수를 사용한 경우가 쓰기 성능이 약간 떨어졌으며 읽기 성능은 약간 좋아였으나 매우 미세한 수치였다.

<br>
<br>

![image](https://user-images.githubusercontent.com/101636590/188542986-bc017ba8-4e05-4f2e-8e06-7f5757fe77f1.png)

이를 수학적으로 보자면 해시 함수를 6개를 사용했을 때와 7개를 사용했을 때

False Positive가 발생할 확률 차이는 0.00024로 매우 미비한 값이며

이는 읽기 중 해시를 한번 더 처리하면서 추가적으로 발생한 overhead와 거의 동등한 수치인 것으로 추정된다.


<br/>
<br/>
<br/>
 <br/>
<br/>
<br/>

# Hypothesis 4

그렇다면, 이론상 최적의 해시 함수 개수와 실제 사용 개수가 비슷한

9 bits ( 6.21 <-> 6 )나 12 bits ( 8.28 <-> 8 )를 사용한다면 성능의 개선이 있을 것이다.

<br/>
<br/>

# Result

![image](https://user-images.githubusercontent.com/101636590/188543384-32643815-612a-4f40-b5b1-4bd67103aacc.png)

측정 결과는 예상과는 달리, 10bits를 사용했을때 가장 좋은 결과가 나왔다.

<br>
<br>
<br>
<br>

![image](https://user-images.githubusercontent.com/101636590/188543466-a3e12c1c-77f9-42ce-a674-dc3214db5c11.png)

일단 12 bits와 8 hashes를 사용한 경우, 

디폴트 값인 10 bits와 6 hashes에 비해 false positive가 발생할 확률은 대폭 감소하였으나

2 bits와 2 hashes를 추가로 처리하는데 필요한 overhead를 감안하면 읽기 성능 차이는 크지 않았고

쓰기 성능은 되려 감소했다.

<br>
<br>
<br>

![image](https://user-images.githubusercontent.com/101636590/188544697-3e50231b-4709-41c5-a111-2c119a036bdf.png)

그리고 6개의 hash와 9개의 bits를 사용한 경우 같은 개수의 해시를 사용한 만큼 

디폴트의 경우와 쓰기 성능은 큰 차이가 없었으나 읽기 성능은 되려 감소하였는데, 

<br>
<br>
<br>

![image](https://user-images.githubusercontent.com/101636590/188544749-24cfabb8-f4ee-48ae-a9aa-379ee1b963d3.png)


이는 내 생각과 달리

최적의 hash 개수는 bloom_bits 값이 정해져있을 때 false positive를 가장 적게 발생 시키는 수치로,

반대로 hash의 개수가 정해져 있다면 false positive는 bloom_bits 값이 클수록 무조건 덜 발생하기 때문이였다.


<br/>
<br/>
<br/>
 <br/>
<br/>
<br/>

# Hypothesis 5

![image](https://user-images.githubusercontent.com/101636590/188545415-3b94a1a0-2577-4a61-83a1-66e939c6d8de.png)


위 실험 결과에서 같은 hash 개수면 bloom_bits 값이 큰 쪽이 읽기 성능이 좋단 점을 알았다.

그리고 hash 개수보다 bloom_bits 값이 false positive에 영향이 크단 점과

블룸 필터의 overhead는 hash의 비중이 크단 점을 고려하였을 때,

hash의 개수를 고정시키고 bloom_bits 값을 늘리면 더 좋은 성능 결과를 얻을 수 있을 것으로 추정된다.



<br/>
<br/>

# Result

![image](https://user-images.githubusercontent.com/101636590/188545779-24ac7ed8-dba5-41b7-b17a-33799ad49b42.png)

6개의 해시 함수에 각각 10개의 비트와 30개의 비트를 사용한 결과,

읽기 성능은 소폭 증가하였으나 쓰기 성능은 감소하였다.

다양한 변수를 조정하며 많은 측정을 진행하였지만,

디폴트 값인 10개의 bits에 6개의 hash 함수를 사용하는 것보다 좋은 결과를 얻는 것은 힘들 것으로 추정된다.




<br/>
<br/>
<br/>
 <br/>
<br/>
<br/>

# Hypothesis 6

False Positive는 일단 읽어야 할 데이터가 존재하지 않아야 발생할 수 있다.

그렇다면, 존재하지 않는 key 만을 찾는 ReadMissing 벤치마크에선 ReadRandom보다 False Positive의 영향이 클 것이다.


<br>
<br>
<br>

# Result
![image](https://user-images.githubusercontent.com/101636590/188546141-49a49cf8-3fc0-4bfd-816c-2900aa7bde5f.png)
![image](https://user-images.githubusercontent.com/101636590/188546151-a06edf9a-93a0-4ed3-b277-f24f39251422.png)

꾸준히 성능이 감소하는 FillRandom,

10 bits를 전후로 성능이 증가했다 감소하는 ReadRandom과 달리

ReadMissing은 블룸필터를 사용하자마자 성능이 대폭 증가하였으며,

비트 수를 계속 늘려도 꾸준히 성능이 좋아지는 것을 볼 수 있다.

이는 가설처럼 ReadMissing에선 False Positive 감소로 인한 성능 증가가

블룸필터 크기 증가로 인한 overhead보다 크기 때문으로 추정된다.






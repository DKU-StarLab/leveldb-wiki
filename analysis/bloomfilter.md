# Bloom Filter 
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424363-05494e10-e230-45b1-9a2a-18f413748970.png)
(출처:https://en.wikipedia.org/wiki/Bloom_filter) 

<br>

블룸 필터는 데이터 블록에 특정 key의 데이터가 존재하는지 확인할 수 있는 확률적 자료 구조이다.

블룸 필터는 데이터를 Write를 할 때에, 각각의 key에 k개의 해시 함수를 수행하여 각 key 마다 k개의 해시 값을 얻고,

해당 값에 해당하는 블룸 필터 배열 칸들의 값을 0에서 1로 수정하여 해당 key가 데이터 블록에 존재함을 나타낸다.

<br>

반대로 Read를 할 때엔 데이터 블록을 하나하나 전부 읽는 대신

읽으려는 데이터에 동일한 k개의 해시 함수를 수행하여 k개의 해시 값을 얻고

해당 배열 칸의 값이 전부 1인 데이터 블록만을 선별하여 읽는 것으로 Read 성능을 향상시키는 것이 가능하다.
<br><br><br><br>
   
# Bloom Filter Location
![sstable_logic](https://user-images.githubusercontent.com/101636590/188339431-c3f219ba-b2f0-4bc5-bbcf-a39a8be35d85.jpg)

(출처:https://leveldb-handbook.readthedocs.io/zh/latest/)

<br>

블룸 필터는 SSTable 내부에 존재하며 해당 SSTable은 위 사진과 같은 구조를 지닌다.

하나의 SSTable엔 n개의 데이터 블록과 1개의 필터 블록, 1개의 메타 인덱스 블록이 존재하며

필터 블록엔 데이터 블록의 개수와 같은 n개의 블룸 필터 배열이, 

메타 인덱스 블록은 각 블룸 필터 배열이 어느 데이터 블록의 배열인지 나타내는데 사용한다.<br>
  
  <br><br><br><br>
   
   
  
# True Negative & False Positive


![1604747913941](https://user-images.githubusercontent.com/101636590/188339451-c0638280-3882-4883-8396-d23c88008068.png)

(출처:https://www.linkedin.com/pulse/which-worse-false-positive-false-negative-miha-mozina-phd) 

<br>


블룸 필터의 가장 큰 장점은 True Negative가 절대로 발생하지 않는단 점이다.

True Negative는 데이터베이스에 존재하는 데이터를 존재하지 않는다 판단하는 것으로,

이는 필터의 신뢰성과 직결된 문제이므로 True Negative가 발생하지 않는단 점은 큰 장점이다.

<br>

다만 반대로 존재하지 않는 데이터를 존재한다 판단하는 False Positive의 경우엔 종종 발생할 수 있는데,

이는 True Negative보단 상대적으로 덜 중요한 문제이지만

False Positive로 인해 읽을 필요 없는 데이터 블록까지 읽어 성능이 떨어지게 되므로

False Positive를 줄이는 것 역시 블룸 필터의 중요한 과제 중 하나이다.
  
   
 
<br/>
<br/>
<br/>
<br/>


# Hash Function

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424697-ef93e101-a865-47a3-9e14-2046590dd9d9.png)

```cpp
void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;
```

LevelDB가 제공하는 벤치마킹 도구인 db_bench를 통해 측정을 진행할 때,

우리는 key 하나당 사용할 비트의 개수(Bloom_bits)나 쓰거나 읽을 데이터의 개수(Num) 등을 조절할 수 있다.

이때 생성되는 블룸 필터 배열의 크기는 (Bloom_bits) * (Num) bits가 된다.

이는 블룸필터의 핵심 코드 파일인 bloom.cc의 CreateFilter 클래스 함수에서 확인할 수 있다.

 
 <br>

참고로 db_bench에선 블룸 필터를 사용하지 않는 것이 디폴트이기에 

Bloom_bits 값을 지정하여 블룸 필터를 켜주어야 하며,

만약 블룸 필터를 사용한다면 LevelDB에서 이상적으로 생각하는 Bloom_bits 값은 10이다.

(이는 쓰기 성능을 크게 떨어뜨리지 않으면서, 읽기 성능을 향상시킬 수 있는 마지노선의 수치이다.)

 

<br/>
<br/>
<br/>
<br/>
 

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424934-cea90e5b-ca24-449a-90f4-53dc6452f539.png)

```cpp
explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
```

또한 해시 함수의 개수 K의 경우 K = ln2 * (M/N) = ln2 * B이란 수식을 사용한다.

이것은 수학적으로 증명된, 블룸 필터의 false positive 발생률을 최소한으로 줄일 수 있는 값이며

이 역시 bloom.cc 파일의 코드에서 구현되어 있는 것을 확인할 수 있다.

(이에 관한 더 자세한 증명은 https://d2.naver.com/helloworld/749531 해당 글을 참조)



<br/>
<br/>
<br/>
<br/>
 

![image](https://user-images.githubusercontent.com/101636590/188341669-1192eb93-3234-463d-8958-bb142251287e.png) <br>
![image](https://user-images.githubusercontent.com/101636590/188341828-490d75b7-17b1-455d-acba-ecabfb06dad3.png) <br>
![image](https://user-images.githubusercontent.com/101636590/188341913-5b0f489f-294a-4d5c-8171-d0ae7fa895cc.png) 

<br>

False positive가 발생할 확률을 수학적으로 정리하면 위와 같으며,

이를 e의 정의를 통해 정리하면 오른쪽 아래와 같은 식으로 표현할 수 있다.

<br>

여기서 k의 값이 ln2 * (m/n)이라면 ( = kn/m 값이 ln2라면)

해당 식은 (1/2)^k의 형태로 정리되므로,

<br>

해당 값이 false positive 발생 확률을 가장 최소화 시킬 수 있는 최적의 수치이며

또한 k의 값이 클수록 false positive가 발생할 확률이 작아짐을 알 수 있다.

(그리고 k = ln2 * b = 0.69 b 이므로 k의 값은 bloom_bits의 값과 비례한다.)


 <br/>
<br/>




# Code Flow of Bloom Filter
[Write : Bloom Filter의 생성](https://github.com/DKU-StarLab/leveldb-wiki/blob/main/analysis/bloomfilter-write.md)  
[Read : Bloom Filter로 특정 key의 존재 여부를 빠르게 확인](https://github.com/DKU-StarLab/leveldb-wiki/blob/main/analysis/bloomfilter-read.md) 





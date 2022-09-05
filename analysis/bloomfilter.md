# Bloom Filter <br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424363-05494e10-e230-45b1-9a2a-18f413748970.png)
(출처:https://en.wikipedia.org/wiki/Bloom_filter)

블룸 필터는 데이터 블록에 특정 key의 데이터가 존재하는지 확인할 수 있는 확률적 자료 구조이다.

블룸 필터는 데이터를 Write를 할 때에 각각의 key에 k개의 해시 함수를 수행하여 각 key 마다 k개의 해시 값을 얻고,

해당 값에 해당하는 블룸 필터 배열 칸들의 값을 0에서 1로 수정하여 해당 key가 데이터 블록에 존재함을 나타낸다.

그럴경우 반대로 Read를 할 때엔 데이터 블록을 하나하나 전부 읽는 대신

읽으려는 데이터에 동일한 k개의 해시 함수를 수행하여 k개의 해시 값을 얻고

해당 배열 칸의 값이 전부 1인 데이터 블록만을 선별하여 읽는 것으로 Read 성능을 향상시키는 것이 가능하다.
<br><br><br><br>
   
# Bloom Filter Format
![sstable_logic](https://user-images.githubusercontent.com/101636590/188339431-c3f219ba-b2f0-4bc5-bbcf-a39a8be35d85.jpg)

(출처:https://leveldb-handbook.readthedocs.io/zh/latest/)

하나의 sstable엔 n개의 데이터 블록과 1개의 필터 블록, 1개의 메타 인덱스 블록이 존재하며

필터 블록엔 n개의 블룸 필터 배열이, 메타 인덱스 블록은 각 블룸 필터 배열이 어느 데이터 블록의 필터인지 저장한다.<br>
  
  <br><br><br><br>
   
   
  
# False Positive


![1604747913941](https://user-images.githubusercontent.com/101636590/188339451-c0638280-3882-4883-8396-d23c88008068.png)

(출처:https://www.linkedin.com/pulse/which-worse-false-positive-false-negative-miha-mozina-phd)

블룸 필터의 장점은 True Negative가 절대로 발생하지 않는단 점이다.

True Negative는 데이터베이스에 존재하는 데이터를 존재하지 않는다 판단하는 것으로,

이는 필터의 신뢰성과 직결된 문제이므로 True Negative가 발생하지 않는단 점은 큰 장점이다.

<br>

다만 반대로 존재하지 않는 데이터를 존재한다 판단하는 False Positive의 경우엔 종종 발생할 수 있는데,

이는 True Negative보단 상대적으로 덜 중요한 문제이지만

False Positive로 인해 읽을 필요 없는 데이터 블록까지 읽어 성능이 떨어지게 되므로

False Positive를 줄이는 것 역시 블룸 필터의 중요한 과제이다.
  
   
 
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

db_bench에선 블룸 필터를 사용하지 않는 것이 디폴트이기에 Bloom_bits 값을 지정하여 블룸 필터를 켜주어야 하며,

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

이것은 수학적으로 증명된, 블룸 필터의 false positive 발생률을 최소한으로 줄일 수 있는 값이다.

이에 관한 더 자세한 증명은 https://d2.naver.com/helloworld/749531 해당 글을 참조하면 좋을 것이다.

이 역시 bloom.cc 파일의 코드에서 구현되어 있는 것을 확인할 수 있다.

<br/>
<br/>
<br/>
<br/>
 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183424987-d8a58b47-0e68-4f13-b3d1-bc7ea413406a.png)


False positive가 발생할 확률을 수학적으로 정리하면 위와 같으며,

이를 e의 정의를 통해 정리하면 오른쪽과 같은 식이 나온다.

여기서 k의 값이 ln2 * (m/n)이라면 해당 식은 (1/2)^k의 꼴로 정리되므로,

해당 값이 false positive 발생 확률을 가장 최소화 시킬 수 있는 최적의 수치이며

또한 k의 값이 클수록 false positive가 발생할 확률이 작아짐을 알 수 있다.

(그리고 k = ln2 * b = 0.69 b 이므로 k의 값은 bloom_bits의 값과 비례한다.)

<br/>
<br/>
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425038-94b8594d-a3c9-4325-8ddc-61f6c90c89b9.png)

이를 db_bench를 통해 실험해 본 결과 bloom_bits 값이 커질수록

false positive가 적게 발생하여 평균보다 성능이 빠른 경우가 더 많이 발생함을 알 수 있다.

(해당 경우는 노란색으로 표시)

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425050-e20cbdb2-f988-45aa-bb71-f423c77236f8.png)

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425059-75c0aa55-ee7a-4b32-9905-e97275d20966.png)



다만 bloom bits의 값이 너무 커질경우 오히려 false positive가 더 많이 발생하는 경향을 보였는데,

이는 bloom.cc 코드에서 해시 함수의 최대 개수를 30개로 제한하고 있기에

k = ln2 * b 라는 공식이 지켜지지 않아 되려 false positive가 증가한 것으로 추정된다.
<br/>
 <br/>
<br/>
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425778-ecfbef99-a13b-4efb-82ad-c666be9dc22e.png)


그렇기에 해시 함수 개수를 제한하는 코드 부분을 제거하고 다시 db_bench를  본 결과,

모든 ReadRandom의 측정값이 평균과 비슷한 값이 나옴을 알 수 있는데

이는 false positive의 발생 확률이 대폭 낮아져서 false positive가 거의 발생하지 않았기 때문으로 추정된다.

허나 이 경우엔 한번의 read에 무려 690번의 해시를 처리해야 하므로 (전자의 경우엔 30번)

false positive 자체는 덜 발생하였으나 이를 위한 더 많은 해시 함수를 처리하는데 필요한 overhead가 더 컸기에

되려 성능이 떨어진 것을 볼 수 있다.

 

 <br/>
<br/>
<br/>
<br/>

# Code of Bloom Filter

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425832-0ca0a6f6-a8d4-471b-8765-44a3ccad1904.png)


LevelDB의 전체 코드엔 약 100개 가량의 메인 함수가 존재한다.

여기서 db_bench의 makefile을 확인해 보면 db_bench의 메인 함수는

db_bench.cc 파일에 존재함을 알 수 있다.

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425880-7bcff039-5faa-4e42-9ad9-763c12f6c614.png)


db_bench.cc 파일의 메인 함수는 크게 2개의 파트로 구성되는데,

첫번째는 db_bench를 실행할때 bloom_bits, num과 같은 파라미터를 읽어들이는 sscanf 파트,

그리고 benchmark.Run()이란 클래스 함수 두 파트로 구성된다.

  <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425930-e7017bb8-d77e-428a-a4c1-fa9e1b5288ca.png)

benchmark.Run()도 크게 3가지 파트로 나눌 수 있는데,

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425988-2d5638f4-ba97-4a54-9f5e-ddc3529dbe7a.png)

벤치마크 클래스에서 filter policy라는 클래스 변수가 선언되며

<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183425998-87fd0957-3bc3-4c7f-9239-ebf597053cf7.png)

이후 해당 클래스의 생성자에서 filter policy 변수의 값을 지정한다.

메인함수에서 sscanf로 읽은 bloom_bits 값이 0 이상이면 

블룸 필터를 사용한다는 것이니 NewBloomFilterPolicy 함수를 호출하고,

아닐 경우엔 블룸 필터를 사용하지 않는다는 것이니 Null을 리턴한다.

(bloom_bits의 디폴트 값은 -1로 따로 설정하지 않으면 블룸 필터를 사용하지 않으며,

0으로 지정한다면 최소한의 크기의 블룸 필터를 사용하게 된다.)

 <br/>
<br/>
<br/>
<br/>


![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426008-acd92e3d-9adf-4d7b-8986-2c248e6705b1.png)

NewBloomFilterPolicy()는 "FilterPolicy" 클래스를 오버라이드한 "BloomFilterPolicy" 클래스를 생성하여 리턴하는 함수인데,

  <br/>
<br/>



```CPP
class LEVELDB_EXPORT FilterPolicy {
 public:
  virtual ~FilterPolicy();

  // Return the name of this policy.  Note that if the filter encoding
  virtual const char* Name() const = 0;

  // Append a filter that summarizes keys[0,n-1] to *dst.
  virtual void CreateFilter(const Slice* keys, int n,
                            std::string* dst) const = 0;

  // CreateFilter() on this class.  This method must return true if
  // the key was in the list of keys passed to CreateFilter().
  virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};
```

FilterPolicy 클래스의 경우

필터의 이름을 리턴하는 Name() 함수

Write할 때 블룸 필터 배열을 생성하는 CreateFilter() 함수 

그리고 Read할 때 배열에 특정 key 값이 존재하는지 확인하는 KeyMayMatch()

3개의 클래스 함수로 구성되어있다.

이를 오버라이드한 BloomFilterPolicy 클래스의 경우

<br/>
<br/>

```CPP
class BloomFilterPolicy : public FilterPolicy {
 public:
  explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }

  const char* Name() const override { return "leveldb.BuiltinBloomFilter2"; }
```

해시 함수의 개수를 정하고 최대 개수를 제한하는 코드 부분과

특정 이름을 리턴하는 Name() 함수

<br/>
<br/>

```CPP
void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

```

그리고 CreateFilter() 함수는 bloom_bits(=bits_per_key)값과 num(=n)값을 곱해

블룸 필터 배열의 크기(=bits)를 정하는 파트와

<br/>
<br/>

```CPP
const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
    char* array = &(*dst)[init_size];
    for (int i = 0; i < n; i++) {
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      for (size_t j = 0; j < k_; j++) {
        const uint32_t bitpos = h % bits;
        array[bitpos / 8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
 ```

"더블 해싱"을 처리하는 파트로 나뉘게 된다.

  <br/>
<br/>
<br/>
<br/>

# Double Hashing

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426381-db205ffc-c946-49af-a75d-2b0869145737.png)
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426385-eb7ead05-1359-4802-9bc6-0f2d892756bb.png)


(출처:https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)

해당 함수의 주석에 작성되어 있는 논문을 참고하면,

기존의 k개의 서로 다른 해시 함수를 사용하는 방식 대신

2개의 해시 함수와 그 중에서 하나를 중복 사용하는 "더블 해싱" 방식을 통해 

해시 함수로 인한 부하와 연산을 대폭 줄이면서 기존의 성능을 유지할 수 있음을 수학적으로 증명하고있다.

<br/>
<br/>

 
![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426427-2a7c8496-b118-42aa-b6da-112205d9b581.png)

LevelDB에서 첫번째 해시 함수는 BloomHash()란 함수로,

두번째 해시 함수는 ( h>>17 ) | ( h<<15 ) 란 비트 연산으로 구현되어있다.

 
<br/>
<br/>

```CPP
namespace {
static uint32_t BloomHash(const Slice& key) {
  return Hash(key.data(), key.size(), 0xbc9f1d34);
}
```
```CPP
uint32_t Hash(const char* data, size_t n, uint32_t seed) {
  // Similar to murmur hash
  const uint32_t m = 0xc6a4a793;
  const uint32_t r = 24;
  const char* limit = data + n;
  uint32_t h = seed ^ (n * m);
```

BloomHash는 key 값을 인자로 특정 값을 리턴하는 전형적인 해시함수의 형태를 취하고 있으며,

 
<br/>
<br/>

![img1 daumcdn](https://user-images.githubusercontent.com/101636590/183426489-099b0b6f-8a1c-4263-a575-60407a8b8c65.png)


쉬프트 연산은 앞의 15자리와 뒤의 17자리를 바꾸는 형태로 구성되어있다.

(참고로 해시값은 uint32_t라는 32비트 자료형을 사용한다.)

이러한 쉬프트 연산은 해시 함수의 성질을 만족하면서도

연산에 필요한 overhead가 적어 사용되는 것으로 추정된다.

  <br/>
<br/>
<br/>
<br/>


```CPP
uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
```


그리고 특이사항으론 hash 하나당 하나의 bit를 1로 바꿔야 하는데,

블룸 필터 배열은 char 배열로 한 칸의 크기가 1 byte = 8 bits 이므로

배열의 한 칸을 8칸으로 쪼개서 사용하고 있음을 알 수 있다.

 <br/>
<br/>
<br/>
<br/>



```CPP
bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8;

    // Use the encoded k so that we can read filters generated by
    // bloom filters created using different parameters.
    const size_t k = array[len - 1];
    if (k > 30) {
      // Reserved for potentially new encodings for short bloom filters.
      // Consider it a match.
      return true;
    }
```


데이터를 Read할때 사용하는 KeyMayMatch 함수의 경우

CreateFilter와 거의 동일한 연산을 사용하나

or  대신 and 연산을 사용하여 필터 내부에 특정 key 값이 존재하는지 확인하는 것을 볼 수 있다.

   <br/>
<br/>
<br/>
<br/>

# Code Flow of Bloom Filter
[Write : Bloom Filter가 만들어지는 과정](https://github.com/DKU-StarLab/leveldb-wiki/blob/main/analysis/bloomfilter-write.md)  
[Read : Bloom Filter로 특정 key를 찾는 과정](https://github.com/DKU-StarLab/leveldb-wiki/blob/main/analysis/bloomfilter-read.md) 





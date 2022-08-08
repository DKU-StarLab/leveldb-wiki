<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@wCvaj/btrJgeUP7nR/0j2FYzv2l9ffNKBhAa8Xc1/img.png|CDM|1.3|{"originWidth":1821,"originHeight":655,"style":"alignCenter","caption":"(사진 출처: https://ko.wikipedia.org/wiki/%EB%B8%94%EB%A3%B8_%ED%95%84%ED%84%B0)"}_##]</p>
<p data-ke-size="size16">블룸 필터는 데이터 블록에 특정 key 값의 존재 여부를 빠르게 확인할 수 있는 확률적 자료 구조이다.</p>
<p data-ke-size="size16">Write를 할 때엔 각각의 key에 k개의 해시 함수를 수행하여 각 key 마다 k개의 해시 값을 얻고,</p>
<p data-ke-size="size16">해당 값에 해당하는 블룸 필터 배열 칸의 값을 0에서 1로 수정하여</p>
<p data-ke-size="size16">해당 데이터 블록에 해당 key 값이 존재함을 나타낸다.</p>
<p data-ke-size="size16">반대로 Read를 할 때엔 읽으려는 key에 똑같은 k개의 해시 함수를 수행하여 k개의 해시 값을 얻고</p>
<p data-ke-size="size16">해당 배열 값이 전부 1인 데이터 블록만을 선별하여 읽는 것으로 읽기 증폭을 대폭 감소시킨다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@bFTVn1/btrJfoRh5xH/kIz7MS97wXKC86aOIKR8CK/img.png|CDM|1.3|{"originWidth":483,"originHeight":447,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">참고로 하나의 sstable엔 n개의 데이터 블록과 1개의 필터 블록, 1개의 메타 인덱스 블록이 존재하며</p>
<p data-ke-size="size16">필터 블록엔 n개의 블룸 필터 배열이, 메타 인덱스 블록은 각 블룸 필터 배열이 어느 데이터 블록의 필터인지 저장한다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_ImageGrid|kage@bj7gnd/btrJfpbAacD/nKIAH8xI1eyBHdvgIVHBTk/img.png,kage@85cyI/btrJe5KQgjO/0f1vpIji0XP0YW45USrf2K/img.png|width="420" height="369" data-origin-width="701" data-origin-height="588" data-is-animation="false" data-widthpercent="58.38" data-filename="blob" style="width: 57.7017%; margin-right: 10px;",data-origin-width="436" data-origin-height="513" data-is-animation="false" data-widthpercent="41.62" style="width: 41.1355%;"|_##]</p>
<p data-ke-size="size16">블룸 필터의 장점은 True Negative가 발생하지 않는단 점이다.</p>
<p data-ke-size="size16">블룸 필터를 통해 특정 key의 값이 존재하지 않는다 판단되면,</p>
<p data-ke-size="size16">해당 key 값은 절대로 해당 데이터 블록에 존재하지 않는다.</p>
<p data-ke-size="size16">이는 필터의 신뢰성과 직결된 문제이므로 True Negative가 발생하지 않는단 점은 큰 장점이다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">다만 반대로 존재하지 않는 값을 존재한다 판단하는 False Positive의 경우 종종 발생할 수 있는데,</p>
<p data-ke-size="size16">이는 True Negative보단 덜 중요한 문제이지만</p>
<p data-ke-size="size16">False Positive로 인해 읽을 필요 없는 데이터 블록까지 읽어 성능이 떨어지므로</p>
<p data-ke-size="size16">False Positive를 줄이는 것 역시 블룸 필터의 중요한 과제이다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@dIQwcF/btrJaV949Xr/MDzaYunhoVB6HyKKNUKgJ1/img.png|CDM|1.3|{"originWidth":858,"originHeight":211,"style":"alignCenter"}_##][##_Image|kage@qJdtt/btrJfquILYy/PAkIbowCqNstCUy9qaD3Dk/img.png|CDM|1.3|{"originWidth":1780,"originHeight":229,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">db_bench를 통해 실험을 돌릴 때,</p>
<p data-ke-size="size16">우리는 key당 bits 값 (Bloom_bits)이나 쓰거나 읽을 데이터의 갯수 (Num)를 조절할 수 있다.</p>
<p data-ke-size="size16">이때 생성되는 블룸 필터 배열의 크기는 B*N bits가 된다.</p>
<p data-ke-size="size16">이는 블룸필터의 핵심 코드 파일인 bloom.cc의 CreateFilter 클래스 함수에서 확인할 수 있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">db_bench에선 블룸 필터를 사용하지 않는 것이 디폴트 값이기에 Bloom_bits 값을 지정하여 블룸 필터를 켜주어야 하며,</p>
<p data-ke-size="size16">만약 블룸 필터를 사용한다면 LevelDB에서 이상적으로 생각하는 Bloom_bits 값은 10이다.</p>
<p data-ke-size="size16">(이는 쓰기 성능을 떨어뜨리지 않으면서, 읽기 성능을 향상시킬 수 있는 마지노선의 수치이다.)</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@DHJxR/btrJfZcxQcD/DgcnSnb8NJohv9DyGpJYjk/img.png|CDM|1.3|{"originWidth":862,"originHeight":198,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@CaMSI/btrJeqhrRfK/d0Ms1eIq2ofkRm7NlKHCZ1/img.png|CDM|1.3|{"originWidth":702,"originHeight":104,"style":"widthContent"}_##]</p>
<p data-ke-size="size16">또한 해시 함수의 개수 K의 경우 K = ln2 * (M/N) = ln2 * B이란 수식을 사용한다.</p>
<p data-ke-size="size16"><span style="background-color: #fcfcfc; color: #404040;"><span>이것은 수학적으로 증명된, 블룸 필터의 false positive를 최소한으로 줄일 수 있는 값이다.</span></span></p>
<p data-ke-size="size16"><span style="background-color: #fcfcfc; color: #404040;"><span>이에 관한 더 자세한 증명은 <a href="https://d2.naver.com/helloworld/749531" target="_blank" rel="noopener">https://d2.naver.com/helloworld/749531</a> 해당 글을 참조.</span></span></p>
<p data-ke-size="size16">이 역시 bloom.cc 파일의 코드에서 구현되어 있는 것을 확인할 수 있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@dvWAhN/btrJfY5OELl/P36KqT6lcMJ7NW1WbRuyok/img.png|CDM|1.3|{"originWidth":835,"originHeight":206,"style":"alignCenter","width":730,"height":180}_##]</p>
<p data-ke-size="size16">false positive가 발생할 확률을 수학적으로 정리하면 위와 같으며,</p>
<p data-ke-size="size16">이를 e의 정의를 통해 정리하면 오른쪽과 같은 식이 나온다.</p>
<p data-ke-size="size16">여기서 k의 값이 ln2 * (m/n)이라면 해당 식은 (1/2)^k의 꼴로 정리되므로</p>
<p data-ke-size="size16">k의 값이 클수록 false positive가 발생할 확률이 작아짐을 알 수 있다.</p>
<p data-ke-size="size16">(그리고 k = ln2 * b = 0.69 b 이므로 b가 증가하면 k가 증가한다.)</p>
<p>[##_Image|kage@oyfOi/btrJgaSpnKe/P1tg4cnge2yClLR578nLX1/img.png|CDM|1.3|{"originWidth":1251,"originHeight":720,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">이를 db_bench를 통해 실험해 본 결과 bloom_bits 값이 커질수록</p>
<p data-ke-size="size16">false positive가 적게 발생하여 평균보다 성능이 빠른 경우가 더 많이 발생함을 알 수 있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@5nTi7/btrI9mNHi0B/9WSkbKK9GfpyzB30Kgi4V0/img.png|CDM|1.3|{"originWidth":1266,"originHeight":728,"style":"alignCenter"}_##][##_Image|kage@nQFjt/btrJgOas3FZ/8xz7Ms5x04VIhCMAEJW3KK/img.png|CDM|1.3|{"originWidth":967,"originHeight":316,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">다만 bloom bits가 너무 커질경우 오히려 false positive가 더 많이 발생하는 경향을 보였는데,</p>
<p data-ke-size="size16">이는 bloom.cc 코드에서 해시 함수의 최대 개수를 30개로 제한하고 있기에</p>
<p data-ke-size="size16">k = ln2 * b 라는 공식이 지켜지지 않아 되려 false positive가 증가한 것으로 추정된다.</p>
<p>[##_Image|kage@C5nY4/btrJcF6012O/nBdJxaMLUADrrFQT53BsBK/img.png|CDM|1.3|{"originWidth":808,"originHeight":478,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">해시 함수 갯수 제한을 제거하고 다시 db_bench를 돌려 본 결과,</p>
<p data-ke-size="size16">모든 read 측정값이 평균과 비슷한 값이 나옴을 알 수 있다.</p>
<p data-ke-size="size16">이는 false positive의 발생 확률이 대폭 낮아져서 false positive가 거의 발생하지 않았기 때문으로 추정된다.</p>
<p data-ke-size="size16">허나 이 경우엔 한번의 read에 690번의 해시를 처리해야 하므로 (전자의 경우엔 30번)</p>
<p data-ke-size="size16">false positive 자체는 덜 발생하였으나 이를 위해 read를 처리하는데 필요한 시간은 오히려 증가하였음을 알 수 있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@cygSBi/btrJgd9xVLC/25aJKpNVxcE0DCUxk9HPi0/img.png|CDM|1.3|{"originWidth":837,"originHeight":320,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">LevelDB 전체 코드엔 약 100개 가량의 메인 함수가 존재한다.</p>
<p data-ke-size="size16">여기서 db_bench의 makefile을 확인해 보면 db_bench의 메인 함수는</p>
<p data-ke-size="size16">db_bench.cc 파일에 존재함을 알 수 있다.</p>
<p>[##_Image|kage@DsmAi/btrI9lnHtHY/638M1wFCKxbSrB8Uiswkuk/img.png|CDM|1.3|{"originWidth":1831,"originHeight":799,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">db_bench.cc 파일의 메인 함수는 크게 2개의 부분으로 구성되는데,</p>
<p data-ke-size="size16">첫번째는 db_bench를 실행할때 bloom_bits, num과 같은 파라미터를 읽어들이는 sscanf 파트,</p>
<p data-ke-size="size16">그리고 benchmark.Run()이란 클래스 함수 수행 두 파트로 구성된다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@dgibfI/btrJaXfOiNc/APi60RIe3hQHtyXfD4GWs1/img.png|CDM|1.3|{"originWidth":676,"originHeight":296,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">benchmark.Run()도 크게 3가지 부분으로 나눌 수 있는데,</p>
<p>[##_Image|kage@bLNAVj/btrJfpinrDo/d4KaAiAy8MZWn9c1wi7d0K/img.png|CDM|1.3|{"originWidth":758,"originHeight":323,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">벤치마크 클래스에서 filter policy라는 클래스 변수가 선언되며</p>
<p>[##_Image|kage@2qPFN/btrJfqBAjon/E41o0KrNSPiwbeEjb8rf6K/img.png|CDM|1.3|{"originWidth":811,"originHeight":280,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">이후 벤치마크 생성자에서 filter policy의 내부를 채운다.</p>
<p data-ke-size="size16">메인함수에서 sscanf로 읽은 bloom_bits 값이 0 이상이면</p>
<p data-ke-size="size16">블룸 필터를 사용한다는 것이니 NewBloomFilterPolicy 함수를 호출하고,</p>
<p data-ke-size="size16">아닐 경우엔 블룸 필터를 사용하지 않는다는 것이니 Null을 리턴한다.</p>
<p data-ke-size="size16">(bloom_bits의 디폴트 값은 -1로 따로 설정하지 않으면 블룸 필터를 사용하지 않는다.)</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@lCsbw/btrJfqn2Bfc/JpA2uL8AXR83xVXNDQsBG1/img.png|CDM|1.3|{"originWidth":729,"originHeight":352,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">NewBloomFilterPolicy()는 FilterPolicy 클래스를 오버라이드한 BloomFilterPolicy 클래스를 생성하여 리턴한다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@b8wA0s/btrI9lBfRmf/Fy20HUacFVMmL9DwFX6Ec0/img.png|CDM|1.3|{"originWidth":1398,"originHeight":587,"style":"alignCenter","caption":"https://github.com/google/leveldb/blob/main/include/leveldb/filter_policy.h"}_##]</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">FilterPolicy 클래스는</p>
<p data-ke-size="size16">필터의 이름을 리턴하는 Name(),</p>
<p data-ke-size="size16">Write할 때 블룸 필터 배열을 생성하는 CreateFilter(),</p>
<p data-ke-size="size16">Read할 때 배열에 특정 key 값이 존재하는지 확인하는 KeyMayMatch() ,</p>
<p data-ke-size="size16">3개의 클래스 함수로 구성되어있다.</p>
<p data-ke-size="size16">이를 오버라이드한 BloomFilterPolicy 클래스의 코드를 살펴보자면</p>
<p>[##_Image|kage@Ya9Sf/btrJgaZa35v/V8Z4AoYp4mHkoJyHVnELkK/img.png|CDM|1.3|{"originWidth":539,"originHeight":211,"style":"alignCenter","caption":"https://github.com/google/leveldb/blob/main/util/bloom.cc"}_##]</p>
<p data-ke-size="size16">해시 함수의 갯수를 정하고 최대 갯수를 제한하는 코드 부분과</p>
<p data-ke-size="size16">이름을 리턴하는 Name() 함수</p>
<p>[##_Image|kage@nNbEV/btrJhjBsZoF/5Lz05GkK3gUuKeU9iWvCw1/img.png|CDM|1.3|{"originWidth":531,"originHeight":214,"style":"widthContent"}_##]</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">그리고 CreateFilter 함수는 bloom_bits(bits_per_key)값과 num(n)값을 곱해</p>
<p data-ke-size="size16">블룸 필터 배열의 크기(bits)를 정하는 파트와</p>
<p>[##_Image|kage@cZ4VlK/btrI3yHl0JA/H0fOCkJrY1kIKulgWQAyX1/img.png|CDM|1.3|{"originWidth":1306,"originHeight":706,"style":"widthContent"}_##]</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">"더블 해싱"을 처리하는 파트로 나뉜다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@I4T60/btrJik1bbiC/UXQq8amKazSNH9zm5uJtmk/img.png|CDM|1.3|{"originWidth":1587,"originHeight":198,"style":"alignCenter"}_##][##_Image|kage@WOj9O/btrJeqV3OJB/fkURNQI09jQw4S98LX27K0/img.png|CDM|1.3|{"originWidth":678,"originHeight":222,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16"><a href="https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf" target="_blank" rel="noopener">https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf</a></p>
<p data-ke-size="size16">해당 함수의 주석에 작성되어 있는 논문을 참고하면,</p>
<p data-ke-size="size16">하나의 key에 k개의 해시 함수를 사용하는 대신 2개의 해시 함수를, 그 중에서 하나는 중복 사용하는</p>
<p data-ke-size="size16">"더블 해싱"을 통해 기존의 블룸 필터와 비슷한 false positive 확률을 유지한 채&nbsp;</p>
<p data-ke-size="size16">해시 함수로 인한 부하와 연산을 대폭 줄일 수 있음을 수학적으로 증명하고있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@mO2bN/btrJgcvTS1W/XgCgLzkZS9qkFCZh3p2kr1/img.png|CDM|1.3|{"originWidth":786,"originHeight":355,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">LevelDB에서 첫번째 해시 함수는 BloomHash()란 함수로,</p>
<p data-ke-size="size16">두번째 해시 함수는 ( h&gt;&gt;17 ) | ( h&lt;&lt;15 ) 란 비트 연산으로 구현되어있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@vcams/btrJik1boKg/KTrZWYznxCtKhoUhCMCSJ0/img.png|CDM|1.3|{"originWidth":1190,"originHeight":358,"style":"alignCenter","width":651,"height":196}_##][##_Image|kage@lwbLx/btrJaVPQfpZ/dvl4thaL0KTEdTtj0GKz80/img.png|CDM|1.3|{"originWidth":1117,"originHeight":253,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">BloomHash는 key 값을 인자로 특정 값을 리턴하는 전형적인 해시함수의 형태를 취하고 있으며,</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@EJFP0/btrJgxfFi25/RT6b36jAdXpYdGlc43A42K/img.png|CDM|1.3|{"originWidth":681,"originHeight":154,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">쉬프트 연산은 앞의 15자리와 뒤의 17자리를 바꾸는 형태로 구성되어있다.</p>
<p data-ke-size="size16">(참고로 해시값은 uint32_t라는 32비트 자료형을 사용한다.)</p>
<p data-ke-size="size16">이러한 쉬프트 연산은 input 값에 따라 output 값이 정해져있으나</p>
<p data-ke-size="size16">input 값이 유사해도 전혀 다른 output 값이 나온다는 해시 함수의 조건을 만족하면서도</p>
<p data-ke-size="size16">필요한 오버헤드가 가장 적어서 사용되는 것으로 추정된다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@GpHL5/btrJgw8SNEI/RCTmW57kkuXlXVqIcaUSW1/img.png|CDM|1.3|{"originWidth":472,"originHeight":191,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">그리고 특이사항으론 hash 하나당 하나의 bit를 1로 바꾸는데,</p>
<p data-ke-size="size16">블룸 필터 배열은 char 배열로 한 칸의 크기가 1 byte = 8 bits 이므로</p>
<p data-ke-size="size16">배열의 한 칸을 8칸으로 쪼개서 사용하고 있음을 알 수 있다.</p>
<p>[##_Image|kage@em0p6K/btrJaWOODn1/zuGy0mK4JicL8EsWvrx3A1/img.png|CDM|1.3|{"originWidth":541,"originHeight":506,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">데이터를 Read할때 사용하는 KeyMayMatch 함수의 경우</p>
<p data-ke-size="size16">CreateFilter와 거의 동일한 연산을 사용하나</p>
<p data-ke-size="size16">or&nbsp; 대신 and 연산을 사용하여 필터 내부에 특정 key 값이 존재하는지 확인하는 것을 볼 수 있다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@l5sZP/btrJcGrsdld/lkkVy0E2EOzeQC12IuqAg1/img.png|CDM|1.3|{"originWidth":669,"originHeight":291,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">다시 코드 흐름을 살펴보자면,</p>
<p data-ke-size="size16">class Benchmark에서 클래스 변수 filterpolicy를 선언하고</p>
<p data-ke-size="size16">생성자 Benchmark()에서 해당 변에 Bloomfilterpolicy 값을 채웠다.</p>
<p data-ke-size="size16">이제 이를 바탕으로 Run() 클래스 함수를 실행하게 된다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@bJJQG9/btrJe633Ig1/npR1dpJPq7Ajx2HyQjzCrK/img.png|CDM|1.3|{"originWidth":665,"originHeight":317,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">Run() 클래스 함수 역시 크게 3가지 부분으로 나뉘는데</p>
<p>[##_Image|kage@WL5Vg/btrJgduY9Ad/KL0nfcn14CE1ET4TD9sEnK/img.png|CDM|1.3|{"originWidth":778,"originHeight":355,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">PrintHeader()는 db_bench를 돌리는 환경이나 key-vaule 설정을 출력해주는 함수이다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@sTyb5/btrI6c47teS/kOyRFwjXRK7GrBf45YPYsK/img.png|CDM|1.3|{"originWidth":591,"originHeight":364,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">그 다음 Open 함수의 경우</p>
<p data-ke-size="size16">Options이란 클래스를 새롭게 생성하여</p>
<p data-ke-size="size16">filterpolicy를 포함하여 캐시나 버퍼 사이즈 등 변수를 저장하고</p>
<p data-ke-size="size16">이를 인자로 DB::Open 함수를 실행한다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@bhG0gK/btrJhj2zlta/G0aeBVVVoT92YYvysBXwbk/img.png|CDM|1.3|{"originWidth":710,"originHeight":248,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">이 다음 DB::Open의 경우도 인자로 받은 options의 값들을 impl이란 변수에 넣고</p>
<p data-ke-size="size16">이를 인자로 MaybeScheduleCompaction() 함수를 수행한다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@yG7Ai/btrJfYdOpKD/Fg38FNUCbWc0fUdb9yyVa1/img.png|CDM|1.3|{"originWidth":790,"originHeight":186,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">디 다음 흐름은 위와 같은데 Compaction을 진행하고</p>
<p data-ke-size="size16">필터 블록을 포함하여 SSTable을 생성한다.</p>
<p data-ke-size="size16">이 과정에선 filter_policy는 이전 함수에서 다음 함수로 값을 전달 받기만 하므로 자세한 설명은 생략한다.</p>
<p>[##_Image|kage@c8TA1v/btrJaWBfqCL/YZxoW5jDKKcpe9J9wfKBU0/img.png|CDM|1.3|{"originWidth":1361,"originHeight":522,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">*참고로 블룸 필터는 SSTable에 생성되므로,</p>
<p data-ke-size="size16">전체 데이터 크기가 적어 compaction이 발생하지 않는다면</p>
<p data-ke-size="size16">BGWork()가 아닌 NeedsCompaction()이 수행되어 블룸 필터가 생성되지 않는다.</p>
<p data-ke-size="size16">&nbsp;</p>
<p data-ke-size="size16">&nbsp;</p>
<p>[##_Image|kage@cazRdS/btrJgdodxUO/K6vrK8E7LY4AcS2NvFScO1/img.png|CDM|1.3|{"originWidth":552,"originHeight":361,"style":"alignCenter"}_##]</p>
<p data-ke-size="size16">이후 마지막 GenerateFilter 함수에서,</p>
<p data-ke-size="size16">지금까지 유지해온 filterpolicy의 createFilter 함수를 호출하여 write를 진행한다.</p>
<p data-ke-size="size16">&nbsp;</p>

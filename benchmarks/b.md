
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

 

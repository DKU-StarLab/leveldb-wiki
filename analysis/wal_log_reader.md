# Log format 로그의 형식
로그의 구조는 32KB 크기의 블럭의 시퀀스로 되어있다.     
한 블럭 안에는 ```Checksum(4bytes), Length(2bytes), Type(1byte), Data``` 부분으로 나뉘어져 있다.      
쓰고자 하는 데이터의 크기가 한 블럭에 담긴다면 ```Type``` 부분에 ```Full(1)```이 표기되겠지만,       
그러지 못하고 데이터가 여러 블럭에 담겨야 할 때 그 데이터가 기록되는 가장 앞 부분의 블럭의 ```Type```에는 ```First(2)```가 표기되고       
가장 마지막 블록은 ```Last(4)```가 표기된다.       
그리고 그 중간에 담기는 모든 블럭들은 ```Middle(3)```로 표기된다.    
<img src="https://drive.google.com/u/1/uc?id=1LoOrR3a4E2kP5DGBzcFwPASw05O5yLEK&export=download" width="460" height="280">    
       
​
# log::reader 파일    
```log::reader``` 파일에는   
```Reader``` 클래스와 몇 가지 함수가 있다.     
<img src="https://drive.google.com/u/1/uc?id=1VQ_Q20Z1ZtrY39Lead8EDtCSbr9z-BIS&export=download" width="460" height="280">    
나는 이 파일의 흐름을 살펴보고자 노력했다.    
   
   
```ReadRecord()``` 함수가 실행되면 이니셜 블록의 위치를 찾은 후 ```while```문을 돌면서 읽고자 하는 데이터를 읽는다.  
```ReadRecord()``` 함수의 매개변수로는 ```record```와 ```scratch```를 받는데,      
읽고자 하는 데이터가 여러 블록으로 나뉘어져 있을 경우 ```scratch```에 각 블럭의 데이터를 ```append```해주었다가 마지막 블럭을 만나면 한번에 ```record```에 준다.   
![a-3](https://drive.google.com/u/1/uc?id=1hWZOM3mO0TeymflMS6knKOzybNslOmhE&export=download)  
​
읽으려는 데이터가 몇 개의 블럭에 걸쳐 있는지 알기 위해서 ```switch```문에서는 각 블럭의 ```type```을 읽고 블럭을 더 읽을 것인지 판단하며    
에러가 있으면 이 또한 처리하는 과정이 있다.      
```kFullType```의 경우 데이터를 읽어 바로 ```record```에 준다음 ```true```를 반환하여 함수를 빠져나온다.   
```kFirstType```의 경우 ```scratch```에 데이터를 ```append```한 후        
```in_fragmented_record``` 변수를 ```true```로 설정하여 뒤에 블럭이 더 있음을 알리는 역할을 한다.     
![a-4](https://drive.google.com/u/1/uc?id=1hpQesw4cq7697jc_H0pcu_sP-1igFiiK&export=download)   
​
```kMiddleType```의 경우 ```scratch```에 데이터를 ```append``` 해준다.     
```kLastType```의 경우 ```scratch```에 데이터를 ```append``` 해준 후      
여태 추가했던 데이터가 담겨있는 ```scratch```를 ```Slice``` 객체로 바꿔서 ```record```에 준다.   
```kEof```의 경우는 더이상 읽을 블럭이 없을 때이다.     
![a-5](https://drive.google.com/u/1/uc?id=1nPL8OqHRJ03CzOKO6cWtbdU1i-T_fCiC&export=download)  
​
​
 ```kBadRecord```의 경우 ```checksum```이 맞지 않거나 레코드의 길이가 0이거나 등     
 오류가 있어 물리 레코드에서 읽어오지 못한 경우 처리된다.
 ```모든 keyType에 해당되지 않는 경우```는 오류를 알린다.   
![a-6](https://drive.google.com/u/1/uc?id=1wa81Pks8xtTKhS7hz0gK579JJK1gAKMg&export=download)   
​
​
​
​

# compaction 
> Compaction은  db에서 가장 복잡한 프로세스 중 하나이며 db의 성능에 큰 영향을 주기도 합니다.

> 내부 데이터 중첩 및 통합 메커니즘이며 읽기 및 쓰기 속도의 균형을 맞추는 효과적인 수단이기도 합니다.


## Compaction Type
### 1. Minor Compaction
- In Level db call “Flush”

![image](https://user-images.githubusercontent.com/86946575/181177580-415e1214-edfc-4180-b072-b36b8827ca1f.png)

- Trivial move  

![image](https://user-images.githubusercontent.com/86946575/181178432-ba39014c-a4a7-4d2e-ad15-5333109bdb22.png)

### 2. Major Compaction
- In Level db call “Compaction”
- Leveled Compaction

![image](https://user-images.githubusercontent.com/86946575/181183259-327818ac-1a2d-4e0e-91c9-24bc99cd1c3b.png)

### Another
- Tiered Compaction
- FIFO Conpaction
etc.

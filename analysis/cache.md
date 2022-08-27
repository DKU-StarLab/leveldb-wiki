# LevelDB Cache Analysis

### <center> Contents 
  - [1. What is Cache](#1-what-is-cache)
  - [2. What is LRU](#2-what-is-lru)
  - [3. LRU Cache Structure in LevelDB](#3-lru-cache-structure-in-leveldb)
  - [4. Overrall Cache Flow](#4-overrall-cache-flow)
  - [5. Cache Source Code Analysis](#5-cache-source-code-analysis)
---
## 1. What is Cache

 캐시는 하드웨어의 한 장치로서 메인 메모리나 디스크에 접근을 하기 이전에 거치는 메모리 저장 장치이다. 
  
캐시의 장점은 먼저 접근하는 속도가 굉장히 느린 메인 메모리에 비해서 접근 속도가 굉장히 빨라서 메인 메모리의 값을 미리 저장을 해두어서 하웨어의 처리 속도를 굉장히 효율적으로 줄이는게 가능하다. 
  
하지만 캐시의 가장 큰 단점은 빠른 접근 속도로 인한 하드웨어의 비싼 가격이다. 비싼 가격으로 인해 큰 용량을 캐시에서는 사용하지 못하는 단점도 발생한다.
  
하지만 빠른 처리 속도로 인해 꼭 필요한 장치이다.

<img width="754" alt="스크린샷 2022-08-27 오후 2 15 50" src="https://user-images.githubusercontent.com/74447373/187015861-02659a06-9a5a-4951-8487-97f5d17ee9ea.png">


---
## 2. What is LRU
  LRU캐시란 말 그대로 Least Recently Used 알고리즘을 칭하는 말로 LRU캐시는 이 알고리즘을 사용하여 캐시를 처리하는 캐시를 말한다.

<img width="661" alt="스크린샷 2022-08-27 오후 2 23 03" src="https://user-images.githubusercontent.com/74447373/187016130-35ee76ac-7b3e-4430-aa4e-80b1ec5a0506.png">

 LRU캐시의 장점으로는 먼저 빠른 액세스 타임과 그리고 빠른 업데이트이다 탐색과 업데이트에서 각각 최악의 경우에서는 캐시를 탐색하는 O(n)의 시간복잡도를 필요로 함으로 빠름 탁색과 업데이트가 가능하다. 
  
 
  LUR 캐시의 단점은 위 그림과 같은 방식으로 캐시가 처리가 되며 가장 오래 참조하지 않는 캐시의 위치에 새로운 값을 삽입하는 방식으로 알고리즘을 처리한다. 이 방식의 단점은 먼저 어떤 캐시의 위치가 가장 오래 참조를 하지 않았는지 표시를하는 방식이 필요로 한다. 
  
---
## 3. LRU Cache Structure in LevelDB

내용작성

글

그림


---
## 4. Overrall Cache Flow

내용작성

글

그림

## 5. Cache Source Code Analysis

내용작성

글

그림


---

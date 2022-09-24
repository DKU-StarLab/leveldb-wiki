# LevelDB WIKI
[DKU System Software Lab](https://sslab.dankook.ac.kr/)에서 진행한 [2022 LevelDB 스터디](https://github.com/DKU-StarLab/leveldb-study)를 통해 작성된 LevelDB wiki입니다.  
학생들이 스터디를 통해 LevelDB를 공부한 내용을 정리하여 작성한 문서입니다.

이 문서는 LevelDB의 배경, 구조, 분석 그리고 분석하는 방법에 대해 설명합니다.  
DKU System Software Lab의 홈페이지에서 [LevelDB WIKI를 전자책](https://sslab.dankook.ac.kr/leveldb-wiki/)으로 편하게 읽어보실 수 있습니다.

문서 내용에 오류가 있거나, 추가사항이 있으시다면 언제든지 Pull Request를 통해 기여해주시면 감사하겠습니다.

## Authors
* Student:
  - 1.WAL/Manifest: [Isu Kim](https://github.com/gooday2die), [Seyeon Park ](https://github.com/SayOny), [Suhwan Shin](https://github.com/Student5421)
  - 2.Memtable: [Taegyu Cho ](https://github.com/HASHTAG-YOU), [Zhu Yongjie](https://github.com/arashio1111)
  - 3.Compaction: [Sangwoo Kang](https://github.com/aarom416), [Seoyoung Park](https://github.com/seo-0), [Zhao Guangxun](https://github.com/ErosBryant)
  - 4.SSTable: [Jongki Park](https://github.com/JongKI-PARK), [Sanghyun Cho](https://github.com/Cho-SangHyun)
  - 5.Bloom Filter: [Hansu Kim](https://github.com/gillyongs)
  - 6.Cache: [Seungwon Ha](https://github.com/ha-seungwon), [Subin Hong](https://github.com/sss654654)
* Assistant: [Min-guk Choi](https://github.com/korea-choi)
* Senior Assistant: [Sounghyoun Lee](https://github.com/shl812), [Hojin Shin](https://github.com/shinhojin)
* Professor: [Jongmoo Choi](http://embedded.dankook.ac.kr/~choijm/), [Seehwan Yoo](https://sites.google.com/site/dkumobileos/members/seehwanyoo)

## Contents
### 1. LevelDB Code Analysis
1. [WAL](./analysis/wal.md)
2. [Memtable](./analysis/memtable.md)
3. Compaction
    - [Compaction](./analysis/compaction/compaction.md)
    - [Major Compaction](./analysis/compaction/Major-Compaction.md)
    - [Minor Compaction](./analysis/compaction/Minor-Compaction.md)
4. SSTable
    - [SSTable Format](./analysis/sstable/sstable.md)
    - [SSTable Write](./analysis/sstable/sstable-write.md)
    - [SSTable Read](./analysis/sstable/sstable-read.md)
5. Bloom Filter
    - [Bloom Filter](./analysis/bloomfilter/bloomfilter.md)
    - [Bloom Filter Write](./analysis/bloomfilter/bloomfilter-write.md)
    - [Bloom Filter Read](./analysis/bloomfilter/bloomfilter-read.md)
6. [Cache](./analysis/cache.md)
7. [Manifest](./analysis/manifest.md)

### 2. Benchmark Experiment
- WAL
    - [Experiment 1](./benchmarks/wal_1.md)
    - [Experiment 2](./benchmarks/wal_2.md)
    - [Experiment 3](./benchmarks/wal_3.md)
- [Memtable](./benchmarks/memtable.md)
- [Compaction](./benchmarks/compaction.md)
- [SSTable](./benchmarks/sstable.md)
- [Bloom Filter](./benchmarks/bloomfilter.md)
- [Cache](./benchmarks/cache.md)

### 3. YCSB Tuning Contest
 - [Tuning Contest Guide](https://github.com/DKU-StarLab/leveldb-study/blob/main/tuning/README.md)
 - [Team SSTable Report](./tuning/%5BTuning%5Dteam_SSTable_report.md)
 - [Team Bloom Filter Report](./tuning/%5BTuning%5Dteam_bloomfilter_report.md)
 - [Team WAL/Manifest Report](./tuning/%5BTuning%5Dteam_WAL%2CManifest_report.md)
 - [Team Memtable Report](./tuning/%5BTuning%5Dteam_memtable_report.md)
 - [Team Cache Report](./tuning/%5BTuning%5Dteam_cache_report.md)
 - [Team Compaction Report](./tuning/%5BTuning%5Dteam_Compaction_report.md)


### 4. Pratice 
  - [Question](https://github.com/DKU-StarLab/leveldb-study/issues/6)
  - [Solution](https://github.com/DKU-StarLab/leveldb-study/blob/main/introduction/homework_solution.md)  

## 사진
<img src="./image/photo.png" width="100%">

## 포스터
<img src="./image/poster_kor.png" width="50%">

## 참고문헌
#### 1. Documents
  - [LevelDB Document](https://github.com/google/leveldb/blob/main/doc)
  - [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)
  - [Jongmoo Choi,『Key-Value Store: Database for Unstructured Bigdata』, 2021](https://github.com/DKU-StarLab/leveldb-study/blob/761b550973ab6d1e88189190e66c0ee19a52aa12/introduction/Jongmoo%20Choi,%20Key-Value%20Store%20-%20Database%20for%20Unstructured%20Bigdata,%202021.pdf)
  - [Fenggang Wu, 『LevelDB Introduction』, 2016](https://www-users.cselabs.umn.edu/classes/Spring-2020/csci5980/index.php?page=presentation)
  - [rjl493456442, 『leveldb-handbook (CHS)』, 2022](https://leveldb-handbook.readthedocs.io/zh/latest/)
  - [rsy56640, 『read_and_analyse_levelDB (CHS)』](https://github.com/rsy56640/read_and_analyse_levelDB/tree/master/reference)
  - [FOCUS,『LevelDB fully parsed (CHS)』](https://www.zhihu.com/column/c_1258068131073183744)
  - [bloomingTony, 『Research on Network and Storage Technology(CHS)』](https://www.zhihu.com/column/c_180212366)
  - [木鸟杂记,『Talking about LevelDB data structure (CHS)』, 2021 ](https://www.qtmuniao.com/categories/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/)
#### 2. Lecture
  - [Jongmoo Choi, 『Key-Value Store: Database for Unstructured Bigdata (KOR)』,  2021](https://mooc.dankook.ac.kr/courses/61d537a3b6b71841651153b3)
  - [GL Tech Tutorials, 『LSM trees』, 2021](https://youtube.com/playlist?list=PLRNjlOFk-f0lJJZVoSAmcwZgVtp64tXaX)
  - [Wei Zhou, LevelDB YouTube playlist](https://youtube.com/playlist?list=PLaCN8MYUet0tR1xn5d8ZtCumHKtP6Wkeq)
#### 3. Analysis Tools
  - [GDB](https://www.sourceware.org/gdb/)
  - [Understand](https://licensing.scitools.com/download)
  - [Uftrace](https://github.com/namhyung/uftrace)
  - [Draw.io](https://www.draw.io)
#### 4. Previous Study
  - [DKU RocksDB Festival, 2021](https://github.com/DKU-StarLab/RocksDB_Festival)

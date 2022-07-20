# leveldb-wiki (KOR)
-------------
## Background
#### 1. What is a key-value store?
#### 2. Why open source?
#### 3. Job market

-------------
## Analysis
#### 0. Overall
#### 1. Key-Value Interface
#### 2. [WAL](./analysis/wal.md)
#### 3. [Memtable](./analysis/memtable.md)
#### 4. [Compaction](./analysis/compaction.md)
#### 5. [SSTable](./analysis/sstable.md)
#### 6. [Bloom Filter](./analysis/bloomfilter.md)
#### 7. [Cache](./analysis/cache.md)
#### 8. [Manifest](./analysis/manifest.md)

-------------
## Apenddix
####  1. How to Install LevelDB
#### 2. Analysis Tools
* Understand
* GDB (shell script)
* Uftrace (shell script)
#### 3. Homework (db_bench practice)
* [Question](https://github.com/DKU-StarLab/leveldb-study/issues/6)
* [Solution](https://github.com/DKU-StarLab/leveldb-study/blob/main/introduction/homework_solution.md)
#### 4. Benchmarks Analysis
- [WAL](./benchmarks/wal.md)
- [Memtable](./benchmarks/memtable.md)
- [Compaction](./benchmarks/compaction.md)
- [SSTable](./benchmarks/sstable.md)
- [Bloom Filter](./benchmarks/bloomfilter.md)
- [Cache](./benchmarks/cache.md)
#### 5. Real-world Workload Tuning
- Twitter Trace
-------------
## References

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
#### 4. Real-World Workload
  - [Twitter cache trace](https://github.com/twitter/cache-trace)
  - [Facebook ZippyDB](https://github.com/facebook/rocksdb/wiki/RocksDB-Trace%2C-Replay%2C-Analyzer%2C-and-Workload-Generation)
#### 5. Previous Study
  - [DKU RocksDB Festival, 2021](https://github.com/DKU-StarLab/RocksDB_Festival)
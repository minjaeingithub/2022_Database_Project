## RocksDB

### RocksDB 특징과 구조

- RocksDB는 persistent key-value 를 저장한다. 이는, 기존의 SSD 기술보다, read/write rate이 높아져서, 향상된 성능을 최대로 이용하기 위함이다.
  - performance와 scalability 향상에 집중하여, server workload에 최적화되어있다.(후술)

- DB engine에는 log-structured 구조이며, C++로 작성되었다.

RocksDB의 기본적인 구성 요소를 알아보자.

1. **Memtable**

   in-memory DS이며, MySQL에서 버퍼와 같은 역할을 한다. 이 곳에서 임시적으로 write된 데이터들을 보관해둔다.

2. **Logfile**

   순차적으로 storage에 쓰이는 파일

3. **SSTable(SST file)**

   - storage에 저장된, 순차적이고, 정렬된 테이블들이며, key를 빨리 찾기 위한 구조이다.
   - key-value pair로 저장되어있고, level로 구조화되어있으며,
   - lifetime동안에는 immutable하다.(영구적이며, 저장되면 수정불가능)

   ![Screen Shot 2022-10-19 at 1.46.10 PM](/Users/jominjae/Desktop/Screen Shot 2022-10-19 at 1.46.10 PM.png)

   > SST의 기본적 포맷인 BBT이다.

**간단한 작동원리**

Write 요청이 오면, memtable에 먼저 써지고, logfile에 순차적으로 써진다.

memtable이 가득차게 되면, memtabledms SST file에 flush되며, 해당 memtable과 관련된 logfile은 삭제된다.

SSTable의 인덱스는 항상 메모리에 올라가있으며, read 요청은 먼저 memtable을 접근하고, SSTable의 인덱스에 접근한다.

SSTable은 주기적으로 merge한다(후술)

​	update/delete 레코드가 이전 데이터들을 overwrite/remove하게 된다.

<img src="/Users/jominjae/Desktop/Screen Shot 2022-10-19 at 1.46.37 PM.png" alt="Screen Shot 2022-10-19 at 1.46.37 PM" style="zoom:50%;" />

## B+ tree vs. Log Structured Merge tree (LSM tree)

<img src="/Users/jominjae/Desktop/Screen Shot 2022-10-19 at 1.53.13 PM.png" alt="Screen Shot 2022-10-19 at 1.53.13 PM" style="zoom:50%;" />

#### B+ tree

**in-place update** 방식(그림의 (a))으로 업데이트한다. 레코드의 가장 최신 버전이 overwritten되었기 때문에 **Read-optimized**한 구조이다. 그러나, 이런 업데이트 방식은 random I/O를 유발하기 때문에 write 성능을 희생하게 된다.

#### LSM-Tree

<img src="/Users/jominjae/Desktop/Screen Shot 2022-10-19 at 1.53.21 PM.png" alt="Screen Shot 2022-10-19 at 1.53.21 PM" style="zoom:50%;" />

**out-of-place update** 방식(그림의 (b))으로 업데이트한다. Logical tree를 여러 개의 물리적인 piece들로 나누는 작업이며, 가장 최신에 새로 업데이트된 데이터도 메모리의 트리구조에 남게 된다. 즉, logfile(메타데이터)과 memtable을 사용하여 random write을 sequential write로 바꾸기 때문에 더 빠르다. 

RocksDB는 LSM tree를 채택하고 있으며, write에 최적화되어있다.(후술)

## DBMS space management

> 현재 데이터센터의 동향은 SSD가 영구적인 데이터를 보관하는 저장소로 쓰이는 추세이다. 페이스북의 경우, 온라인 데이터의 10s of petabytes를 저장하고 있다. 그럼에 따라 storage space에 병목이 생기는 것이 문제이다. 자원의 효율성을 높여, scalability를 보장하는 것이 현안이 되었다. FYI) 그 동안 DBMS는 작은 DRAM 사이즈와 magnetic HDD가 느렸었고, 이를 바탕으로 DBMS를 만들고, 유지보수하는 것이 이뤄져왔다.

### MySQL/InnoDB space management

MySQL DBMS의 인덱스 데이터 구조는 B+ tree를 채택하고 있다. 고정된 크기의 페이지를 사용하여, fan-out이 높기 때문에(==자식이 많다), Disk I/O 요청이 비교적 낮았다. 그러나, MySQL 을 SSD에서 사용하게 된다면, 성능과 reliability가 좋지만, space/write amplification에서 비효율성을 확인할 수 있다.

### the lower, the better!

$$
Write\:Amplification = {number\:of\:physical\:writes\:to\:flash\:memory \over number\:of\:logical\:writes\:from\:the\:host}
$$

$$
Space\:Amplification = {The\:size\:of\:database \over The\:size\:of\:the\:data\:in\:the\:database}
$$

MySQL의 경우, 1 Byte modification은 1 page write을 발생시킨다(4KB~16KB).

그리고, B+tree의 경우, 70% 이하로만 avg.fill factor를 유지하기 때문에, 그 이상이 넘어가면 leaf node를 분할 현상이 발생한다.

따라서, space utilization을 떨어뜨린다.



### RocksDB Compaction

#### Compaction이란, 같은 key에 대한 multiple copies를 제거, SST file을 더 큰 SST file로 merge하는 것이다.

두 가지 방법을 지원한다. (1) Leveled compaction, (2) Universal compaction

**1. Leveled compaction**

<img src="/Users/jominjae/Library/Application Support/typora-user-images/image-20221019142335332.png" alt="image-20221019142335332" style="zoom:50%;" />

SST file은 여러 level로 구성된다. 

L0은 memtable에서 flush된 file들을 저장한다. 

각 level은(L0 제외) 정렬된 데이터의 연속이고, 내부에는 데이터가 여러 SST file로 범위가 나누어져있다.
0이 아닌 모든 level에는 타겟 크기가 있는데, compaction의 목표는 level의 크기를 타겟 크기보다 작게 만드는 것이다.
compaction은 L0의 파일 수가 특정 숫자를 넘었을 때 발생하고, L0의 파일들은 L1에 merge된다. 일반적으로 L0의 파일들은 key값이 겹치기 때문에, 모든 L0 파일들은 compaction 대상 파일로 선택해야 한다.

DB의 data는 multiple levels로 구성돼있음, 각 레벨의 타겟 크기는 이전 레벨보다 10배 크다.
	recent data -> L0
	oldest data -> Lmax
L0: overlapping keys, **flush time**으로 정렬 돼있음
L1 ~ Lmax: Non-overlapping keys, **key**로 정렬 돼있음



**2. Univeral Compaction**

- write-heavy workload에서, level compaction은 **병목현상**이 발생함.
- universal compaction은 write amplification을 줄이기 위해 고안 됨.
- universal compaction에선 모든 files는 L0에 **time order**로 정리되어있음.
- universal compaction은 일시적으로 amplification size를 증가시킴(space amplification을 잠시 높인다.)

1. 시간순으로 인접한 few files를 pick

2. pick up한 few files를 merge

3. **L0의 new file로** 교체함

   


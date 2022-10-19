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

![Screen Shot 2022-10-19 at 1.46.37 PM](/Users/jominjae/Desktop/Screen Shot 2022-10-19 at 1.46.37 PM.png)


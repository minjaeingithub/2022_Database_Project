## MySQL

### MySQL server

- open-source RDBMS
- 범용성이 뛰어나고, OLTP에 최적화되어있음

### OLTP vs. OLAP

- **OLTP**
  - 간단한 쿼리를 사용해서 SIUD 빠르게 처리
  - Mixed random r/w workloads
  - **index scan** 을 사용한다.
  - examples : ATM machine, online banking, booking, shopping e.t.c.
- **OLAP**
  - 복잡한 쿼리를 사용해서 데이터 분석 및 집계를 통해 사용자 의사결정에 도움을 주는 것이 가장 큰 목적
  - Heavy **read** workloads
  - **Full table scan** 을 사용한다. 
  - examples: sales analysis, market research, forecasting e.t.c.

- ** 두 방식 모두 index를 사용하지만, OLTP는 실시간으로 업데이트되는 대용량 데이터를 읽는 와중에 특정 데이터를 찾아야하기 때문에 index scan을 사용하고, 많은 데이터를 전부 읽어야하는 OLAP는 full table scan방식을 사용한다.



## TPC-C

우리는 tpc-c로 벤치마킹을 돌려 성능을 측정하고 평가했다. 벤치마크는 무엇이고, 왜 필요한가?

벤치마킹은 특정 애플리케이션의 성능을 측정하고, 다른 유사한 workload와 비교하여 DBMS performance에 gap 이 있는지 발견하고, 성능 자체를 향상시키기도 한다. TPC-C는 OLTP 벤치마크이다.

따라서, 벤치마크는 domain-specific하다. 도메인별로 특화되어있기 때문에, 범용성이 높은 벤치마크는 좋은 벤치마크가 되지 않는다. 벤치마크의 특징은 다음과 같다 :

- Relevant : meaningful within the target domain
- Understandable/acceptable : vendors and users embrace it.
- Scalable/coverable : not oversimplify the typical environment.

### TPC-C 성능 측정 지표

#### 	Throughput of TPC-C 는 분당 실행된 **New-order transaction의 수**다.

##### 	-> TPmC(transactions per minute count)

### 5 types of well defined transactions:

1. New-order(R/W)

   warehouse로부터 평균 10개의 item을 주문 -> 주문 insert -> 주문에 대한 해당 재고 수준 update

2. Payment transaction(R/W)

   customer의 지불 처리 -> 잔액 및 기타 데이터 update

3. Order status(Read only)

   customer의 마지막 주문 상태를 반환

   

4. Delivery transaction(R/W)

   process(read, update, delete) orders corresponding to 10 pending orders, one for each district, with 10 items per order.

5. Stock level(Read only)

   한 district의 마지막 20개 주문 별로 주문한 아이템의 재고 수량을 읽고 반환



## DB I/O architecture

데이터가 메모리에 있으면 읽는 속도가 빨라진다. 그러나, 캐시에 없다면, 디스크에서 찾아와야 하는 번거로움이 있다. write의 경우에는 그 방향이 반대로 작용한다. 

![img](https://velog.velcdn.com/images/rhtaegus17/post/cfe4b7da-f137-4306-a885-9ab3370e80b8/image.png)

Transaction 이란, 유저 입장에서는 일련의 SQL 쿼리이다. 물리적인 입장에서 보면, 일련의 read/write operations이다. DB개론 수업에서 배웠듯이, select는 page를 읽는 것이고, IDU는 읽어온 페이지가 dirty해지게 되는 명령어로, 저장을 하고 싶다면 storage에 WRITE 를 해야한다. memory에 page를 할당된 용량만큼 올려놓는 것이 buffer이다. 즉, page caching 역할을 한다.

Page가 buffer 안에 있으면 Hit, 없어서 disk에서 불러와야 한다면 miss이다. 이 때, buffer pool이 DBMS 성능과 직결되고, 후술된다.

Buffer hit의 경우 빠른 DRAM에서 읽어오고, Miss의 경우 느린 Disk I/O가 필요하다. Transaction의 response time의 경우, 대부분 disk I/O이므로, 이를 최소화해야한다.



## Buffer Manager

![img](https://velog.velcdn.com/images/rhtaegus17/post/95227c5d-cda2-44eb-9dc5-0a250b835948/image.png)

Disk의 데이터 블록을 가지고 있는 buffer manager cache에는 Frequently access data가 있기 때문에, 성능향상을 위해서 빠른DRAM에 접근해야한다.

Buffer에 요청한 페이지가 있다면 : 그 페이지를 pin하고 페이지 위치가 있는 주소를 리턴한다.

Buffer에 요청한 페이지가 없다면: replacement policy에 따라 victim buffer frame을 선정한다. 	

- 클린한 페이지라면, 그 프레임의 데이터가 디스크에 반영되었다는 말이기 때문에, 단순히 데이터를 버려서 free buffer frame을 만든다. 
- 더티한 페이지라면, 우선 디스크에 변경된 사항을 적용하여 write를 하고, 요청된 페이지를 victim buffer frame에 올려놓고, 해당 프레임 주소를 반환한다.

> $$
> Hit\:Ratio = {number\: of\: hits \over number\:of\:page\:requests}
> $$
>
> 

- page가 하나 MISS되는 것은 one or two disk I/O를 의미하고, disk I/O는 디램에 비해 1000배 성능 저하가 일어난다. 따라서, 1%의 Hit ratio가 증가하는 Miss가 하나 줄었다고 가정하면, 10%의 성능향상 효과를 보이는 것과 같다.



## InnoDB Architecture

![img](https://velog.velcdn.com/images/rhtaegus17/post/62418544-31e3-4eec-bc6c-ef1c820f1101/image.png)

MySQL는 InnoDB storage engine을 사용하고 있다. 위 사진은 InnoDB 구조이며, 자체적으로 메인 메모리 안에 데이터 캐싱과 인덱싱을 위한 버퍼 풀을 관리한다.



## Buffer Pool and Replacement policy

- Buffer Pool

  InnoDB가 접근할 때 테이블 및 인덱스 데이터를 캐시하는 메인 메모리 영역이다.

  LRU를 사용한다.

- Replacement Policy

  - Miss가 났을 때, victim buffer frame을 선정하는 정책이다.

  - ex. random, FIFO, LRU, MRU, LFU, Clock e.t.c.

  - 이 정책은 I/O 횟수에 영향을 미치며, 좋은 페이지 교체 정책은 buffer hit ratio를 올리는 정책이다.

    

## LRU algorithm

- MySQL은 modified LRU를 사용하며, two sublists로 버퍼 풀을 관리한다.

  > LRU list를 두 개로 나눠서 관리하는 이유는
  >
  > - 자주 접근되는 hot page가 buffer pool에 계속 남겨지는 것을 보장하기 위해
  > - never accessed again 의 특성을 가지는 버퍼 풀 스캔이 buffer pool에 저장되는 양을 최소화하고, buffer pool 변동을 최소화하기 위해

  ![img](https://velog.velcdn.com/images/rhtaegus17/post/a03ededc-8970-42ab-bd3f-69ca14588dde/image.png)

  	1. Page read request가 들어오면, 제일 먼저 midpoint에 삽입된다(Midpoint insertion).

      > Read-ahead/Large scan/where절 없는 select쿼리 등은 스캔하는 동안 한 번 접근되고 다시 접근되지 않는 특성을 가지고 있기 때문에, new sublist 의 head에 추가되면, old sublist 의 tail로 가기까지 오랜 시간이 걸려, 버퍼 풀을 낭비하게 되는 것이기 때문이다.	

  	2. Page HIT가 일어나면

      * Old sub-list : 페이지를 new sublist head로 옮긴다.
      * New sub-list : 페이지가 new sublist head에서 특정 거리만큼 떨어져 있으면, head로 옮긴다.

​		3. Least Recently Used pages는 리스트의 tail로 가고, evicted 된다.

## Buffer pool configuration

> Lab experiment를 떠올려보자. Buffer size, scan depth size를 my.cnf 파일에서 변경할 수 있었다.

1. **Buffer pool size**

   - 바꿀 수 있었던 변수 :

      ```innodb_buffer_pool_size ```

     ``` innodb_buffer_pool_chunk_size ```

     ​	buffer pool is allocated in chunks.

      So,  ```innodb_buffer_pool_size ``` = ``` innodb_buffer_pool_chunk_size ``` * ```innodb_buffer_pool_instances```

   - ```innodb_buffer_pool_instance ``` = # of cores * 2

     buffer pool을 separate instances들로 나누면, 병행으로 데이터 접근이 가능해진다. 즉, 다른 쓰레드가 같은 페이지에 r/w하는 트랜잭션 간 lock contention 을 줄일 수 있다.

     **나눠진 buffer pool instance들은 각각 free list, flush list, LRU list를 가지고 있다.(후술)**

2. **Scan resistant Buffer Pool**

   ``` innodb_old_blocks_time```: specifies the time window after the first access to a page during which it can be accessed w/o being moved to the front(most recently used end) of LRU list.

   ```innodb_old_blocks_pct```: specifies the approximate percentage of the InnoDB buffer pool used for the old block sub-list

3. **Buffer Pool prefetching**

   **Read-ahead**: 곧 필요할 것 같은 page들을 buffer pool에서 **비동기적**으로 미리 가져오도록 하는 I/O request이다.

   > <Methods>
   >
   > Linear : buffer pool에 있는 순서대로 접근
   >
   > ​	<code>innodb_read_ahead_threshold </code>		
   >
   > ​	: controls how sensitive InnoDB is in detecting patterns of sequential page access	
   >
   > Random: page access order에 상관없이, 이미 buffer pool에 있는 페이지를 바탕으로, 어떤 페이지가 필요할 지 예측
   >
   > ​	<code> innodb_random_read_ahead </code>
   >
   > ​		: random read-ahead feature를 사용할 수 있도록 한다.

4. **Buffer Pool Flushing**

   Buffer Pool의 dirty page는 **background thread**에 의해 flush 된다.

   <code> innodb_lru_scan_depth </code>

   : # of. page flushed in LRU tail by the page cleaner thread

   - 백그라운드로 돌고 있는 page_cleaner_thread가 버퍼풀의 LRU list 중 dirty page를 얼마나 많이 disk로 내릴지(==scan depth) 정하는 옵션이다.

   - 주로 버퍼풀이 큰 상황에서, 평소 I/O load를 감당할 정도라면 늘려도 좋지만, I/O 용량을 상회한다면, 설정을 낮추는 것이 좋다. (Default = 1024)

## Buffer Pool Monitoring Method

``` sql
SHOW ENGINE INNODB STATUS
```

![img](https://velog.velcdn.com/images/rhtaegus17/post/674255b0-9e45-4c2f-abd4-18584c03843d/image.png)




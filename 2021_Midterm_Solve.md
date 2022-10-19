List five transactions in TPC-C and briefly explain the business logic each transaction carries out.

``` 
New order transaction(r/w):
Payment transaction(r/w):
Order status transaction(r):
Delivery transaction(r/w):
Stock level transaction(r):
```

Briefly explain at least two tools you can use to monitor system statistics (e.g., I/O, CPU, memory, etc.) and list what metrics you can get from each tool

```
mpstat, htop => cpu status
	%usr: app level
	%sys: system level
	%iowait: percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request
	%idle: percentage of time that the CPU or CPUs were idle and the system did not have an outstanding disk I/O request
	
vmstat:memory status
	Procs
		r: The number of runnable processes (running or waiting for run time)
		b: The number of processes blocked waiting for I/O to complete
	Memory
		swpd: the amount of swap memory used
		free: the amount of idle memory
		buff: the amount of memory used as buffers
	Swap
		si: Amount of memory swapped in from disk (/s)
		so: Amount of memory swapped to disk (/s)
  IO
		bi: Blocks received from a block device (blocks/s)
		bo: Blocks sent to a block device (blocks/s)
	System
		in: The number of interrupts per second, including the clock
		cs: The number of context switches per second
	CPU
    us: Time spent running non-kernel code (user time, including nice time)
		sy: Time spent running kernel code (system time)
    id: Time spent idle
		wa: Time spent waiting for IO
		st: Time stolen from a virtual machine
	
	iostat: io status
	r/s, w/s, util
	util: Percentage of elapsed time during which I/O requests were issued to the device 		  
	      (bandwidth utilization for the device).
```

Interpret the meaning of each metric (trx, 95%, 99%, TpmC) in the TPC-C experimental result below.

```shell
10, trx: 493, 95%: 764.362, 99%:1011.230, ...
20, ...
<TpmC>
  3634.300 TpmC
```

```
10초 단위로 transaction이 493개 발생했으며, 10초당 95% response time이 764.362이고, 99%의 response time은 1011.230이다.3643.300 TpmC는, 분 당 완료된 new order transaction 수가 3643.300개임을 말하고 있다.
```

Describe how TPC-C throughput changes as the MySQL's buffer size increases from 10% to 50% of the DB size. Explain the expected result and why.

``` 
Buffer Pool Size를 DB size의 10%, 20%, 30%, 40%, 50%로 늘리면서 TpmC와 Buffer hit rate가 어떻게 변화하는지를 관찰한다면, buffer pool size가 증가할 수록 TpmC와 buffer hit rate가 증가할 것이다. 할당된 DB size가 늘어날수록 데이터를 cache할 수 있는 용량이 커지므로 hit rate가 늘어나고 그에 따라 transaction per minute도 증가할 것이다.
```

Describe how TPC-C throughput varies with scan depth sizes? Explain the expected result and why.

```
LRU scan depth의 변화에 따라 buffer miss senario의 step1, step2, step3의 비율이 달라지고, buffer manager operation에 영향을 줄 것이다. step1은 free list를 search하는 과정인데, 이때 free block이 있다면 이를 반환한다. step2는 LRU list의 tail부터 scan depth만큼 LRU list를 scan하며 clean page를 찾고, clean page가 있다면 이를 반환한다. step3는 LRU tail의 dirty page를 flush하여 free list에 삽입한다.
```

Buffer pool hit rate metric를 확인할 수 있는 tool

```
SHOW INNODB ENGINE STATUS
```

RocksDB ./db_bench metric

```
readrandomwriterandom : 53.084 micros/op 18838 ops/sec; ( reads:10172700 writes:1130299 total:1000000 found:4076910)
```

**해석**

- `micros/op`: Microseconds spent processing one operation
- `ops/sec`: Processed operations per second
- readandwriterandom: benchmark type
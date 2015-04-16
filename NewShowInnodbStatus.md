# Introduction #

We have added more output to **SHOW INNODB STATUS**, reordered the output so that the list of transactions is printed list and increased the maximum size of the output that may be returned.

# Background threads #

All of this is new:
  * srv\_master\_thread\_loops - counts work done by main background thread
  * spinlock delay displays the number of milliseconds that the spinlock will spin before going to sleep
  * fsync callers displays the source of calls to fsync()

```
----------
BACKGROUND THREAD
----------
srv_master_thread loops: 28488 1_second, 28487 sleeps, 2730 10_second, 1182 background, 761 flush
srv_master_thread log flush: 29146 sync, 2982 async
srv_wait_thread_mics 0 microseconds, 0.0 seconds
spinlock delay for 5 delay 20 rounds is 5 mics
fsync callers: 1034231 buffer pool, 39227 other, 73053 checkpoint, 10737 log aio, 80994 log sync, 0 archive
```

# Semaphores #

New output includes:
  * lock wait timeouts counter
  * number of spinlock rounds per OS wait for a mutex

```
----------
SEMAPHORES
----------
Lock wait timeouts 0
...
Spin rounds per wait: 2.90 mutex, 1.27 RW-shared, 3.04 RW-excl
```

# Disk IO #

New output includes:
  * number of pages read/written
  * number of read/write system calls used to read/write those pages
  * time in milliseconds to complete the IO requests

```
--------
FILE I/O
--------
I/O thread 0 state: waiting for i/o request (insert buffer thread) reads 24 writes 0 requests 14 io secs 0.033997 io msecs/request 2.428357 max_io_wait 18.598000
I/O thread 1 state: waiting for i/o request (log thread) reads 0 writes 10737 requests 10737 io secs 30.626824 io msecs/request 2.852456 max_io_wait 710.588000
I/O thread 2 state: waiting for i/o request (read thread) reads 136659 writes 0 requests 118296 io secs 3093.412099 io msecs/request
26.149761 max_io_wait 2631.029000
I/O thread 3 state: waiting for i/o request (read thread) reads 91262 writes 0 requests 71709 io secs 1900.155508 io msecs/request 26.498145 max_io_wait 1626.209000
I/O thread 6 state: waiting for i/o request (write thread) reads 0 writes 1847360 requests 7065434 io secs 1063.904923 io msecs/request 0.150579 max_io_wait 2569.244000
```
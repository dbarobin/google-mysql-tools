There are more details on InnoDB status in the output from _SHOW INNODB STATUS_ and _SHOW STATUS_.

# SHOW INNODB STATUS #

New details include:
  * frequency at which the main background IO thread runs
  * IO latency for each background IO thread
  * per-file IO statistics
  * insert buffer prefetch reads
  * statistics on checkpoint related IO
  * statistics on prefetches
  * statistics on sources of background IO

## Main background IO thread ##

This includes:
  * srv\_master\_thread loops - number of iterations of the main background loop including the tasks per second (1\_second) and the tasks per 10 seconds (10\_second).
  * Seconds in background IO thread: number of seconds performing different background IO tasks

```
BACKGROUND THREAD
----------
srv_master_thread loops: 1623 1_second, 1623 sleeps, 162 10_second, 1 background, 1 flush
srv_master_thread log flush: 1785 sync, 1 async
srv_wait_thread_mics 0 microseconds, 0.0 seconds
spinlock delay for 5 delay 20 rounds is 2 mics
Seconds in background IO thread: 5.10 insert buffer, 49.02 buffer pool, 0.00 adaptive checkpoint, 52.34 purge
fsync callers: 0 buffer pool, 189 other, 1323 checkpoint, 263 log aio, 5179 log sync, 0 archive
```

## Background IO thread statistics ##

This includes:
  * reads, writes - number of pages read and written
  * requests - number of pwrite/pread system calls. There may be fewer of these than reads and writes because of request merging.
  * msecs/r - average number of milliseconds per **request**. For the **io:** section this is the time for the pwrite/pread system call. For the **svc:** section this is the time from when the page is submitted to the background thread until it is completed.
  * secs - total seconds for all pread/pwrite calls
  * old - number of pages for which the service time is greater than 2 seconds
  * Sync reads, Sync writes - IO operations done synchronously. These share code with the background IO threads, but the IO calls are done directly rather than begin put in the request array and handled by a background IO thread.

```
FILE I/O
--------
I/O thread 0 state: waiting for i/o request (insert buffer thread) reads 2177 writes 0 io: requests 125 secs 2.99 msecs/r 23.90 max msecs 82.07 svc: 106.58 msecs/r 48.96 max msecs 128.76 old 0
I/O thread 1 state: waiting for i/o request (log thread) reads 0 writes 263 io: requests 263 secs 0.13 msecs/r 0.49 max msecs 30.43 svc: secs 0.14 msecs/r 0.54 max msecs 30.48 old 0
I/O thread 2 state: doing file i/o (read thread) reads 116513 writes 0 io: requests 35777 secs 564.96 msecs/r 15.79 max msecs 251.04 svc: secs 7643.21 msecs/r 65.60 max msecs 2492.18 old 111 ev set
I/O thread 6 state: waiting for i/o request (write thread) reads 0 writes 391586 io: requests 256597 secs 1169.16 msecs/r 4.56 max msecs 336.70 svc: secs 104498.79 msecs/r 266.86 max msecs 3001.04 old 169
Sync reads: requests 10126259, pages 10126278, bytes 165912465408, seconds 171656.02, msecs/r 16.95
Sync writes: requests 2849234, pages 3029512, bytes 11289789952, seconds 77.81, msecs/r 0.03
```

## File IO statistics ##

This includes statistics per file. It is much more useful when InnoDB is run with _innodb\_file\_per\_table_. The first two columns are the tablespace name and tablespace ID. There are separate sections for reads and writes per file:
  * pages - number of pages read or written
  * requests - number of pwrite/pread system calls. There may be fewer of these than reads and writes because of request merging.
  * msecs/r - average number of milliseconds per **request**
  * secs - total seconds for all pread/pwrite calls

```
File IO statistics
  ./test/warehouse.ibd 10 -- read: 3 requests, 3 pages, 0.01 secs, 4.36 msecs/r, write: 30 requests, 30 pages, 0.11 secs, 3.70 msecs/r
  ./ibdata1 0 -- read: 1123 requests, 3349 pages, 22.97 secs, 20.46 msecs/r, write: 2662 requests, 86526 pages, 32.86 secs, 12.34 msecs/r
  ./test/orders.ibd 29 -- read: 26301 requests, 28759 pages, 450.63 secs, 17.13 msecs/r, write: 82089 requests, 101564 pages, 425.44 secs, 5.18 msecs/r
  ./test/customer.ibd 28 -- read: 333186 requests, 338048 pages, 5955.39 secs, 17.87 msecs/r, write: 185378 requests, 200494 pages, 883.61 secs, 4.77 msecs/r
  ./test/stock.ibd 27 -- read: 902675 requests, 1179864 pages, 16036.91 secs, 17.77 msecs/r, write: 577970 requests, 790063 pages, 2473.27 secs, 4.28 msecs/r
  ./test/order_line.ibd 25 -- read: 74232 requests, 92644 pages, 1217.65 secs, 16.40 msecs/r, write: 141432 requests, 274155 pages, 643.97 secs, 4.55 msecs/r
  ./test/new_orders.ibd 22 -- read: 4642 requests, 4960 pages, 81.02 secs, 17.45 msecs/r, write: 11482 requests, 60368 pages, 103.86 secs, 9.05 msecs/r
  ./test/history.ibd 21 -- read: 8006 requests, 11323 pages, 123.86 secs, 15.47 msecs/r, write: 24640 requests, 52809 pages, 119.01 secs, 4.83 msecs/r
  ./test/district.ibd 18 -- read: 14 requests, 14 pages, 0.14 secs, 10.35 msecs/r, write: 39 requests, 249 pages, 0.43 secs, 10.96 msecs/r
  ./test/item.ibd 16 -- read: 2892 requests, 3033 pages, 51.96 secs, 17.97 msecs/r, write: 0 requests, 0 pages, 0.00 secs, 0.00 msecs/r
  ./ib_logfile0 4294967280 -- read: 6 requests, 9 pages, 0.00 secs, 0.02 msecs/r, write: 314701 requests, 316680 pages, 6.73 secs, 0.02 msecs/r
```

## Insert Buffer Statistics ##

New output includes:
  * Ibuf read pages - number of requested and actual prefetch reads done to merge insert buffer records. InnoDB chooses entries to merge at random. If the number requested is much higher than the actual number, then the random algorithm is inefficient.
  * Ibuf merge - rate at which work is done for the insert buffer

```
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 3776, free list len 1895, seg size 5672,
984975 inserts, 454561 merged recs, 58782 merges
Ibuf read pages: 32960 requested, 36280 actual
Ibuf merge: 1229.9 requested_io/s, 39269.4 records_in/s, 19639.5 records_out/s, 2210.6 page_reads/s
```

## Log Statistics ##

This includes:
  * Foreground (Background) page flushes - page flushes done synchronously (asynchronously) by user sessions to maintain a small number of clean buffer pool pages.

```
LOG
---
Foreground page flushes:  sync 0 async 0
Background adaptive page flushes: 0
Foreground flush margins: sync 3025130459 async 2823455095
Space to flush margin:     sync 3000381113 async 2798705749
Current_LSN - Min_LSN     24749346
Checkpoint age            25858470
Max checkpoint age        3226805822
```

## Buffer Pool Statistics ##

This includes:
  * LRU\_old pages - number of **old** pages on the LRU list
  * Total writes - sources of pages for dirty page writes
  * Write sources - callers from which dirty page write requests are submitted
  * Foreground flushed dirty - number of dirty page writes submitted from a user session because the main background IO thread was too slow
  * Read ahead - number of prefetch read requests submitted because random or sequential access to an extent was detected
  * Pct\_dirty - percent of pages in the buffer pool that are dirty

```
BUFFER POOL AND MEMORY
----------------------
LRU_old pages      48109
Total writes: LRU 23271, flush list 1491363, single page 0
Write sources: free margin 23271, bg dirty 552374, bg lsn 0, bg extra 2742, recv 0, preflush 0
Foreground flushed dirty 935903
Read ahead: 44312 random, 355282 sequential
Pct_dirty 25.83
```

# SHOW STATUS #
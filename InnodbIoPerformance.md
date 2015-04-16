# Introduction #

It is one thing to publish performance results. It is another to understand them. The results here need more analysis and the code needs to be tested by others in the community.

This describes work to make InnoDB faster on IO bound workloads. The goal is to make it easy to use InnoDB on a server that can do 1000 to 10000 IOPs. A lot of problems must be fixed for that to be possible, but this is a big step towards that goal. These changes improve performance by 20% to more than 400% on several benchmarks. At a high level, these changes make InnoDB:
  * more efficient when processing IO requests
  * more likely to use available IO capacity
  * better at balancing different IO tasks
  * easier to monitor

One day, Heikki will write the _Complete Guide to InnoDB_, until then you need to consult multiple sources to understand the internals. It also helps to read the source code. These may help you to understand it:
  * [slides](http://docs.google.com/Presentation?id=dhngrkwh_7fn256bdj) from the Percona Performance Conference
  * [insert buffer in the MySQL manual](http://dev.mysql.com/doc/refman/5.0/en/innodb-insert-buffering.html)
  * blog post on [IO lag](http://mysqlha.blogspot.com/2008/07/how-do-you-know-when-innodb-gets-behind.html)
  * [another blog post on InnoDB IO performance](http://mysqlha.blogspot.com/2008/12/other-performance-problem.html)
  * [details on new my.cnf parameters for InnoDB IO performance](http://mysqlha.blogspot.com/2008/10/innodb-background-io.html)
  * [notes on my.cnf parameters for more background IO threads](http://mysqlha.blogspot.com/2008/10/more-background-io-threads-for-innodb.html)

# Features #

  * [displays more data](InnodbIoPerfStatus.md) in SHOW INNODB STATUS and SHOW STATUS including per-file IO statistics
  * [uses less CPU](InnodbIoPerfCpu.md) while processing background IO requests
  * [enforces innodb\_max\_dirty\_pages\_pct](InnodbIoPerfMaxDirty.md)
  * [prevents the insert buffer](InnodbIoIbuf.md) from getting full (and becoming useless)
  * [changes the main background IO thread](InnodbIoPerfMain.md) to be simpler and use more asynchronous IO
  * [adds many my.cnf variables](InnodbIoPerfConfig.md)
  * Reimplements [adaptive checkpoints](InnodbIoAdaptiveCheckpoint.md) as first described by Percona
  * Changes the computation of the percentage of dirty buffer pool pages. Before this change the percentage exluded pages borrowed from the buffer pool for other uses. While that may be more accurate, it also requires the caller to lock/unlock a hot mutex. It also made the percentage vary a bit too much as the insert buffer grew and shrank. The v4 patch doesn't exclude the borrowed pages. As most of the borrowed pages should be used in the insert buffer and the insert buffer should be smaller (thanks to _ibuf\_max\_pct\_of\_buffer_), this is _probably_ a good thing.

# Performance #

  * [in-memory OLTP](InnodbIoOltpMemory.md)
  * [disk-bound OLTP](InnodbIoOltpDisk.md)
  * [insert benchmark](InnodbIoInsert.md)
  * [results with InnoDB read prefetch enabled and disabled](InnodbIoPrefetch.md)
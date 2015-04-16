# SMP Performance Improvements #

This describes performance improvements from changes in the v2 Google patch. While the changes improve performance in many cases, a lot of work remains to be done. It improves performance on SMP servers by:
  * disabling the InnoDB memory heap and associated mutex
  * replacing the InnoDB rw-mutex and mutex on x86 platforms
  * linking with tcmalloc

While tcmalloc makes some of the workloads much faster, we don't recommend its use yet with MySQL as we are still investigating its behavior.

## Database reload ##

This displays the time to reload a large database shard on a variety of servers (HW + SW). Unless otherwise stated, my.cnf was optimized for a fast (but unsafe) reload with the following values. Note that **innodb\_flush\_method=nosync** is only in the Google patch and is **NOT** crash safe (kind of like MyISAM). This uses a real data set that produces a 100GB+ database.
  * innodb\_log\_file\_size=1300M
  * innodb\_flush\_method=nosync
  * innodb\_buffer\_pool\_size=8000M
  * innodb\_read\_io\_threads=4
  * innodb\_write\_io\_threads=2
  * innodb\_thread\_concurrency=20

The data to be reloaded was in one file per table on the db server. Each file was compressed and reloaded by a separate client. Each table was loaded by a separate connection except for the largest tables when there was no other work to be done. 8 concurrent connections were used.

The smpfix RPM is MySQL 5.0.37 plus the v1 Google patch and the SMP fixes that include:
  * InnoDB mutex uses atomic ops
  * InnoDB rw-mutex uses lock free methods to get and set internal lock state
  * tcmalloc is used in place of glibc malloc
  * the InnoDB malloc heap is disabled

The base RPM is MySQL 5.0.37 and the v1 Google patch. It does not have the SMP fixes.

The servers are:
  * 8core - the base RPM on an 8-core x86 server
  * 4core-128M - the base RPM on a 4-core x86 server with innodb\_log\_file\_size=128M
  * 8core-tc4 - the base RPM on an 8-core x86 server with innodb\_thread\_concurrency=4
  * smpfix-128M - the smpfix RPM with innodb\_log\_file\_size=128M
  * 4core - the base RPM on a 4-core x86 server
  * smpfix-4core - the smpfix RPM on a 4-core x86 server
  * smpfix-512M - the smpfix RPM on an 8-core x86 server with innodb\_log\_file\_size=512M
  * smpfix - the smpfix RPM on an 8-core x86 server
  * onlymalloc - the base RPM on an 8-core x86 server with the InnoDB malloc heap disabled
  * smpfix-notcmalloc - the smpfix RPM on an 8-core x86 server without tcmalloc

![http://google-mysql-tools.googlecode.com/files/reload_time.v1.png](http://google-mysql-tools.googlecode.com/files/reload_time.v1.png)

## Sysbench readwrite ##

[sysbench](http://sysbench.sourceforge.net) includes a transaction processing benchmark. The readwrite version of the sysbench OLTP test is measured here using 1, 2, 4, 8, 16, 32 and 64 threads.

### Configuration ###

Command line
```
# N is 1, 2, 4, 8, 16, 32 and 64
 --test=oltp --oltp-table-size=1000000 --max-time=600 --max-requests=0 --mysql-table-engine=innodb --db-ps-mode=disable --mysql-engine-trx=yes --num-threads=N
```

MySQL Options
```
innodb_buffer_pool_size=8192M
innodb_log_file_size=1300M
innodb_read_io_threads = 4
innodb_write_io_threads = 4
innodb_file_per_table
innodb_flush_log_at_trx_commit=2
innodb_log_buffer_size = 200m
innodb_thread_concurrency=0
log_bin
key_buffer_size = 50m
max_heap_table_size=1000M
max_heap_table_size=1000M
tmp_table_size=1000M
max_tmp_tables=100
```

### MySQL servers ###

The servers are:
  * base - MySQL 5.0.37 without the smp fix
  * tc4 - MySQL 5.0.37 without the smp fix, innodb\_thread\_concurrency=4
  * smpfix - MySQL 5.0.37 with the smp fix and tcmalloc
  * notcmalloc - MySQL 5.0.37 with the smp fix, not linked with tcmalloc
  * onlymalloc - MySQL 5.0.37 with the InnoDB malloc heap disabled
  * my4026 - unmodified MySQL 4.0.26
  * my4122 - unmodified MySQL 4.1.22
  * my5067 - unmodified MySQL 5.0.67
  * my5126 - unmodified MySQL 5.1.26
  * goog5037 - the same as base, MySQL 5.0.37 without the smp fix

### sysbench rw, 4-core server ###

![http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_4-core_server.png](http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_4-core_server.png)

### sysbench rw, 8-core server ###

![http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_8-core_server.v1.png](http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_8-core_server.v1.png)

### sysbench rw, 16-core server ###

![http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_16-core_server.png](http://google-mysql-tools.googlecode.com/files/sysbench_readwrite_on_16-core_server.png)

## Sysbench readonly ##

[sysbench](http://sysbench.sourceforge.net) includes a transaction processing benchmark. The readonly version of the sysbench OLTP test is measured here using 1, 2, 4, 8, 16, 32 and 64 threads.

### Configuration ###

Command line
```
# N is 1, 2, 4, 8, 16, 32 and 64
 --test=oltp --oltp-read-only --oltp-table-size=1000000 --max-time=600 --max-requests=0 --mysql-table-engine=innodb --db-ps-mode=disable --mysql-engine-trx=yes --num-threads=N
```

MySQL Options
```
innodb_buffer_pool_size=8192M
innodb_log_file_size=1300M
innodb_read_io_threads = 4
innodb_write_io_threads = 4
innodb_file_per_table
innodb_flush_log_at_trx_commit=2
innodb_log_buffer_size = 200m
innodb_thread_concurrency=0
log_bin
key_buffer_size = 50m
max_heap_table_size=1000M
max_heap_table_size=1000M
tmp_table_size=1000M
max_tmp_tables=100
```

### MySQL servers ###

The servers are:
  * base - MySQL 5.0.37 without the smp fix
  * tc4 - MySQL 5.0.37 without the smp fix, innodb\_thread\_concurrency=4
  * smpfix - MySQL 5.0.37 with the smp fix and tcmalloc
  * notcmalloc - MySQL 5.0.37 with the smp fix, not linked with tcmalloc
  * onlymalloc - MySQL 5.0.37 with the InnoDB malloc heap disabled
  * my4026 - unmodified MySQL 4.0.26
  * my4122 - unmodified MySQL 4.1.22
  * my5067 - unmodified MySQL 5.0.67
  * my5126 - unmodified MySQL 5.1.26
  * goog5037 - the same as base, MySQL 5.0.37 without the smp fix

### sysbench ro, 4-core server ###

![http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_4-core_server.png](http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_4-core_server.png)

### sysbench ro, 8-core server ###

![http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_8-core_server.v1.png](http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_8-core_server.v1.png)

### sysbench ro, 16-core server ###


![http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_16-core_server.png](http://google-mysql-tools.googlecode.com/files/sysbench_readonly_on_16-core_server.png)

## Concurrent joins ##

This test runs a query with a join. It is run using concurrent sessions. The data fits in the InnoDB buffer cache. The query is:
```
select count(*) from T1, T2 where T1.j > 0 and T1.i = T2.i
```

The data for T1 and T2 matches that used for the sbtest table by sysbench. This query does a full scan of T1 and joins to T2 by primary key.

The servers are:
  * base - MySQL 5.0.37 without the smp fix
  * tc4 - MySQL 5.0.37 without the smp fix, innodb\_thread\_concurrency=4
  * smpfix - MySQL 5.0.37 with the smp fix and tcmalloc
  * notcmalloc - MySQL 5.0.37 with the smp fix, not linked with tcmalloc
  * onlymalloc - MySQL 5.0.37 with the InnoDB malloc heap disabled
  * my4026 - unmodified MySQL 4.0.26
  * my4122 - unmodified MySQL 4.1.22
  * my5067 - unmodified MySQL 5.0.67
  * my5126 - unmodified MySQL 5.1.26
  * goog5037 - the same as base, MySQL 5.0.37 without the smp fix

### joins, 4-core server ###

TODO

### joins, 8-core server ###

Note, lower values for Time are better.

With data from the worst case:
![http://google-mysql-tools.googlecode.com/files/joins_on_8-core_server.v1.png](http://google-mysql-tools.googlecode.com/files/joins_on_8-core_server.v1.png)

Without data from the worst case:
![http://google-mysql-tools.googlecode.com/files/joins_on_8-core_server.o.v1.png](http://google-mysql-tools.googlecode.com/files/joins_on_8-core_server.o.v1.png)

### joins, 16-core server ###

Note, lower values for Time are better.

With data from the worst case:
![http://google-mysql-tools.googlecode.com/files/joins_on_16-core_server.png](http://google-mysql-tools.googlecode.com/files/joins_on_16-core_server.png)

Without data from the worst case:
![http://google-mysql-tools.googlecode.com/files/joins_on_16-core_server%2C_v2.png](http://google-mysql-tools.googlecode.com/files/joins_on_16-core_server%2C_v2.png)

## Concurrent inserts ##

This test reloads tables in parallel. Each connection inserts data for a different table. Tests were run using 1, 2, 4, 8 and 16 concurrent sessions. The regression for 5.0.37 is in the parser and was fixed by 5.0.54.

A separate table is used for each connection. DDL for the tables is:
```
create table T$i (i int primary key, j int, index jx(j)) engine=innodb
```

Multi-row insert statements are used that insert 1000 rows per insert statement. Auto-commit is used. The insert statements look like:
```
INSERT INTO T1 VALUES (0, 0), (1, 1), (2, 2), ..., (999,999);
```

The servers are:
  * base - MySQL 5.0.37 without the smp fix
  * tc4 - MySQL 5.0.37 without the smp fix, innodb\_thread\_concurrency=4
  * smpfix - MySQL 5.0.37 with the smp fix and tcmalloc
  * notcmalloc - MySQL 5.0.37 with the smp fix, not linked with tcmalloc
  * onlymalloc - MySQL 5.0.37 with the InnoDB malloc heap disabled
  * my4026 - unmodified MySQL 4.0.26
  * my4122 - unmodified MySQL 4.1.22
  * my5067 - unmodified MySQL 5.0.67
  * my5126 - unmodified MySQL 5.1.26
  * goog5037 - the same as base, MySQL 5.0.37 without the smp fix

MySQL 5.0.37 has a performance regression in the parser. This was fixed in 5.0.54.

### inserts, 4-core server ###

Note, lower values for Time are better.
![http://google-mysql-tools.googlecode.com/files/inserts_on_4-core_server.png](http://google-mysql-tools.googlecode.com/files/inserts_on_4-core_server.png)

### inserts, 8-core server ###

Note, lower values for Time are better.

With data from the worst case:
![http://google-mysql-tools.googlecode.com/files/inserts_on_8-core_server.v1.png](http://google-mysql-tools.googlecode.com/files/inserts_on_8-core_server.v1.png)

Without data from the worst case:
![http://google-mysql-tools.googlecode.com/files/inserts_on_8-core_server.o.v1.png](http://google-mysql-tools.googlecode.com/files/inserts_on_8-core_server.o.v1.png)

### inserts, 16-core server ###

Note, lower values for Time are better.

With data from the worst case:
![http://google-mysql-tools.googlecode.com/files/inserts_on_16-core_server.png](http://google-mysql-tools.googlecode.com/files/inserts_on_16-core_server.png)

Without data from the worst case:
![http://google-mysql-tools.googlecode.com/files/inserts_on_16-core_server%2C_v2.png](http://google-mysql-tools.googlecode.com/files/inserts_on_16-core_server%2C_v2.png)
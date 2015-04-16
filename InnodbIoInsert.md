These run the [insert benchmark](http://www.tokutek.com/benchmark.php) from Tokutek and measure the time to insert 50m rows into a table that starts with 50m rows. Four sessions continuously run queries concurrent with the inserts for the duration of the test.

The test configuration is:
  * 8-core server, 10 disks, SW RAID 0, 1MB stripe, ext2, 16G RAM

The results list:
  * average rows inserted per second for the duration of the test
  * number of seconds to run the test
  * average value for the percent of dirty buffer pool pages. This is computed using the static size of the buffer pool. It does not account for the usage of buffer pool pages for the insert buffer and elsewhere. Even if I wanted to include that, I cannot because only the Google patch has the insert buffer size in SHOW STATUS output.

Binaries tested:
  * v4-5037 -- v4 Google patch for MySQL 5.0.37
  * [xtradb-5134](http://www.percona.com/percona-lab.html) -- XtraDB for MySQL 5.1.34
  * mysql-5075 -- unmodified MySQL 5.0.75

## Results for innodb\_max\_dirty\_pages\_pct=20 ##

Notes:
  * v4-5037 was better at enforcing innodb\_max\_dirty\_pages\_pct, but this may be explained by the notes above.
  * for mysql-5075, the percentage of dirty pages varied between 20% and 50%

| **Binary** | **Inserts per second** | **Time** | **Avg pct dirty** |
|:-----------|:-----------------------|:---------|:------------------|
| v4-5037 | 1022 | 48928 | 24% |
| xtradb-5134 | 982 | 50905 | 29% |
| mysql-5075 | 306 | 163591 | 29% |

## Results for innodb\_max\_dirty\_pages\_pct=80 ##

  * xtradb-5134 may have been slower because it did not allow as many pages to be dirty. Given that the size of the insert buffer was limited to 200MB, the explanation should not be that 50% of the buffer pool pages were used for the insert buffer. But I don't have SHOW INNODB STATUS output from the test and XtraDB does not list insert buffer size in SHOW STATUS output, so the real answer must wait.

| **Binary** | **Inserts per second** | **Time** | **Avg pct dirty** |
|:-----------|:-----------------------|:---------|:------------------|
| v4-5037 | 1615 | 30959 | 53% |
| xtradb-5134 | 786 | 63598 | 31% |
| mysql-5075 | 305 | 164049 | 33% |

## Analysis ##

This provides more performance data:
  * Reads/s, Writes/s - average disk reads/writes per second from iostat
  * QPS - queries per second from the query sessions
  * CPU% - user + sys time from vmstat
  * Idle% - idle time from vmstat
  * Wait% - wait time from vmstat

| **Binary** | **Pct dirty** | **Reads/s** | **Writes/s** | **QPS** | **CPU%** | **Idle%** | **Wait%** |
|:-----------|:--------------|:------------|:-------------|:--------|:---------|:----------|:----------|
| xtradb-5134 | 20% | 581 | 353 | 43 | 8.6% | 47.0% | 45.0% |
| v4-5037 | 20% | 526 | 452 | 43 | 9.2% | 48.0% | 43.0% |
| xtradb-5134 | 80% | 684 | 190 | 50 | 7.9% | 43.3% | 48.3% |
| v4-5037 | 80% | 597 | 328 | 43 | 10.3% | 43.8% | 46.0% |
| mysql-5075 | 80% | 536 | 120 | 46 | 5.6% | 53% | 42% |

## Configuration ##

for v4-5037
```
innodb_buffer_pool_size=2G
innodb_log_file_size=1900M
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
innodb_io_capacity=1000
innodb_read_io_threads=4
innodb_write_io_threads=4
innodb_max_dirty_pages_pct=20 or 80
innodb_ibuf_max_pct_of_buffer=10
innodb_doublewrite=0
innodb_ibuf_flush_pct=40
skip_innodb_ibuf_reads_sync
innodb_check_max_dirty_foreground
innodb_adaptive_checkpoint
innodb_thread_concurrency=0
innodb_file_per_table
allow_view_trigger_sp_subquery
```

for xtradb-5134
```
innodb_log_file_size=1900M
innodb_buffer_pool_size=2G
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
innodb_io_capacity=1000
innodb_use_sys_malloc=0
innodb_read_io_threads=4
innodb_write_io_threads=4
innodb_max_dirty_pages_pct=20 or 80
innodb_ibuf_max_size=200M
innodb_ibuf_active_contract=1
innodb_ibuf_accel_rate=400
innodb_doublewrite=0
innodb_file_per_table
innodb_thread_concurrency=0
```

for mysql-5075
```
innodb_buffer_pool_size=2G
innodb_log_file_size=1900M
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
innodb_max_dirty_pages_pct=80
innodb_doublewrite=0
innodb_thread_concurrency=0
innodb_file_per_table
```
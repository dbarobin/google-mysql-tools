This compares performance for tpcc-mysql with a data set that does not fit in the InnoDB buffer cache.

The test configuration is:
  * 100 warehouses
  * 16, 32 and 48 concurrent users
  * 1200 second warmup
  * 14400 second runtime
  * 8-core server, 10 disks, SW RAID 0, 1MB stripe, ext2, 16G RAM

Tests were done with innodb\_max\_dirty\_pages\_pct set to 20 and 80. Only the v4 Google patch was able to come close to enforcing that limit.

Binaries tested:
  * v4-5037 -- v4 Google patch for MySQL 5.0.37
  * [xtradb-5134](http://www.percona.com/percona-lab.html) -- XtraDB for MySQL 5.1.34
  * mysql-5075 -- unmodified MySQL 5.0.75

The results list:
  * binary used
  * Throughput in TpmC
  * average value for the percent of dirty buffer pool pages. This is computed using the static size of the buffer pool. It does not account for the usage of buffer pool pages for the insert buffer and elsewhere. Even if I wanted to include that, I cannot because only the Google patch has the insert buffer size in SHOW STATUS output.

## Results with innodb\_max\_dirty\_pages\_pct=20 ##

Notes:
  * avg pct dirty is measured at 48 users
  * xtradb-5134 did not enforce innodb\_max\_dirty\_pages\_pct. Given the limits on the size of the insert buffer, the notes above do not explain this. I need to run another test and capture SHOW INNODB STATUS output.

| **Binary** | **TpmC @ 16 users** | **TpmC @ 32 users** | **TpmC @ 48 users** | **Avg pct dirty** |
|:-----------|:--------------------|:--------------------|:--------------------|:------------------|
| v4-5037 | 3386 | 3496 | 3516 | 26% |
| xtradb-5134 | 2727 | 2900 | 2883 | 45% |

## Results with innodb\_max\_dirty\_pages\_pct=80 ##

  * avg pct dirty is measured at 48 users
  * With XtraDB, performance gets worse at 80% dirty than at 20%. A similar result occurs for the insert benchmark. I don't understand why (yet).

| **Binary** | **TpmC @ 16 users** | **TpmC @ 32 users** | **TpmC @ 48 users** | **Avg pct dirty** |
|:-----------|:--------------------|:--------------------|:--------------------|:------------------|
| v4-5037 | 3857 | 4097 | 4168 | 55% |
| xtradb-5134 | 3078 | 2131 | 1888 | 63% |
| mysql-5075 | 3421 | 3548 | 3543 | 56% |

## Analysis ##

This provides more performance data for the test with 48 concurrent users:
  * Reads/s, Writes/s - average disk reads/writes per second from iostat
  * CPU% - user + sys time from vmstat
  * Idle% - idle time from vmstat
  * Wait% - wait time from vmstat

Notes:
  * CPU% is low for pct\_dirty=20% and v4-5037 on this test and the 20 warehouse tpcc-mysql result. TODO -- explain that

| **Binary** | **Pct dirty** | **Reads/s** | **Writes/s** | **CPU%** | **Idle%** | **Wait%** |
|:-----------|:--------------|:------------|:-------------|:---------|:----------|:----------|
| xtradb-5134 | 20% | 678 | 430 | 30.2% | 21.8% | 48.1% |
| v4-5037 | 20% | 731 | 648 | 17.2% | 32.6% | 50.2% |
| xtradb-5134 | 80% | 412 | 201 | 26.8% | 43.4% | 30.0% |
| v4-5037 | 80% | 893 | 490 | 20.8% | 25.6% | 53.8% |
| mysql-5075 | 80% | 797 | 432 | 47.4% | 26.5% | 27.5% |

90% percentile response time in seconds:

| **Binary** | **Pct dirty** | **New-Order** | **Payment** | **Order-Status** | **Delivery** | **Stock-Level** |
|:-----------|:--------------|:--------------|:------------|:-----------------|:-------------|:----------------|
| xtradb-5134 | 20% | 1.40 | 0.20 | 0.40 | 2.20 | 2.60 |
| v4-5037 | 20% | 1.40 | 0.60 | 0.40 | 0.80 | 1.00 |
| xtradb-5134 | 80% | 1.60 | 0.40 | 0.60 | 3.40 | 3.60 |
| v4-5037 | 80% | 0.80 | 0.20 | 0.40 | 1.40 | 2.40 |
| mysql-5075 | 80% | 1.00 | 0.20 | 0.40 | 1.40 | 2.20 |

Max response time in seconds:

| **Binary** | **Pct dirty** | **New-Order** | **Payment** | **Order-Status** | **Delivery** | **Stock-Level** |
|:-----------|:--------------|:--------------|:------------|:-----------------|:-------------|:----------------|
| xtradb-5134 | 20% | 7.49 | 3.32 | 4.48 | 7.70 | 14.74 |
| v4-5037 | 20% | 4.49 | 3.63 | 1.74 | 4.15 | 8.80 |
| xtradb-5134 | 80% | 9.33 | 3.31 | 3.57 | 8.54 | 22.08 |
| v4-5037 | 80% | 2.95 | 2.62 | 2.21 | 3.69 | 6.52 |
| mysql-5075 | 80% | 8.29 | 6.51 | 4.11 | 6.52 | 16.06 |



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
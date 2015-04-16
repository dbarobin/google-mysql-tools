This compares performance for tpcc-mysql with a data set that fits in the InnoDB buffer cache.

The test configuration is:
  * 20 warehouses
  * 1200 second warmup
  * 8, 16 and 32 concurrent users
  * 14400 second runtime
  * 8-core server, 10 disks, SW RAID 0, 1MB stripe, ext2

Tests were done with innodb\_max\_dirty\_pages\_pct set to 20 and 80. Only the v4 Google patch was able to come close to enforcing that limit. That is why XtraDB was faster at innodb\_max\_dirty\_pages\_pct=20.

Binaries tested:
  * v4-5037 -- v4 Google patch for MySQL 5.0.37
  * [xtradb-5134](http://www.percona.com/percona-lab.html) -- XtraDB for MySQL 5.1.34
  * mysql-5075 -- unmodified MySQL 5.0.75

The results list:
  * binary used
  * Throughput in TpmC
  * average value for the percent of dirty buffer pool pages. This is computed using the static size of the buffer pool. It does not account for the usage of buffer pool pages for the insert buffer and elsewhere. Even if I wanted to include that, I cannot because only the Google patch has the insert buffer size in SHOW STATUS output.

## Results with innodb\_max\_dirty\_pages\_pct=20 ##

  * avg pct dirty is measured at 32 users
  * v4-5037 is better at enforcing innodb\_max\_dirty\_pages\_pct

| **Binary** | **TpmC @ 8 users** | **TpmC @ 16 users** | **TpmC @ 32 users** | **Avg pct dirty** |
|:-----------|:-------------------|:--------------------|:--------------------|:------------------|
| v4-5037 | 24433 | 28560 | 28464 | 24% |
| xtradb-5134 | 24422 | 32231 | 33235 | 28% |

## Results with innodb\_max\_dirty\_pages\_pct=80 ##

  * avg pct dirty is measured at 32 users

| **Binary** | **TpmC @ 8 users** | **TpmC @ 16 users** | **TpmC @ 32 users** | **Avg pct dirty** |
|:-----------|:-------------------|:--------------------|:--------------------|:------------------|
| v4-5037 | 27701 | 36674 | 36053 | 35% |
| xtradb-5134 | 24748 | 32798 | 33618 | 34% |
| mysql-5075 | 26274 | 32847 | 29632 | 32% |

## Analysis ##

This provides more performance data for the test with 32 concurrent users:
  * Reads/s, Writes/s - average disk reads/writes per second from iostat
  * CPU% - user + sys time from vmstat
  * Idle% - idle time from vmstat
  * Wait% - wait time from vmstat

Notes:
  * Given that Idle is high for v4-5037 with pct\_dirty=80% there might be too much mutex contention

| **Binary** | **Pct dirty** | **Reads/s** | **Writes/s** | **CPU%** | **Idle** | **Wait** |
|:-----------|:--------------|:------------|:-------------|:---------|:---------|:---------|
| xtradb-5134 | 20% | 20 | 212 | 93.8% | 5.2% | 1.2% |
| v4-5037 | 20% | 13 | 312 | 83.8% | 14.2% | 2.0% |
| xtradb-5134 | 80% | 19 | 128 | 94.3% | 4.9% | 0.7% |
| v4-5037 | 80% | 14 | 110 | 93.8% | 5.5% | 0.7% |
| mysql-5075 | 80% | 13 | 76 | 92.7% | 6.6% | 0.4% |


90% percentile response time in seconds:

| **Binary** | **Pct dirty** | **New-Order** | **Payment** | **Order-Status** | **Delivery** | **Stock-Level** |
|:-----------|:--------------|:--------------|:------------|:-----------------|:-------------|:----------------|
| xtradb-5134 | 20% | 0.20 | 0.20 | 0.20 | 0.20 | 0.20 |
| v4-5037 | 20% | 0.20 | 0.20 | 0.20 | 0.20 | 0.20 |
| xtradb-5134 | 80% | 0.20 | 0.20 | 0.20 | 0.20 | 0.20 |
| v4-5037 | 80% | 0.20 | 0.20 | 0.20 | 0.20 | 0.20 |
| mysql-5075 | 80% | 1.00 | 0.20 | 0.40 | 1.40 | 2.20 |

Max response time in seconds:

| **Binary** | **Pct dirty** | **New-Order** | **Payment** | **Order-Status** | **Delivery** | **Stock-Level** |
|:-----------|:--------------|:--------------|:------------|:-----------------|:-------------|:----------------|
| xtradb-5134 | 20% | 2.10 | 2.05 | 0.72 | 3.13 | 3.87 |
| v4-5037 | 20% | 2.23 | 1.34 | 0.61 | 0.83 | 4.95 |
| xtradb-5134 | 80% | 9.50 | 5.60 | 1.75 | 2.88 | 3.55 |
| v4-5037 | 80% | 2.95 | 2.62 | 2.21 | 3.69 | 6.52 |
| mysql-5075 | 80% | 8.29 | 6.51 | 4.11 | 6.52 | 16.06 |

## Configuration ##

for v4-5037
```
innodb_buffer_pool_size=8G
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
innodb_buffer_pool_size=8G
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
innodb_buffer_pool_size=8G
innodb_log_file_size=1900M
innodb_flush_log_at_trx_commit=2
innodb_flush_method=O_DIRECT
innodb_max_dirty_pages_pct=80
innodb_doublewrite=0
innodb_thread_concurrency=0
innodb_file_per_table
```
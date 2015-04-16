# Introduction #

The tests from InnodbIoOltpMemory, InnodbIoOltpDisk and InnodbIoInsert were repeated with InnoDB read prefetch enabled and disabled. This is done to determine whether prefetch improves performance.

First, there are cases where InnoDB prefetch improves performance. A query that does a full table scan was run with and without prefetch enabled. The query was 35% faster with prefetch enabled.

For these benchmarks, disabling prefetch improved performance for all but one configuration. This lists the performance with prefetch disabled relative to it with prefetch enabled. A value greater than 1.0 means that performance is faster with it disabled.
  * 0.96, 1.03 for the insert benchmark
  * 1.10, 1.11 for the OLTP in-memory benchmark
  * 1.12, 1.20 for the OLTP disk-bound benchmark

The test configuration is:
  * 8-core server, 10 disks, SW RAID 0, 1MB stripe, ext2, 16G RAM

Binaries tested:
  * v4-5037 -- v4 Google patch for MySQL 5.0.37 with options to enable/disable prefetch
    * --innodb\_readahead\_random - enable/disable prefetch during random access
    * --innodb\_readahead\_sequential -- enable/diable prefetch during sequential access

# Results for the insert benchmark #

The test is described in InnodbIoInsert. Results are only provided for MySQL 5.0.37 with the v4 Google patch.

The results list:
  * value for innodb\_max\_dirty\_pages\_pct
  * whether Innodb prefetch is enabled or disabled
  * average rows inserted per second for the duration of the test
  * number of seconds to run the test
  * ratio of inserts per second (prefetch off / prefetch on)
  * number of page reads for random and sequential prefetch
  * total number of reads

| **innodb\_max\_dirty\_pages\_pct** | **prefetch** | **Inserts per second** | **Time** | **ratio** | **random prefetch** | **sequential prefetch** | **total reads** |
|:-----------------------------------|:-------------|:-----------------------|:---------|:----------|:--------------------|:------------------------|:----------------|
| 20% | on | 1030 | 48537 | - | 50618 | 4941247 | 36101926 |
| 20% | off | 992 | 50409 | 0.96 |  0 | 0 | 30331187 |
| - | - | - | - | - | - | - | - |
| 80% | on | 1612 | 31022 | - | 29737 | 2537532 | 27038651 |
| 80% | off | 1666 | 30004 | 1.03 | 0 | 0 | 23439753 |

# Result for the OLTP in-memory benchmark #

The test is described in InnodbIoOltpMemory. Results are only provided for MySQL 5.0.37 with the v4 Google patch.

The results list:
  * value for innodb\_max\_dirty\_pages\_pct
  * whether InnoDB read prefetch is enabled or disabled
  * TpmC for the test
  * ratio of TpmC (prefetch off / prefetch on)
  * number of page reads for random and sequential prefetch
  * total number of reads

| **innodb\_max\_dirty\_pages\_pct** | **prefetch** | **TpmC** | **ratio**| **random prefetch** | **sequential prefetch** | **total reads** |
|:-----------------------------------|:-------------|:---------|:|:--------------------|:------------------------|:----------------|
| 20% | on | 33209 | - | 355 | 3100 | 110826 |
| 20% | off | 36512 | 1.10 | 0 | 0 | 107411 |
| - | - | - | - | - | - | - |
| 80% | on | 34043 | - | 415 | 30544 | 110460 |
| 80% | off | 37661 | 1.11 | 0 | 0 | 107494 |

# Result for the OLTP disk-bound benchmark #

The test is described in InnodbIoOltpDisk. Results are only provided for MySQL 5.0.37 with the v4 Google patch.

The results list:
  * value for innodb\_max\_dirty\_pages\_pct
  * whether InnoDB read prefetch is enabled or disabled
  * TpmC for the test
  * ratio of TpmC (prefetch off / prefetch on)
  * number of page reads for random and sequential prefetch
  * total number of reads

| **innodb\_max\_dirty\_pages\_pct** | **prefetch** | **TpmC** | **ratio**|  **random prefetch** | **sequential prefetch** | **total reads** |
|:-----------------------------------|:-------------|:---------|:|:---------------------|:------------------------|:----------------|
| 20% | on | 3312 | - | 87207 | 714499 | 3421592 |
| 20% | off | 3718 | 1.12 | 0 | 0 | 2713576 |
| - | - | - | - | - | - | - |
| 80% | on | 3876 | - | 134158 | 804011 | 4061826 |
| 80% | off | 4662 | 1.20 | 0 | 0 | 3552835 |
# Introduction #

The my.cnf option rpl\_event\_checksums enables the use of binlog event checksums. Note that both slaves and masters must used this patch when this is enabled. It is disabled by default.


# Details #

From tests I ran in the past, the use of checksums reduced performance by 10%. For tests that I run today it reduces performance by 3% to 7% at high levels of concurrency. The checksum is computed with crc32. The use of adler32 from zlib in place of crc32 does not change performance. These are results from sysbench readwrite on a 16-core server for 1 to 64 concurrent users:

| **checksum** | **1 user** | **2 users** | **4 users** | **8 users** | **16 users** | **32 users** | **64 users** |
|:-------------|:-----------|:------------|:------------|:------------|:-------------|:-------------|:-------------|
| none | 170 | 379 | 702 | 1113 | 1227 | 1191 | 1103 |
| crc32 | 171 | 371 | 688 | 1064 | 1205 | 1153 | 1067 |
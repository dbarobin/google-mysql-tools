# Details #

InnoDB was changed to reduce mutex contention on the transaction log and buffer pool mutexes (log\_sys->mutex, buffer\_pool->mutex). It was also changed to reduce contention between them because on commit a session would lock log\_sys->mutex and then lock buffer\_pool->mutex. Now it locks log\_sys->mutex and then locks buffer\_pool->flush\_list\_mutex.

Changes:
  * added flush\_list mutex to protect buffer\_pool->flush list independent of buffer\_pool->mutex.
  * added array of mutexes to protect the hash table of pages (buffer\_pool->page\_hash) independent of buffer\_pool->mutex

# Query Performance #

Changes were made to reduce contention on log\_sys->mutex and buffer\_pool->mutex. At high levels of concurrency this improves performance by 20% on sysbench readwrite with a 16-core server.

| **binary** | **1 user** | **2 users** | **4 users** | **8 users** | **16 users** | **32 users** | **64 users** |
|:-----------|:-----------|:------------|:------------|:------------|:-------------|:-------------|:-------------|
| without fix | 173 | 379 | 697 | 1105 | 1233 | 1174 | 1095 |
| with fix | 179 | 388 | 700 | 1175 | 1453 | 1491 | 1263 |

The same data [graphed](http://chart.apis.google.com/chart?chs=400x200&cht=lc&chxt=x,y,x,y&chd=t:173,379,697,1105,1233,1174,1095|179,388,700,1175,1453,1491,1263&chdl=without_fix|with_fix&chtt=&chco=FF0000,00FF00&chds=0,1491&chxr=1,0,2000&chxl=0:|1|2|4|8|16|32|64|2:|Concurrent%20users||3:|Throughput|&chxp=2,50|3,50&chg=10,10)

# Database reload performance #

This lists the time to reload a production database server with ~130GB of data for a real and complex schema into InnoDB.

| **binary** | **seconds to reload** | **speedup from base** |
|:-----------|:----------------------|:----------------------|
| 5.0.37 | 49331 | NA |
| 5.0.37 & v2 google patch | 8298 | 5.9X |
| 5.0.37 & v3 google patch | 7248 | 6.8X |
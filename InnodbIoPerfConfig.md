These InnoDB my.cnf variables are new in the Google patches:
  * innodb\_max\_merged\_io - maximum number of IO requests merged to issue large IO from background IO threads.
  * innodb\_read\_io\_threads - number of background read I/O threads in InnoDB.
  * innodb\_write\_io\_threads - number of background write I/O threads in InnoDB.
  * innodb\_adaptive\_checkpoint - makes the background IO thread flush dirty pages when are there old pages that will delay a checkpoint. OFF provides traditional behavior.
  * innodb\_check\_max\_dirty\_foreground - make user sessions flush some dirty pages when innodb\_max\_dirty\_pages\_pct has been exceeded. OFF provides traditional behavior.
  * innodb\_file\_aio\_stats - compute and export per-file IO statistics for InnoDB.
  * innodb\_flush\_adjacent\_background - when background IO threads flush dirty pages, flush adjacent dirty pages from the same extent. ON provides traditional behavior.
  * innodb\_flush\_adjacent\_foreground - when user sessions flush dirty pages, flush adjacent dirty pages from the same extent. ON provides traditional behavior.
  * innodb\_ibuf\_flush\_pct - percent of innodb\_io\_capacity that should be used for prefetch reads used to merge insert buffer entries.
  * innodb\_ibuf\_max\_pct\_of\_buffer - soft limit for the percent of buffer cache pages that can be used for the insert buffer. When this is exceeded background IO threads work harder to merge insert buffer entries. The hard limit is 50%. The traditional value is 50%.
  * innodb\_ibuf\_reads\_sync - use sync IO to read blocks for insert buffer merges. ON provides traditional behavior.
  * innodb\_io\_capacity - maximum number of concurrent IO requests that should be done to flush dirty buffer pool pages. CAUTION -- setting this too high will use a lot of CPU to schedule IO requests and more than 1000 might be too high. The traditional value is 100.
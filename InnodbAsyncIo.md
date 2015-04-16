# Introduction #

InnoDB supports asynchronous IO for Windows. For Linux, it uses 4 threads to perform background IO tasks and each thread uses synchronous IO. There is one thread for each of:
  * insert buffer merging
  * log IO
  * read prefetch requests
  * writing dirty buffer cache pages

InnoDB issues prefetch requests when it detects locality in random IO and when it detects a sequential scan. However, it only uses one thread to execute these requests. Multi-disk servers are best utilized when more IO requests can be issued concurrently.

For deployments that use buffered IO rather than direct IO or some type of remote disk (SAN, NFS, NAS), there is not much of a need for more write threads because writes complete quickly into the OS buffer cache. However, as servers with many GB of RAM are used, it is frequently better to use direct IO.

We have changed InnoDB to support a configurable number of background IO threads for read and write requests. This is controlled by the parameters:
  * **innodb\_max\_merged\_io** - Max number of IO requests merged to issue large IO from background IO threads
  * **innodb\_read\_io\_threads** - the number of background IO threads for read prefetch requests
  * **innodb\_write\_io\_threads** - the number of background IO threads for writing dirty pages from the buffer cache


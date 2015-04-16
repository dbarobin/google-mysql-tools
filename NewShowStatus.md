# Overview #

We have added extra values for monitoring. Much of the data from **SHOW INNODB STATUS** is now available in **SHOW STATUS**.

We have also added rate limiting for both **SHOW STATUS** and **SHOW INNODB STATUS** to reduce the overhead from overzealous monitoring tools. This limits how frequently the expensive operations are done for these SHOW commands.

# General #

The new values include:
  * Binlog\_events - number of replication events written to the binlog
  * Binlog\_largest\_event - larget event in the current binlog
  * Denied\_connections - number of connection attempts that fail because of the max\_connections limit
  * Malloc\_sbrk\_bytes\_alloc, Malloc\_chunks\_free, Malloc\_mmap\_chunks\_alloc, Malloc\_mmap\_bytes\_alloc, Malloc\_bytes\_used, Malloc\_bytes\_free - values reported from mallinfo()
  * Gettimeofday\_errors - errors for gettimeofday calls (yes, this happens)
  * Sort\_filesort\_old - number of times the old filesort algorithm is used
  * Sort\_filesort\_new - number of times the new filesort algorithm is used

# Replication #
  * Replication\_fail\_io\_connections - on a slave, number of times the IO thread has disconnected from the master because of an error
  * Replication\_total\_io\_connections - number of connections made by the IO thread to the master
  * Replication\_last\_event\_buffered - on a slave, time when last replication event received
  * Replication\_last\_event\_done - on a slave, time when last replication event replayed

# Semi-synchronous replication #
  * Rpl\_semi\_sync\_clients - number of semi-sync clients connected to a master
  * Rpl\_semi\_sync\_net\_avg\_wait\_time(us) - average time to wait for an acknowledgement of a replication event from a semi-sync slave
  * Rpl\_semi\_sync\_net\_wait\_time - total time waiting for acknowledgement
  * Rpl\_semi\_sync\_net\_waits
  * Rpl\_semi\_sync\_no\_times
  * Rpl\_semi\_sync\_no\_tx - number of transactions not acknowledged by semi-sync slaves
  * Rpl\_semi\_sync\_status - indicates whether semi-sync is enabled
  * Rpl\_semi\_sync\_slave\_status
  * Rpl\_semi\_sync\_timefunc\_failures
  * Rpl\_semi\_sync\_tx\_avg\_wait\_time(us) - average time a sessions waits for commit to finish
  * Rpl\_semi\_sync\_tx\_wait\_time
  * Rpl\_semi\_sync\_tx\_waits
  * Rpl\_semi\_sync\_wait\_pos\_backtraverse
  * Rpl\_semi\_sync\_wait\_sessions
  * Rpl\_semi\_sync\_yes\_tx - number of transactions acknowledged by semi-sync slaves
  * Rpl\_transaction\_support

# Innodb #

  * Innodb\_dict\_size - number of bytes used for the InnoDB dictionary
  * Innodb\_have\_atomic\_builtins - indicates whether InnoDB uses atomic memory operations in place of pthreads synchronization functions
  * Innodb\_heap\_enabled - indicates  whether the InnoDB malloc heap was enabled -- see [Bug38531](http://bugs.mysql.com/bug.php?id=38531)
  * Innodb\_long\_lock\_wait - set when there is a long lock wait on an internal lock. These usually indicate an InnoDB bug. They also occur because the adaptive hash latch is not always released when it should be (such as during an external sort).
  * Innodb\_long\_lock\_waits - incremented once for each internal long lock wait
  * Innodb\_os\_read\_requests - from SHOW INNODB STATUS
  * Innodb\_os\_write\_requests - from SHOW INNODB STATUS
  * Innodb\_os\_pages\_read - from SHOW INNODB STATUS
  * Innodb\_os\_pages\_written - from SHOW INNODB STATUS
  * Innodb\_os\_read\_time - from SHOW INNODB STATUS
  * Innodb\_os\_write\_time - from SHOW INNODB STATUS
  * Innodb\_time\_per\_read - average microseconds per read
  * Innodb\_time\_per\_write - average microseconds per write
  * Innodb\_deadlocks - application deadlocks, detected automatically
  * Innodb\_transaction\_count - from SHOW INNODB STATUS
  * Innodb\_transaction\_purge\_count - from SHOW INNODB STATUS
  * Innodb\_transaction\_purge\_lag - count of work to be done by the InnoDB purge thread, see [InnodbLag](http://mysqlha.blogspot.com/2008/07/how-do-you-know-when-innodb-gets-behind.html)
  * Innodb\_active\_transactions - from SHOW INNODB STATUS
  * Innodb\_summed\_transaction\_age - from SHOW INNODB STATUS
  * Innodb\_longest\_transaction\_age - from SHOW INNODB STATUS
  * Innodb\_lock\_wait\_timeouts - count of lock wait timeouts
  * Innodb\_lock\_waiters - from SHOW INNODB STATUS
  * Innodb\_summed\_lock\_wait\_time - from SHOW INNODB STATUS
  * Innodb\_longest\_lock\_wait - from SHOW INNODB STATUS
  * Innodb\_pending\_normal\_aio\_reads - from SHOW INNODB STATUS
  * Innodb\_pending\_normal\_aio\_writes - from SHOW INNODB STATUS
  * Innodb\_pending\_ibuf\_aio\_reads - from SHOW INNODB STATUS
  * Innodb\_pending\_log\_ios - from SHOW INNODB STATUS
  * Innodb\_pending\_sync\_ios - from SHOW INNODB STATUS
  * Innodb\_os\_reads - from SHOW INNODB STATUS
  * Innodb\_os\_writes - from SHOW INNODB STATUS
  * Innodb\_os\_fsyncs - from SHOW INNODB STATUS
  * Innodb\_ibuf\_inserts - from SHOW INNODB STATUS
  * Innodb\_ibuf\_size - counts work to be done by the insert buffer, see [InnodbLag](http://mysqlha.blogspot.com/2008/07/how-do-you-know-when-innodb-gets-behind.html)
  * Innodb\_ibuf\_merged\_recs - from SHOW INNODB STATUS
  * Innodb\_ibuf\_merges - from SHOW INNODB STATUS
  * Innodb\_log\_ios\_done - from SHOW INNODB STATUS
  * Innodb\_buffer\_pool\_hit\_rate - from SHOW INNODB STATUS
# Note #

This code has been built on an x86-64 server that uses gcc and Linux 2.6. I don't know if it works on other platforms.

The code is [here for 5.1.26](http://code.google.com/p/google-mysql-tools/source/browse/trunk/mysql-patches/mutexstats-5.1.26.patch) and [here for 5.1.31](http://code.google.com/p/google-mysql-tools/source/browse/trunk/mysql-patches/mutexstats-5.1.31).

# Introduction #

This patch provides mutex contention statistics for MySQL via the SQL command SHOW GLOBAL MUTEX STATUS. It is similar to the output provided for InnoDB mutexes by the SQL command [SHOW MUTEX STATUS](http://dev.mysql.com/doc/refman/5.0/en/show-mutex-status.html). It optionally reports lock waits by caller.

To enable it, apply this patch, run _configure_ with the **--with-fast-mutexes** option and build. To get lock waits by caller, run _configure_ with **-with-fast-mutexes C\_EXTRA\_FLAGS=-DMY\_COUNT\_MUTEX\_CALLERS**.

The fast mutex code uses busy-wait loops before blocking on lock requests. If the lock is not obtained during the busy-wait loop, the code assumes that the caller has to block on the lock request. The duration of the busy-wait loop is determined by the my.cnf variable **mysql\_spin\_wait\_loops**. The default value is 100 which produces a ~6 microsecond delay on a current x86\_64 CPU. The delay is measured at mysqld startup and printed in the database error log. It is also displayed in the SHOW STATUS variable **Mysql\_spin\_wait\_microseconds**.

This patch also changes all pthread\_mutex\_init calls to use MY\_MUTEX\_INIT\_FAST.

I won't claim that fast mutexes improve performance. But they make it possible to estimate mutex contention. I use this for performance testing builds.

The data is exported by the SQL command SHOW GLOBAL MUTEX STATUS. Sample output is displayed below. The output has several columns:
  * **Locks** is the number of times a blocking pthread call had to be used to lock the mutex or rw-lock. This means that the lock could not be obtained by the non-blocking trylock commands during the busy-wait loop.
  * **Spins** is the number of rounds done by the busy-wait loop. This is at most 4 times **Locks**.
  * **Sleeps** is the number of blocking lock attempts. This is <= **Locks**. When MySQL is built to count mutex contention by lock callers, values are positive when the line reports contention for a mutex and negative when the line reports contention for a lock caller.
  * **Name** is the name of the file at which the mutex or rw-lock is created.
  * **Line** is the line number in the file at which the mutex or rw-lock is created.
  * **Users** is the number of times the mutex or rw-lock is created.

Sample output:
```
Locks   Spins   Sleeps  Name    Line    Users
1       0       0       my_thr_init.c   287     68
172     0       0       mysqld.cc       3081    1
0       0       0       slave.h 453     1
4229999 114473  1883    thr_lock.c      319     111742
0       0       -225    thr_lock.c      325     0
```

This displays the estimated busy-wait time.
```
show status like "%spin%";
Variable_name	Value
Mysql_spin_wait_microseconds	6
```

The statistics may be inaccurate:
  * locks are not used to read or write the statistics counters
  * a fixed-size array of counters is used. An array entry may be used by multiple mutexes and rw-locks because the entry to use for a given mutex or rw-lock is determined by computing a hash function from the filename and line number from the code that creates the lock

# Thanks #

I copied code from InnoDB for hashing and random number generation. Thanks InnoDB.
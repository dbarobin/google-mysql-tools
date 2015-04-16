# Introduction #

Author: Ben Handy

InnoDB has been changed to run faster on SMP servers. The improvements are significant on servers with 8+ cores. The changes include:
  * use atomic memory operations for the Innodb mutex
  * use tcmalloc and disable the InnoDB memory heap (see [Bug38531](http://bugs.mysql.com/bug.php?id=38531))
  * use atomic memory operations for the Innodb rw-mutex.

# Details #

TODO
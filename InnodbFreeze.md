# Commands #

  * **set global innodb\_disallow\_writes=ON**
  * **set global innodb\_disallow\_writes=OFF**

# Purpose #

These enable and disable all Innodb file system activity except for reads. If you want to take a database backup without stopping the server and you don't use LVM, ZFS or some other storage software that provides snapshots, then you can use this to halt all destructive file system activity from InnoDB and then backup the InnoDB data files. Note that it is not sufficient to run **FLUSH TABLES WITH READ LOCK** as there are background IO threads used by InnoDB that may still do IO.

[5.0.37 Patch](http://code.google.com/p/google-mysql-tools/source/browse/trunk/mysql-patches/innodb_disallow_writes-5.0.37.patch)
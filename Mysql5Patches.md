# Introduction #

The code has been changed to make MySQL more manageable, available and scalable. Many problems remain to be solved to improve SMP performance. This is a good start. The v3 patch and all future patches will be published with a BSD license which applies to code we have added and changed. Original MySQL sources has a GPL license.

These have the same functionality as the MySQL 4 patches. There are several patch sets:
  * the [v1 patch](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/mysql-5.0.37-patches) published in 2007
  * the [v2 patch](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/all.v2-mysql-5.0.37.patch.gz) with all of our changes for MySQL 5.0.37
  * the [v3 patch](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/all.v3-mysql-5.0.37.patch.gz) with all of our changes for MySQL 5.0.37 as of May 6, 2009. This adds global transaction IDs, row-change logging and more InnoDB SMP performance fixes.
  * the [v4 patch](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/all.v4-mysql-5.0.37.patch.gz) as of June 1, 2009. [This makes InnoDB faster](InnodbIoPerformance.md) on IO bound loads.
  * the [semi-sync replication V1](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/mysql-5.0.37_semisync.patch) patch published in 2007.
  * the [mutex contention statistics patch](http://code.google.com/p/google-mysql-tools/source/browse/trunk/old/mysql-patches/mutexstats-5.1.26.patch) for MySQL 5.1.26
  * the [patch to improve SMP performance](http://code.google.com/p/google-mysql-tools/source/browse/trunk/old/mysql-patches/smp-5.0.67.patch) for MySQL 5.0.67. This has two changes:
    * use atomic memory instructions for the InnoDB mutex and rw-mutex. This is only done for x86 platforms that use a recent (>= 4.1) version of GCC.
    * disable the InnoDB memory heap. Thisi s done for all platforms.
  * the [patch to improve SMP performance](http://code.google.com/p/google-mysql-tools/source/browse/trunk/old/mysql-patches/smp_plugin_1.0.1.patch) for the InnoDB 1.0.1 plugin in MySQL 5.1
  * [disable/enable writes to InnoDB files](http://code.google.com/p/google-mysql-tools/source/browse/old/trunk/mysql-patches/innodb_disallow_writes-5.0.37.patch) via _set global innodb\_disallow\_writes=(0|1)_
  * [Patch](http://code.google.com/p/google-mysql-tools/source/browse/old/trunk/mysql-patches/rwlock_mutex_5.0.patch) to use pthread\_mutex\_t instead of mutex\_t for rw\_lock\_struct::mutex in InnoDB.
  * the [global transaction IDs and binlog event checksums stand-alone patch](http://google-mysql-tools.googlecode.com/svn/trunk/old/mysql-patches/global_trx_ids-5.0.68.patch.gz) extracted out of the big V3 patch and ported to mysql-5.0.68 as of May 12, 2009.
  * Notes on implementing synchronous replication -- MysqlSyncReplication

SmpPerformance has results for a variety of MySQL versions on 4, 8 and 16 core servers.

# Feedback, Problems and Comments #

Use the [google-mysql-tools group](http://groups.google.com/group/google-mysql-tools)

# Disclaimer #

We have changed a lot of code. Not all of the changes are described here and some of the changes to default behavior from new my.cnf variables can break your applications. Unless your name rhymes with Domas, it might be better to take pieces of the patch rather than try to use all of it.

The code has been tested on 32-bit and 64-bit Linux x86. We may have broken the build for other platforms.

The embedded server, **--with-embedded-server**, cannot be built with these changes. We have broken the build for it.

Many of the Makefile.in and Makefile.am files have been changed in the big patch because we changed InnoDB to use the top-level configure.

If you try to install the big patch, treat it like [installing from a source tree](http://dev.mysql.com/doc/refman/5.0/en/installing-source-tree.html).

# Authors #

A lot of people have contributed to this:
  * Wei Li
  * Gene Pang
  * Eric Rollins
  * Ben Handy
  * Justin Tolmer
  * Larry Zhou
  * Yuan Wei
  * Robert Banz
  * Chip Turner
  * Steve Gunn
  * Mark Callaghan

# The v2 patch #

This has many new features and a few non-features. Embedded MySQL will not work with this patch.
  * SqlChanges
  * SemiSyncReplication
  * InnodbSmp
  * NewShowStatus
  * NewShowInnodbStatus
  * NewConfiguration
  * UserTableMonitoring
  * TransactionalReplication
  * MysqlRoles
  * MysqlRateLimiting
  * MoreLogging
  * InnodbAsyncIo
  * FastMasterPromotion
  * MirroredBinlogs
  * InnodbSampling
  * NewSqlFunctions
  * InnodbStatus
  * LosslessFloatDump
  * MysqlHttp
  * InnodbIoTuning
  * MutexContentionStats
  * FastMutexes
  * InnodbFreeze

# The v3 patch #

This has many new features and a few non-features. Embedded MySQL will not work with this patch. Also, I generated the patch after running 'make distclean' so there are some files that must be regenerated after this patch is applied, including sql\_yacc.cc and sql\_yacc.h. By doing this, the patch diff is smaller but maybe a bit confusing. Also, I did not update any of the files in libmysqld/ that are copied from sql/.

  * GlobalTransactionIds
  * OnlineDataDrift
  * BatchKeyAccess
  * InnodbMutexContention2
  * BinlogEventChecksums

# The v4 patch #

This makes InnoDB much faster on IO bound workloads and fixes bugs in new features.

  * InnodbIoPerformance

# Not yet released #
  * MysqlThreadPool
# Introduction #

These are patches for MySQL 4.0.26. They add many features that enhance the manageability and reliability of MySQL.

In a perfect world, each feature would be provided as a separate patch and all code would be as portable as MySQL. We are not there yet. These have been implemented and deployed on Linux. Also, some of these features only work with InnoDB, because we use InnoDB. They could be extended to support other storage engines.

The patch should be applied to MySQL 4.0.26 source code. That source is available from the download section for this project. The patch can be applied to the source as:
```
  tar xf mysql-4.0.26.tar.gz
  cd mysql-4.0.26
  patch --strip=1 --fuzz=1 < ../mysql-4.0.26-patches
```

# Features #

The patches include a few big features and many enhancements. The big features are:
  * SemiSyncReplication - block commit on a master until at least one slave acknowledges receipt of all replication events.
  * MirroredBinlogs - maintain a copy of the master's binlog on a slave
  * TransactionalReplication - make InnoDB and slave replication state consistent during crash recovery
  * UserTableMonitoring - monitor and report database activity per account and table
  * InnodbAsyncIo - support multiple background IO threads for InnoDB
  * FastMasterPromotion - promote a slave to a master without restart

Other enhancements include:
  * LosslessFloatDump - support dump and restore of float/double without loss of precision
  * Use 8X less memory for account and table privileges
  * Use fastest compression rather than the default level for client/mysqld networking
  * InnodbSampling - control the number of leaf blocks sampled for optimizer statistics
  * InnodbStatus - display more statistics in **show innodb status**
  * Reduced number of calls to fsync when the InnoDB background IO thread is active
  * Changed InnoDB to recover when InnoDB and MySQL data dictionaries are inconsistent
  * NewSqlFunctions - functions for checksums and floating point to string conversion
  * Backported **START SLAVE UNTIL**
  * Sort float columns with the order: **-INF < negative < 0 < positive < +INF < NaN**
  * Change **long\_query\_time** to be dynamic and log all queries that run for greater than or equal this number of seconds rather than greater than.
  * Count connection attempts tha are denied because of **max\_connections** and display the count as **denied\_connections**
  * MoreLogging - log actions done on specified tables and **SUPER** users
  * **rpl\_always\_enter\_innodb** boosts the priority of the slave SQL thread (for replication) in InnoDB by making it ignore the InnoDB concurrency limits
  * **rpl\_event\_buffer\_size** sets the fixed size buffer that is allocated in the master for each connected slave.  The buffer is used for replication events smaller than the buffer. This reduces memory allocation done to copy replication events from the master.
  * Backported **sync-binlog**
  * Added **reserved\_super\_connections** to reserve the final N connections for users with the SUPER privilege
  * NewShowStatus - many new variables in **SHOW STATUS**
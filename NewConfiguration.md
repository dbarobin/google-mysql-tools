# Introduction #

  * http\_enable - start the embedded HTTP demon when ON, see MysqlHttp
  * http\_port - port on which HTTP listens, see MysqlHttp
  * innodb\_max\_merged\_io - max number of IO requests merged into one large request by a background IO thread
  * innodb\_read\_io\_threads, innodb\_write\_io\_threads - number of background IO threads for prefetch reads and dirty page writes, see InnodbAsyncIo
  * show\_command\_compatible\_mysql4 - make output from some SHOW commands match that used by MySQL4
  * show\_default\_global - make SHOW STATUS use global statistics
  * global\_status\_update\_interval - the interval at which per-thread stats are read for SHOW STATUS. When SHOW STATUS is run more frequently cached values are used rather than locking and reading data from each thread.
  * google\_profile[=name] - enable profiling using Google Perftools and write output to this file. Server must have been compiled to use Google Perftools.
  * equality\_propagation - enables use of equality propagation in the optimizer
  * trim\_trailing\_blanks - trim trailing blanks on varchar fields when set
  * allow\_view\_trigger\_sp\_subquery - allow use of views, triggers, stored procedures and subqueries when set
  * allow\_delayed\_write - allow use of delayed insert and replace statements
  * local-infile-needs-file - LOAD DATA LOCAL INFILE requires the FILE privilege when set
  * audit\_log[=name] - log logins, queries against specified tables, and startup
  * audit\_log\_tables=name - log queries that use these tables to the audit log (comma separated)
  * log\_root - log DML done by users with the SUPER privilege
  * repl\_port[=#] - extra port on which mysqld listens for connections from users with SUPER and replication privileges
  * rpl\_always\_reconnect\_on\_error - slave IO thread always tries to reconnect on error when set
  * rpl\_always\_enter\_innodb - slave SQL thread always enter innodb when set regardless of innodb concurrency ticket count
  * rpl\_event\_buffer\_size=# - size of the per-connection buffer used on the master to copy events to a slave. Avoids allocating/deallocating a buffer for each event.
  * reserved\_super\_connections=# - number of reserved connections for users with SUPER privileges.
  * rpl\_always\_begin\_event - always add a BEGIN event at the beginning of each transaction block written to the binlog. This fixes a bug.
  * rpl\_semi\_sync\_enabled - enable SemiSyncReplication on a master
  * rpl\_semi\_sync\_slave\_enabled - enable SemiSyncReplication on a slave
  * rpl\_semi\_sync\_timeout - timeout in milliseconds for SemiSyncReplication in the master
  * rpl\_semi\_sync\_trace\_level - trace level for debugging SemiSyncReplication
  * rpl\_transaction\_enabled - use TransactionalReplication on a slave
  * innodb\_crash\_if\_init\_fails - crash if InnoDB initialization fails
  * innodb\_io\_capacity - number of disk IOPs the server can do, see InnodbIoTuning
  * innodb\_extra\_dirty\_writes - flush dirty buffer pages when dirty pct is less than max dirty pct
  * connect\_must\_have\_super - only connections with SUPER\_ACL, REPL\_SLAVE\_ACL or REPL\_CLIENT\_ACL are accepted (yes, this is dynamic)
  * readonly\_databases - prevents writes to any DB except for mysql
  * readonly\_mysql - prevents writes to mysql DB will fail.
  * fixup\_binlog\_end\_pos - fix for MySQL [bug 23171](https://code.google.com/p/google-mysql-tools/issues/detail?id=3171) which updates the end\_log\_pos of  binlog events as they are written to the  bin log.
  * log\_slave\_connects - log connect and disconnect messages for replication slaves
  * mapped\_users - use the mapped\_user table to map users to roles
  * xa\_enabled - enable support for XA transactions (I like to disable this)
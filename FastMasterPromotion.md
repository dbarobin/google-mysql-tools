# Introduction #

These commands allow fast promotion of a slave to a master. It is fast because it can be done without restarting the slave. Storage engines with dirty pages, such as InnoDB, can take a long time (more than a minute) to shutdown.

# Features in the V1 and V2 patch #

  * MAKE MASTER REVOKE SESSION
  * MAKE MASTER REVOKE SESSION WITH KILL
  * MAKE MASTER GRANT SESSION

# Features only in the V1 patch #

SQL statements include:
  * MAKE MASTER MASTER\_LOG\_FILE=

<log\_file>

, MASTER\_SERVER\_ID=

&lt;id&gt;

 [BINLOG](WITH.md)
  * MAKE MASTER MASTER\_LOG\_FILE=

<log\_file>

, MASTER\_SERVER\_ID=

&lt;id&gt;

, INDEX=<log\_file.index> [BINLOG](WITH.md)

## MAKE MASTER MASTER\_LOG\_FILE ... ##

This enables the use of the binlog on a slave without restarting mysqld.

## MAKE MASTER REVOKE SESSION ##

This prevents non-SUPER users from connecting. When the **WITH KILL** option is used, current non-SUPER connections are killed.

## MAKE MASTER GRANT SESSION ##

This allows non-SUPER users to connect.
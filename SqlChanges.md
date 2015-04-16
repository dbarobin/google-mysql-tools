# Introduction #

This describes changes to SQL parsed by MySQL.

# New tokens #

  * CLIENT\_STATISTICS
  * TABLE\_STATISTICS
  * USER\_STATISTICS
  * INDEX\_STATISTICS
  * IF\_IDLE
  * MAKE
  * MAPPED
  * MAX\_QUERIES\_PER\_MINUTE
  * NEW\_PASSWORD
  * ROLE
  * SLOW
  * TCMALLOC
  * IEEE754\_TO\_STRING
  * LAST\_VALUE
  * ORDERED\_CHECKSUM
  * UNORDERED\_CHECKSUM

# New SQL functions #

See NewSqlFunctions for more details:
  * ORDERED\_CHECKSUM
  * UNORDERED\_CHECKSUM
  * LAST\_VALUE
  * HASH
  * IEEE754\_TO\_STRING
  * NEW\_PASSWORD

# New options for existing statements #

**KILL**

&lt;id&gt;

 IF\_IDLE**can be used to kill a connection but only if it is idle.**

**MAX\_QUERIES\_PER\_MINUTE** can be used in place of **MAX\_QUERIES\_PER\_HOUR**. This version of MySQL enforces query limits per minute rather than per hour and the value stored in the MySQL privilege table is the rate per minute.

**CREATE MAPPED USER 'foo' ROLE 'bar'** and **DROP MAPPED USER 'foo'** support mapped users. See MysqlRoles for more details.

**SHOW PROCESSLIST WITH ROLES** and **SHOW USER\_STATISTICS WITH ROLES** use the role name rather than the user name in results.

# New statements #

See UserTableMonitoring for more details:
  * **SHOW USER\_STATISTICS**
  * **SHOW TABLE\_STATISTICS**
  * **SHOW INDEX\_STATISTICS**
  * **SHOW CLIENT\_STATISTICS**
  * **FLUSH TABLE\_STATISTICS**
  * **FLUSH INDEX\_STATISTICS**
  * **FLUSH CLIENT\_STATISTICS**

See MysqlRateLimiting for more details:
  * **MAKE USER 'foo' DELAYED 1000**
  * **MAKE CLIENT '10.0.0.1' DELAYED 2000**
  * **SHOW DELAYED USER**
  * **SHOW DELAYED CLIENT**

**SHOW TCMALLOC STATUS** displays the status of tcmalloc when MySQL hash been linked with it and compiled with -DUSE\_TCMALLOC. This displays the output from MallocExtension::GetStats.

**CAST** supports cast to **DOUBLE**.

**SHOW INNODB LOCKS** provides more details on InnoDB lock holders and waiters.

**FLUSH SLOW QUERY LOGS** rotates the slow query log.

**MAKE MASTER REVOKE SESSION** disconnects all sessions but the current one and prevents future connections from all users unless they have SUPER, REPL\_CLIENT or REPL\_SLAVE privileges. **MAKE MASTER GRANT SESSION** undoes this.
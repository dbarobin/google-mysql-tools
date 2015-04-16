# Introduction #

We have added code to measure database activity and aggregate the results per account, table and index. We have also added SQL statements to display these values.

One of these days we will integrate this with the information schema.

# Details #

Note that **rows changed** includes rows from insert, update, delete and replace statements.

The commands are:
  * **SHOW USER\_STATISTICS**
  * **SHOW TABLE\_STATISTICS**
  * **SHOW INDEX\_STATISTICS**
  * **SHOW CLIENT\_STATISTICS**
  * **FLUSH TABLE\_STATISTICS**
  * **FLUSH INDEX\_STATISTICS**
  * **FLUSH CLIENT\_STATISTICS**

# SHOW USER\_STATISTICS #

This displays resource consumption for all sessions per database account:
  * number of seconds executing SQL commands (wall time and CPU time)
  * number of concurrent connections (the current value)
  * number of connections created
  * number of rollbacks
  * number of commits
  * number of select statements
  * number of row change statements
  * number of other statements and internal commands
  * number of rows fetched
  * number of rows changed
  * number of bytes written to the binlog
  * number of network bytes sent and received
  * number of rows read from any table
  * number of failed attempts to create a connection
  * number of connections closed because of an error or timeout
  * number of access denied errors
  * number of queries that return no rows

# SHOW CLIENT\_STATISTICS #

This has the same values as **SHOW USER\_STATISTICS** but they are aggregated by client IP address rather than by database account name.

# SHOW TABLE\_STATISTICS #

This displays the number of rows fetched and changed per table. It also displays the number of rows changed multiplied by the number of indexes on the table.

# SHOW INDEX\_STATISTICS #

This displays the number of rows fetched per index. It can be used to find unused indexes.

# Flush commands #

Each of the flush commands clears the counters used for the given SHOW command.
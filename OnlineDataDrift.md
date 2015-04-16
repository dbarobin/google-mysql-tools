# Overview #

This describes a tool for generating checksums on tables in a MySQL database. It can be used to check for data drift between copies of tables on a master and slave. The tool is intended to be easy to use and efficient. The tool is easy to use because it can run concurrent with the normal workload on a master or slave, it does not require an idle server, and servers do not have to be drained or restored when it runs. The tool is efficient because it computes the checksum incrementally over a period of time. The rate at which the check is made is configurable.

In order to be efficient and easy to use, this tool does not compute the checksum for all tables or all rows in one table at the same point in time. The tool computes a checksum for each chunk of rows in a table. It is likely that this will skip a few rows from some tables. This is a trade-off. By using short-running statements, the statements can be run on a primary and get replayed on the slaves.

This is similar to [mk-table-checksum](http://maatkit.sourceforge.net/doc/mk-table-checksum.html) but uses features added to the Google MySQL patch. The big difference is that this uses the [last\_value](NewSqlFunctions.md) aggregate function to avoid estimating row ranges.

## Design ##

The tool does the following:
  * estimate the number of rows in the database to determine how fast it should run
  * for each table, run a sequence of statements to compute the checksum of a chunk of rows and insert the result into a table

When the tool is done, the result tables can be compared between a master and slaves. If differences are found, then there has been data drift.

The checksum statements are run directly on masters. They are written to the binlog and replayed on all slaves by MySQL replication. They are executed on the primary and replicas at the same point in time as measured by transactions committed.

### Row Estimation ###

Run **SHOW TABLE STATUS** for each database to get an estimate on the number of rows in each table. Determine the rate at which the checksum SQL statements should be run from the estimated row count, the number of rows per chunk (**--rows\_per\_chunk**) and the time in which the job should finish (**--hours\_to\_run**). Statements are run sequentially. If **--hours\_to\_run** is too small, the checksum job will not finish in the expected time. Statements will not be run concurrently to catch up.

### Checksum SQL ###

The tool scans a table in primary key order a chunk of rows at a time. This will scan some rows multiple times because the last row scanned from the previous statement is the first row scanned from the statement that follows. It may miss some rows because multiple statements are used for many tables and the same snapshot is not used for all statements.

The scan is done by the following. This ignores the code to compute the checksum for simplicity.
  * assume the primary key index for Foo is on (p1, p2) and c1, c2 are the other columns in Foo
  * set (next\_p1, next\_p2) to the value for the primary key columns of the first row in the table
  * repeat until reaching the end of the table
    * scan at most rows\_per\_chunk rows starting at the row with primary key (next\_p1, next\_p2)
    * set (next\_p1, next\_p2) to the value of the primary key rows from the last row scanned by the previous statement

SQL to determine the primary key values for the first row in the table:
```
select @next_p1 := p1, @next_p2 := p2 from Foo order by p1, p2 limit 1
```

SQL to scan one chunk of rows assuming rows\_per\_chunk is 1000. This requires a new SQL aggregate function, **last\_value**, that returns the last value of the input expression per group. The end of the table has been reached when the returned count is less than rows\_per\_chunk. The next scan starts at the primary key value at which this scan stops. Including this row in the checksum for two chunks is not a problem.
```
-- be paranoid, do not trust evaluation for mutating user variables
set @start_p1 := @next_p1
set @start_p2 := @next_p2
select @next_p1 := last_value(p1), @next_p2 := last_value(p2), count(*)
from 
    (select * from Foo force index (PRIMARY) 
     where (p1 = @start_p1 and p2 > @start_p2) or p1 > @start_p1  limit 1000) f
```

The checksums for all columns must be computed. The SQL aggregate function **ordered\_checksum** is used for this. The following must be added to the SQL above:
```
ordered_checksum(p1), ordered_checksum(p2), ordered_checksum(c1), ordered_checksum(c2)
```

The result table has a fixed number of columns but the number of expression produced depends on the number of columns in the checked tables. This is resolved by concatenating expressions together and storing the result as a string in the Offsets and Checksums columns of the result table. The result table has the columns:
  * DatabaseName varchar(255) - the database for the checked table
  * TableName varchar(255) - the checked table
  * Chunk int - the chunk number per table, a value between 1 and ceil(#rows / rows\_per\_chunk)
  * JobStarted Datetime - the time at which the checksum job begins
  * ChunkDone Datetime - the time when the SQL statement for this chunk was run
  * Offsets text - the range of values per primary key column for this chunk
  * Checksums text - the checksums per column for this chunk
  * Count int - number of rows in this chunk
  * primary key (Name, Chunk, JobStarted)

Finally, all of this is to be stored in a result table, so a replace statement is used. The statement for the example table is:
```
replace into tc (DatabaseName, TableName, Chunk, JobStarted, ChunkDone, Offsets, Checksums, Count) 
select 'test', 'Foo', 1, @job_started, now(),
concat_ws(':', 'p1', @start_p1, @next_p1 := last_value(p1),
               'p2', @start_p2, @next_p2 := last_value(p2)),
concat_ws(':', 'p1', ordered_checksum(p1),
               'p2', ordered_checksum(p2),
               'c1', ordered_checksum(c1),
               'c2', ordered_checksum(c2)),
@count := count(*)
from 
    (select * from Foo force index (PRIMARY) 
     where (p1 = @start_p1 and p2 > @start_p2) or p1 > @start_p1 
     order by p1, p2 limit @rows_per_chunk) f
```

## Tool options ##

Configuration options for the tool:
  * --databases\_to\_check - the databases for which checksums are computed. It might be easier to list the databases to skip.
  * --tables\_to\_skip - the tables for which checksums are not computed. The table used for results, --result\_table, is always skipped.
  * --engines\_to\_check - tables are skipped unless they use one of these storage engines
  * --column\_types\_to\_skip - columns within a table are skipped when they use one of these datatypes
  * --result\_table - the table in which the checksums are written. This should be an InnoDB table.
  * --db\_user
  * --db\_host
  * --db\_password - path to password file. The tool will prompt for a password if this is not specified.
  * --rows\_per\_chunk - number of rows to examine per chunk (and per SQL statement)
  * --hours\_to\_run - hours in which the job should complete
  * --force\_job\_started - provide the value of the JobStarted column for entries in the result table, used when restarting a failed job

## Open Issues ##

### Floating point columns ###

The values of _float_ and _double_ columns might be different between the primary and replica for acceptable reasons. For example, the order of evaluation for aggregation may be different when a statement is executed on a replica. If rounding is required during the computation, then the result may be different. We don't know the extent of this problem. If it generates a lot of false warnings, then we should either ignore some floating point columns or provide a new SQL function that formats such columns as with a reduced amount of precision. This can be done by formatting as a decimal string with fewer digits of precision (3 for float, 6 for double).

### Tables without primary keys ###

There are a few options to handle tables without primary keys:
  * scan all rows in one chunk if the table is small
  * scan by a secondary index if the table is large. The tool will get stuck on a key value if the number of rows with that key value is greater than the chunk size. Something must be done to survive that case.
  * ignore the table
# Introduction #

Replication state on the slave is stored in two files: relay-log.info and master.info. The slave SQL thread commits transactions to a storage engine and then updates these files to indicate the next event from the relay log to be executed. When the slave mysqld process is stopped between the commit and the file update, replication state is inconsistent and the slave SQL thread will duplicate the last transaction when the slave mysqld process is restarted.

# Details #

This feature prevents that failure for the InnoDB storage engine by storing replication state in the InnoDB transaction log. On restart, this state is used to make the replication state files consistent with InnoDB.

The feature is enabled by the configuration parameter **rpl\_transaction\_enabled=1**. Normally, this is added to the **[mysqld](mysqld.md)** section in **/etc/my.cnf**.

The state stored in the InnoDB transaction log can be cleared setting a parameter and then committing a transaction in InnoDB. For example:
```
set session innodb_clear_replication_status=1;
create table foo(i int) type=InnoDB;
insert into foo values (1);
commit;
drop table foo;
```

Replication state is updated in the InnoDB transaction log for every transaction that includes InnoDB. It is updated for some transactions that don't include InnoDB. When the replication SQL thread stops, it stores its offset in InnoDB.

# The Dream #

We would love to be able to kill the slave (kill -9) and have it always recover correctly. We are not there yet for a few reasons:
  1. We don't update the state in InnoDB for some transactions that do not use InnoDB
  1. DDL is not atomic in MySQL. For **drop table** and **create table** there are two steps: create or drop the table in the storage engine and create or drop the frm file that describes the table. A crash between these steps leaves the storage engine out of sync with the MySQL dictionary.
  1. Other replication state is not updated atomically. When relay logs are purged, the files are removed and then the index file is updated. A crash before the index file update leaves references to files that don't exist. Replication cannot not be started in that case. Also, the index file is not updated in place rather than atomically (write temp file, sync, rename).
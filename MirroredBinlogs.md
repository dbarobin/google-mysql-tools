# Note #

This is only in the V1 patch. It has been removed from all other patches. GlobalTransactionIds should be used instead of it.

# Introduction #

MySQL replication is great. It is efficient, stable and easy to use. Unfortunately, it is not easy to use for hierarchical replication. The most important replication state on a slave is the name and offset for the file on the server from which it copies replication events.

When hierarchical replication is used, that state cannot be transferred without translation. For example, suppose there is one master, two slaves that replicate from the master and generate a binlog, these two are the replication proxy slaves. There are also many slaves that replicate from the two replication proxy slaves. If one of the replication proxy slaves fails, the slaves that replicated from the proxy cannot transparently failover to the other replication proxy. The binlogs written by the proxies might not have the same names, and replication events stored at the same offsets are not the same.

# Notes #

We have modified the slave IO thread to maintain a copy of the master's binlog as it writes the relay log. By **copy**, I mean that the file has the same name and same contents. When this is done, slave can transparently failover between replication proxy slaves as long as the proxies all mirror the binlog.

When this is first enabled, the slave must download all of the current binlog. This can take some time. New events are not appended to the relay log until this has finished.

Parameters:
  * **rpl\_mirror\_binlog\_enabled** enables this
  * **rpl\_mirror\_binlog\_no\_replicate**, enables mirroring of the binlog on a slave but prevents the slave from serving it to other slaves.
  * **sync-mirror-binlog** is equivalent to **sync-binlog** but for the mirror binlog
# Introduction #

Is it possible to implement synchronous replication for MySQL without major surgery on the code? On a master there are two parts to the replication flow:
  1. write the binlog - this is done using XA to keep a storage engine (InnoDB) and the binlog in sync. For PREPARE SQL statements are written to the binlog and the InnoDB transaction log is (optionally) forced to disk. At this step there should be an option to foce the binlog to disk (sync\_binlog?). For COMMIT an XID event is written to the binlog and a commit record is written to the InnoDB transaction log. On crash recovery InnoDB is provided a list of all XID events from the current binlog and decides whether to commit or rollback transactions in state PREPARED.
  1. copy events from the binlog to the slave - this is done after commit.

Semi-sync replication interrupts the second part of the replication flow (copy events from the binlog to the slave). Synchronous replication must interrupt the first part.

# My naive idea #

  1. add another resource manager (binlog, storage engine, new code for this) into the commit protocol
  1. on prepare write all tx binlog events elsewhere
  1. on commit write XID event elsewhere

I have not defined 'elsewhere'.  The best place is the relay log on another slave. If the XA approach is used (first write tx events, then write XID) it might be possible to use the relay log approach as the slave cannot commit that transaction until it gets
the XID event. And if the relay log is used, can existing code be reused to move the event from the master to a slave?

# The expert responds #

```
In short, yes I think that it is possible.

Hmm... a relay log... Here are some ideas which aren't the most elegant and aren't completely thought through, but they may be the least work...
```

## First step ##

Add a pre-binlog which is an instance of the MYSQL\_LOG class just like the binlog and so is using almost all the same code. As a first iteration I'm advocating that the log on disk be a complete duplicate of the actual binlog. That's a lot of duplication, but since the binlog an integral part of crash recovery it's safer for the first round. Then change ha\_commit\_trans so that after the prepares, but before the tc\_log->log() call you call pre\_binlog\_end\_trans() using the transaction cache stored at (IO\_CACHE**)thd->ha\_data[binlog\_hton.slot];.**

Note that the server's current group\_id is stored at mysql\_bin\_log.group\_id and so the fact that mysql\_pre\_bin\_log.group\_id runs a little ahead shouldn't be a problem though

## Second step ##

Move the semi-sync wait to happen at the end of pre\_binlog\_end\_trans() and run with --rpl-semi-sync-always-on and --rpl-semi-sync-timeout=<some very large value>.

## Third step ##

Add COM\_PREBINLOG\_DUMP. Refactor mysql\_binlog\_send and its callers so that the MYSQL\_LOG to use is a parameter instead of hardcoded to mysql\_bin\_log.

## Fourth step ##

Either change the slaves' IO thread to use COM\_PREBINLOG\_DUMP or add another slave IO thread (with thread function handle\_slave\_io) to do so. Change request\_dump so that it uses the correct COM**.**

## Summary ##

Those steps should result in commits being blocked utnil there is a relay log on a slave with the full transaction. How to then make use of that relay log isn't fully thought out.


  * How to handle failure from binlog\_end\_trans?. If the slaves' existing IO thread is changed to read from the pre-binlog then all the slaves are now inconsistent with the master because they have the transaction. Note that there may also be a similar problem today if binlog\_commit succeeds but InnoDB's commit fails. Looking at innobase\_commit shows that it always returns 0 so apparently never fails. What about other storage engines though?
  * If, instead, there's a second slave IO thread, slave is generating 2 sets of relay logs containing all the same data. When should the slave switch to use the pre-binlog relay log? How does it know?
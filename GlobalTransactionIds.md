# MySQL Hierarchical Replication & Global Group IDs #

Author: Justin Tolmer

## Objective ##

To make the process of pointing a replication slave to a new master straightforward so that we can setup replication in a hierarchical topology. This change would allow us to reduce the number of slaves replicating directly from the primaries.

## Background ##

This is the detailed design document. We should also publish a user's guide.

Note that this document is written with MySQL 5.0 in mind. Any functions, source files and source line numbers are taken from the Google branch of the 5.0 code base.

A primer on MySQL replication can be found on [MysqlForge](http://forge.mysql.com/wiki/MySQL_Internals_Replication).

### Current State of Hierarchical Replication ###

One can setup hierarchical replication today. To support such chained replication, it is required for slaves which are also masters to be running with both **--log-bin** and **--log-slave-updates**. There are, however, some shortcomings with the current implementation, which, of course, is what this document is all about. Let's start by examining an example with the current implementation. In this example we have 3 MySQL servers, chained as A -> B -> C where -> means "replicates to". I have C running with both **--log-bin** and **--log-slave-updates** for consistency in configuration though it isn't technically required. It does help with my example though because it means C has a bin log I can examine rather than having to setup A -> B -> C -> D just to illustrate my points. Here are 2 updates made on a master as the corresponding events make their way into the bin logs of all the 3 servers:

#### Server A, the master ####

```
# at 499
#080424 16:14:38 server id 11306  end_log_pos 618       Query   thread_id=6    exec_time=0      error_code=0
SET TIMESTAMP=1209078878/*!*/;
insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10)/*!*/;
# at 618
#080424 16:14:56 server id 11306  end_log_pos 712       Query   thread_id=6    exec_time=0      error_code=0
SET TIMESTAMP=1209078896/*!*/;
insert into t2 select * from t1/*!*/;
```

#### Server B, first level slave ####

```
# at 98
#080424 16:14:38 server id 11306  end_log_pos 217       Query   thread_id=6    exec_time=12     error_code=0
use test/*!*/;
SET TIMESTAMP=1209078878/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=1, @@session.unique_checks=1/*!*/;
SET @@session.sql_mode=0/*!*/;
/*!\C latin1 *//*!*/;
SET @@session.character_set_client=8,@@session.collation_connection=8,@@session.collation_server=8/*!*/;
insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10)/*!*/;
# at 217
#080424 16:14:56 server id 11306  end_log_pos 311       Query   thread_id=6    exec_time=0      error_code=0
SET TIMESTAMP=1209078896/*!*/;
insert into t2 select * from t1/*!*/;
```

#### Server C, second level slave ####

```
# at 296
#080424 16:14:38 server id 11306  end_log_pos 415       Query   thread_id=6    exec_time=24     error_code=0
SET TIMESTAMP=1209078878/*!*/;
insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10)/*!*/;
# at 415
#080424 16:14:56 server id 11306  end_log_pos 509       Query   thread_id=6    exec_time=6      error_code=0
SET TIMESTAMP=1209078896/*!*/;
insert into t2 select * from t1/*!*/;
```

There are some things to notice about the above example:
  * The server ID in the bin log is always the ID of the originating server regardless of in which server's bin log it shows up.
  * The timestamp of the event is also always the value from the originating server.
  * The end\_log\_pos is always the pos of the bin log relative to the server writing the log. (Actually, this isn't always true. See below on the impact of transactions and the use of bin log caches).

The shortcomings of the current implementation become evident when you consider the scenario of B failing and needing to point C to start replicating from A. Currently one has to use some combination of server ID and the query text (and possibly timestamp) in order to find the last event from A's bin log(s) which was been played on C's replication SQL thread in order to determine which log and log\_pos of A to use in the **CHANGE MASTER TO ...** command on C to get it to replicate from A without either losing or duplicating events. Using the text of the query is non-optimal because it may not be unique. Timestamp can help disambiguate, but, with highly utilized servers, there can be many events with the same timestamp and so one needs to be careful to select the correct one.

If your topology was instead, B->A, B->C when B failed, you would instead be faced with electing one of A or C to be the new master. You do this by finding which of the two is the most current w.r.t. the failed master B by executing **SHOW SLAVE STATUS** on A and C and then picking the one which is the most current based on later master log file index and/or larger master log pos. Assume that it was A and that A was promoted to the new master. Then, the problem of configuring C to replicate from A is the same as the above.

### Transactions and Bin Log Caches ###

There is another issue, unrelated to chaining replication, of which you need to be aware. When using transactions the end\_log\_pos in the header of events between the **BEGIN** and **COMMIT** is incorrect.

```
#080430  9:57:20 server id 11306  end_log_pos 782       Query   thread_id=11    exec_time=0     error_code=0
SET TIMESTAMP=1209574640/*!*/;
BEGIN/*!*/;
# at 782
#080430  9:51:39 server id 11306  end_log_pos 112       Query   thread_id=11    exec_time=0     error_code=0
SET TIMESTAMP=1209574299/*!*/;
insert into test.t1 values (1,2), (3,4), (5,6), (7,8)/*!*/;
# at 894
#080430  9:57:20 server id 11306  end_log_pos 921       Xid = 64
```

Note that updates to InnoDB tables are always wrapped in **BEGIN** / **COMMIT**. If the user doesn't explicitly start a transaction, one is implicitly done inside MySQL. The reason the end\_log\_pos of events inside  **BEGIN** / **COMMIT** is wrong is because they aren't relative to the bin log, but to a temporary file, a bin log cache, used to hold the events until they are actually committed. A temporary file is used because, if the events are rolled back instead of committed, then they never make it to the bin log at all. At commit time, the entire contents of the temporary file are copied into the bin log en masse; there is no processing of each individual, cached event.

The fact that these end\_log\_pos values are not correct is [Bug23171](http://bugs.mysql.com/bug.php?id=23171).

### Bin Log Groups ###

Events in the bin log occur in groups. The nature of bin log groups are that the position of the replication SQL thread cannot advance until the final event in the group is executed. That is, if the server crashes, is shut down or the replication SQL thread terminates, replication must start playing back from the first event of the group; it can't start in the middle. Groups may be explicit (e.g. a transaction) or implicit and may also only contain a single event (e.g. create table). Transactions are easy to understand as a group because if the playback is interrupted before a transaction is committed, the transaction is rolled back. For the SQL thread to start back up, it obviously needs to do so from the beginning. The set of events comprising the group in the bin log are identifiable by the surrounding BEGIN / COMMIT events. An example of an implicit group is an insert to a MyISAM table which uses a user-defined variable. In the bin log there are 2 events which comprise such a group though there isn't any clear indication in the bin log that these events are tied together:

```
# at 2663
#080610 12:23:15 server id 11306  end_log_pos 2718      User_var
SET @`my_var`:=1007/*!*/;
# at 2718
#080610 12:23:15 server id 11306  end_log_pos 2825      Query   thread_id=7     exec_time=0     error_code=0
SET TIMESTAMP=1213125795/*!*/;
insert into test.t1 values (@my_var)/*!*/;
```

### Bin Log Format ###

Some good data on the format of events in the bin log, and how the header is extensible, can be found in [Worklog3610](http://forge.mysql.com/worklog/task.php?id=3610) in the Low Level Design section.

Each bin log currently starts with a 4 byte header, **BINLOG\_MAGIC**. Immediately followed by this header comes a **Format\_description\_log\_event** (FDE). The FDE is the method by which the extensibility of the bin log is achieved. Part of the cost of that extensibility is that the FDE's header size cannot be changed.

When a new bin log is created the FDE written to the log has its flags field set to 0x1 indicating that the file is in use (sql/log.cc, line 677). On a clean shut down of the file, the flags field of this first event is overwritten with 0 marking that the file was shut down cleanly (sql/log.cc, line 2279).

### Bin Log Processing at Server Start ###

At server start, here's what happens with the bin logs and bin log index files:

```
  TC_LOG_BINLOG::open(const char *)         // process existing bin logs & index file
    MYSQL_LOG::find_log_pos
    do {
      MYSQL_LOG::find_next_log
    } while entries in index file           // figure out the most recent bin log file
    open_binlog                             // open the most recent bin log 
    if (FDE bit set marking file in use) {
      TC_LOG_BINLOG::recover                // do crash recovery
    }
  MYSQL_LOG::open(const char *, enum_log_type, const char *, ...)    // create new bin log
```

The main data point to understand by reading the above pseudo code is that on crash recovery it is only the most recent bin log which is 'recovered'; older bin logs and their entries in the index file are not processed.

### rpl\_transaction\_enabled ###

TransactionalReplication is an option we use to allow a slave's replication progress to be persisted in InnoDB rather than in solely in the file relay-log.info. During crash recovery scenarios InnoDB's state often overrides the information in relay-log.info in determining the correct position with which to start replicating again from the master. Specifically, if a slave crashes between the transaction getting committed and the flushing of relay-log.info, MySQL's replication state can become inconsistent. There is existing [Bug26540](http://bugs.mysql.com/bug.php?id=26540) tracking this issue. **rpl\_transaction\_enabled** fixes this issue for us because it allows InnoDB's replication state to override that of relay-log.info.

Unfortunately, there is currently some incorrect behavior with our implementation of this option. Specifically, if the the last event(s) executed by the SQL thread before a server crash is not one which changes an InnoDB table, InnoDB will not have knowledge of the event and the crash recovery code will likely try to replay the event because it will chose InnoDB's state over that of relay-log.info. I made things slightly better with the work I did for Bug:1169545, but my changes only addressed normal operating scenarios, not crash recovery, and so there are still issues. For example, if you have a sequence of events occur on the replication SQL thread such as

  1. begin;
  1. Insert values into InnoDB table;
  1. commit;
  1. Insert values into MyISAM table;
  1. Crash
  1. Restart Server

The statement at 4 will be executed twice. Once before the crash and then again after server restart because InnoDBs replication state from the crash recovery would restart replication after the commit at 3. Since we're not currently running slaves with **--log-bin** enabled, fixing it requires updating the large number of code paths which call into the binlog code.

## Overview ##

I am going to add a new ID to the header of events which get written to the bin log. Events comprising a group will share the same ID and each group will have an ID which is unique and greater than the ID of preceding groups. This new ID also needs be written to the relay log of the slave, read by the slave's SQL thread, passed down as the SQL thread executes the event and written, unchanged, to the slave's bin log. Additionally, when failing over to a new master, the new master will continue the ID sequence where the old master left off.

## Detailed Design ##

### Non-goals ###

  * Support for circular, or other multi-master, replication topologies.
  * Fully automated failover contained within mysqld.

### Nice-to-haves ###

  * Single my.cnf file for all machines in the topology.
  * Would be nice if piping the output of mysqlbinlog into the mysql client would preserve the group IDs. (Not currently addressed. Maybe can do something similar to **SET TIMESTAMP ...**. which mysqlbinlog currently outputs.)

### Filtering non-master events ###

One of the potential problems with hierarchical replication is that bin log events would be generated on the slaves due to operations in the scratch database. Thus, if things are left as is, the further down the replication chain a slave is, the larger the accumulation of changes it will have which didn't originate on the root master. What we really want are for only the updates done on the root master to be percolating through the system. As Mark put it, we want all servers except the root master to behave as if they were running with a new option, **--log-only-slave-udpates**, which means, on a slave, only put events from the replication SQL thread into the bin log.

Such an option, if running everywhere except for the root master, would give the behavior we want. Changes to the scratch database which originate on the root master get replicated down the tree, and they need to be replicated. On the root master one can do work in the scratch database and then **INSERT INTO adsN.some\_table SELECT `*` FROM scratch.some\_scratch\_table**. With statement based replication, the events affecting the scratch database need to be replicated to the slaves for that **INSERT** to succeed. However, any changes made to the scratch database on any other server, where the adsN database is readonly and such an **INSERT ... SELECT ...** can't be done to cause us replication problems, wouldn't get replicated down the chain.

There are several possible methods to accomplish the goal:
  * Give servers in the topology explicit knowledge of which server\_id is the current root master. This could be done by adding a new command which puts an event in the bin log. When the slave processes that event on its SQL thread, it updates it's understanding of the current master. With the knowledge of which server\_id is the root master servers could filter events they put into their bin log or those that they play back on their replication SQL thread. This approach has multiple drawbacks, some of which are:
    * It requires a command to be run on the master at the correct point in the failover process.
    * Slaves have bookkeeping work they need to do.
    * Slaves being inserted into the replication topology after the event has made its way through the system would need additional configuration done to insert them correctly.
  * Add a configuration table to the mysql database. This configuration table would be pushed at the same time as permissions during failover. When getting ready to log an event to the bin log, the server checks the current configuration to see if it is the root master or not and acts accordingly. Disadvantages:
    * Adds a new table.
    * Requires adding code to read the configuration table from inside the server.
    * Changes the existing failover process.
  * Have the server determine whether it is the root master by simply checking if it currently has a master. This check is done by examining **active\_mi** and switching on the values it contains. Advantages:
    * No new tables.
    * No new commands.
    * No bookkeeping.
    * No changes to the failover process.
    * Essentially the same check as that done in **show\_master\_info** in response to receiving a **SHOW SLAVE STATUS** command.

Because of the advantages, I have elected to use the third approach to implement the desired filtering. I will add a new option, **--rpl-hierarchical**. Different from the hypothetical **--log-only-slave-udpates** option, this option can, and should, be set on all servers in the replication topology, including the root master. For the new option to function correctly **--log-slave-updates** must also be set, just as **--log-slave-updates** requires **--log-bin** to be set. Therefore, I will add a check to **init\_server\_components** (sql/mysqld.cc, line 3225) verifying that the options are set consistently similar to the existing check for **--log-slave-updates**. The actual event filtering will happen in **MYSQL\_LOG::write** (sql/log.cc, line 1743) where I will check:
```
  // if hierarchical replication is enabled, and the event was generated on this server,
  // and this server has a master, we do not want it to go into the bin log.
  // Note that if the event didn't originate on this server, that is, it is being generated
  // by the replication SQL thread, we do want it to go to the bin log. Hence the up-front
  // server_id check to allow us to avoid taking locks.
  if (rpl-hierarchical &&
      event_info->server_id == ::server_id) {
    DBUG_ASSERT(!thd->slave_thread);      // events on the replication SQL thread should have a different server_id.
    pthread_mutex_lock(&LOCK_active_mi);
    if (active_mi.host[0]) {              // if we have a master host configured, we're a slave
      pthread_mutex_unlock(&LOCK_active_mi);
      pthread_mutex_unlock(&LOCK_log);
      DBUG_PRINT("info",("not logging event due to rpl-hierarchical"));
      DBUG_RETURN(0);
    }
    pthread_mutex_unlock(&LOCK_active_mi)
  }  
```

UPDATE: During implementation, the check above was simplified so that it didn't require taking any locks. MYSQL\_LOG has a new member, have\_master, which is updated in init\_slave and change\_master and the filtering check examines the value of have\_master rather than active\_mi.

### ID Generation ###

On the root master, ID generation will be done inside **MYSQL\_LOG::write(Log\_event `*`)** (sql/log.cc, line 1708) and **MYSQL\_LOG::write(THD `*`, IO\_CACHE `*`, Log\_event `*`)** (sql/log.cc, line 1973). The ID will be represented by a 64-bit, unsigned integer, hereafter named group\_id. The in-memory group\_id variable will track the value of the last consumed ID. Note that writes to the bin log are serialized by locking **MYSQL\_LOG::LOCK\_log** and we will use this lock to ensure group\_ids of events in the bin log are sequential.

All events written to the bin log, except for the FDE, will have a group\_id. As stated above, FDEs have a fixed header size which cannot be changed. Therefore they will neither have nor consume a group\_id.

UPDATE: Like the FDE, **Rotate\_log\_event** events currently have a frozen header and so also will not have a group\_id. I also changed the code so that **Stop\_log\_event** events are not issued when hierarchical replication is enabled. Stop events would consume an ID, which could cause the master and slave to get out of sync by 1. Since **Stop\_log\_event::exec\_event** doesn't do any work the absence of the stop event in the bin log doesn't cause any problems. An alternative would have been to change the stop event so that it didn't have a group\_id, but that made the code behind **SHOW BINLOG INFO FOR** much more complicated.

The value of the last group\_id consumed needs to be persisted so that a server doesn't reissue the same group\_id between sessions. The group\_id will be persisted in the bin log's index file. Doing so will require changing the format of the index file as right now each line of the file contains only a full path to a file. The new format will be "full\_path\_to\_file|groupd\_id|".

Places in the code which need to be changed to read and write the new index format are:
  * At creation of a new bin log, **MYSQL\_LOG::open** (sql/log.cc, line 758). The group\_id which will get written to the index for the new file will be that of the last event of the preceeding bin log file because the FDE which was just written to the file has no group\_id and it is the only event in the log at this point. The other alternative, which I didn't take, was to not write the group\_id to the index file in this case.
  * **MYSQL\_LOG::find\_log\_pos** (sql/log.cc, line 915) which has the purpose of finding the byte position in the index file of a given bin log file name.
  * **MYSQL\_LOG::find\_next\_log** (sql/log.cc, line 979) where entries are read out of the index file.
  * **MYSQL\_LOG::close** (sql/log.cc, line 2302) needs to be changed to fix the entry in the index file corresponding to the bin log file so that it has the group\_id of the last event contained in log.

On server start after a clean shutdown, the index file is processed, the entries read out, and the server will know the value of the last group\_id it issued by the value of the last entry in the index file. On the other hand, on server start after a crash, things are more complicated. Remember from above that recovery in the bin log is detected by checking the file-in-use bit of the flags field of the FDE at the start of the bin log. For crash recovery there are multiple scenarios I've considered:
  * Normal case is that **TC\_LOG\_BINLOG::recover** (sql/log.cc, line 3206) is run, all events in the dirty bin log are processed and the last issued group\_id is the last one in the file.
  * If the dirty bin log only contained an FDE, which doesn't have a group\_id, then we will use the group\_id of that log's entry in the index file. At the time the server created the new bin log file and wrote its entry to the index file, it knew the correct group\_id to write, and so that is the one to use.
  * The server could be repeatedly crashing at start and the start up code will only ever "recover" the most recent bin log. There are 2 sub-scenarios:
    * The server could repeatedly crash before writing a new bin log. In that case we'll continue to process the same bin log, and determine the same, correct group\_id each crash recovery using above steps.
    * The server could be repeatedly crashing after writing a new bin log. In this case, before creating the new bin log, recovery was run on the old bin log, the correct group\_id determined and that group\_id was written into the index file for the new bin\_log. In the new bin log, there are 2 sub-sub-scenarios.
      * The only event is the FDE, which means we use the group\_id from the index file.
      * There are additional events which means we do the normal case recovery.
In the crash recovery case, there is an incentive to fix-up any incorrect group\_ids in the index file, so I will probably do so. Reason will be explained below.

Since there are many index files already in production in the current format, I need to write the code which reads the index file lines in such a way that it can handle the absence of the delimiter and group\_id. If the group\_id is not found, the code will assume that the server is switching to the new format for the first time and will initialize the in-memory group\_id variable to the default value of 0.

On the slave, the index file for the bin logs will be treated just as it is on the root master as the slave needs all the same information. However, the index file associated with the relay logs will not have group\_ids stored in it. During failover when pointing a slave to a new master one of the steps done is to **RESET SLAVE** which clears out the replication state, including deleting any existing relay logs. It is the position of the replication SQL thread which determines the log name and pos on the new master to use in the **CHANGE MASTER** command. Thus, the group\_id of the IO thread isn't that important for us to track and not doing so removes the bookkeeping cost as well as the work of ensuring proper behavior in crash recovery scenarios which would interact with **rpl\_transaction\_enabled**. The code which writes the index file doesn't currently have access to the THD. Therefore, the check to determine whether the index file should include the group\_id will be accomplished by adding a new entry to **enum\_log\_type**, **LOG\_RELAY**. In **init\_relay\_log\_info** where the relay log is opened, it will be changed to pass in the new **LOG\_RELAY** instead of the old **LOG\_BIN**. Handling of **LOG\_RELAY** will be exactly the same as **LOG\_BIN** expect for the behavior of not writing the group\_id.

Open Issue: I'm thinking of using '|' as my delimiter in the index file, but I'm open to better suggestions. '|' is an illegal character on windows and so won't show up in the file name. Seems unlikely to be used on Linux, but it is valid, so the use of '|' could hit problems. I thought of making the character to use an option so that it could be overridden from the default, but did not because once index files are created with one delimiter, changing the delimiter to a different character would create many problems.

During failover when a server which was a slave becomes the root master, we want it to continue the group\_id sequence where the old master left off, which is possible because we'll be running with **--rpl-hierarchical** so that the only events being written to the bin logs in our system are those which originated on the root master. The alternative is to have each server track and issue its own series of group\_ids and to use the pair (server\_id, group\_id) to uniquely identify the events in the system. The reason I didn't go with the latter approach is because of scenarios around backup, restore and failed servers. Consider the scenario of a root master, which was running and issuing group\_ids, suffering a catastrophic hard drive failure losing all data on disk. An unplanned failover would take place and a new server would be made root master. Eventually the failed hardware would be repaired or replaced, a restore done to the server and the server put back into the replication topology as a slave, very possibly with the same server\_id. Further assume that at some point later the server is once again made root master due to a failover. What group\_id should it start issuing and how is the server supposed to determine it? The most correct behavior would have been for someone, at the time of the failure, to record the last group\_id which the server generated by extracting the information from the most current slave. Then, later, when the machine was put back into service, it would have needed to be configured with the previously recorded group\_id. In all, it seemed like a fragile scenario likely to result in the server beginning to issue group\_ids starting at 0 again. Instead, we'll have a system where group\_ids are unique in the system rather than per server. When a server is switched from a slave to be made root master, and needs to determine which group\_id to start issuing, it just needs to look to see the last group\_id it played back on its SQL thread and issue a value larger by 1. Such an approach alleviates any complicated interaction with backup and restore preserving or resetting the group\_id space of a server. It also makes book keeping easier because it alleviates any ambiguity about what the group\_ids stored in the index files are tracking as well as eliminating any differences in the tracking done by masters and slaves. If each server had its own group\_id space, the index files (or alternate persistence mechanism) likely would have to track lists of (server\_id, group\_id) pairs instead of a single value.

Given the above, it follows that only persisting a single group\_id being a valid solution depends on the filtering done by the **--rpl-hierarchical** option to be active. Therefore, the functionality of generating and storing group\_ids will be put behind the same option. Putting the group\_id generation behind the option has the additional benefit of decoupling the deployment of a build with the new hierarchical replication support from the ability to turn the functionality on and off.

MySQL's use of bin log cache temporary files for events within transactions presents some difficulties for the ID generation. If we give the events a group\_id at the time they are written to the bin log cache, then events in the actual bin log are not guaranteed to be be sorted by group\_id. On the other hand, if, at commit time, we want to fix up the group\_ids of all the cached events, it means we have to change the code in **MYSQL\_LOG::write(THD `*`, IO\_CACHE `*`, Log\_event `*`)** (sql/log.cc, line 1973) so that it reads each individual event out of the cache, transforms it, and then puts it into the bin log rather than doing the bulk copy currently being done. Fortunately, it turns out that MySQL has fixed [bug 23171](https://code.google.com/p/google-mysql-tools/issues/detail?id=3171) in 5.1 by doing the latter. My approach is going to be to backport the fix which does the header fix-up and then to augment it to handle the new group\_ids. Thus, the implementation will be that all events written to the transaction cache are written with group\_ids of 0. When the transaction is committed, in the to-be-ported MYSQL\_LOG::write\_cache where the end\_log\_pos is fixed up, we will assign the events their actual group\_ids as we write them to the bin log.

Previous iterations of this design had strong incentives for needing to do this fixup, but there currently aren't any beyond the correctness of all events having a valid group\_id. It would be possible to proceed without doing the fixup and we should still be able to achieve all the functionality contained in this document as the replication SQL thread position always advances on group boundaries, not by individual events within the transaction.

Update: the fix for [Bug23171](https://code.google.com/p/google-mysql-tools/issues/detail?id=171) was ported.

Update: When running in a mixed-mode environment where the slave has **--rpl-hierarchical** set, but the master does not, the slave will generate a group\_id of 0 for all events. 0 is used in this scenario as we do not want slaves to generate a valid looking ID sequences because multiple slaves with the option enabled connected to the same master would be generating independent sequences making the group\_ids useless for any failover scenarios. Note that events with a group\_id of 0 would not otherwise appear in the bin log because under normal operation the first group\_id to show up in the log is 1, not 0.

### Adding group\_id to bin log ###

Fortunately the format of the bin log events is such that new fields can be added to the event header without breaking the file format. This extensibility is because the first event in every bin log is a **Format\_description\_log\_event** which tells the reader of the log the size of the event headers the log contains. This extensibility means that there won't be any problems rolling out a build with the new group\_id field in the event header. Servers which don't know about the new field will just skip over the additional data. They will lose the group\_id the header contains, but in that case, while we're in the process of deploying a build with the group ID work, we're no worse off than we are today.

To add the new field to the header, I need to update **LOG\_EVENT\_HEADER\_LEN**, add a member to the **Log\_event** class and update the implementations of **Log\_event::write\_header** and **Log\_event::print\_header**.

### Preserving group\_id on slave ###

Once the group\_id is in the bin log of the master, it needs to make it to the slave's relay and bin logs. Most of this is automatically accomplished by the above changes to add the field to Log\_event. The missing piece is making sure that the value read out of the relay log on the slave makes it unchanged to the slave's bin log. Fortunately there are already 2 similar values being passed down on the **THD**, **THD::server\_id** which is set in **exec\_relay\_log\_event** and **THD::user\_time** set in **Query\_log\_event::exec\_event**. I will just have to do something similar with the new group\_id and add a check to **MYSQL\_LOG::write(Log\_event `*`)** so that it takes appropriate action with the group\_id variable tracking the server's state. As stated above, on the master the value of the variable is used to determine the group\_id to be given to the new event to be written. However, on the SQL thread of the slave, the group\_id variable will be updated with the value from the THD. This is how the slave keeps track of the last group\_id it has seen so that on failover to become a master it can continue the id sequence uninterrupted.

### Interaction with rpl\_transaction\_enabled ###

Unfortunately, turning on the bin log on the slaves introduces an issue similar to [Bug26540](http://bugs.mysql.com/bug.php?id=26540). On the replication SQL thread, when playing a **COMMIT** with the bin log on, the order of operations is:
  1. writing transaction to the bin log
  1. commit the transaction in InnoDB, which updates it's trx replication state.
  1. flush relay-log.info
As stated above, **rpl\_transaction\_enabled** provides correct behavior if the slave crashes between the 2nd and 3rd steps. However, replication will be wrong if the slave crashes between the 1st and 2nd steps, a problem which didn't previously exist in our environment because the slaves didn't have the bin log turned on.

**TC\_LOG\_BINLOG::recover** loops through the events of the last bin log creating a collection of **XID\_EVENTs**. It then calls **ha\_recover** to play these events, trying to make sure that the database matches the contents of the bin log. **trx\_recover\_for\_mysql** does not update InnoDB's replication state during this recovery because a) it isn't happening on the replication SQL thread and b) the relay log information is lost anyway since the events are coming out of the bin log, not the relay log. Unfortunately, MySQL also does not update relay-log.info with the fact that these events were recovered and so will try to replay those same events when the replication SQL thread starts up. My proposed solution for the time being is to halt replication when this condition is detected. As part of the crash recovery we will find the last group\_id written to the bin log. If, we later get an event on the replication SQL thread with a lower group\_id, we will halt replication with an appropriate error so that we do not play events multiple times. Replication can be repaired by issuing an appropriate **CHANGE MASTER** command to restart replication at the correct pos. Note, since it is the last group\_id in the bin log which concerns us, and that is written before the transaction is committed, we do not need to change **rpl\_transaction\_enabled** to track the group\_id. A future feature could be to automatically repair replication using the group\_id.

Well, if turning on the bin log made things worse on one hand, it has also made things better on the other. One of the reasons a full fix for Bug:1169545 wasn't implemented is that the scope of the changes to fix it would have been too large. However, with --log-bin enabled on the slaves, it provides us with a single function which would need to be updated, **MYSQL\_LOG::write(Log\_event `*`)**, in order to make sure that InnoDB's transaction replication state is updated for every event which gets written to the bin log, fully fixing the bug.

### Changes to mysqlbinlog ###

Because we want the new group\_id to be human readable, mysqlbinlog needs to output it as part of the standard header comment, resulting in output something like this:

```
# at 296
#080424 16:14:38 server id 11306  end_log_pos 415  group_id 2657       Query   thread_id=6    exec_time=24     error_code=0
SET TIMESTAMP=1209078878/*!*/;
insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10)/*!*/;
# at 415
#080424 16:14:56 server id 11306  end_log_pos 509  group_id 2658       Query   thread_id=6    exec_time=6      error_code=0
SET TIMESTAMP=1209078896/*!*/;
insert into t2 select * from t1/*!*/;
```

This turns out to be really easy to do as it is automatically done by making changes in **Log\_event::print\_header** which I mentioned above as needing to be updated.

As a possible future improvement, to support piping the output of mysqlbinlog to the mysql client, we could do additional work to convert the group\_id into a **SET ...** command, but careful consideration needs to be done before implementing such a feature because it creates the possibility of a server's log having duplicate or incorrectly ordered group IDs in the bin log.

### MySQL Commands ###

To support group\_ids I will be changing a couple existing MySQL commands as well as adding some new ones. First, **SHOW SLAVE STATUS** will be changed so that it will output the last group\_id processed by the replication SQL thread. Technically, displaying the information doesn't require any changes to disk files as the information is already contained on disk in the bin log and in memory in the group\_id variable. However, I may flush the group\_id to relay-log.info so that it is more easily accessible to our backup framework which sometimes reads replication state from the info files on disk rather requiring running mysqlbinlog or examining the bin log index file. The second changed command is **SHOW MASTER STATUS**. It will be changed to output the group\_id of the last group written to the bin log. Note that on mid-tier servers, which are both masters and slaves, the group\_id output by **SHOW SLAVE STATUS** and **SHOW MASTER STATUS** are the same. Thus, we could get by with only adding it to **SHOW MASTER STATUS**, but I expect many people will look for it in slave status and so I'm planning to have it in both places. Third, **SHOW BINLOG INFO FOR some\_group\_id** is a new command which will take a group\_id and output the bin log name and end\_log\_pos of the last event in the group with the matching ID in the server's bin logs.

Update: I am not flushing the group\_id to relay-log.info; it is only in the index file.

The usage of these commands is centered around failover. **SHOW MASTER STATUS** can be used to pick from the pool of potential new masters the one with the largest group\_id and thus is furthest ahead in the replication stream. Then, on the slaves which have to be failed over, use **SHOW SLAVE STATUS** to see the last event executed by the slave's replication SQL thread. On the new master which was picked, run **SHOW MASTER STATUS** to verify that the new master is at or ahead of the  slave in the replication stream. Assuming that it is, run the **SHOW BINLOG INFO FOR** command on the new master to find the correct bin log name and pos to be used in the **CHANGE MASTER ...** command on the slave in order to point it to the new master. It is assumed that a **RESET SLAVE** or similar action was taken on the slave to clean the slave's previous relay logs and thus ensure that the same event(s) aren't pulled down from both the old and new masters.

Let's walk through an example. First, let's look at a snippet of a master's bin log:

```
# at 499
#080424 16:14:38 server id 11306  end_log_pos 618  group_id 2657     Query   thread_id=6    exec_time=0      error_code=0
SET TIMESTAMP=1209078878/*!*/;
insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10)/*!*/;
# at 618
#080424 16:14:56 server id 11306  end_log_pos 712  group_id 2658     Query   thread_id=6    exec_time=0      error_code=0
SET TIMESTAMP=1209078896/*!*/;
insert into t2 select * from t1/*!*/;
```

Assume that when **SHOW SLAVE STATUS** is run on the slave the group\_id reported for the SQL thread is 2657, indicating that the slave has successfully processed the **insert into t1 values (1,2), (3,4), (5,6), (7,8), (9,10);** statement. Back on the master **SHOW BINLOG INFO FOR 2657** is run. As stated above, it returns the end\_log\_pos from the the last event of the group with the matching group\_id, which is 618. This is the correct value to use in **CHANGE MASTER** on the slave because we want the slave to start replicating with the **insert into t2 select `*` from t1;** statement.

The implementation of finding the bin log name and pos will depend on whether the group\_ids in the index file are trustworthy. That is, whether crash recovery fixes incorrect values or leaves them be. If they can be trusted, then the group\_ids in the index file can be used to determine the correct bin log file. If they can't, the correct bin log file can be determined by reading the start of each bin log file to find the group\_id at the top. Once the correct file is found, finding the correct pos is a matter of sequentially reading the events out of the file looking for the matching one. I can leverage the code which fixes [bug 23171](https://code.google.com/p/google-mysql-tools/issues/detail?id=3171) to do this scan.

The last command is a utility to reset a server's current group\_id to a specific value, **SET BINLOG\_GROUP\_ID**. This command requires global **SUPER\_ACL** in order to be used as care must be taken when using this command. Incorrectly setting the value could cause replication to halt or, worse, cause slaves to replicate from the incorrect position in the replication stream resulting in data differences between servers in the topology. As part of processing this command, the server will update the bin log index file and, if the server is a slave, relay-log.info and flush both files to disk. These need to be flushed so that if the command is run and the server shutdown before processing any events it will use the correct value when started up again.

### Interaction with backup ###

I'm going to make an assumption that a server never goes from being freshly restored immediately to being the root master; it will always spend some time as a slave first. If this assumption is true, then the code and my.cnf changes in this document can be rolled out without requiring any immediate changes to the current backup & restore scripts though there will be a bug that exists. Since the backup doesn't include the bin log index file, a freshly restored server which hasn't yet received any replication events will report a group\_id of 0 in **SHOW SLAVE STATUS**.

Restore should use **SET BINLOG\_GROUP\_ID** to the value it saved with the backup so that **SHOW SLAVE STATUS** reports the correct information immediately rather than having to wait until the server receives an event to be correct.

## Caveats ##

### New event type ###

Previous iterations of this design proposed using event\_id instead of group\_id. That is, all events, even those comprising a group, in the bin log would have had a unique ID. Requirements have since changed and more investigation found a problem with the event\_id approach and so the design was switched to group\_id, but it left the proposal that the header of all events be updated to include the ID. As an alternative, it may be possible to achieve the goals by instead creating a new event type. This new event type would be similar to some of the existing event types, like **User\_var\_log\_event**, in that they would set state on the **THD** which would then apply to the events following it, resulting in logs something like:

```
# at 2647
#080610 12:23:15 server id 11306  end_log_pos 2663      Group_id
SET group_id=25486
# at 2663
#080610 12:23:15 server id 11306  end_log_pos 2718      User_var
SET @`my_var`:=1007/*!*/;
# at 2718
#080610 12:23:15 server id 11306  end_log_pos 2825      Query   thread_id=7     exec_time=0     error_code=0
SET TIMESTAMP=1213125795/*!*/;
insert into test.t1 values (@my_var)/*!*/;
```

The fact that such an approach doesn't require changes to the format of any existing events is a big advantage, but since the switch from event\_ids to group\_ids is a late one I have not fully considered such an option to know if it is viable. Comments in the MySQL code indicate that they planned to update the header of events to include an ID:

```
/*
   Fixed header length, where 4.x and 5.0 agree. That is, 5.0 may have a longer
   header (it will for sure when we have the unique event's ID), but at least
   the first 19 bytes are the same in 4.x and 5.0. So when we have the unique
   event's ID, LOG_EVENT_HEADER_LEN will be something like 26, but
   LOG_EVENT_MINIMAL_HEADER_LEN will remain 19.

   Update: LOG_EVENT_HEADER_LEN may now be either 19 or 27.
*/
#define LOG_EVENT_MINIMAL_HEADER_LEN 19
```

Also, an obvious problem with this approach is the issue I mentioned above about piping the output of mysqlbinlog to the mysql client easily leading to duplicate / unordered group\_ids in the bin log.

### Multi-master ###

In topologies with a single master, the group\_id is sufficient to uniquely identify an event in the binlog of any server in the system because only a single server is generating group\_ids. However, with multiple masters my approach does not guarantee unique group\_ids throughout the whole system. Instead, the pair (server\_id, group\_id) needs to be used to uniquely identify events as the multiple servers generating original bin log events will all be doing so starting at 0, producing identical group\_ids, but that is outside the focus of this document.

### Using log\_index & log\_pos ###

My initial plan on generating a unique ID was going to be to use the triple (server\_id, orig\_log\_index, orig\_log\_pos). These data are all currently known at the time events are written to the bin log and have the advantage of not requiring the server to track or persist any new data. I didn't use them because MySQL indicated that they wanted us to use the (server\_id, group\_id) pair, group\_id works better for exporting row changes and the triple (server\_id, log\_index, log\_pos) isn't guaranteed to be unique because, when the log\_index overflows, log\_index will get reused.

With this alternate solution, both orig\_log\_index & orig\_log\_pos would have been added as new fields in the event header. Recall that the current log\_pos is that of the server writing the bin log, not the server where the event originated, and so th orig\_log\_pos would have to be added. Also, as I'm planning to do with group\_id, these new fields would have needed to be passed from the relay log to the bin log of the slave via the **THD**.

### Using timestamp ###

As already stated above, the timestamp of the master is retained in the relay and bin logs of the slaves and so could be used as part of the tuple which uniquely identifies events. One problem with using the timestamp is that it is doesn't have sufficient granularity; multiple events in a bin log can have the same timestamp:
```
# at 1441
#080428 11:34:05 server id 11306  end_log_pos 1532      Query   thread_id=289   exec_time=0     error_code=0
SET TIMESTAMP=1209407645/*!*/;
insert into test.t1 values (1,2)/*!*/;
# at 1532
#080428 11:34:05 server id 11306  end_log_pos 1623      Query   thread_id=289   exec_time=0     error_code=0
SET TIMESTAMP=1209407645/*!*/;
insert into test.t1 values (3,4)/*!*/;
# at 1623
#080428 11:34:05 server id 11306  end_log_pos 1714      Query   thread_id=289   exec_time=0     error_code=0
SET TIMESTAMP=1209407645/*!*/;
insert into test.t1 values (5,6)/*!*/;
# at 1714
#080428 11:34:05 server id 11306  end_log_pos 1805      Query   thread_id=289   exec_time=0     error_code=0
SET TIMESTAMP=1209407645/*!*/;
insert into test.t1 values (7,8)/*!*/;
# at 1805
#080428 11:34:05 server id 11306  end_log_pos 1897      Query   thread_id=289   exec_time=0     error_code=0
SET TIMESTAMP=1209407645/*!*/;
insert into test.t1 values (9,10)/*!*/;
DELIMITER ;
```

The granularity problem could be resolved by appending an index onto the event which effectively resets back to zero each second. With such an approach, our unique group ID then becomes the triple (server\_id, timestamp, index). Such an approach has the advantage that it doesn't require the master to persist any new data; the index to use at server start would always be 0. Unfortunately, such an approach still isn't guaranteed to yield unique IDs as a machine's current time can be changed back to a time which it has already seen.

## Testing Plan ##

MySQL has a unit test framework which already includes many replication test cases. Ensuring those tests all continue to pass after I made my code changes ensures that my changes don't break any existing functionality, but those cases will be limited to testing replicating from a server with my changes to another server also with my changes. To verify that there won't be any event log compatibilities rolling out a build with my changes, I will also setup multi-version replication scenarios:

  * Server without my changes -> server with my changes
  * Server with my changes -> server without my changes

We will want to test out the new features in a production environment without actually having to enable the **--rpl-hierarchical** option on the primaries. To accomplish that we'll need an option, **--rpl-hierarchical-act-as-root-master**, to allow a slave to pretend that it is the root master and to generate group\_ids for events it is playing on its replication SQL thread. We could then setup a topology such as:

TODO -- upload png

where P is a current primary and [R1](https://code.google.com/p/google-mysql-tools/source/detail?r=1) through RN are the current production replicas without any changes to the existing topology or their my.cnf settings. FM and A through E are testing servers not receiving any production traffic. FM is the fake master running with **--rpl-hierarchical-act-as-root-master** as well as **--rpl-hierarchical**. Servers A through E are all running with **--rpl-hierarchical**. With this environment we can do things like:
  * Write a script to periodically swap the master of C, D and/or E.
  * Run data drift between any of A through E.
  * kill -9 mysqld on any of A through E and verify crash recovery picks up the correct group\_id.
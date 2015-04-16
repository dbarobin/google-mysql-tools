# Introduction #

Heikki Tuuri worked on this first, but there wasn't much demand for it, beyond my request. Solid will offer their version of this later in 2007. We couldn't wait and implemented it.

The MySQL replication protocol is asynchronous. The master does not know when or whether a slave gets replication events. It is also efficient. A slave requests all replication events from an offset in a file. The master pushes events to the slave when they are ready.

# Usage #

We have extended the replication protocol to be semi-synchronous on demand. It is on demand because each slave registers as **async** or **semi-sync**. When semi-sync is enabled on the master, it blocks return from commit until either at least one semi-sync slave acknowledges receipt of all replication events for the transaction or until a configurable timeout expires.

Semi-synchronous replication is disabled when the timeout expires. It is automatically reenabled when slaves catch up on replication.

# Configuration #

The following parameters control this:
  * **rpl\_semi\_sync\_enabled** configures a master to use semi-sync replication.
  * **rpl\_semi\_sync\_slave\_enabled** configures a slave to use semi-sync replication. The IO thread must be restarted for this to take effect.
  * **rpl\_semi\_sync\_timeout** is the timeout in milliseconds for the master

# Monitoring #

The following variables are exported from SHOW STATUS:
  * **Rpl\_semi\_sync\_clients**: number of semi-sync replication slaves
  * **Rpl\_semi\_sync\_status**: whether semi-sync is currently ON/OFF
  * **Rpl\_semi\_sync\_slave\_status**: TBD
  * **Rpl\_semi\_sync\_yes\_tx**: how many transaction got semi-sync reply
  * **Rpl\_semi\_sync\_no\_tx**: how many transaction do not get semi-sync reply
  * **Rpl\_semi\_sync\_no\_times**: TBD
  * **Rpl\_semi\_sync\_timefunc\_failures**: how many gettimeofday() function fails
  * **Rpl\_semi\_sync\_wait\_sessions**: how many sessions are waiting for replies
  * **Rpl\_semi\_sync\_wait\_pos\_backtraverse**: how many time we move waiting position back
  * **Rpl\_semi\_sync\_net\_avg\_wait\_time(us)**: the average network waiting time per tx
  * **Rpl\_semI\_sync\_net\_wait\_time**: total time in us waiting for ACKs
  * **Rpl\_semi\_sync\_net\_waits**: how many times the replication thread waits on the network
  * **Rpl\_semi\_sync\_tx\_avg\_wait\_time(us)**: the average transaction waiting time
  * **Rpl\_semi\_sync\_tx\_wait\_time**: TBD
  * **Rpl\_semi\_sync\_tx\_waits**: how many times transactions wait
  * **Rpl\_semi\_sync\_timefunc\_failures**: #times gettimeofday calls fail

# Design Overview #

Semi-sync replication blocks any COMMIT until at least one replica has acknowledged receipt of the replication events for the transaction. This ensures that at least one replica has all transactions from the master. The protocol blocks return from commit. That is, it blocks after commit is complete in InnoDB and before commit returns to the user.

This option must be enabled on a master and slaves that are close to the master. Only slaves that have this feature enabled participate in the protocol. Otherwise, slaves use the standard replication protocol.

# Deployment #

Semi-sync replication can be enabled/disabled on a master or slave without shutting down the database.

Semi-sync replication is enabled on demand. If there are no semi-sync replicas or they are all behind in replication, semi-sync replication will be disabled after the first transaction wait timeout. When the semi-sync replicas catch up, transaction commits will wait again if the feature is not disabled.

# Implementation #

The design doc is at SemiSyncReplicationDesign.

Each replication event sent to a semi-sync slave has two extra bytes at the start that indicate whether the event requires acknowledgement. The bytes are stripped by the slave IO thread and the rest of the event is processed as normal. When acknowledgement is requested, the slave IO thread responds using the existing connection to the master. Acknowledgement is requested for events that indicate the end of a transaction, such as commit or an insert with autocommit enabled.
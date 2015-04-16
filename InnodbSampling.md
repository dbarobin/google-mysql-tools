# Introduction #

InnoDB uses sampling to determine optimizer statistics. It uses index keys from 8 leaf nodes. For some tables, more samples are needed. We added a session parameter that sets the number of leaf blocks that are sampled. The default for the parameter is 8. The name of the parameter is **innodb\_btr\_estimate\_n\_pages**.
# Introduction #

This is a backport of the [batch key access](http://dev.mysql.com/doc/refman/6.0/en/bka-optimization.html) feature from MySQL 6. It works great for a limited workload. There is a bug that produces intermittent wrong query results when used for InnoDB.

# Details #

These details are courtesy of Igor B.

This feature is controlled by the my.cnf parameter **join\_cache\_level**. It supports the following values:
  * **0** - do not use join buffering
  * **1** - use join buffer only for inner cross joins employing Blocked Nested Loops (BNL)Join Algorithm as is normally done by MySQL.
  * **2** - use the linked version of this algorithm. What is this? **TODO**
  * **3,4** - use a BNL variant for outer joins and semi-joins. BNL was not used for these before. Nested outer joins and nested semi-joins cannot work with a join buffer of level 3.
  * **5** -use a join buffer for all fields referenced in the query. Do not eliminate duplicate keys stored in the join buffer.
  * **6** -use a join buffer for the fields from the joined table. The fields from the previous tables are reached by a reference to the join buffers attached to these tables. These join caches are called linked. Do not eliminate duplicate keys in the join buffer.
  * **7,8** - levels 7 and 8 are the same as 5 and 6 except that duplicate keys are eliminated before being passed to the MRR interface. Falcon requires this level.

Nested outer joins and nested semi-joins (outer joins and semi-joins with more than 1 inner table) can use only join caches of level 6 and 8 (linked caches).
# Global Transaction IDs #

GlobalTransactionIds were originally developed against MySQL 5.0. Work has begun to port the functionality to MySQL 5.1. This patch should be considered to be of in-development status; it has not run in any production environment yet. Some additional notes
  * The feature requires --binlog-format=STATEMENT
  * The feature disallows
    * Use of --binlog-do-db
    * Use of --binlog-ignore-db
    * Creation of temporary tables on the master
  * Be very careful with the use of triggers / stored procedures; they may break the feature.

[Click here to download the patch.](http://google-mysql-tools.googlecode.com/svn/trunk/mysql-5.1-patches/mysql-5.1.58-global-trx-ids.patch.gz)
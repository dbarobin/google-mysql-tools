# Notes #

**show innodb status** has been enhanced:
  * The current transaction section is printed last in case it is too large and gets truncated
  * The maximum amount of data returned by it has been increased from 64kb to 128kb
  * More data is provided in **show innodb status**

Extra statistics are provided for:
  * average time per read and write request
  * sources of calls to fsync
  * sources of calls to sync the InnoDB transaction log
  * counts of work done by the background IO thread
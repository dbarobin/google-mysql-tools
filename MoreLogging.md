# Introduction #

These enable different types of audit logging.

Parameters:
  * audit\_log - log logins, queries against specified tables, and startup
  * log\_tables - log queries that use these table to the audit log (comma seperated)
  * log-update - log DDL and DML by users with the SUPER privilege

# Audit Log #

**audit\_log** is a new parameter. When set, entries are written to an audit log for every connection, query to a specified table and startup. Another variable, **log\_tables**, is used to list the tables for which all accesses are logged.

# Super User Log #

**log-update** logs all DDL and DML (create, drop, insert, update, delete and replace statements) done by users with the SUPER privilege. DML statements are only logged when they modify at least 1 row after statement completion. A statement that changes 1 row but is never committed, still writes data to the log.

This paramter is not new. This patch changes the meaning of the parameter.
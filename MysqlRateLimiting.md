# Introduction #

This describes SQL commands added to rate limit queries per account and per client IP.

## Per account rate limiting ##

Per-account query delays use new SQL commands to set a query delay for an account. The delay is the number of milliseconds to sleep before running a SQL statement for the account. These values are transient and all reset to zero delay on server restart. The values are set by the command **MAKE USER 'user' DELAYED 100** where the literals _user_ and _100_ are the account and number of milliseconds to sleep. There is no delay when the value is 0. The values are displayed by the command **SHOW DELAYED USER**.

MySQL had a feature to limit the number of queries per hour for an account. This is done by setting the _user.max\_questions_ column for the account. We have changed this to be the max queries per minute so that when an account reaches the limit, it doesn't have to wait for an hour for the reset.

These don't change the behavior for existing connections. For the problem account to reconnect to get the new values.

## Per client IP rate limiting ##

Per-client rate limiting is done by the command **MAKE CLIENT 'IP-address' DELAYED 100** where the literal Ip-address is the exact match for the client IP that should be delayed and 100 is the number of milliseconds to delay each statement. The delays are displayed by the command **SHOW DELAYED CLIENT**.
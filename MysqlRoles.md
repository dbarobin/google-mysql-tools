# MySQL roles and mapped users #

The access control model in MySQL does not scale for a deployment with thousands of accounts and thousands of tables. The problems are that similar privileges are specified for many accounts and that the only way to limit an account from accessing a table is to grant privileges at the table or column level in which case the _mysql.user_ table has millions of entries.

Privileges may be associated once with a role, and then many accounts may be mapped to that role. When many accounts have the same privileges, this avoids the need to specify the privileges for each account.

We have implemented _mapped users_ in the MySQL access control model. These are used to simulate roles and solve one of these problems. A _mapped user_ provides authentication credentials and is mapped to a _role_ for access control. A new table, mysql.mapped\_user, has been added to define mapped users. Entries in an existing table, mysql.user, are reused for roles when there are entries from mysql.mapped\_user that reference them.

To avoid confusion:
  * mapped user - one row in mysql.mapped\_user
  * role - one row in mysql.user referenced by at least one row in mysql.mapped\_user

This provides several features:
  * multiple passwords per account
  * manual password expiration
  * roles
  * transparent to users (_mysql -uuser -ppassword_ works regardless of whether authentication is done using entries in _mysql.mapped\_user_ or _mysql.user_)

## Use Case ##

Create a role account in _mysql.user_. Create thousands of private accounts in _mysql.mapped\_user_ that map to the role. By map to I mean that the value of _mysql.mapped\_user.Role_ is the account name for the role.

## Implementation ##

Authentication in MySQL is implemented using the _mysql.user_ table. mysqld sorts these entries and when a connection is attempted, the first entry in the sorted list that matches the account name and hostname/IP of the client is used for authentication. A challenge response protocol is done using the password hash for that entry.

A new table is added to support mapped users. This table does not have columns for privileges. Instead, each row references an account name from mysql.user that provides the privileges. The new table has a subset of the columns from _mysql.user_:
  * User - the name for this mapped user
  * Role - the name of the account in _mysql.user_ from which this account gets its privileges
  * Password - the password hash for authenticating a connection
  * PasswordChanged - the timestamp when this entry was last updated or created. This is intended to support manual password expiration via a script that deletes all entries where PasswordChanged less than the cutoff.
  * ssl\_type, ssl\_cipher, x509\_issuer, x509\_subject - values for SSL authentication, note that code has yet to be added in the server to handle these values

DDL for the new table:
```
CREATE TABLE mapped_user (
  User char(16) binary DEFAULT '' NOT NULL,
  Role char(16) binary DEFAULT '' NOT NULL,
  Password char(41) character set latin1 collate latin1_bin DEFAULT '' NOT NULL,
  PasswordChanged Timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL,
  ssl_type enum('','ANY','X509','SPECIFIED') character set utf8 NOT NULL default '',
  ssl_cipher blob NOT NULL,
  x509_issuer blob NOT NULL,
  x509_subject blob NOT NULL,
  PRIMARY KEY (User, Role, Password)
) engine=MyISAM
CHARACTER SET utf8 COLLATE utf8_bin
comment='Mapped users';
```

## Authentication ##

Entries from mysql.mapped\_user are used to authenticate connection attempts only when authentication fails with entries in mysql.user. The failure may have occurred because there was no entry in mysql.user for the user/host or because the password was wrong. If authentication succeeds using an entry in mysql.mapped\_user, the mysql.mapped\_user.Role column in that entry and the client's hostname/IP are used to search mysql.user for a matching entry. And if one is found, that entry provides the privileges for the connection. By _provides the privileges_ I mean that:
  * the values of mysql.user.User and mysql.user.Host are used to search the other privilege tables
  * the global privileges stored in mysql.user for the matching entry are used

The mysql.mapped\_user table supports multiple passwords per account. When a user tries to create a connection with a username that is in the mysql.mapped\_user table and there are multiple entries with a matching value in mysql.mapped\_user.User, then authentication is attempted for one entry at a time using the password hash in mysql.mapped\_user.Password until authentication succeeds or there are no more entries. Note that the order in which the entries from mysql.mapped\_user are checked is **not** defined, but this is only an issue when there are entries in mysql.mapped\_user with the same value for _User_ and different values for _Role_ and that deployment model should not be used. Also note that this does not require additional RPCs during client authentication.

Entries are ignored from mysql.mapped\_user when:
  * Role is the empty string.
  * User is the empty string
  * Password is the empty string

There is no constraint between the values in mysql.mapped\_user.User and mysql.user.User.  Thus, a bogus mapping (Role references an account that does not exist in mysql.user) can be created. In that case, the entry in mysql.mapped\_user cannot be used to create connections and will get access denied errors.

There is a primary key index on mysql.mapped\_user, but that is not sufficient to enforce all of the integrity constraints that are needed. Entries with the same values for _User_ and _Role_ but different passwords are allowed, and the primary key forces the password to be different. Entries with the same value for _User_ but different values for _Role_ should not be allowed. However, this can only be enforced with a check constraint on the table and MySQL does not enforce check constraints. We can write a tool to find such entries.

## SQL Interfaces ##

Roles can be added via the _create mapped user_ command that is similar to _create user_ but extended to support options for SSL connections. Roles can be dropped by the _drop mapped user_ command that is similar to _drop user_. These commands update internal data structures and update the mysql.mapped\_user table. There is no need to run _flush privileges_ with these commands.

The following have been changed to print the value of _mysql.mapped\_user.User_ rather than the value of _mysql.user.User_ when a role is used to create a connection.
  * error messages related to access control
  * _select current\_user()_
  * _select user()_
  * _show user\_statistics_
  * _show processlist_

The output of _show grants_ has not been changed and will display the privileges for the role (the entry in _mysql.user)._

_set password = password(STRING)_ fails for accounts that use a role. The only way to change a password for an entry in _mysql.mapped\_user_ is by an insert statement.

**show processlist with roles** displays the role for connections from mapped users rather than the mapped user name. **show processlist** displays the value from mysql.mapped\_user.

**show user\_statistics with roles** displays statistics aggregated by role for connections from mapped users. _show user\_statistics_ displays values aggregated by the value from mysql.mapped\_user.

Mapped users can be created by inserting into mysql.mapped\_user and then running FLUSH PRIVILEGES. They are also created by the _create mapped user_ command. An example is **create mapped user mapped\_readonly identified by 'password' role readonly**.

Mapped users can be dropped by deleting from mysql.mapped\_user and then running FLUSH PRIVILEGES. They are also dropped by the _drop mapped user_ command. An example is **drop mapped user foo**. This drops all entries from mysql.mapped\_user with that user name. A delete statement must be used to drop an entry matching either (username, role) or (username, role, password).

**select user()** displays the value of the mapped user name when connected as a mapped user. **select current\_user()** displays the value of the role when connected as a mapped user. This is done because **current\_user()** is defined to return the name of the account used for access control.

MysqlRateLimiting via _make user delayed_ is done on the value of the account name. It does not matter whether the account is listed in mysql.user or mysql.mapped\_user.

mysql.mapped\_user does not have columns for resource limits such as max connections and max QPS. Limits are enforced per role.

This feature is only supported when the configuration variable **mapped\_users** is used (add to /etc/my.cnf). This feature is disabled by default. Also, the mysql.mapped\_user table must exist. This table does not exist in our current deployment. It must be created before the feature is enabled. The scripts provided by MySQL to create the system databases will create the table, but we do not use those scripts frequently.

The value of the mysql.user.Host column applies to any mapped users trying to create a connection. This can be used to restrict clients to connect from prod or corp hosts.

## Open Requests ##
  * Add a unique index on (User, Password)
  * Add an email column to mysql.mapped\_user
  * Inherit limits (hostname/IP address from which connections are allowed, connection limits, max queries per minute limit) from the mysql.user table.
  * Implement support for SSL -- the mysql.mapped\_user table has columns for SSL authentication. Code has not been added to the server to handle them.
RCP 1 - Multi users AUTH and ACLs for Redis.
===

```
Author: Salvatore Sanfilippo <antirez@gmail.com>
Creation date: 12 Dec 2014
Update date: 13 Dec 2014
Status: open
```

Rationale
---

 Even if it's lovely to think at Redis as a do-one-thing
daemon, in complex production environments you end needing basic ACLs,
credentials rotations, and so forth.

Notes: This proposal only deals with AUTH + ACLs in the context of
users configured inside Redis. However the ability to authenticate the
user via external systems (LDAP or similar) will be covered in a
different proposal.

Commands introduced
---

New form of the AUTH command:

AUTH <username> <password>

An old form of AUTH will still be supported, with a single argument,
for compatibility with the past. Eventually it will get deprecated.

The default user, before calling AUTH, is the special "default" user
which is considered to be always authenticated once a connection is
created with Redis.
In general, AUTH always succeeds, even if we have an existing
authentication in place. The semantics is last-AUTH-wins. If AUTH
fails, the previous authentication level is retained.

Configuration directive
---

A single configuration directive is added, called users-file, that
specifies the path of a file that lists users, passwords, ACLs.

Example:

users-file /var/redis/users.txt

The users file has the following syntax:

<username> <password> <acls>

As with redis.conf, it is possible to use quotes for strings
containing spaces. Example:

antirez "my password" xrwa
default "" x

ACLs
---

ACLs are specified as the third argument in the password. Each
character enables a class of commands.

x: AUTH command.
r: Read only commands.
w: Write commands.
d: Dangerous write commands (FLUSHALL, FLUSHDB)
a: Admin commands (CONFIG, MONITOR, SLAVEOF, MIGRATE, RESTORE, ...)

If no users file is provided, or no "default" user is found inside,
new connections have the maximum level of access, like it happens
today.

Reloading users
---

Password file is re-read and the local state updated using the "CONFIG
reload-users" command, or when setting a new password file via "CONFIG
SET users-file /new/or/old.path".

Security and hashed passwords
---

Passwords in the Redis password file are non hashed since passwords
usually don't represent user credentials but just different classes of
clients accessing to Redis, and for the high performance nature of
Redis by default it should be possible to authenticate users in a very
small amount of time, and weak hashing like SHA1() is pointless
(secure hashing requires the authenticator to consume a non trivial
amount of CPU). However because of the support for external
authentication methods, not discussed here, the user will be able to
switch to much more secure authentication methods easily.

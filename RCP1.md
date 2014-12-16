RCP 1 - Multi users AUTH and ACLs for Redis.
===

```
Author: Salvatore Sanfilippo <antirez@gmail.com>
Creation date: 2014-12-12
Update date: 2014-16-12
Status: open
Version: 1.2
```

History
---

* Version 1.0 (2014-12-12): Initial version.
* Version 1.1 (2014-13-12): Slow commands ACL flag added.
* Version 1.2 (2014-16-12): Fine grained command ACLs.

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

New form of the `AUTH` command:

    AUTH <username> <password>

An old form of `AUTH` will still be supported, with a single argument,
for compatibility with the past. Eventually it will get deprecated.

The default user, before calling `AUTH`, is the special "default" user
which is considered to be always authenticated once a connection is
created with Redis.
In general, `AUTH` always succeeds, even if we have an existing
authentication in place. The semantics is last-AUTH-wins. If AUTH
fails, the previous authentication level is retained.

Configuration directive
---

A single configuration directive is added, called users-file, that
specifies the path of a file that lists users, passwords, ACLs.

Example:

    users-file /var/redis/users.txt

The users file has the following syntax:

    <username> <password> [<acl> <acl> ... <acl>]

As with redis.conf, it is possible to use quotes for strings
containing spaces. Example:

    antirez "my password" +#readonly -#slow
    default ""

ACLs
---

ACLs specify the set of commands the user is able to call.
ACLs are specified as multiple arguments. If zero ACL arguments are provided
the default is that the set of the commands the specified user can use is
zero. For example the line `default ""` means that the default user (not
authenticated) can't execute anything, with the exception of the `AUTH`
command itself which is always allowed.

ACLs for an user are processed left to right, and are prefixed with a
plus or minus character. The plus is used in order to add commands to the
set of commands the user is able to execute, while the minus command remove
commands. For example the following command allows the default user to
execute the `PING` and `INFO` commands:

    default "" +ping +info

It is possible to add whole classes of commands by providing the name of the
class prefixed by a `#` character. For example the following configuration
line adds all the read-only commands for the user `guest` but disallows
commands that have a time complexity greater than logarithmic:

    guest mypassword +#readonly -#slow

Another example where the user is given read only access and some write access:

    guest mypassword +#readonly +sadd +zadd +del

The following classes of commadns are available:

* #readonly -- All the non administrative read only commands.
* #write -- All the non administrative write commands.
* #slow -- #readonly and #write commands with a time complexity greater than log(N).
* #admin -- Administrative commands (`CONFIG`, `SHUTDOWN`, `BGSAVE`, ...).
* #string -- String type commands, including bitmaps (RW).
* #list -- List type commands (RW).
* #set -- Set type commands (RW).
* #zset -- Sorted Set type commands (RW).
* #hash -- Hash type commands (RW).
* #hyperloglog -- HyperLogLog commands (RW).
* #scan -- Scan family commands.
* #pubsub -- Pub/Sub commands.
* #transaction -- Transaction related commands.
* #scripting -- Lua scripting commands.

If no users file is provided, or no "default" user is found inside,
new connections have the maximum level of access, like it happens
today.

Reloading users
---

Password file is re-read and the local state updated using the "CONFIG
reload-users" command, or when setting a new password file via "CONFIG
SET users-file /new/or/old.path".

Implementation details
---

In order to implement ACLs with good performances, each command is flagged
with a progressive ID at startup. When users are represented in the Redis
memory, each user is associated with a bitmap composed of 4 unsigned 64 bit
integers, representing each a bit corresponding to a single command ID, for
a maximum of 256 commands (currently we have 157 commands).

Clients also have a set of 4 64 bit integers, which by default are populated
by copying the flags of the `default` user, which is referenced by Redis
in a direct way (no lookup needed) in the `server` structure:

    client->cmdacl[0] = server.default_user->cmdacl[0];
    client->cmdacl[1] = server.default_user->cmdacl[1];
    client->cmdacl[2] = server.default_user->cmdacl[2];
    client->cmdacl[3] = server.default_user->cmdacl[3];
    /* ... Set the bit for the AUTH command here. */

Each time an user tries to execute a command, the corresponding bit (by
computing `2^command-id`) is checked in the client `cmdacl` array. The
command execution is refused with an `-ACL` error if the bit is not found
to be set.

Notes about implementations and behavior when ACLs are modified at runtime
---

Using the implementation described above, changing the ACL for an user at
runtime via `CONFIG SET` does not automatically changes the access level of the currently connected clients.
If this is not desireable, a more complex alternative can
be implemented, where the client has a reference to the user object. However
this means that users must be reference counted (since they may change because
of `CONFIG SET`, or be deleted at all) and when existing users are updated,
they should be updated in place. Moreover this means that the semantics of
a given connection may change at runtime, that may not be great for rotation
of clients privileges: old clients should experiment the same behavior as
usually until reconnection.

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

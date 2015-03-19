# RCP 3 - Lazy write operations

```
Author: Sebastian Waisbrot <seppo0010@gmail.com>
Creation date: 2015-03-19
Update date: 2015-03-19
Status: Open
Version: 1.0
Implementation: none
```

## History

* Version 1.0 (2015-03-19): Initial version.

## Rationale

Some write commands may be slow, and their result may not be immediately
required. The client could ask for a the command to run at some point in
the future.

## Commands introduced

LAZY command:

    LAZY <command> <arguments...>

The command arity will be checked, and then enqueued internally for
processing. The command will return the status LAZY-ENQUEUED if no error is
detected.

Any new command issued that uses keys that would be modified by the command
will automatically run enqueued commands and discard their responses.

### Example

    LAZY SINTERSTORE result bigset1 bigset2
    "LAZY-ENQUEUED"
    SCARD result
    10214

Notice this changes *when* the command is executed. The `"LAZY-ENQUEUED"`
response is immediate and does not depend on the complexity of the operation.

### Restrictions

Time sensitive commands, such as EXPIRE, will fail to be executed lazily.

Commands that do not affect any key, such as PUBLISH, cannot be executed
lazily.

When running on a cluster, the restrictions for multi key commands are
enforced.

### Execution

Once enqueued, the lazy command may be executed either actively or passively.

Active execution happens when a non lazy command is issued that requires the
value of the key.

For passive execution a list of keys with lazy commands is kept. When
redis-server is idle (no commands received), a random key with queued data
is fetched, and the first enqueued command for that keys is executed and
removed.

### Persistence and replication

Commands enqueued persistence should be guaranteed as if the command was issue
in a no lazy manner.

For AOF and replication, the command will be logged as non-lazy when received,
and when the command is actually executed it will not be logged.

For RDB, the lazy queue will be fully executed before dumping data to the file.

## Configuration directive

A new directive is added to enable the passive execution of the queue.

    lazy-queue-enabled 1

## Implementation details

`redisDb` will have a new `dict *lazy_commands` that will map keys to a
`list *` with the commands per key.

Each key will have a queue will be a list of commands and arguments per key.

Lazy commands can be run on keys with expiration, and if they are still
enqueued when the key is evicted, the commands will be discarded.

Notice that executing a lazy command may trigger a chain of execution of
another lazy commands. However no dead-lock is possible since the commands
per key are enqueued in order.

## The good, the bad, the ugly

### The good

* Expensive operations can be executed with no client waiting for a response.
* The feature is completely opt-in and adds little overhead for people who
  do not want to use it.

### The bad

* It does add a few checks for every single key read.
* It adds additional complexity that might need to be checked in several
  places.

### The ugly

* Developers may run into long execution time for commands that were supposed
  to be fast if lazy commands need to run first.

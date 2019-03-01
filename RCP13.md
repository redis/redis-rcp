RCP 13 - Key locks for modules "no GIL" threaded operations
===

    Author: Salvatore Sanfilippo <antirez@gmail.com>
    Creation date: 2019-02-28
    Update date: 2019-02-28
    Status: open
    Version: 1.0
    Implementation: none

## Problem background

Redis modules allow to perform the execution of a command in a different thread
than the Redis process main thread. The way this is accomplished is via the
following process:

* The command execution calls an API to block the client. Such API returns an handle to later send some reply to the client and unblock it. The command execution immediately returns to the caller after passing the execution of the command to some thread that receives such handler.
* The thread executes operations related to the command. In order to access the Redis dataset it is required to create a thread-safe context and lock the server using a global lock. However to just send replies to the blocked client handle no lock is needed.
* Finally the thread calls the API to unlock the client: the reply accumulated in the blocked client handle is transferred to the actual client in the main thread (at no cost, just swapping a pointer), and the client execution continues.

This setup is perfect if the background operation to perform is slow to compute
but does not need to access much data from Redis. An example to show this point
is a command that takes a key containing a number, factorize the number in its
prime factors, and later returns a list containing the prime factors to the
client. The operation can be slow to perform, but there is very little data
access to perform. The command would work like that:

* Read the key value, block the client, send the value and the blocked client handle to the thread.
* The background thread performs the factorization.
* Finally the background thread replies sending data to the blocked client handler (without holding any lock, since the blocked client handle is a copy not linked to the main thread), and unblocks the client (also a fast operation).

This is efficient and the majority of the work was carried in the background thread without blocking the main thread in any way.

## Limits of the current implementation

In the example above we found an efficient implementation of the blocking
operation to factorize integers. However given that data itself is only accessible
via a global lock, other commands reading or modifying large values would not
experience such improvement using a threaded implementation.

Note that currently the lock is managed so that Redis allows the other threads
to access the data space only when it is busy in the kernel waiting for the
file descriptors to get ready, so many times the threaded operations don't make
the main process itself slower. However this architecture is not scalable.

Let's imagine to implement a version of `SMEMBERS` that can serve the client in
another thread if the value is found to be big (for small values to serve the
client synchronously can still be the best strategy).

Assuming that not all the clients are calling `SMEMBERS` against the same
*hot key*, we could make such implementation scalable allowing the thread to
lock the key. This is what would happen roughly:

* `BACKGROUND-SMEMBERS` is called from a module implementing it.
* The command implementation attempts to lock the key for reading: once the request succeeded we'll receive a key object that will not be modified until the key is unlocked.
* Now the key is marked as locked in the main thread: clients trying to perform work on such key that is incompatible with the locking type (for instance clients trying to perform `SADD`) are blocked, and will be resumed later once the key is accessible again.
* The command implementation calls `RedisModule_BlockClient` and processes the request knowing that the locked will not be touched. We can read from the key and put data in the blocked client handle without hitting any global lock: we are just blocking other clients that want to write to the same key.
* Finally the command implementation unlocks the key and unblocks the client.
* The pending clients for that key are served.

So read-write locks with key granularity will be able to make Redis threaded
for all the slow operations that in the past used to block the server, in the
present can be performed concurrently using thread-safe contexts but still don't
scale, and are not able anyway to return point-in-time data.

## Implementation strategies

We can't increase the CPU time nor the memory of Redis servers that will not
use such mechanism. Currently it's not clear if the Redis core itself will
implement certain slow operations in terms of this new API (probably yes -- either
with a module that is shipped as part of the server, or natively). However
let's say that if nobody uses slow operations we don't want to pay any speed
or memory penalty.

In order to fullfil this requirement the simplest solution is to have a
dictionary of locked keys. When a client is executing a command, if the size
of the dictionary of the locked keys is empty (a very fast check to perform)
we don't try at all to extract the keys from the command to see if there could
be locked keys. Otherwise we do the following:

* Some client is executing some command.
* The locked keys dictionary is non-zero in size.
* We extract the keys from the command argument vector.
* If there are no keys that are locked, we can continue.
* If there are keys that are locked, the client is blocked before the command is executed at all, and is put into the dictionary of locked keys, in the `blocked_clients` field.
* Once the key is unlocked, or its locking type changes and becomes compatible with what certain clients are trying to do, we can resume the clients in queue.

## Anatomy of the locked keys dictionary

The dictionary, one per every Redis database, has the key name as the dictionary
key. The value is a structure that has many fields:

1. The type of the current lock.
2. The list of clients that are currently using the key for reading.
3. The client that is currently using the key for writing.
4. The queue of clients that are blocked because the key is currently locked.

So inside every Redis database, the redisDB structure we'll have:

    dict *locked_keys;
    dict *locked_keys_fork_queue;
    uint64_t locked_keys_read;  /* How many read locks are in effect? */
    uint64_t locked_keys_write; /* How many write locks are in effect? */
    pthread_mutex_t locked_keys_mutes; /* Protect the above data structures. */

The second dictionary is used when we want to persist, read later.
The first dictionary is the one described above, so its values will be
structures like the following:

    struct lockedKey {
        int lock_type;  /* LOCKED_KEY_READ,
                           LOCKED_KEY_WRITE,
                           LOCKED_KEY_PENDING_FORK */
        list *reading_clients; /* List of clients that locked for read. */
        client *writing_client; /* The client that locked for write. */
        list *queue;            /* Clients blocked since key is locked. */
    };

Please note that we don't have a queue of the clients that are blocked on the
key because the module API (or core API at some point) tried to lock
the key without success, because the API to lock the key is synchronous.
Why this is sufficient is explained later in the API section.

## State required in the client structure

At the same time we also need to remember, in the client structure, the keys
we locked or that we are waiting for is blocked. Potentially we may reuse
some other data structure that the client structure already has for similar
tasks due to other kinds of blocking operations.

## Calling fork() to persist

If there are just client locks currently active, we are free to fork()
because the dataset at the moment of the fork is consistent: there are no
modifications going on in the locked keys, nor there are in the rest of the
data set, since we only perform syncrhonous operations otherwise.

However if we have keys that are in the process of being modified in the
background we are no longer able to fork: the saving child will access keys
locked for writing that may be in some inconsistent state.

So when we need to actually fork and persist on the disk because of AOF
rewriting or RDB snapshotting, no new write locks should be allowed, and
at the same time, we should wait for the count of write locks to drop to
zero: when this happens we can finally fork. To do so we raise a flag
in the global server structure;

    int write_locks_denied; /* Name may vary */

So that if `server.write_locks_denied` is true we can do the following with
clients trying to lock keys for writing: we put them into the queue, like
we do if there is already a client with a read or write lock. However
additionally we put the key into the dictionary declared above:

    dict *locked_keys_fork_queue;

This way once we fork and clear the flag `server.write_locks_denied`, we
can run the list and keys that may have clients pending to be blocked.

In this case the key in the locked dictionary will have the type
`LOCKED_KEY_PENDING_FORK`, since it's not locked for reading nor for
clients, yet the entry exists in the locked dictionary.

## Client structure interaction

When we execute commands with multiple keys, that may be locked at the same
time, such as `MGET A B`, we may end in the situation where the client
is blocked waiting for multiple keys to be unlocked in order to continue the
execution of the command. Since in the client structure we have the list of
keys we are waiting for, we can understand when all the keys are ready since
once the last was unlocked, our list length will drop to zero: now it's time
to execute the command.

When the client is freed while blocked waiting for locked clients, we run the
list of keys the client is blocked for, and can perform the cleanup in the
locked keys queue as well.

However there is a more complex case: the client may invoke a command that
blocks after locking some key, executing some operation in a different thread
while the key is locked. In the meantime the client may get freed for some
reason due to network errors or the execution of the `CLIENT KILL` command
or other similar reasons. When this happens, the blocked client structure that
was generated when the command implementation called `RedisModule_BlockClient()`
is modified in order to continue the execution in a different thread (or
later when a callback is called and so forth), is modified in order to set its
client reference to NULL.

However when the blocked client handle is finally unblocked (even if the real
client it belongs no longer exists), we should be able to do the cleanup
correctly: calls to unlock the keys should work as expected, even if now the
original client locking the keys no longer exist. We could do one of the following
things to solve this problem:

1. At least initially allow to lock the keys only in the context of a blocked client handle. This one must exist for all the time the asynchronous request is completed.
2. Alternatively, when the client is freed, we could replace its references in the locked keys data structures with the one of the fake client that we use in order to accumulate the replies in the blocked client handle:

    typedef struct RedisModuleBlockedClient {
	... more fields ...
	client *reply_client;           /* Fake client used to accumulate replies
					   in thread safe contexts. */
	... more fields ...
    } RedisModuleBlockedClient;

Similarly we could link the lists that we have in the original client about what
keys we are locking there in the `reply_client` field. At the end, when we
detect that the client field is NULL, we know that we should use the `reply_client`
to do the cleanup instead.

## What happens to the key when it's locked for writing

When the key is locked for writing, and hence should never be touched by the
main thread in any way, it is probably a good idea to *force* the fact that the
key is not accesible. This could be done efficiently by setting the pointer
to the object that is stored in the key to a special pointer like 0xFF112233 or
any other pointer that maps to an address likely unmapped in the process address
space, and that will immediately be noticed on crashes.

Moreover lookup functions should explicitly have asserts to check if they are
returning such pointer, and abort the Redis process in such case.

## Modules API

## Optimizations and defensive strategies

TODO: Copy the object on write lock when the object is small?

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

Please note that we don't have a queue of the clients that are blocked on the
key because of the module API (or core API at some point) that tried to lock
the key without success, because the API to lock the key is synchronous.
Why this is sufficient is explained later in the API section.

## Modules API

## Optimizations and defensive strategies

1. Copy instead of write lock when object is small.
2. Use a dummy object instead of NULL.
3. Having a key flag in the new key objects in order to know if the key is locked without having to do an extra lookup, so that we can assert in LookupKey() API.

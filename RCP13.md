RCP 13 - Key locks for modules "no GIL" threaded operations
===

    Author: Salvatore Sanfilippo <antirez@gmail.com>
    Creation date: 2019-02-28
    Update date: 2019-02-28
    Status: open
    Version: 1.0
    Implementation: none

BACKGROUND
==========

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



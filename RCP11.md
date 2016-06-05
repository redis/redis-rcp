RCP 11 - The stream data type
===

    Author: Salvatore Sanfilippo <antirez@gmail.com>
    Creation date: 2016-06-04
    Update date: 2016-06-04
    Status: open
    Version: 1.0
    Implementation: none

History
---

* Version 1.0 (2016-06-04): Initial version.

Rationale
---

During the morning (CEST) of May 20th 2016 I was spending some time in the
`#redis` channel on IRC. At some point Timothy Downs, nickname *forkfork* wrote
the following messages:

    <forkfork> the module I'm planning on doing is to add a transaction log style data type - meaning that a very large number of subscribers can do something like pub sub without a lot of redis memory growth
    <forkfork> subscribers keeping their position in a message queue rather than having redis maintain where each consumer is up to and duplicating messages per subscriber

Now this is not very far from what Apache Kafka provides, from the point of view
of an abstract data structure. At the same time it is somewhat similar to what
Redis lists and Pub/Sub already provide in one way or the other. All this
functionality overlapping in the past prevented me from considering such
a data structure.

However after Timothy message I started to think about it. Such a simple data
structure actually allows users to model problems using Redis that are
currently very hard to model both using lists and Pub/Sub.

* Lists cannot be accessed efficiently in the middle, since the seek complexity is O(N).
* There is no notion of offset in the list, so if old elements are evicted, there is no longer a way for clients to read only new elements, or to rewind to a past known position.
* A log is inherently more compact and memory efficient, since we don't need to account for removal of elements in the middle.
* Pub/Sub has no efficient way to persist an history of messages. There were ideas to implement such a feature, but it always looks far fetched since the whole Pub/Sub mechanism in Redis is designed towards fire-and-forget workloads.
* Pub/Sub has a cost which is related to the number of clients listening for a given channel.
* There is no *consumer group* concept in lists and Pub/Sub, which is a very interesting abstraction in order to have groups of clients that receive different messages, yet another group can receive the same set of messages if needed. If you are not familiar with Kafka, consumer groups are sets of clients sharing the offset of the latest consumed offset, so that all the clients in the same group will receive different messages. Yet each of these clients can independently rewind if they want to consume the same messages again.

The above shortcomings make the existing data structures a problem when trying
to implement *streams of data*. Streams should have the following characteristic:

* Clients should have control on what they want to read, it should be possible to rewind back in time, consumer groups should be implemented.
* Blocking operations should be provided so that a client may block waiting for entires having an offset greater than a given one.
* Each entry in the log should have an unique offset that remains the same if old entries are evicted.
* Memory efficiency should be very good in order to take a big amount of history in memory without problems. Since the data structure should not have a big overhead due to nodes and pointers of other data structures, this should be possible.
* Efficient access of elements in the middle should be possible even with many millions of entires. Let's say that with 100 million entries still to seek in the middle should not be obviously slow.
* It should be possible to gradually evict old entries.
* The log should be efficiently persisted on RDB and AOF files to avoid to be ephemeral like Pub/Sub is.

The above features allow multiple clients to read from the same stream of data
efficiently, without requiring any polling, and with the ability to restart
from a given point after a disconnection.

However certain goals stated above have tensions. For example in order to be able to evict old data, and still have good access time when accessing entries in the middle of a very large log, seems to require a linked data structure that is not memory efficient. We'll explore possible options in the implementation details option.

Also more advanced features should be considered, starting from the experience
of what is not possible with Apache Kafka but what would be useful in Redis
streams: since Redis operates in memory and most of the usages would not require
very strong data retention abilities, we could probably try to do more in order
to maximize what it is possible to do with such a data structure in a system
like Redis.

Logical offsets
---

My first feeling is that, given we are in memory and can do funny things,
offsets of Redis streams should be logical instead of being real offsets
in the log, so that each successive entry in the log is guaranteed to
have an incremental offset.

For example given an offset, the client can easily decide to jump back 10
messages just subtracting 10 to the current offset.

This requirement may be hard to obtain while keeping the memory usage
the smallest possible, but is one I would not easily give away for a bit
of memory efficiency.

Commands introduced
---

    TWRITE key entry
    TWRITE key [BACKLOG <count>] ENTRIES entry [entry entry ... entry]

This command creates a new stream at `key` if it does not already exist, and
adds the specified `entry` at the end of the log. It returns the offset
at which the entry was stored.

The command has two forms: a simple form and a more complex from where
both options and multiple entires can be specified.

Currently only one option is planned:

    BACKLOG <count>

It automatically deletes all the old data so that **at least** `<count>`
items remain in the log. Note that the command may take more than the
specified items in the log, since it is likely that an implementation strategy
for the streams data structure is to have blocks of messages, so we may
evict just on a per-block basis.

    TREAD key <offset> <count> [BLOCK <milliseconds>] [GROUP <keyname>] [WITHINFO]

Read *count* messages starting from *offset*. The command returns an array
of messages:

    ["foo", "bar", ...]

Since the user knows what offset it demanded usually this is enough, however
if the `WITHINFO` option is passed, the first element is an array that
specifies the first offset available in the stream, the last offset, the
offset of the first message in the reply, and the number of replies.

By using a `count` of 0 it is possible to obtain just the informations, in order
to start reading only the new data in the next call, or to fetch the available
history, and so forth.

When the user requests an offset which is already evicted, the missing
messages are reported as NULL entries.

The `BLOCK <ms>` option blocks if there are no messages having an offset
greater or equal the specified offset, and unblocks the client returning
data as soon as messages are available with such an offset. This is useful
in order to read messages and block again until new messages are available.

Given that offsets are planned to be logical, a client may decide, based
on what it is receiving, to ignore the next N messages, and block for
an offset of `old_offset + N`.

The `GROUP` option implements consumer groups, in a similar fashion as Kafka.
When a group is specified, which is actually just a Redis key name, the
specified offset is disregarded. Instead in order to provide messages to the
client the offset to use is read from the group key name. The group key
is updated to the new offset at the same time, so that the next client asking
for messages with the same group will receive new messages and so forth.

Note that `GROUP` option must play well with `BLOCK`: when multiple clients
are blocked with the same group we must guarantee that the one that we
serve first will return as the last one in the queue of clients blocked, so that
we can efficiently route the messages evenly to all the clients that are
participating to the group.

    TEVICT key <offset>

This removes old data from the log. If offset is positive it just removes
entries that are less or equal to the specified offset, however the command
may not evict all the data the user requested if there the offset is
in the currently active block, depending on implementation details.

If a negative number is specified, it means to retain at least the specified
number of messages.

Behavior of clients groups with empty keys
---

One problem to solve with the `GROUP` option is what happens the first time
a new group is used, when the group key still is empty and contains no offsets.
The two obvious solutions would be:

* Initialize it to the offset of the first message in the stream.
* Initialize it to the offset of the last message in the stream, plus 1.
* Provide two `GROUP` options with the different behavior.

Initializing it with the first message offset could be a valid solution since,
if needed, the client could query the stream and set the key value with
a `SET` call when a new group is created, however this is not very
respectful of the Redis *no initializations* API pattern, so potentially
to provide two `GROUP` variants could be a better choice.

Implementation details
---

The data structure used to represent streams should:

* Be memory efficient.
* Allow the eviction of old entries in a very fast way.
* Allow efficient seek in the middle.
* Support logical message offsets.
* Permit the sequential traversal both ways without overhead in order to return messages and in order to efficiently persist the structure.

After evaluating different candidates, my initial proposal is to use a skiplist
with the following special characteristics:

* Double linked skiplist, so that it's possible to traverse elements sequentially in both directions. We already use a similar trick for sorted sets.
* Each skiplist node is actually a **macro node**, composed of 1000 stream messages. This is a fundamental point in order to reduce memory usage, make eviction of old messages very fast since the number of allocations is small even for millions of messages, and to also guarantee very fast seek times, since log(N) will be very small even with very large streams, so that seek will be effectively be constant time (even at 100 million messages we'll use 100k nodes).

Each macro node would be composed of a single blob with all the messages and
a contiguous *offset table* of unsigned 32 bit integers that locate each
message. Something like the following:

    struct macronode {
        uint64_t initial_logical_offset;
        uint32_t offsets_table[1000];
        uint32_t num_messages;
        char *messages;
        struct macronode *previous;
        struct macronode **forward; /* skiplist forward nodes. */
        int skiplist_level;         /* number of forward pointers. */
    };

The use of 32 bit offsets is in order to avoid 64 bit pointers, and at the same time, to make the serialization of the structure into RDB files a simple copy operation (we can keep messages in little endian regardless of the architecture in use, a non-op in most systems). Messages up to 4GB should be enough for all usages.

Note that the message size can always be obtained using the difference between two successive offsets.

One problem with the above implementation is that a stream with just a few
messages has a large memory overhead if we pre allocate the offsets table. However
it is possible to handle this in a special way: if the stream is composed
of a single node, we can reallocate the offsets table at each write, while
starting from the second node we allocate it at creation time in order to
save time.

Nodes compression
---

Because each node has many contiguous messages, compression of inner nodes (all the nodes but the currently *active* one, that can still be target for writes) can be a big win, so each node could also have an access time field. When we traverse the skiplist in order to access data, we could randomly check if a node is a good candidate for compression, and compress it on the fly, to decompress it on accesses. Nodes compression could allow to retain a lot more history for free.

Public design process
--

This Redis feature will be handled via a public design process:

1. This draft is published at https://reddit.com/r/redis as a sticky post.
2. Feedbacks are encouraged into the comments.
3. I'll carefully listen to all the feedbacks and reply. We'll try to find alternatives together to shortcomings in this design.
4. At the end I'll evaluate all the feedbacks, and write a new version of this spec.
5. I'll write a first implementation of the data type for users to test in real world to find new possible issues.



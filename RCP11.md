RCP 11 - The stream data type
===

    Author: Salvatore Sanfilippo <antirez@gmail.com>
    Creation date: 2016-06-04
    Update date: 2017-02-07
    Status: open
    Version: 1.2
    Implementation: none

TODO
---

* Time vs number of messages eviction policies.

History
---

* Version 1.0 (2016-06-04): Initial version.
* Version 1.1 (2016-06-22): Message format change and messages acknowledges.
* Version 1.2 (2017-02-07): Proposal mostly rewritten. Many important changes to make the data structure also a good fit for time series.

Rationale
---

The Stream data structure should provide a way to model time series and other data, with an API that can be used as a vanilla abstract data structure, but also with streaming functionalities.
The streaming functionalities should allow different clients to efficiently read the same stream of messages, and allow clients to start reading again after a disconnection starting from the last
message received. Moreover it should be possible for the clients to inspect past messages if needed.
Since the data structure should also work well for time series, including IOT (Internet Of Things) data collection, sensors and other similar use cases, the messages created into a stream are structured into a set of field-value pairs, and are keyed by an unique identifier that is related to the insertion
time of the entry. The structure should allow to efficiently query for ranges of times as well.

The use cases covered by streams have some overlapping with Lists and Pub/Sub, and even sorted sets in the case of time series.

However these pre-existing primitives in Redis are not efficient at modeling the features exposed above, for the following reasons:

* Lists cannot be accessed efficiently in the middle, since the seek complexity is O(N).
* There is no notion of offset in the list, so if old elements are evicted, there is no longer a way for clients to read only new elements, or to rewind to a past known position.
* A log is inherently more compact and memory efficient, since we don't need to account for removal of elements in the middle.
* Pub/Sub has no efficient way to persist an history of messages. There were ideas to implement such a feature, but it always looks far fetched since the whole Pub/Sub mechanism in Redis is designed towards fire-and-forget workloads.
* Pub/Sub has a cost which is related to the number of clients listening for a given channel.
* There is no *consumer group* concept in lists and Pub/Sub, which is a very interesting abstraction in order to have groups of clients that receive different messages, yet another group can receive the same set of messages if needed. If you are not familiar with Kafka, consumer groups are sets of clients sharing the offset of the latest consumed offset, so that all the clients in the same group will receive different messages. Yet each of these clients can independently rewind if they want to consume the same messages again.
* Sorted sets do not allow to add repeating elements. Scores must be computed client side and do not work well enough as messages IDs. The memory usage is bigger than needed for most use cases where certain sorted sets features (like rank operations) are useless. There is no automatic entries eviction, nor blocking operations are supported.

In comparison, streams should have the following characteristics:

* Clients should have control on what they want to read, it should be possible to rewind back in time, consumer groups should be implemented.
* Blocking operations should be provided so that a client may block waiting for entires having an offset greater than a given one.
* Each entry in the log should have an unique offset that remains the same if old entries are evicted.
* Memory efficiency should be very good in order to take a big amount of history in memory without problems. Since the data structure should not have a big overhead due to nodes and pointers of other data structures, this should be possible.
* Efficient access of elements in the middle should be possible even with many millions of entires. Let's say that with 100 million entries still to seek in the middle should not be obviously slow.
* It should be possible to gradually evict old entries.
* The log should be efficiently persisted on RDB and AOF files to avoid to be ephemeral like Pub/Sub is.

Messages are ordered collections of field-value pairs
---

A message in a Redis Stream is conceptually similar to a Redis Hash, it is
composed of multiple field-value pairs. However such field-value pairs are
in this case ordered, and are usually small both in the number of fields and
items size, while the Hash data type supports easily tens of millions of
elements per key without noticeable performance issues.

This is very clear in the API. In order to add data to the stream we specify
fields and names like in the case of an hash:

    TAPPEND key sensor 01 temperature 35.6
    > 1486475519747.0

Entry IDs
---

Each added entry has an ID that works as a *logical offset* inside the stream,
the format of the entry ID is the time in milliseconds when the entry was added
followed by a dot and a counter which is just an incremental number that marks
entries added in the same millisecond.

    <milliseconds-time>.<counter>

If after a clock skew, an entry is added when the current time is
smaller than the last entry in the stream, the entry is added by reusing
the previous entry time stamp and incrementing just the entry counter. So
entries are guaranteed to monotonically always increase semantically.

However note that we do not imply here that there is such a guarantee
when the system loses data because of a restart, a fail over or other
conditions, entries IDs are just always incrementing from the point
of view of the current content of the stream key.

The internal representation of the offset is a 128 bit number which
is actually stored as two `uint64_t` numbers:

    struct stream_offset_t {
        uint64_t ms;
        uint64_t seq;
    };

So the same milliseconds can account for 2^64-1 entries.

Commands introduced
---

**TAPPEND command**

    TAPPEND key field value ... field value
    > 1486475519748.3

    TAPPEV key (COUNT|TIME) <count> field value ... field value
    > 1486475519747.0

This command creates a new stream at `key` if it does not already exist, and
adds the specified entry at the end of the log. It returns the offset
at which the entry was stored.

The `TAPPEV` command is just an *append and evict* command that can be used
in order to remove older entries so that the total entry count is N or that
only entries having less than the specified number of milliseconds of age
are left (the age is computed according to their offset).

The `TAPPEV` variant may not include any field-value list, in order to
just evict.

**TREAD command**

    TREAD key <last-received-entry-ID> <count> [BLOCK <milliseconds>] [GROUP <name> <ttl>] [RETRY <rerty-ms> <expire-ms>] [WITHINFO]

Read *count* messages starting from the next message after `last-received`. The
command returns an array of messages, where each message contains the ID and the
message itself.

    [["1486475519748.3","key","val", ...],["1486475519747.0","key","val"]]

The following are the options the `TWRITE` command supports:

    WITHINFO

If the `WITHINFO` option is passed, the first element of the reply is an array
that specifies the first and last offset available in the stream.

By using a `COUNT` of 0 it is possible to obtain just the informations, in order
to start reading only the new data in the next call, or to fetch the available
history, and so forth.

When the user requests an offset which does not exist, the missing messages
are reported as NULL entries.

    BLOCK <milliseconds>

The `BLOCK <ms>` option blocks if there are no messages having an offset
greater or equal the specified offset, and unblocks the client returning
data as soon as messages are available with such an offset. This is useful
in order to read messages and block again until new messages are available.

When `BLOCK` is used, the `last-received_entry-ID` can be specified as an
empty string to signal we just want entries starting from the next one that
will arrive, without any history.

    GROUP <name> <ttl>

The `GROUP` option implements consumer groups, in a similar fashion as Kafka.
Instead in order to provide messages to the client the offset to use is read
from the group meta data stored inside the stream structure. The group
is updated to the new offset at the same time, so that the next client asking
for messages with the same group will receive new messages and so forth.

The basic idea of groups is to return a different subset of a stream of messages
to different clients participating to the same group.

The group TTL is used in order to destroy the group when the specified
amount of milliseconds have elapsed without any request in that group.
When a group is used with a different TTL compared to the past one, the
group TTL is set to the new value.

Normally when a `GROUP` option is passed, the `last-received-entry-ID` is set
to the empty string, and is up to the group to decide what entries to return
to the client. In this case, the group will just serve new entries that will
arrive in the stream, without serving any history.

However it is possible for the client to still pass the `last-received-entry-ID`
option when `GROUP` is used: if the group was not known before, such an ID
will be used as the initial offset of the group. Otherwise if the group already
exists the option is ignored. This way if clients in a group want some history
they could use `TREAD` in order to get the stream informations and request
some history.

Note that the `GROUP` option must play well with `BLOCK`: when multiple clients
are blocked with the same group we must guarantee that the one that we
serve first will return as the last one in the queue of clients blocked, so that
we can efficiently route the messages evenly to all the clients that are
participating to the group.

    RETRY <retry-ms> <expire-ms>

If a retry time is specified, and only if also a group name is specified,
the returned messages need to be acknowledged, otherwise they'll be
provided to clients of the specified group after the amounts of milliseconds
specified (or more). Clients must acknowledge messages using the `TACK`
command documented later.

The expire time provided gives a time to live for messages that are not
acknowledged in time. After the expire time elapses, the message is no
longer retained in the list of pending messages and gets destroyed instead.

Basically you can think at messages served with `RETRY` as messages that
are not just returned to the client, but also memorized in a group
specific list of pending messages. The `TACK` command will remove them from
that pending list, otherwise the messages will be re-scheduled for delivery.

Pending messages in groups are both persisted on disk and replicated to
slaves.

**TACK command**

    TACK key groupname id1 id2 ... idN

The TACK command just remove the messages from the pending list.
IDs not in the pending list are ignored.

**TRANGE command**

    TRANGE key start-id end-id [COUNT <count>]

The TRANGE command is able to just fetch entries between two IDs. The
IDs can be just specified as unix times in milliseconds without requiring
the entry count, in order to just make time range queries.

Streams internal representation
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
* Each skiplist node is actually a **macro node**, composed of 100-1000 (actual value configurable) stream messages. This is a fundamental point in order to reduce memory usage, make eviction of old messages very fast since the number of allocations is small even for millions of messages, and to also guarantee very fast seek times, since log(N) will be very small even with very large streams, so that seek will be effectively be constant time (even at 100 million messages we'll use 100k nodes).

Macro nodes will be represented by [Listpacks](https://gist.github.com/antirez/66ffab20190ece8a7485bd9accfbc175).

Public design process
---

This Redis feature will be handled via a public design process:

1. This draft is published at https://reddit.com/r/redis as a sticky post.
2. Feedbacks are encouraged into the comments.
3. I'll carefully listen to all the feedbacks and reply. We'll try to find alternatives together to shortcomings in this design.
4. At the end I'll evaluate all the feedbacks, and write a new version of this spec.
5. I'll write a first implementation of the data type for users to test in real world to find new possible issues.


Triva
---

This proposal originates from an user hint:

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

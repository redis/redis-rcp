Redis Change Proposals.
===

In order to get a big change to Redis discussed, and potentially implemented
and accepted into the core, the new process (starting from December 2014)
is to write a Redis Change Proposal (RCP).

1. Write an RCP following the same schema used in the existing RCPs. When you are ready to publish your RCP, please obtain your RCP number by opening an issue in the redis/redis-rcp repository asking for a new number. The first to ask for a given number wins automatically, no need to get an acknowledge, we'll close those issues only when we'll update the list of RCPs with your number marked as *work in progress* in this README file.
2. Post a pull request here to get your RCP added.
3. Post a link to your RCP with a clear subject in the [redis-dev mailing list](https://groups.google.com/forum/#!forum/redis-dev). 
4. Wait for the RCP to get discussed, and eventually denied or accepted.

Note: we suggest to don't provide a reference implementation if it's a lot of work to write one, unless you already have one.

RCPs possible statuses are:

* Open: discussion in progress.
* Refused: the change was not accepted.
* Accepted: the change was accepted.
* Implemented: the change was accepted and an implementation merged into the `unstable`.
* Production: the change was merged into a stable version of Redis.

List of RCPs
===

1. RCP1: [Multi user AUTH with ACLs.](https://github.com/redis/redis-rcp/blob/master/RCP1.md)
1. RCP3: [Lazy write operations](https://github.com/redis/redis-rcp/blob/master/RCP3.md)
1. RCP4: [Sentinel Flushconfig command](https://github.com/redis/redis-rcp/blob/master/RCP4.md)
1. RCP10: [Additional CONFIG Subcommands](https://github.com/redis/redis-rcp/blob/master/RCP10.md)

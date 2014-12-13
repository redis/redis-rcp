Redis Change Proposals.
===

In order to get a big change to Redis discussed, and potentially implemented
and accepted into the core, the new process (starting from December 2014)
is to write a Redis Change Proposal (RCP).

1. Write an RCP following the same schema used in the existing RCPs.
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

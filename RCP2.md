RCP 1 - Track and Make Available The Frequency of Commands Rejected Due to Lack of Memory
===

```
Author: Bill Anderson <therealbill@me.com>
Creation date: 2015-22-01 
Update date: 2015-22-01 
Status: open
Version: 1.0
Implementation: none
```

History
---

* Version 1.0 (2015-22-01): Initial version.


Proposal 
---

Every time Redis rejects a command due to memory limitations it should store
the timestamp of the event and memory usage in bytes at that time in the
memory log and expose this information similarly to how it makes slowlog
events available using a new command `MEMLOG`.

Additionally the `INFO MEMORY` section should display the current length of
the memory log.


Rationale
---

While knowing current memory usage is important, knowing if and how often Redis
rejects a command due to being at it's configured memory limit is very useful. 
Having clients track this themselves does not provide a broad-based view of
the situation.

By making this available at a server level people responsible for managing the
server, ie. operations crew, can monitor not just how much memory they are
using but if they are experiencing memory limitations in actual use case. This
can be particularly useful if the memory usage is due to client buffers as
opposed to data size.

This change will enable better operational control and understanding of Redis
as it runs and especially during a review of an "event" or issue.

Commands introduced
---

New Command: `MEMLOG`
This command provides a means to view the history of "not enough memory"
rejections of commands. What it does is dependent on the subcommand used with it.

Subcommands: 
`MEMLOG RESET`: Resets the memory event log
`MEMLOG HISTORY [count]`: Returns the last [count] memory events

Example output:
```
		1) 1) (integer) 1405067822
		   2) (integer) 12345678
		2) 1) (integer) 1405067941
		   2) (integer) 12345675
```

`MEMLOG LEN`: Returns the current length of the MEMLOG log.

As with `SLOWLOG` when the log is full the oldest is removed to make room and
the new event added.


Configuration directive
---

A single configuration directive is added It controls how many events to store:
`memlog-max-len <maximum> `. The default is 128, the minimum is 0 which
disabled it. Access to the directive via `CONFIG` is permitted.


INFO Section Addition
---

The Memory section of the `INFO` command should have the following entry
added:
`memlog-length: N` where N is the current length of the memory log.



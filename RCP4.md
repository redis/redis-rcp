RCP 4 - Add "flushconfig" sub-command to Sentinel command
===

```
Author: Bill Anderson <therealbill@me.com>
Creation date: 11/15/05  
Update date: 2015-15-05 
Status: open
Version: 1.0
Implementation: https://github.com/therealbill/redis/tree/sentinel-flushconfig-command
```

History
---

* Version 1.0 (2015-14-05): Initial version.


Proposal 
---
Add a command to Sentinel called "flushconfig" which causes sentinel to write
it's current in-memory config to disk.


Rationale
---

There are cases where you need to cause Sentinel to write it's current config/state to disk. A few examples are:
  * A package manager overwrote the config file but has not restarted the daemon
  * You are needing to audit the config file and want to ensure you have a current copy of the config
  * Someone or some process has wiped out the config file and you need to get it back
  * You are wanting to backup or clone the sentinel config and want a current copy


Commands introduced
---

New Sentinel Subcommand: 
`SENTINEL FLUSHCONFIG`: Triggers a sentinel flush of the config to disk. Returns and OK response.



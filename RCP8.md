RCP 8 - Add aliasing to LUA scripts in SCRIPT LOAD
===

```
Author: Julian Grinblat <julian@dotcore.co.il>
Creation date: 2015/09/22  
Update date: 2015-09-22 
Status: open
Version: 1.0
```

History
---

* Version 1.0 (2015-09-22): Initial version.


Proposal 
---
Basically, it's the discussion that was had in this thread: https://github.com/antirez/redis/issues/2185

Being able to assign an alias to a lua script when loading it, supporting `SCRIPT LOAD "script" "alias"`, and also supporting calling back with `EVALALIAS "alias" #KEYS key1 key2 ... keyN arg1 arg2 ... argN`

Rationale
---

Good libraries like ioredis in Node.JS which I am currently using will abstract these custom commands easily enough, so that is not a problem at all. The actual problem is, if you want to use a lua script that you wrote inside another lua script. For example, in my case, I wrote a simple mdel command which deletes multiple keys based on a pattern given to it. However, which resorting to using a sha, I cannot use this command inside other lua scripts that do other things, making lua scripts much less useful

Commands introduced
---

`SCRIPT LOAD "script" "alias"` - Gives custom alias to a loaded script which can be used later

`EVALALIAS "alias" #KEYS key1 key2 ... keyN arg1 arg2 ... argN` - Executes the command with the given alias as you would when using `EVALSHA`


RCP 4 - Add "defaults" and "diff" sub-commands to config
===

```
Author: Bill Anderson <therealbill@me.com>
Creation date:  2016-01-27
Update date: 2016-01-27
Status: Open
Version: 1.0
Implementation: 
```

History
---

* Version 1.0 (2016-01-27): Initial version.


Proposal 
---
Add two new options to the config command. The first is to enable the user to
pull the default values for all of the configuration directives. The second is
for pulling all directive/value pairs which are *not* the default.


Rationale
---

Often when trying to help a user in the community, or to see what has been
modified when troubleshooting an instance dirctly, it is very useful to see
what has been altered or modified from the defaults. An example of a system
which uses this is Postfix' `postconf` command which has an option to only
display changes from default. A clear and valuable use for this would be in to
limit the data in a config dump used for posting to StackExchange, the mailing
list, Reddit, etc.. 

For the `config defaults` sub-command this is useful for tools which manage or
discover Redis instances. In combination with the `config diff` subcommand this
can be very useful for tracking default changes and values across versions.
This can be useful for users looking to upgrade their Redis version to detect
changes which may affect their setup. This command returns what the compiled in
defaults are - not what the current values are. Combining this with the `config
diff` command would allow administration tools to "reset" a value to the
default for that version of Redis, for example.


Commands introduced
---

New Config Subcommand: 
`CONFIG DIFF`: Returns a list of config directives which are not the defaults.
Other than returning just the changed values it should behave and repsond like
a `CONFIG GET *` would.
`CONFIG GET DEFAULTS`: Returns all the *default* values for all config
directives - regardless of any changes made.



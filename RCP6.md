# RCP 6 - SCAN-bound Operations.

```
Author: Itamar Haber <itamar@redislabs.com>
Creation date: 2015-06-06
Update date: 2015-06-06
Status: open
Version: 1.0
Implementation: none
```

## History
- Version 1.0 (2015-06-06): Initial version.

## Proposal
Extend the `SCAN` family of commands to perform operations on the reply.

## Rationale
Iterating through the keyspace (or the contents of most data structures) is possible with the `SCAN` command (and associated its datatype-specific variants). However, the use of `SCAN` usually translates to the following:
1. Allocating and initializing a local `cursor` variable to 0
2. Executing a `repeat...until cursor == 0` control structure, invoking `SCAN` with each iteration
3. Each bulk reply from `SCAN` is then
  1. Iterated over, possibly executing one or more operations per key name (or datatype element), or
  2. Processed as bulk, possibly invoking multi-key (variadic) commands with it

While some of control structures for the above could be provided by the client library, these patterns are inefficient as they require considerable chatter between the client and the Redis server. Specifically, by delegating step #3 to the server, the potential gains in overall performance ("data gravity") are quite substantial.

While this could theoretically be mitigated to a degree using Lua scripting, `SCAN`'s random nature prevents it from being used in any meaningful way in that context (consider the trivial use case for deleting keys that match a pattern).

## Commands changed
In order to refrain from API bloat, this proposal proposes to extend the existing set of `SCAN` commands with two optional, mutually-exclusive switches - `FORALL` and `FOREACH`. This is proposed despite the modification modifying the commands' replies, since the new replies will be returned only when one of the new switches is used, thus preserving backwards-compatability.

Both new switches require a single string parameter, `operation`. That parameter specifies a Redis operation that will be executed on the reply.

When used with one of the new switches, `SCAN` et al will behave as they do today. However, upon completion, their reply would undergo further processing. When invoked with the `FORALL` switch, a `SCAN`-type command will execute the `operation` provided only once. Using the `FOREACH` switch will cause `operation` to be executed for each element in the reply.

Similar to `SORT...GET`, patterns can be used in the `operation`'s specification to reference contents and properties of the reply. Specifically, when `FOREACH` is used:
- `*` is substituted with the current element
- `@` is substituted with the current element's position in the reply
- `#` is substituted with the total number of elements in the reply

When called with `FORALL`, pattern replacement logic changes as follows:
- `*` is substituted with the entire reply
- `@` is always substituted with 0

### Changes to `SCAN`'s reply
When used with the new switches, `SCAN` commands will accept a new optional switch - `WITHREPLIES`. Omitting that switch will cause `SCAN` to reply in the same way as it does today. Including the `WITHREPLIES` switch with the `FOREACH` switch would cause the return value's second element - the multi-bulk with array of elements to be as follows:
- `SCAN` array of elements, each element an array of two elements - keyname and reply from `opeation`
- `SSCAN` array of elements, each element an array of two elements - set member and reply from `operation`
- `HSCAN` array of elements, each element an array of two elements - field and reply from `operation`
- `ZSCAN` array of elements, each element an array of two elements - member and reply from `operation`

When using `FORALL` with `WITHREPLIES`, the return value will be made up of a multi-bulk with three elements:
1. String representing an unsigned 64-bit number (the cursor) - unchanged
2. Reply from the `operation`
3. Multi-bulk string with array of elements - unchanged

## Executed operation
The operation executed can be virtually any of the supported API commands. That said, it could be wise to put some obvious  safe guards in place:
- `SCAN`s cannot call other `SCAN`s

### Return value handling
Since the operation can be made up of any Redis command, the operation's return value is determined by it. When using the `WITHREPLIES` switch, the user is responsible for correctly handling the reply.

## Usage examples
### Example 1 - deleting keys by pattern

```
SCAN 0 MATCH inactive_user:* FORALL 'DEL *'
```

## Additional open ends and considerations
- When substituting `*` in `FORALL`, a customizable delimiter could come in handy
- Cluster: should be ok as long as hashtag are used properly since `SCAN`s are run on each instance
- Replication: despite `SCAN`'s chaotic nature, since `operation` is just another Redis operation, these can be safely added to the replication stream when data is modified.

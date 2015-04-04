# Distributed Locking/Presence with Consul

Diego defers to Consul/etcd for the distributed systems magic.  We have two basic needs that are continuing to give us trouble.  These are both very different and we should pick the best tool/abstraction for the job.

## Distributed Locking

We have a handful of processes that are actually meant to be singletons: `nsync-bulker`, `tps-watcher`, `converger`, `auctioneer`, and `route-emitter`.

Some terms:

- **worker** will refer to a process that is doing work.
- **slumberer** will refer to a process that is not doing work.  **slumberers** wait to become **workers** in the event of failure.
- *immediately* means "ASAP" or "in under a second"
- *eventually* means within some predefined threshold.  Ideal targets here are 5-10 seconds.

The desired behavior here is:

1. Given a set of processes, at most one may be the **worker**
2. If the **worker** shuts down cleanly (`SIGINT`, `SIGTERM`, `monit stop`, etc..) exactly one **slumberer** should become the next **worker** *immediately*.
3. If the **worker** dies hard (`SIGKILL`, VM explodes, etc...) exactly one **slumberer** should become the next **worker** *eventually*.
4. If a **worker** loses confidence that it is the *only* **worker**, the **worker** should shut down.

#### Where we are today:

1 and 2 are working.  3 is taking ~30s despite TTLs of 10s having been explicitly specified.  Onsi has not reproduced 4 yet.

## Presence

Cells maintain their presence in a centralized store.  This serves two purposes:
1. the converger uses this information to determine whether the Cell is still running.  If it is not, then applications associated with that Cell are rescheduled.
2. the auctioneer uses this information to discover and communicate with the Cell.

The desired behavior here is:

1. When a Cell dies hard (`SIGKILL`, VM explodes, etc...):
    a) the converger should be notified within 5-10 seconds. i.e. the converger should not wait until a convergence loop to act.  If it is notified that a cell has disappeared it should act immediately.
    b) when the convergence loop runs the Cell should not be present
    c) when the auctioneer schedules work it should not attempt to talk to the (now missing) Cell
2. When a Cell is shut down cleanly (`SIGTERM`, `monit stop`, rolling deploy):
    a) the converger should not be notified immediately.  There is no need to trigger a convergence loop.
    b) when the convergence loop runs the Cell should not be present
    c) when the auctioneer schedules work it should not attempt to talk to the (now missing) Cell
    
#### Where we are today:

2b and 2c are working.  2a does not work as specified (a convergence loop is triggered when the cell presence is taken away) -- this is OK if not ideal.

1a, 1b, and 1c are not working at all.  This is a substantial regression and we should add inigo coverage to make sure we have this important edge case covered.

## Current implementation of locks using Consul API

Distributed Locking and Presence are using the same set of APIs. The Consul Lock API is used to acquire and release some piece of data. In the case of a lock, the data is meaningless. For presence, any useful presence information is stored.

At the center of Consul is the concept of a Session. A session some important characteristics including a TTL, LockDelay, Checks, and a Behavior. 
If a session expires, exceeds its TTL, it is invalidated. 
The Behavior describes how data owned by the session is handled when invalidated. It can be released (default) or deleted. 
The LockDelay is used to mark certain locks as unacquirable. When a lock is forcefully released (failing health check, destroyed session, etc), it is subject to the LockDelay impossed by the session. This prevents another session from acquiring the lock for some period of time as a protection against split-brains.
It is not clear how Checks interact with Sessions. They do refer to defined checks but I am not sure if they are on the node or a service, so we will ignore them for now since I do not believe we need to be concerned with them (yet).

The current code creates locks to represent a lock or presence. The lock api will create a session if one is not provided. If a session is created, it is also automatically renewed on an interval of ttl/2. A key/value is then acquired. If already acquired, the lock will keep retrying until it becomes available. Any error will fail the Lock(). A conflict may be returned if you are locking a semaphore. The retry has a fixed delay of 5s. On acquisition, the lock is monitored. If for any reason, this fails, you are notified that the lock is lost. Note that if the connection to the agent times out (which it does), you will think you lost the lock prematurely.

An Unlock() does the reverse. If a session is being automatically renewed, it is stopped. The key/value is released, not deleted.

A Destroy() assumes you have Unlock()ed and remove the key/value if possible.

## Where do we need to go

Its pretty obvious that we need to center around a Session. There should only be one session per process. The ideal solution would allow us to create or re-acquire the session for a process. Our behavior would be to delete rather than release. For presence, a lock delay of 0 makes perfect sense since there is no concern for split brain. We can argue whether we need some lock delay for locks but we can begin with 0. If we require a lock delay for locks, then we will need a separate session for locks.

Locks and presence become nothing more than acquiring and releasing a key/value beyond this point. The focus is all about keeping the session current.

A limitation or possible enhancement to discuss with Hashicorp would be to perform a release&delete which is currently two operations.

## Known issues
[Leader lock re-acquisition](https://github.com/hashicorp/consul/issues/430)

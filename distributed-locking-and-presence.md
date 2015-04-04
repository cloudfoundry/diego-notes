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

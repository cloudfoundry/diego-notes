# Faster Missing Cell Recovery

Today, Cell reps maintain a presence key in etcd.  This key has a TTL of `PRESENCE_TTL` (currently set to 30s).  When a Cell disappears this key eventually expires.  When the Converger next performs its convergence loop (every `CONVERGENCE_INTERVAL` seconds) it notices that there are [ActualLRPs](https://github.com/pivotal-cf-experimental/diego-dev-notes#harmonizing-desiredlrps-with-actual-lrps-converger) and [Tasks](https://github.com/pivotal-cf-experimental/diego-dev-notes#maintaining-consistency-converger) in the BBS that claim to be running on non-existing Cells.  It responds by restaring the ActualLRPs and failing the Tasks.

Today, both `PRESENCE_TTL` and `CONVERGENCE_INTERVAL` are set to 30s.  In the worst case scenario, the time between a Cell going down and the ActualLRPs being restarted can be as high as `PRESENCE_TTL + CONVERGENCE_TTL` -- i.e. 1 minute.  We can do better.

#### Why bother?

- Doing better makes for good demos.
- Doing better is better :stuck_out_tongue_winking_eye:.

## A better way

One option is to reduce `PRESENCE_TTL` and `CONVERGENCE_INTERVAL`.  The fomer is practical, the latter is not:

- Since we *only* heartbeat the presence information into etcd we can probably easily sustain `PRESENCE_TTL` in the neighborhood of `5 seconds`.  This knob may need to be tweaked down as the number of Cells increases, however.
- It is, however, impractical to substantially decrease the `CONVERGENCE_INTERVAL`.  Convergence is a very expensive operation that places heavy strain on etcd.

I propose the following:

- We decrease `PRESENCE_TTL` to 5 seconds.
- We modify the Converger to watch for expiring Cell presences and respond immediately:
    + This would entail adding a new method that is triggered by an expiring Cell presence.
    + The Converger's convergence loop would maintain its responsiblity for identifying missing Cells (in case a watch event is missed)

In this way we can reduce the maximum recovery time from 1 minute to ~5 seconds.

#### But I thought we were moving away from etcd watches?

Yes and no.

We are moving away from etcd watches for three reasons:

- They easily lead to the thundering herd problem (e.g. a Task is complete, all receptors head to etcd to mark the Task as RESOLVING)
- They are generally unperformant under heavy write loads
- They make it harder to move away from etcd

But we are only primarily concerned with not watching for changes in our *data* (DesiredLRPs, ActualLRPs, Tasks).

Watching for *services* (e.g. CellPresence) is fine.  In the event that we move our data out of etcd we are likely to continue to use etcd (or consul, which also supports watch-like semantics) for our services.  Note that moving our data out of etcd also dramatically reduces the write-pressure which makes reducing the `PRESENCE_TTL` to 5 seconds a lot more feasible even in very large deployments (>1000 cells).

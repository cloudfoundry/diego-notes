# Diego Dev Notes

This describes the Diego state machine for running Tasks and LRPs in detail.

This document is not concerned with:

- the Receptor API (documented [here](https://github.com/cloudfoundry-incubator/receptor/blob/master/doc/README.md))
- details around how the Executor runs Actions

## The Rep-Executor(-Garden) Cycle

The Rep is responsible for representing the containers in Garden to BBS.  The Rep gets updates about container status by **polling** the executor and by listening for **events**.  In both cases, the Rep ends up with a container object that it must reconcile with the BBS.  It does this by either:

- Modifying the BBS
- Deleting the container
- Doing nothing

The Tasks and LRPs sections below detail how this reconciliation should occur.

#### Container State

The container object has a `state`.  Here are the possible `states`:

State | Meaning
------|--------
`RESERVED`| The container has been allocated in the Executor's registry but has not been created yet.
`INITIALIZING` | A container is being created in Garden
`CREATED` | A container has been created and the executor is starting to run the Setup/Action/Monitor actions. If there is no Monitor action the container will transition to `RUNNING` almost immediately.  If there is a Monitor action it will only transition when the Monitor says the container is ready.
`RUNNING` | The container is running.
`COMPLETED` | The container is no longer running.  `container.Result` will include details around whether the work the container performed succeeded or failed, or if the container was explicitly stopped.

Allowed transitions:

- `RESERVED -> INITIALIZING`
- `INITIALIZING -> CREATED | COMPLETED`
- `CREATED -> COMPLETED | RUNNING`
- `RUNNING -> COMPLETED`

It is the Rep's responsibility to delete containers.  Containers can be deleted regardless of current state.

### RunResult

When a container is in `COMPLETED` state, information about its result is available in `RunResult`.

* `Failed`: true if container's recipe completed with failure
* `FailureReason`: if `Failed` is true this contains a message describing the failure
* `Stopped`: true if container was explicitly stopped. If true, `Failed` and `FailureReason` will not be set.


#### Container Events

The executor emits the following events:

- `ContainerReservedEvent`: triggered when the container is initially reserved
- `ContainerRunningEvent`: triggered when the container enters the running state
- `ContainerCompletedEvent`: triggered when the container enters the completed state

The rep should handle events exactly as it handles polling.  Given a container in a particular state, reconcile with the BBS.  The events are simply an optimization on top of polling.

#### Polling vs Events

The Rep keeps the BBS up-to-date via two mechanisms: polling and events.  This is complicated by the fact that events can come in while the poller is *fetching* state from its two sources (BBS & Executor).

As a result, when polling the Rep may have a stale copy of the containers and/or a stale copy of the BBS.  To be safe it should always validate a decision it is about to make.  This is *especially* important when deleting an `INITIALIZING/CREATED/RUNNING` container.

## LRPs

All things LRP.  Here's an outline:

- **DesiredLRPs** discusses the DesiredLRP side of things.
- **ActualLRPs** discusses the various states that ActualLRPs can be in.
- **LRP Lifecycle** An overview
- **Computing ∆s** discusses the rules used to determine how Desired and Actual ought to be reconciled.  The Receptor and Converger do this.
- **Starting/Stopping ActualLRPs** discusses how start and stop requests are handled.  The Receptor and Converger can request stops.  The Receptor, Converger, and Rep can request starts.
- **Distributing ActualLRPs** discusses how the Auctioneer distributes LRPs.
- **Crashes** discusses the lifecycle associated with a crashing ActualLRP.  The Rep and Converger play a role here.
- **Evacuation** discusses how a Rep evacuates.  Only the Rep plays a role here.
- **Harmonizing DesiredLRPs with Actual LRPs: Converger** discusses the Converger's responsibilities.
- **Harmonizing ActualLRPs with Container State: Rep** outlines (exhaustively) how the Rep keeps the ActualLRPs in the BBS and the set of running containers on its Cell in sync.

### DesiredLRPs

- `DesiredLRP`s can be created and deleted freely.
- `DesiredLRP`s can be updated freely, but only a subset of fields may be updated (`instances`, `routes`, `annotation`).  This subset corresponds to changes that can be accommodated *without* restarting any ActualLRPs
- **Only** consumers can modify `DesiredLRP`s.  Therefore, only the Receptor is allowed to manipulate `DesiredLRP`.
- `DesiredLRP`s are organized into domains - arbitrary groupings for the consumer's convenience.  Domains have a notion of "freshness" meaning that the Desired state is up-to-date.  The Converger avoids taking incorrect destructive actions by only shutting down undesired ActualLRPs if the Desired state is "fresh" (i.e. up-to-date).

### ActualLRPs

Diego attempts to keep one ActualLRP running per desired index.  To ensure this we use the BBS to walk an individual ActualLRP through a state machine.

The BBS stores the ActualLRP under `/v1/actual/:process_guid/:index/instance`

When it is evacuating, the BBS stores the ActualLRP under `/v1/actual/:process_guid/:index/evacuating`.  Only during evacuation is it possible/legal for (at most) 2 ActualLRPs to run at a given index.

Indices are in the range `0..N-1` where `N` is the number of desired instances specified in the DesiredLRP.

Here is the ActualLRP state machine:

State | Meaning
------|--------
`UNCLAIMED`| The ActualLRP is being scheduled by an auction
`CLAIMED` | The ActualLRP has been assigned to a Cell and is being started
`RUNNING` | The ActualLRP is runing on a Cell and is ready to receive traffic/work.
`CRASHED` | The ActualLRP has crashed and is no longer on a Cell.  It should be restarted (eventually).

In addition, the ActualLRP includes two pieces of data: `CrashCount` (keeping track of the number of crashes) and `Since` (keeping track of the time when `State` was last updated). These are used to implement a backoff policy when restarting crashes.

An evacuating ActualLRP will always be in the `RUNNING` state.

### LRP Lifecycle

Here is a happy path overview of the LRP lifecycle, just to set the stage.  Many more details will follow.

- A DesiredLRP is created/modified by an incoming request to the Receptor
- The Receptor compares the DesiredLRP to the Actual state and computes a ∆
- Starts are requested and sent to the Auctioneer
	- The Rep starts the ActualLRP and updates its state in the BBS
- Stops are sent directly to the Reps
	- The Rep stops the ActualLRP and updates its state in the BBS

### Computing ∆s

Both the Converger and the Receptor need to compute ∆s between Desired and Actual state.  The Receptor does this when a new/updated DesiredLRP comes in.  The Converger does this when convering LRPs.  Here's how ∆s are computed:

Given a DesiredLRP that requests `N` instances.

And a set of ActualLRPs (living under `/v1/actual/:process_guid/*/instance`) associated with that DesiredLRP.

- If an ActualLRP has index > `N-1`: Stop the ActualLRP (see below)
- If there is no ActualLRP for some index `< N`: Start the ActualLRP (see below)

Note: the *state* of the ActualLRP is irrelevant.

### Starting/Stopping ActualLRPs

The following actions can be taken to start/stop an ActualLRP.

#### Starting ActualLRPs

When an ActualLRP needs to be started or restarted (in the case of crashes/evacuations):

- if starting a missing/new ActualLRP:
	- an UNCLAIMED ActualLRP is CREATED *(if this fails: do nothing, there's already an ActualLRP there which means Diego is on it)*
- if restarting a crashed ActualLRP:
	- CAS from CRASHED to UNCLAIMED *(if this fails: do nothing, it means the ActualLRP is already being restarted)*
- in both cases: upon success, a start is sent to the Auctioneer
- the Auctioneer picks a Rep *(if this fails: the Auctioneer retries)*
- the Rep is told to start the ActualLRP
- the Rep creates a container reservation *(if this fails: the Auctioneer is told and placement is retried)**
- upon success, the Rep responds to the Auctioneer successfully (this ensure the Auctioneer isn't kept waiting)
- the Rep then CAS the ActualLRP from `UNCLAIMED` to `CLAIMED`  *(upon failure: the Rep aborts and cleans up the reservation and we rely on the converger to try again)*
- upon success, the Rep then starts running the container *(upon failure: the Rep aborts, cleans up the reservation, and deletes the ActualLRP.  We rely on the converger to try again)*
- eventually a `ContainerRunningEvent` causes the Rep to CAS from CLAIMED to RUNNING

#### Stopping ActualLRPs

When an ActualLRP should be stopped:

- for `CLAIMED` or `RUNNING` ActualLRPs:
	- a stop is sent to the corresponding Rep.
	- the Rep deletes the container *(if this fails: the rep aborts - the converger will pick this up later)*
	- the Rep then removes the ActualLRP from the BBS *(if this fails: the rep aborts - the rep will retry later in its polling loop)*
- for `UNCLAIMED` or `CRASHED` ActualLRPs:
 	- compare-and-delete the ActualLRP *(if this fails: the converger will pick it up later)*

### Distributing ActualLRPs: Auctioneer

The Auctioneer is responsible for distributing ActualLRPs optimally and efficiently.  There is only one Auctioneer ever handling requests.  Requests are handled on demand: when a request to start work (an ActualLRP or Task) arrives the Auctioneer immediately fetches state from all Cells, picks the optimal placement, and then tells the corresponding Cell to perform its assigned work.

Doing this for each individual piece of work would incur a large communication overhead.  As such the entire flow of information around starting Tasks and LRPs supports batching.  There are two notions of "batch":

1. When a component knows it must request many starts it submits a batch request ot the Auctioneer.  For example, if the Receptor needs to scale an application to 100 instances it submits a single batched request to the Auctioneer.  Similarly if the Converger needs to start a number of missing LRPs, it sends a single batch request to the Auctioneer.  The Auctioneer in turn submits work to Reps in batches.
2. Work sent to the Auctioneer is added to the next batch of work.  The entire batch is scheduled on the next scheduling loop.  Batches are heterogenous and include both ActualLRPs and Tasks.  Batches are deduped before being scheduled -- so identical units of work in a single batch will not be scheduled multiple times.

Here are some details around the scheduling loop.  When a new ActualLRP arrives:

- the Auctioneer fetches the state of each Cell.  This includes information about the available capacity on the Cell, the Cell's stack, and the set of ActualLRPs currently running on the Cell.
- the Auctioneer then distributes the ActualLRPs across the Cells (in-memory).  It ensures that ActualLRPs are only placed on Cells with matching `stack`s that have sufficient resources to host the ActualLRP.  Once the Auctioneer identifies the set of Cells matching this criteria it choses the winning Cell by optimizing for (in increasing priority):
	- an even distribution of memory, disk, and container usage across Cells (all with equal weighting)
	- minimizing colocation of ActualLRP instances of a given DesiredLRP on the same Cell (takes precedence over the distribution of memory/disk/etc.).
	- minimizing colocation of ActualLRP instances of a given DesiredLRP in the same Availability Zone
- the Auctioneer then submits the allocated work to all Cells
	- any work that could not be allocated is carried over into the next batch
	- if the Cell responds saying that the work could not be performed, the auctioneer carries the failed work over into the next batch
	- if the Cell *fails to respond* (e.g. a connection timeout elapses), the auctioneer *does **not*** carry the work over into the next batch.  This is a case of partial failure and the auctioneer defers to the Converger to figure out what to do.
- any work carried over into the next batch is merged in with work that came in during the previous round of scheduling and the auction repeats the scheduling loop

#### Auction Work Prioritization

When the Auctioneer receives a batch of work it must decide the order in which the work must be distributed.  This is important.  When scheduling heterogenous work the Auctioneer could, incorrectly, fill the Cells with small units of work before attempting to place large units of work.  As a result, these large units may fail to place even if - in principal - there is sufficient capacity in the cluster. However, naively prioritizing large units of work over small units of work could lead to a situation in which, for example, a single large application fills the cells and prevents a smaller application from having *any* instances running.

To mitigate these issue the Auctioneer first sorts the batch of work it is operating on in the following order (high priority to low priority):

1. Index 0 LRPs
2. Tasks
3. Index 1 LRPs
4. Index 2 LRPs
5. etc...

Within each priority group, work is sorted in order of decreasing memory (so larger units of work are placed before smaller units of work).

It is important to note that this prioritization only applies within an Auctioneer's single batch of work.  Since applications can arrive in arbitrary order the effectiveness of this scheme is somewhat limited.  However, since communication around starts is generally distributed in batches - this prioritization will apply when (for example) a cell is evacuating or when the converger is submitting a batch of work.

#### Communicating Fullness

When an ActualLRP cannot be placed because there are no resources to place it, the Auctioneer leaves the `ActualLRP` in the `UNCLAIMED` state and sets the `PlacementError` field.  Whenever the `ActualLRP` transitions out of the `UNCLAIMED` state the `PlacementError` should be cleared.

There are multiple reasons why an `ActualLRP` may fail to be placed; these should be communicated to the user.  For example, in the presence of placement pool rules it is possible that the Auctioneer simply cannot find appropriate host Cells (`PlacementError="found no compatible cells"`).  Alternatively, the Auctioneer may *find* appropriate Cells but all those Cells might be full (`PlacementError="insufficient resources"`).

Diego continues to attempt to schedule `UNCLAIMED` `ActualLRP`s.  Should an operator add spare capacity, Diego will automatically schedule the `ActualLRP`s.

### Crashes

When an ActualLRP instance crashes, Diego is responsible for restarting the instance.  There is a tension, however, between restarting a failed instance immediately and overloading the system with endless immediate restarts of instances that never succeed.

To strike this balance Diego immediately restarts a crashed instance up to 3 times.  We perform immediate restarts for a few reasons:
- they make for good demos (look it crashed -- look it's back!)
- they handle cases where a healthy well-written application has truly crashed and shold be restarted ASAP.

Sometimes, however, an instance crashes because some external dependency is down. In these cases it makes more sense for Diego to wait (and alleviate the strain of a thrashing instance hopping from one Cell to another) between restarts.  So for crashes 3 - 8 Diego applies an exponential backoff in the waittime between restarts.

Finally, we've learned from existing large public installations (PWS, Bluemix) that the vast majority of crashed instances in the system are poorly-written instances that never even manage to come up.  Instead of endlessly restarting them (which puts a substantial strain on the system) Diego will (eventually) give up on these instances.

All that remains is the question of how we reset the exponential backoff.  Instances that thrash and place a heavy load on the system typically crash quickly.  So we apply a simple heuristic: if an instance manages to stay running for > 5 minutes (defined below), we reset its crash count.

When the container associated with an ActualLRP enters the `COMPLETED` state the Rep takes actions to ensure the ActualLRP gets restarted.  Let's call this the `RepCrashDance`:

- If `CrashCount` < 3:
	-  Increment the `CrashCount` and bump `Since`,  CAS the ActualLRP to `UNCLAIMED`, and emit a start to the Auctioneer.
- If `CrashCount` > 3:
	- Increment the `CrashCount` and bump `Since`.  CAS the ActualLRP to `CRASHED`.

The `CRASHED` ActualLRP is eventually restarted by the converger.  The `WaitTime` is computed like so:

- If `CrashCount < 8`
	- the Converger should restart the ActualLRP `N` seconds after `Since` (exponential backoff in [this story](https://www.pivotaltracker.com/story/show/83638710))
- If `CrashCount >= 8`
	- the Converger should restart the ActualLRP `MaxWaitTime = 16` minutes after `Since`
- If `CrashCount > 200`
	- the ActualLRP is never restarted

It is important that the `CrashCount` be reset eventually.  The Rep does this when marking an ActualLRP as crashed:

- If, at the time the crash occurs, the ActualLRP in the BBS is in the `RUNNING` state with a `Since` time that is `>= 5 minutes` ago:
	- Reset the `CrashCount` to 0 and the then do the `RepCrashDance`
- Otherwise
	- Do the `RepCrashDance`


> Note: the Rep is responsbile for immediately restarting `ActualLRP`s that have `CrashCount < 3`.  These `ActualLRP`s never enter the `CRASHED` state.  The Converger is responsible for restarting `ActualLRP`s that are in the `CRASHED` state.

### Evacuation

When a Cell must be evacuated its ActualLRPs must first transfer to another Cell.  Here's how this works.

#### The Rep's Role during Evacuation

After the rep is signaled to evacuate, it communicates the evacuating state to the rest of its components and periodically attempts to resolve its containers. For the details of all the resolution cases, consult the ["Harmonizing during evacuation"](#harmonizing-during-evacuation) section below. The rep shuts down once all of its containers have been destroyed or it reaches its evacuation timeout.


#### The Receptor's Role during Evacuation

When queried for ActualLRPs the receptor follows the following rules:

- If there is no `evacuating` instance, always return the `instance` instance.
- If there *is* an `evacuating` instance, return it if:
	+ There is no `instance` instance
	+ The `instance` instance is `UNCLAIMED` or `CLAIMED` (to be clear: if `instance` is `RUNNING` or `CRASHED`, return `instance` instead of `evacuating`)

#### The Converger's Role during Evacuation

None.  As far as the Converger is concerned `/evacuating` need not exist.

### Harmonizing DesiredLRPs with Actual LRPs: Converger

Messages get lost.  Connections time out.  Components fail.  Bugs happen.

The converger is responsible for bringing Diego into eventual consistency in the face of these eventualities.  The converger only operates on BBS data and strives to bring the actual state (the set of ActualLRPs) into consistency with the desired state (DesiredLRPs).

Here are it's responsibilities and the actions it takes:

1. Reaping ActualLRPs on failed Cells
	- Cells periodically maintain their presence in the BBS.  If a Cell disappears the converger will notice and CAS `CLAIMED` and `RUNNING` ActualLRPs associated with the missing Cell to `UNCLAIMED`.  The converger will also emit starts for these missing ActualLRPs.
	- In addition to polling periodically, the Converger actively watches for Cells disappearing.  When a Cell goes away the convergence loop is triggered immediately.  This was covered in #11.
2. Starting missing ActualLRPs
	- If there is no ActualLRP in the BBS for a particular DesiredLRP index, the Converger CREATEs an `UNCLAIMED` ActualLRP and requests a start.
3. Stopping extra ActualLRPs
	- If there is an ActualLRP (in any state) that does not correspond to a DesiredLRP index, the Converger stops the ActualLRP (see above)
	- This action is *only* taken if ActualLRP's Domain must be fresh.
4. Restarting `CRASHED` ActualLRPs
	- It is the converger's responsibilities to restart `CRASHED` ActualLRPs (see above)  
5. Re-emitting start requests
	- The request to start the ActualLRP may have failed.  This can be detected if the ActualLRP remains in the `UNCLAIMED` state for longer than some timescale.  The Converger acts by resubmitting a start request.

### Harmonizing ActualLRPs with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.  It does this by periodically fetching containers from the executor and taking actions.  Here is an outline of how the Rep should act to reconcile the ActualLRPs in the BBS with the set of containers.

When performing reconciliation the Rep compares the container associated with a `ProcessGuid` and `Index` with the entry in the BBS for that `ProcessGuid` and `Index`.  It then reconciles the state of the container (`RESERVED`, `INITIALIZING`, `CREATED`, `RUNNING`, `COMPLETED`) with the state of the `ActualLRP` (`UNCLAIMED, CLAIMED, RUNNING, CRASHED`).  In addition it must reconcile the `InstanceGuid` and `CellID` associated with the container and the BBS (this covers cases where (e.g.) the ActualLRP at a given index ends up running on two different Cells, or (e.g.) a single Cell has multiple ActualLRPs running on it).  

In the following table we describe the actions the Rep must take to bring the BBS and the containers on a given Cell into harmony.  This table is from the vantage point of a Receptor running on `CellID=A` with a container with `InstanceGuid=A` -- we'll call the `(InstanceGuid=A,CellID=A)` pair `α`.  It's possible that the Rep will see some other pair of identifiers in the BBS.  We call such a pair `ω`: this could be any other pairing including `(InstanceGuid B, Cell B)`, `(InstanceGuid B, Cell A)` or `(InstanceGuid A, Cell B)` -- the same reconciliation logic applies to either case.

Container State | ActualLRP State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | No ActualLRP | Delete Container | **Conceivable**: This ActualLRP is no longer desired (maybe because the DesiredLRP was scaled down or deleted)
`RESERVED` | `UNCLAIMED` | CAS to `CLAIMED α` **THEN** Run Container | **Expected**: α has a reservation for this LRP and should claim it and then run it in the container.
`RESERVED` | `CLAIMED α` | Run Container | **Conceivable**: We already claimed it and should make sure we've started running the container
`RESERVED` | `CLAIMED ω` | Delete Container | **Conceivable**: Don't run this container, since a different rep got farther than us
`RESERVED` | `RUNNING α` | CAS to `CLAIMED α` **THEN** Run Container | **Inconceivable**: We shouldn't have been able to get to this state, but if we have correct the BBS and run anyway
`RESERVED` | `RUNNING ω` | Delete Container | **Conceivable**: Don't run this container, since a different rep got farther than us
`RESERVED` | `CRASHED` | Delete Container | **Conceivable**: Don't run this container
`INITIALIZING/CREATED` | No ActualLRP | Delete Container | **Conceivable**: This ActualLRP is no longer desired (maybe because the DesiredLRP was scaled down or deleted)
`INITIALIZING/CREATED` | `UNCLAIMED` | CAS to `CLAIMED α` | **Inconceivable**: We should have claimed the LRP before running it, but correct the BBS if this happens
`INITIALIZING/CREATED` | `CLAIMED α` | Do Nothing | **Expected**: The Cell is starting this ActualLRP, no need to write to the BBS
`INITIALIZING/CREATED` | `CLAIMED ω` | Delete Container | **Conceivable**: The ActualLRP is starting as ω (probably on some other Cell), stop starting it on this Cell.
`INITIALIZING/CREATED` | `RUNNING α` | CAS to `CLAIMED α` | **Inconceivable**: This should not be possible, but the BBS should be made to reflect the truth
`INITIALIZING/CREATED` | `RUNNING ω` | Delete Container | **Conceivable**: The ActualLRP is running elsewhere, stop starting it on this Cell
`INITIALIZING/CREATED` | `CRASHED` | Delete Container | **Conceivable**: This Cell is incorrectly starting the instance - some other Cell will pick this up later.
`RUNNING` | No ActualLRP | CREATE `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, let Diego know so it can take action appropriately (we don't allow blank ActualLRPs to shut down containers as this would lead to catastrophic fail should the BBS be accidentally purged).
`RUNNING` | `UNCLAIMED` | CAS to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, no need to start it elsewhere
`RUNNING` | `CLAIMED α` | CAS to `RUNNING α` | **Expected**: This Cell is running the ActualLRP
`RUNNING` | `CLAIMED ω` | CAS to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, no need to start it elsewhere
`RUNNING` | `RUNNING α` | Do Nothing | **Expected**: This Cell is running the ActualLRP, no need to write to the BBS
`RUNNING` | `RUNNING ω` | Delete Container | **Conceivable**: The ActualLRP is running elsewhere, stop running it on this Cell
`RUNNING` | `CRASHED` | CAS to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP. It's not crashed and need not be restarted.
`COMPLETED (crashed)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | This Cell just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (crashed)` | `UNCLAIMED` | Delete Container | Instance will be scheduled elsewhere
`COMPLETED (crashed)` | `CLAIMED α` | Perform `RepCrashDance` then Delete Container | Instance crashed on this Cell while starting
`COMPLETED (crashed)` | `CLAIMED ω` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (crashed)` | `RUNNING α` | Perform `RepCrashDance` (see Crash section above for details around resetting the CrashCount) then Delete Container | Instance crashed on this Cell while running
`COMPLETED (crashed)` | `RUNNING ω` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (crashed)` | `CRASHED` | Delete Container | The crash has already been noted
`COMPLETED (shutdown)` | No ActualLRP | Delete Container | **Conceivable**: Nothing to be done, this Cell was asked to shut the ActualLRP down.
`COMPLETED (shutdown)` | `UNCLAIMED` | Delete Container | **Conceivable**: Nothing to be done
`COMPLETED (shutdown)` | `CLAIMED α` | CAD ActualLRP then Delete Container | **Conceivable**: This Cell was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `CLAIMED ω` | Delete Container | **Conceivable**: The Instance is starting elsewhere, leave it be
`COMPLETED (shutdown)` | `RUNNING α` | CAD ActualLRP then Delete Container | **Expected**: This Cell was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `RUNNING ω` | Delete Container | **Conceivable**: Instance is running elsewhere, leave it be
`COMPLETED (shutdown)` | `CRASHED` | Delete Container | **Conceivable**: Nothing to do
No Container | `CLAIMED α` | CAD ActualLRP | **Conceivable**: There is no matching container, delete the ActualLRP and allow the converger to determine whether it is still desired
No Container | `RUNNING α` | CAD ActualLRP | **Conceivable**: There is no matching container, delete the ActualLRP and allow the converger to determine whether it is still desired

Some notes:

- `COMPLETED` comes in two flavors.  `crashed` implies the container died unexpectedly.  `shutdown` implies the container was asked to shut down (e.g. the Rep was sent a stop). This can be determined by looking at `RunResult.Stopped`.
- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell.  Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible.  However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence.  An exhaustive analysis of all possible combinations (such as this) ensures eventual consistency... eventually.
- When the Action described in this table fails, the Rep should log and do nothing.  In this way we defer to the next polling cycle to retry actions.

Alternate view of table above:

BBS | Reserved | I/C | Running | Shutdown | Crashed | No Container
---|---|---|---|---|---|---
Missing | Do nothing | Delete Container | Create Running | Delete Container | RCD + Delete Container |
Unclaimed | Do Nothing | CAS Claimed | CAS Running | Delete Container | Delete Container |
Claimed-α | Do Nothing | Do nothing | CAS Running | CAD + Delete Container | RCD + Delete Container | CAD
Claimed-ω | Do Nothing | Delete Container | CAS Running | Delete Container | Delete Container |
Running-α | Do Nothing | CAS Claimed | Do Nothing | CAD + Delete Container | RCD + Delete Container | CAD
Running-ω | Do Nothing | Delete Container | Delete Container | Delete Container | Delete Container |
Crashed | Do Nothing | Delete Container | CAS Running | Delete Container | Delete Container |

```
I/C = Initializing/Created
CAS = Compare and Swap
CAD = Compare and Delete
RCD = RepCrashDance
```

#### Harmonizing during evacuation

- the Rep is told to evacuate.
- the Rep subsequently refuses to take on any new work:
	+ when the auctioneer requests the Rep's State, the Rep informs the Auctioneer that it is evacuating which takes it out of the pool
	+ if any work comes in from the auctioneer, the rep immediately returns it all as failed work
- the Rep then periodically attempts to resolve its remaining containers according to their container states:


Assuming a `RUNNING` container on α, the α-rep performs these actions, always ensuring that keys under /evacuating have their TTL set to the evacuation timeout. `β` and `ω` represent other cells in the cluster. `ε` indicates that the UNCLAIMED ActualLRP has a placement error set.

Instance key state | Evacuating key state | Action | Reason
---|---|---|---
`UNCLAIMED` | - | CREATE /e: `RUNNING-α` | **Inconceivable?**: Ensure routing to our running instance
`UNCLAIMED+ε` | - | Delete Container | **Inconceivable?**: No one won the auction, so this instance will not evacuate successfully. Accept the inevitable.
`UNCLAIMED` | `RUNNING-α` | Do Nothing | **Expected**: Waiting for other rep to win auction
`UNCLAIMED+ε` | `RUNNING-α` | CAD /e && Delete Container | **Conceivable**: No one won the auction, so this instance will not evacuate successfully. Accept the inevitable.
`UNCLAIMED` | `RUNNING-β` | Delete container | **Conceivable**: β ran our evacuated instance, then evacuated itself
`CLAIMED-α` | - | CREATE /e: `RUNNING α`, CAS /i: `UNCLAIMED` | **Conceivable**: α has a RUNNING container but didn't get to update the BBS to `RUNNING` yet
`CLAIMED-α` | `RUNNING-α` | CAS /i: `UNCLAIMED` | **Conceivable**: α failed to update the BBS to UNCLAIMED while evacuating
`CLAIMED-α` | `RUNNING-β` | CAS /e: `RUNNING α`, CAS /i: `UNCLAIMED` | **Conceivable**: β evacuated the container, α CLAIMED it, ran it and began evacuating but hasn't yet updated the BBS to `RUNNING`
`CLAIMED-ω` | - | CREATE /e: `RUNNING α` | **Inconceivable?**: Ensure routing to our running instance
`CLAIMED-ω` | `RUNNING-α` | Do Nothing | **Expected**: Waiting for ω to start running instance
`CLAIMED-ω` | `RUNNING-β` | Delete container | **Conceivable**: β ran our evacuated instance, then evacuated itself
`RUNNING-α` | - | CREATE /e: `RUNNING α`, CAS /i: `UNCLAIMED` | **Expected**: This is the initial action during evacuation
`RUNNING-α` | `RUNNING-α` | CAS /i: `UNCLAIMED` | **Conceivable**: α failed to update the BBS to UNCLAIMED while evacuating
`RUNNING-α` | `RUNNING-β` | CAS /e: `RUNNING α`, CAS /i: `UNCLAIMED` | **Conceivable**: β evacuated the container, α CLAIMED it, ran it, and then began evacuating
`RUNNING-ω` | - | Delete container | **Conceivable**: The actualLRP is now running elsewhere but the /e was somehow lost
`RUNNING-ω` | `RUNNING-α` | CAD /e && Delete container | **Expected**: Cleanup after successful evacuation
`RUNNING-ω` | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, ω CLAIMED it, ran it, and then began evacuating, and then α noticed
`CRASHED` | - | Delete container | **Conceivable**: The actualLRP is now running elsewhere but the /e was somehow lost
`CRASHED` | `RUNNING-α` | CAD /e && Delete container | **Expected**: Cleanup after successful evacuation (but then the new instance crashed)
`CRASHED` | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, some rep CLAIMED it, ran it, `CRASHED` it, and then α noticed
- | - | Delete container | **Conceivable**: The actualLRP is now running elsewhere but the /e was somehow lost
- | `RUNNING-α` | CAD /e && Delete container | **Expected**: Cleanup after scaling down during evacuation
- | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, the actualLRP was scaled down, and then α noticed

When the /instance ActualLRP changes to the `UNCLAIMED` state, the BBS will also request a new auction for it.

- In container states that are not `RUNNING` or `COMPLETED`, the rep destroys the container, CADs the /evacuating ActualLRP if is `RUNNING-α`, and request a new auction for the ActualLRP by CASing /instance to `UNCLAIMED`
- In a `COMPLETED (SHUTDOWN)` container state, the rep destroys the container and CADs the /evacuating ActualLRP if is `RUNNING-α`, and CADs the /instance ActualLRP as usual.
- In a `COMPLETED (CRASHED)` container state, the rep destroys the container and CADs the /evacuating ActualLRP if is `RUNNING-α`, and performs the RepCrashDance on the /instance ActualLRP as usual.

- the Rep shuts down when either all containers have been destroyed OR an evacuation timeout is exceeded:
	+ in either case, the Rep ensures that any ActualLRPs associated with it are removed from `/evacuating` and that any tasks it still has running transition to `COMPLETED`.


## Tasks

All things Tasks.  Here's an outline:

### Task States

The lifecycle of a Task in Diego is quite different from that of an LRP.  Tasks are guaranteed to run at most once: they're never restarted, they don't "crash", they don't move during an evacuation.  Instead Tasks `COMPLETE`.  Once in the `COMPLETE` state the Tasks have a notion of whether they have failed or not.  Failed Tasks include a `FailureReason`; succesful Tasks (may) include a `Result` (the contents of a file requested by the user).

Here are the states for the Tasks:

State | Meaning
------|--------
PENDING | The Task is being scheduled by the Auctioneer
RUNNING | The Task is running on a Cell
COMPLETED | The Task is complete
RESOLVING | The Task is being resolved by a Receptor

### Task Lifecycle

Here's a happy path overview of the Task lifecycle:

- When a new Task is created by the Receptor it:
	+ Creates a Task in the `PENDING` state *(if this fails: the receptor returns an unhappy status code)*
	+ Sends a start request to the Auctioneer *(if this fails: the receptor does nothing; the converger will eventually resend the message)*
- The Auctioneer picks a Cell to run the Task *(if this fails: the auctioneer marks the Task completed and failed)*
- The Rep creates a container reservation (and then ACKs the Auctioneer's request) *(if this fails: the Rep tells the Auctioneer that the Task could not be started and the auctioneer adds the Task to the next batch)*
- In its processing loop, the Rep CAS the Task from `PENDING` to `RUNNING` *(if this fails: the Rep deletes the reservation -- the converger will eventually see the `PENDING` task and ask the auctioneer to try again)*
- Upon success the Rep starts the container running *(if this fails: the Rep marks the Task `COMPLETED` and failed)*
- When the container completes (succesfully or otherwise) the Rep CAS the Task to `COMPLETED` *(if the CAS fails, the Rep will try again in its polling cycle)*
	+ If the Task has a `CompletionCallback` URL this sends a message to the Receptor to handle resolving the Task (BBS takes care of this)
- The Receptor CAS the Task to `RESOLVING` and calls the `CompletionCallback` *(if the CAS fails it does not call the `CompletionCallback`)*
- Upon success Receptor Rep CAD the Task *(if the CAD fails the Receptor does nothing: eventually the converger will demote the Task to `COMPLETE` and we'll try again)*

### Distributing Tasks: Auctioneer

Tasks are distributed much like ActualLRPs.  In fact the batch of work performed by the Auctioneer includes both Tasks and ActualLRPs.  The only differences are around how Tasks are optimally placed (unsurprisingly, they are simpler).  For Tasks the Auctioneer only:

- ensures the Task lands on a Cell with matching `stack`
- ensures an even distribution of memory and disk usage across Cells

#### Communicating Fullness

When a Task cannot be allocated the Auctioneer CAS the Task from the `PENDING` state to the `COMPLETED` state, marking it as `Failed` and including a `FailureReason` that explains that the cluster has no capacity for the task.  It is up to the consumer to retry the Task.

### Resolving Completed Tasks

When a Task is `COMPLETED` it is up-to the consumer to delete the Task (though the converger will clean up Tasks that have been `COMPLETED` for a lengthy period of time).  Consumers can either poll Tasks to see them enter the `COMPLETED` state or can register a `CompletionCallback` to be told the Task has been completed:

- When polling:
	+ consumers instruct the Receptor to delete the Task.  This CAS `COMPLETED => RESOLVING` then CAD `RESOLVING`.
- When handling the `CompletionCallback`
	+ the Receptor CAS `COMPLETED => RESOLVING` to indicate that it is handling resolving the Task.  This is necessary to ensure that no two Receptors attempt to call the `CompletionCallback`.
	+ when the `CompletionCallback` returns, the Receptor CAD the Task.

It is an error to attempt to delete a Task that is *not* in the `COMPLETED` state.

> There are a few details around retrying the `CompletionCallback` -- these are documented in the Receptor API docs.

### Cancelling Tasks

Tasks in the `PENDING` and `RUNNING` state can be cancelled via the Receptor at any time:

- If the Task is `PENDING` or `RUNNING`:
	+ The Receptor CAS the Task to `COMPLETED` and `Failed` with `FailureReason = "cancelled"`
	+ The Recetpor then sends a cancellation message to the Rep in question
- The Rep's poller will react to the cancel message and delete its container

The consumer then deletes the `COMPLETED` task.  It is an error to attempt to cancel a Task that is not in the `PENDING` or `RUNNING` states.

> There is a bit of a hole here.  A user could cancel then delete a Task and then request a new Task with the same `TaskGuid` *before* the Rep notices the container should be deleted.  This gap is narrowed by emitting a message to the Rep.  One option would be to generate a unique InstanceGuid so that the Rep can converge correctly, but this is a relatively minor issue that is unlikely to impact CF.

### Evacuation

Tasks never migrate from one Cell to another.  They effectively block Cell evacuation. Here's what happens when a Cell must be evacuated:

- the Rep is told to evacuate.
- the Rep subsequently refuses to take on any new work
- the Rep shuts down when either all containers have been destroyed OR an evacuation timeout is exceeded:
	- in either case, the Rep ensures that any `RUNNING` Tasks associated with it are CAS to the `COMPLETED` state.  These should be marked `Failed` with the `FailureReason = "timed out during cell evacuation"`

### Maintaining Consistency: Converger

Since tasks are guaranteed to run at most once, the Converger never attempts to "restart" Tasks.  Instead it's role is:

1. The converger will CAS Tasks that are `RUNNING` on failed Cells
	- Cells periodically maintain their presence in the BBS.  If a Cell disappears the converger will notice and CAS `RUNNING` Tasks associated with the missing Cell to `COMPLETED` and `Failed`.
2. The converger will re-emit start requests for Tasks stuck in the `PENDING` state
	- If a Task is stuck in the `PENDING` state and has been there for a long period of time a start request is re-emitted to the auctioneer
3. The converger will CAD a `COMPLETED`  Task that has first entered the `COMPLETED` state too long ago (2 minutes ago)
4. The converger will ask a random receptor to resolve a `COMPLETED` Task that has been in the `COMPLETED` state for too long (30 seconds)
5. The converger will CAS a `RESOLVING` Task to the `COMPLETED` state and notify a Receptor if it has been `RESOLVING` for too long
	- Perhaps the resolving Receptor died?  If a Task is stuck `RESOLVING` for too long, the Converger gives it another change to resolve by moving it back to `COMPLETED`

### Harmonizing Tasks with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.  It does this periodically by fetching containers from the executor and taking actions.

Here is an outline of how the Rep should act to reconcile the Tasks in the BBS with the set of containers.  In this example, α will represent the Cell performing the reconciliation and ω will represent a different Cell (possibly empty):

Container State | Task State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | No Task | Delete Container | **Conceivable**: The task has been cancelled, α should not act.
`RESERVED` | `PENDING` | Start Task **THEN** Run Container | **Expected**: α has a reservation for this task and should start the task and run it in the container.
`RESERVED` | `RUNNING on α` | Do Nothing | **Expected**: The container is likely about to be initialized (the Rep tells the BBS running right after it reserves).  Things won't get stuck in this state because the reservation will either enter a new state, or be reaped by the executor.
`RESERVED` | `RUNNING on ω` | Delete Container | **Conceivable**: Don't start running this task
`RESERVED` | `COMPLETED on α` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `COMPLETED on ω` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `RESOLVING on α` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `RESOLVING on ω` | Delete Container | **Conceivable**: Don't start running this Task
`INITIALIZING/CREATED/RUNNING` | No Task | Delete Container | **Conceivable**: The task has been cancelled, α should stop running it.
`INITIALIZING/CREATED/RUNNING` | `PENDING` | CAS the Task to `RUNNING on α` | **Inconceivable**: Make sure BBS knows we're running the Task (but we should already have started the task before running the container).
`INITIALIZING/CREATED/RUNNING` | `RUNNING on α` | Do Nothing | **Expected**: BBS is up-to-date
`INITIALIZING/CREATED/RUNNING` | `RUNNING on ω` | Delete Container (and log loudly) | **Inconceivable**: Apparently this Task is running somewhere else!
`INITIALIZING/CREATED/RUNNING` | `COMPLETED on α` | Delete Container | **Conceivable**: The task is `COMPLETED` (perhaps it was cancelled) - delete the container
`INITIALIZING/CREATED/RUNNING` | `COMPLETED on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`INITIALIZING/CREATED/RUNNING` | `RESOLVING on α` | Delete Container | **Conceivable**: The task is `RESOLVING` (perhaps it was cancelled) - delete the container
`INITIALIZING/CREATED/RUNNING` | `RESOLVING on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`COMPLETED` | No Task | Delete Container | **Conceivable**: The Task has been cancelled - don't worry about it
`COMPLETED` | `PENDING` | CAS the Task to `COMPLETED` then Delete Container | **Inconceivable**: We should already have started the task before running the container.
`COMPLETED` | `RUNNING on α` | CAS the Task to `COMPLETED` then Delete Container | **Expected**: Make sure BBS knows we've completed the Task
`COMPLETED` | `RUNNING on ω` | Delete Container | **Inconceivable**: Apparently this Task is running somewhere else!
`COMPLETED` | `COMPLETED on α` | Delete Container | **Conceivable**: The task has already been marked `COMPLETED` - delete the container
`COMPLETED` | `COMPLETED on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`COMPLETED` | `RESOLVING on α` | Delete Container | **Conceivable**: The task has already been marked `RESOLVING` (perhaps it was cancelled) - delete the container
`COMPLETED` | `RESOLVING on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
No Container | `RUNNING on α` | CAS to `COMPLETED` and `Failed` | **Conceivable**: Diego thinks α is running the instance, but it is not
No Container | `COMPLETED on α` | Do Nothing | **Expected**: The client has not resolved the task
No Container | `RESOLVING on α` | Do Nothing | **Expected**: The client is resolving the task


Some notes:

- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell.  Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible.  However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence.  An exhaustive analysis of all possible combinations (such as this) ensures more safety around the Task lifecycle.
- When the Action described in this table fails, the Rep should log and do nothing.  In this way we defer to the next polling cycle to retry actions.

BBS | Reserved | I/C/Running | Completed | No Container
---|---|---|---|---|---
Missing     | Delete Container          | Delete Container          | Delete Container |
Pending     | Start Task, Run Container | Start Task, Run Container | Complete Task, Delete Container
Running-α   | Do Nothing                | Do Nothing                | Complete Task, Delete Container | Fail Task
Running-ω   | Delete Container          | Delete Container          | Delete Container          |
Completed-α | Delete Container          | Delete Container          | Delete Container          | Do Nothing
Completed-ω | Delete Container          | Delete Container          | Delete Container          |
Resolving-α | Delete Container          | Delete Container          | Delete Container          | Do Nothing
Resolving-ω | Delete Container          | Delete Container          | Delete Container          |

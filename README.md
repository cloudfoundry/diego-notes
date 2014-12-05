# Diego: The Big Picture, The Deep Dive

This describes the world *as it will be* when the next pile of stories are completed.
Also.  I probably missed something(s) ;)

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
`COMPLETED` | The container is no longer running.  `container.Result` will include details around whether the work the container performed succeeded or failed.

Allowed transitions:

- `RESERVED -> INITIALIZING`
- `INITIALIZING -> CREATED | COMPLETED`
- `CREATED -> COMPLETED | RUNNING`
- `RUNNING -> COMPLETED`

It is the Rep's responsibility to delete containers.  Containers can be deleted regardless of current state.

#### Container Events

The executor emits two events:

- `ContainerRunningEvent`: triggered when the container enters the running state
- `ContainerCompletedEvent`: triggered when the container enters the completed state

The rep should handle events exactly as it handles polling.  Given a container in a particular state, reconcile with the BBS.  The events are simply an optimization on top of polling.

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
- `DesiredLRP`s can be updated freely, but only a subset of fields may be updated (`instances`, `routes`, `annotation`).  This subset corresponds to changes that can be accomodated *without* restarting any ActualLRPs
- **Only** consumers can modify `DesiredLRP`s.  Therefore, only the Receptor is allowed to manipulate `DesiredLRP`.
- `DesiredLRP`s are organized into domains - arbitrary groupings for the consumer's convenience.  Domains have a notion of "freshness" meaning that the Desired state is up-to-date.  The Converger only performs destructive operations if the Desired state is up-to-date.

### ActualLRPs

Diego attempts to keep one ActualLRP running per desired index.  To ensure this we use the BBS to walk an individual ActualLRP through a state machine.

The BBS stores the ActualLRP under `/v1/actual/:process_guid/:index/instance`

When it is evacuating, the BBS stores the ActualLRP under `/v1/actual/:process_guid/:index/evacuating`.  Only during evacuation is it possible/legal for (at most) 2 ActualLRPs to run at a given index.

Here is the ActualLRP state machine:

State | Meaning
------|--------
`UNCLAIMED`| The ActualLRP is being scheduled by an auction
`CLAIMED` | The ActualLRP has been assigned to a Cell and is being started
`RUNNING` | The ActualLRP is runing on a Cell and is ready to receive traffic/work.
`CRASHED` | The ActualLRP has crashed and is no longer on a Cell.  It should be restarted (eventually).

In addition, the ActualLRP includes two pieces of data: `CrashCount` and `LastCrashedAt`.  These are used to implement a backoff policy when restarting crashes.

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
	- an UNCLAIMED ActualLRP is CREATED
- if restarting a crashed ActualLRP:
	- CAS to UNCLAIMED ActualLRPAuctioneer
- in both cases: a start is sent to the Auctioneer
	- the Auctioneer picks a Rep
	- the Rep is told to start the ActualLRP
	- the Rep creates a container reservation, then CAS the ActualLRP to CLAIMED, then starts running the container

#### Stopping ActualLRPs

When an ActualLRP should be stopped:

- for `CLAIMED` or `RUNNING` ActualLRPs:
	- a stop is sent to the corresponding Rep.  The Rep will remove the ActualLRP from the BBS.
- for `UNCLAIMED` or `CRASHED` ActualLRPs:
 	- compare-and-delete the ActualLRP.

### Distributing ActualLRPs: Auctioneer

The Auctioneer is responsible for distributing ActualLRPs optimally.  When requests to start ActualLRPs arrive:

- the Auctioneer fetches the state of each Cell.  This includes information about the available capacity on the Cell, the Cell's stack, and the set of ActualLRPs currently running on the Cell.
- the Auctioneer then distributes the ActualLRPs across the Cells (in-memory).  During this process the Auctioneer optimizes for:
	- an even distribution of memory and disk usage across Cells
	- minimizing colocation of ActualLRP instances of a given DesiredLRP on the same Cell.
	- minimizing colocation of ActualLRP instances of a givne DesiredLRP in the same Availability Zone (see below).
- the Auctioneer then submits the allocated work to all Cells.  Work that fails to start/cannot be allocated is placed back in the pool of work to be allocated and is eventually retried

> Note: the Auctioneer never manipulates the BBS.  It simply follows instructions.  The Converger/Rep take care of coming to eventual consistency on the BBS.

#### Managing Availability Zones

There are some important details around how the Auctioneer distributes instances across Availability Zones.  Here are the rules it follows:

- the Auctioneer is told how many AZs there are (`NZones`).  This is a configuration option
- when fetching Cell state, the Auctioneer is given the Cell's `zone`
- when placing instances, the Auctioneer computes the instance's preferred zone (see below)
- the instance is then placed according to the following rules:
	- if the instance `index` is `<= 1` it is preferentially placed in its `PreferredZone`.  If there is no room in this zone it is placed in some other zone.
	- if the instance `index` is `> 1` it must be placed in its `PreferredZone`.  If there is no room in this zone the instance is not placed.  The auctioneer will retry periodically.

The preferred zone is computed as follows.  Assume we have a deterministic mapping from `ProcessGuid` to `zone`: `f(ProcessGuid) ∈ {1..N}`.  Then `PreferredZone(ProcessGuid, Index) = (f(ProcessGuid) + Index) % NZones`

#### Communicating Fullness

TBD - (new ActualLRP state?)

### Crashes

When the container associated with an ActualLRP enters the `COMPLETED` state the Rep takes actions to ensure the ActualLRP gets restarted.  Let's call this the `RepCrashDance`:

- If `CrashCount` < 3:
	-  Increment the `CrashCount` and bump `LastCrashedAt`,  CAS the ActualLRP to `UNCLAIMED`, and emit a start to the Auctioneer.
- If `CrashCount` > 3:
	- Increment the `CrashCount` and bump `LastCrashedAt`.  CAS the ActualLRP to `CRASHED`.

The `CRASHED` ActualLRP is eventually restarted by the converger.  The `WaitTime` is computed like so:

- If `CrashCount < 8`
	- the Converger should restart the ActualLRP `N` seconds after `LastCrashedAt` (exponential backoff in [this story](https://www.pivotaltracker.com/story/show/83638710))
- If `CrashCount >= 5`
	- the Converger should restart the ActualLRP `MaxWaitTime = 16` minutes after `LastCrashedAt`
- If `CrashCount > 200`
	- the ActualLRP is never restarted

It is important that the `CrashCount` be reset eventually.  The Rep does this when marking an ActualLRP as crashed:

- If `LastCrashedAt` is `> 2 * MaxWaitTime` minutes ago then this instance has been running for long enough that it's backoff policy should be reset:
 	- Reset `CrashCount` to 0 then do the `RepCrashDance`.

> Note: the Rep is responsbile for immediately restarting `ActualLRP`s that have `CrashCount < 3`.  These `ActualLRP`s never enter the `CRASHED` state.  The Converger is responsible for restarting `ActualLRP`s that are in the `CRASHED` state.

### Evacuation

When a Cell must be evacuated its ActualLRPs must first transfer to another Cell.  Here's how this works:

- the Rep is told to evacuate.
- the Rep subsequently refuses to take on any new work
- the Rep destroys ActualLRP containers that are not yet `RUNNING` and requests starts for the corresponding ActualLRPs
- the Rep moves its `RUNNING` ActualLRPs from `/v1/actual/:process_guid/:index/instance` to `/v1/actual/:process_guid/:index/evacuating`, requesting corresponding starts.
- the Rep periodically fetches `/v1/actual/:process_guid/:index/instance` from the BBS for the instances it owns under `/evacuating`:
	- if the ActualLRP under `instance` is missing, `RUNNING` or `CRASHED`: the Rep destroys the container and deletes the `/evacuating` entry
	- otherwise, the Rep does nothing
- the Rep shuts down when either all containers have been destroyed OR an evacuation timeout is exceeded:
	- in either case, the Rep ensures that any ActualLRPs associated with it are removed from `/evacuating` and that any tasks it still has running transition to `COMPLETED`.

When fetching ActualLRPs the receptor always returns the `/evacuating` instance if present.

> Note: the Converger plays no special role during evacuation

### Harmonizing DesiredLRPs with Actual LRPs: Converger

Messages get lost.  Connections time out.  Components fail.  Bugs happen.

The converger is responsible for bringing Diego into eventual consistency in the face of these eventualities.  The converger only operates on BBS data and strives to bring the actual state (the set of ActualLRPs) into consistency with the desired state (DesiredLRPs).

Here are it's responsibilities and the actions it takes:

1. Reaping ActualLRPs on failed Cells
	- Cells periodically maintain their presence in the BBS.  If a Cell disappears the converger will notice and CAS `CLAIMED` and `RUNNING` ActualLRPs associated with the missing Cell to `UNCLAIMED`.  The converger will also emit starts for these missing ActualLRPs.
2. Starting missing ActualLRPs
	- If there is no ActualLRP in the BBS for a particular DesiredLRP index, the Converger CREATEs an `UNCLAIMED` ActualLRP and requests a start.
3. Stopping extra ActualLRPs
	- If there is an ActualLRP (in any state) that does not correspond to a DesiredLRP index, the Converger stops the ActualLRP (see above)
	- This action is *only* taken if ActualLRP's Domain must be fresh.
4. Restarting `CRASHED` ActualLRPs
	- It is the converger's responsibilities to restart `CRASHED` ActualLRPs (see above)  
5. Re-emitting start requests
	- The request to start the ActualLRP may have failed.  This can be detected if the ActualLRP remains in the `PENDING` state for longer than some timescale.  The Converger acts by resubmitting a start request.

### Harmonizing ActualLRPs with Container State: Rep

The Rep is responsible for ensuring the the BBS is kept up-to-date with what is running.  It does this by periodically fetching containers from the executor and taking actions.

Here is an outline of how the Rep should act to reconcile the ActualLRPs in the BBS with the set of containers.  In this example, A will represent the Cell performing the reconciliation and O will represent some (any) other cell:

Container State | ActualLRP State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | `ANY` | Do Nothing | A merely has a reservation the ActualLRP, no need to act yet.
`INITIALIZING` or `CREATED` | No ActualLRP | Delete Container | The consumer likely stopped desiring this ActualLRP while it was being auctioned
`INITIALIZING` or `CREATED` | `UNCLAIMED` | CAS to `CLAIMED by A` | A is starting this ActualLRP
`INITIALIZING` or `CREATED` | `CLAIMED by A` | Do Nothing | A is starting this ActualLRP, no need to write to the BBS
`INITIALIZING` or `CREATED` | `CLAIMED by O` | Delete Container | The ActualLRP is starting on O, stop starting it on A
`INITIALIZING` or `CREATED` | `RUNNING on A` | Do Nothing | This should not be possible
`INITIALIZING` or `CREATED` | `RUNNING on O` | Delete Container | The ActualLRP is running on O, stop starting it on A
`INITIALIZING` or `CREATED` | `CRASHED` | Delete Container | A is incorrectly starting the instance - some other Cell will pick this up later.
`RUNNING` | No ActualLRP | CREATE `RUNNING on A` | A is running on this ActualLRP, let Diego know so it can take action appropriately (we don't allow blank ActualLRPs to shut down containers as this would lead to catastrophic fail should the BBS be accidentally purged).
`RUNNING` | `UNCLAIMED` | CAS to `RUNNING on A` | A is running on this ActualLRP, no need to start it elsewhere
`RUNNING` | `CLAIMED by A` | CAS to `RUNNING on A` | A is running this ActualLRP
`RUNNING` | `CLAIMED by O` | CAS to `RUNNING on A` | A is running this ActualLRP, no need to start it elsewhere
`RUNNING` | `RUNNING on A` | Do Nothing | A is running on this ActualLRP, no need to write to the BBS
`RUNNING` | `RUNNING on O` | Delete Container | Instance is running on O, stop running it on A
`RUNNING` | `CRASHED` | CAS to `RUNNING on A` | A is succesfully running this ActualLRP. It's not crashed and need not be restarted.
`COMPLETED (crashed)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | A just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (crashed)` | `UNCLAIMED` | Delete Container | Instance will be scheduled elsewhere
`COMPLETED (crashed)` | `CLAIMED by A` | Perform `RepCrashDance` then Delete Container | Instance crashed on A while starting
`COMPLETED (crashed)` | `STARING on O` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (crashed)` | `RUNNING on A` | Perform `RepCrashDance` then Delete Container | Instance crashed on A while running
`COMPLETED (crashed)` | `RUNNING on O` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (crashed)` | `CRASHED` | Delete Container | The crash has already been noted
`COMPLETED (shutdown)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | A just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (shutdown)` | `UNCLAIMED` | Delete Container | Nothing to be done
`COMPLETED (shutdown)` | `CLAIMED by A` | CAD ActualLRP then Delete Container  | A was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `STARING on O` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (shutdown)` | `RUNNING on A` | CAD ActualLRP then Delete Container | A was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `RUNNING on O` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (shutdown)` | `CRASHED` | Delete Container | Nothing to do
No Container | `CLAIMED by A` | CAS to `UNCLAIMED`, request a start | Diego thinks A is starting the instance, but it is not
No Container | `RUNNING on A` | CAS to `UNCLAIMED`, request a start | Diego thinks A is starting the instance, but it is not

Some notes:
- `COMPLETED` comes in two flavors.  `crashed` implies the container died unexpectedly.  `shutdown` implies the container was asked to shut down (e.g. the Rep was sent a stop).
- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell.  Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible.  However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence.  An exhaustive analysis of all possible combinations (such as this) ensures eventual consistency... eventually.

#### Polling vs Events

The Rep keeps the BBS up-to-date via two mechanisms: polling and events.  This is complicated by the fact that events can come in while the poller is *fetching* state from its two sources (BBS & Executor).

As a result, when polling the Rep may have a stale copy of the containers and/or a stale copy of the BBS.  To be safe it should always validate a decision it is about to make.  This is *especially* important when deleting an `INITIALIZING/CREATED/RUNNING` container.

## Tasks

All things Tasks.  Here's an outline:

### Task States and Lifecycle

The lifecycle of a Task in Diego is quite different from that of an LRP.  Tasks run at most once: they're never restarted, they don't "crash" they don't move during an evacuation.  Instead Tasks `COMPLETE`.  Once in the `COMPLETE` state the Tasks have a notion of whether they have failed or not.  Failed Tasks include a `FailureReason`; succesful Tasks (may) include a `Result` (the contents of a file requested by the user).

Here are the states for the Tasks:

State | Meaning
------|--------
PENDING | The Task is being scheduled by the Auctioneer
RUNNING | The Task is running on a Cell
COMPLETED | The Task is complete
RESOLVING | The Task is being resolved by a Receptor

Here's the happy path for a Task:




### Distributing Tasks: Auctioneer

### Resolving Completed Tasks

### Cancelling Tasks

### Evacuation

### Maintaining Consistency: Converger

### Harmonizing Tasks with Container State: Rep
# Diego: The Big Picture, The Deep Dive

This describes the world *as it will be* when the next pile of stories are completed.

Also.  I probably missed something(s) ;)

This document is not concerned with:

- the Receptor API (we have docs for that)
- the details around how the Exeuctor runs steps/recipes/monitor actions/run actions/etc../etc..
- anything that begins with "CC"
- anything to do with emitting routes (though I should add that since it's going to be relevant)

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
- `DesiredLRP`s can be updated freely, but only a subset of fields may be updated (`instances`, `routes`, `annotation`).  This subset corresponds to changes that can be accomodated *without* restarting any ActualLRPs
- **Only** consumers can modify `DesiredLRP`s.  Therefore, only the Receptor is allowed to manipulate `DesiredLRP`.
- `DesiredLRP`s are organized into domains - arbitrary groupings for the consumer's convenience.  Domains have a notion of "freshness" meaning that the Desired state is up-to-date.  The Converger avoids taking incorrect destructive actions by only shutting down undesired ActualLRPs if the Desired state is "fresh" (i.e. up-to-date).

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
	- CAS to UNCLAIMED
- in both cases: upon success, a start is sent to the Auctioneer
- the Auctioneer picks a Rep
- the Rep is told to start the ActualLRP
- the Rep creates a container reservation
- upon success, the Rep then CAS the ActualLRP from `UNCLAIMED` to `CLAIMED`
- upon success, the Rep then starts running the container 
- eventually a `ContainerRunningEvent` causes the Rep to CAS from CLAIMED to RUNNING

#### Stopping ActualLRPs

When an ActualLRP should be stopped:

- for `CLAIMED` or `RUNNING` ActualLRPs:
	- a stop is sent to the corresponding Rep.
	- the Rep deletes the container
	- the Rep then removes the ActualLRP from the BBS.
- for `UNCLAIMED` or `CRASHED` ActualLRPs:
 	- compare-and-delete the ActualLRP.

### Distributing ActualLRPs: Auctioneer

The Auctioneer is responsible for distributing ActualLRPs optimally.  When requests to start ActualLRPs arrive the Auctioneer performs the scheduling loop:

- the Auctioneer fetches the state of each Cell.  This includes information about the available capacity on the Cell, the Cell's stack, and the set of ActualLRPs currently running on the Cell.
- the Auctioneer then distributes the ActualLRPs across the Cells (in-memory).  During this process the Auctioneer optimizes for:
	- an even distribution of memory and disk usage across Cells
	- minimizing colocation of ActualLRP instances of a given DesiredLRP on the same Cell.
	- minimizing colocation of ActualLRP instances of a givne DesiredLRP in the same Availability Zone (see below).
- the Auctioneer then submits the allocated work to all Cells
	- any work that could not be allocated is carried over into the next batch
	- if the Cell responds saying that the work could not be performed, the auctioneer carries the failed work over into the next batch
	- if the Cell *fails to respond* (e.g. a connection timeout elapses), the auctioneer *does **not*** carry the work over into the next batch.  This is a case of partial failure and the auctioneer defers to the converger to figure out what to do
- any work carried over into the next batch is merged in with work that came in during the previous round of scheduling and the auction repeats the scheduling loop

#### Managing Availability Zones

Diego distributes instances of a given DesiredLRP across availability zones.  This provides high availability in the event of a zone failure.  However, the failure of an entire zone is a catastrophic event and shifting all the load from that zone onto the other zones can easily destabilize the entire system (especially if the number of zones is small).

To avoid this, Diego enforces the following policy when placing ActualLRPs: 
- If an instance has index 0 or 1: always place it.
- If an instance has index > 1: only place it in its preferred zone.

The Auctioneer distributes instances across Availability Zones.  Here are the rules it follows:

- the Auctioneer is told how many AZs there are (`NZones`).  This is a configuration option
- when fetching Cell state, the Auctioneer is given the Cell's `zone`
- when placing instances, the Auctioneer computes the instance's preferred zone (see below)
- the instance is then placed according to the following rules:
	- if the instance `index` is `<= 1` it is preferentially placed according to the `ZonePreferenceRanking`.
	- if the instance `index` is `> 1` it must be placed in its `PreferredZone` (the first elemnt of the `ZonePreferenceRanking`.  If there is no room in this zone the instance is not placed.  The auctioneer will retry periodically.

The Zone Preferrence Ranking is computed as follows.  Assume we have a deterministic mapping `PreferredZone` from `ProcessGuid` to `zone`: `f(ProcessGuid) ∈ {1..N}`.  Then, the rank order of zones for `Index 0` is `f(ProcessGuid), f(ProcessGuid)+1..N,0..f(ProcessGuid)-1`, the rank order of zones for `Index 1` is offset by 1: `f(ProcessGuid)+1..N,0..f(ProcessGuid)`, etc..

In this way, ActualLRPs with `Index > 1` must be placed in `PreferredZone = (f(ProcessGuid) + Index) % NZones`

#### Communicating Fullness (TBD)

When an ActualLRP cannot be placed because there are no resources to place it, the Auctioneer can communicate this back to the user somehow.  The mechanism for this is TBD.  A new `ActualLRP` state (e.g. `FAILED` or `UNPLACEABLE`) could be the best way forward.  Diego could retry starting `FAILED` `ActualLRPs` periodically.  TBD.

### Crashes

When an ActualLRP instance crashes, Diego is responsible for restarting the instance.  There is a tension, however, between restarting a failed instance immediately and overloading the system with endless immediate restarts of instances that never succeed.

To strike this balance Diego immediately restarts a crashed instance up to 3 times.  We perform immediate restarts for a few reasons:
- they make for good demos (look it crashed -- look it's back!)
- they handle cases where a healthy well-written application has truly crashed and shold be restarted ASAP.

Sometimes, however, an instance crashes because some external dependency is down. In these cases it makes more sense for Diego to wait (and alleviate the strain of a thrashing instance hopping from one Cell to another) between restarts.  So for crashes 3 - 8 Diego applies an exponential backoff in the waittime between restarts.

Finally, we've learned from existing large public installations (PWS, Bluemix) that the vast majority of crashed instances in the system are poorly-written instances that never even manage to come up.  Instead of endlessly restarting them (which puts a substantial strain on the system) Diego will (eventually) give up on these instances.

All that remains is the question of how we reset the exponential backoff.  Instances that thrash and place a heavy load on the system typically crash quickly.  So we apply a simple heuristic: if an instance manages to stay running for > 5 minutes, we reset its crash count.

When the container associated with an ActualLRP enters the `COMPLETED` state the Rep takes actions to ensure the ActualLRP gets restarted.  Let's call this the `RepCrashDance`:

- If `CrashCount` < 3:
	-  Increment the `CrashCount` and bump `LastCrashedAt`,  CAS the ActualLRP to `UNCLAIMED`, and emit a start to the Auctioneer.
- If `CrashCount` > 3:
	- Increment the `CrashCount` and bump `LastCrashedAt`.  CAS the ActualLRP to `CRASHED`.

The `CRASHED` ActualLRP is eventually restarted by the converger.  The `WaitTime` is computed like so:

- If `CrashCount < 8`
	- the Converger should restart the ActualLRP `N` seconds after `LastCrashedAt` (exponential backoff in [this story](https://www.pivotaltracker.com/story/show/83638710))
- If `CrashCount >= 8`
	- the Converger should restart the ActualLRP `MaxWaitTime = 16` minutes after `LastCrashedAt`
- If `CrashCount > 200`
	- the ActualLRP is never restarted

It is important that the `CrashCount` be reset eventually.  The Rep does this when marking an ActualLRP as crashed:

- If the ActualLRP had been `RUNNING` for `>= 5 minutes`:
	- Reset the `CrashCount` to 0 and the then do the `RepCrashDance`
- Otherwise
	- Do the `RepCrashDance`


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
	- The request to start the ActualLRP may have failed.  This can be detected if the ActualLRP remains in the `UNCLAIMED` state for longer than some timescale.  The Converger acts by resubmitting a start request.

### Harmonizing ActualLRPs with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.  It does this by periodically fetching containers from the executor and taking actions.

Here is an outline of how the Rep should act to reconcile the ActualLRPs in the BBS with the set of containers.  In this example, α will represent the Cell performing the reconciliation and ω will represent some (any) other cell.

Container State | ActualLRP State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | `ANY` | Do Nothing | α merely has a reservation for the ActualLRP, no need to act yet.
`INITIALIZING|CREATED` | No ActualLRP | Delete Container | The consumer likely stopped desiring this ActualLRP while it was being auctioned
`INITIALIZING|CREATED` | `UNCLAIMED` | CAS to `CLAIMED by α` | α is starting this ActualLRP
`INITIALIZING|CREATED` | `CLAIMED by α` | Do Nothing | α is starting this ActualLRP, no need to write to the BBS
`INITIALIZING|CREATED` | `CLAIMED by ω` | Delete Container | The ActualLRP is starting on ω, stop starting it on α
`INITIALIZING|CREATED` | `RUNNING on α` | Do Nothing | This should not be possible
`INITIALIZING|CREATED` | `RUNNING on ω` | Delete Container | The ActualLRP is running on ω, stop starting it on α
`INITIALIZING|CREATED` | `CRASHED` | Delete Container | α is incorrectly starting the instance - some other Cell will pick this up later.
`RUNNING` | No ActualLRP | CREATE `RUNNING on α` | α is running on this ActualLRP, let Diego know so it can take action appropriately (we don't allow blank ActualLRPs to shut down containers as this would lead to catastrophic fail should the BBS be accidentally purged).
`RUNNING` | `UNCLAIMED` | CAS to `RUNNING on α` | α is running on this ActualLRP, no need to start it elsewhere
`RUNNING` | `CLAIMED by α` | CAS to `RUNNING on α` | α is running this ActualLRP
`RUNNING` | `CLAIMED by ω` | CAS to `RUNNING on α` | α is running this ActualLRP, no need to start it elsewhere
`RUNNING` | `RUNNING on α` | Do Nothing | α is running on this ActualLRP, no need to write to the BBS
`RUNNING` | `RUNNING on ω` | Delete Container | Instance is running on ω, stop running it on α
`RUNNING` | `CRASHED` | CAS to `RUNNING on α` | α is succesfully running this ActualLRP. It's not crashed and need not be restarted.
`COMPLETED (crashed)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | α just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (crashed)` | `UNCLAIMED` | Delete Container | Instance will be scheduled elsewhere
`COMPLETED (crashed)` | `CLAIMED by α` | Perform `RepCrashDance` then Delete Container | Instance crashed on α while starting
`COMPLETED (crashed)` | `CLAIMED on ω` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (crashed)` | `RUNNING on α` | Perform `RepCrashDance` (see Crash section above for details around resettign the CrashCount) then Delete Container | Instance crashed on α while running
`COMPLETED (crashed)` | `RUNNING on ω` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (crashed)` | `CRASHED` | Delete Container | The crash has already been noted
`COMPLETED (shutdown)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | α just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (shutdown)` | `UNCLAIMED` | Delete Container | Nothing to be done
`COMPLETED (shutdown)` | `CLAIMED by α` | CAD ActualLRP then Delete Container  | α was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `CLAIMED on ω` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (shutdown)` | `RUNNING on α` | CAD ActualLRP then Delete Container | α was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `RUNNING on ω` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (shutdown)` | `CRASHED` | Delete Container | Nothing to do
No Container | `CLAIMED by α` | CAD ActualLRP | There is no matching container, delete the ActualLRP and allow the converger to determine whether it is still desired
No Container | `RUNNING on α` | CAD ActualLRP | There is no matching container, delete the ActualLRP and allow the converger to determine whether it is still desired

Some notes:
- `COMPLETED` comes in two flavors.  `crashed` implies the container died unexpectedly.  `shutdown` implies the container was asked to shut down (e.g. the Rep was sent a stop).
- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell.  Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible.  However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence.  An exhaustive analysis of all possible combinations (such as this) ensures eventual consistency... eventually.

In principal it is possible for one Cell to end up running more than one instance of an ActualLRP (i.e. same `ProcessGuid` and `Index` but different `InstanceGuid`).  At most one of these ActualLRPs will be in the BBS.  The table above still applies like so: when processing a container whose `InstanceGuid` is *not* in the BBS, the Rep treats the ActualLRP as if it were on some other Cell (the `ω` cases here).  To be concrete: assume the Cell has two running containers, one with `InstanceGuid` `ig1`, the other with `ig2`.  Suppose further that the BBS lists an ActualLRP running on `α` with `InstanceGuid` `ig1`.  When processing `ig1` the Rep will do nothing.  When processing `ig2` the Rep will see that there is already an ActualLRP running and will delete the container.

It is impossible for multiple containers with the same `InstanceGuid` to run on a Cell as the `InstanceGuid` is the unique-identifier associated with the container.

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
	+ Creates a Task in the `PENDING` state
	+ Sends a start request to the Auctioneer
- The Auctioneer picks a Cell to run the Task
- The Rep creates a container reservation (and then ACKs the Auctioneer's request)
- Upon success the Rep CAS the Task from `PENDING` to `RUNNING`
- Upon success the Rep starts the container running.
- When the container completes (succesfully or otherwise) the Rep CAS the Task to `COMPLETED`
	+ If the Task has a `CompletionCallback` URL this sends a message to the Receptor to handle resolving the Task.
- The Receptor CAS the Task to `RESOLVING` and calls the `CompletionCallback`
- Upon success Receptor Rep CAD the Task

### Distributing Tasks: Auctioneer

The Auctioneer is responsible for distributing Tasks optimally.  When requests to start Tasks arrive the Auctioneer performs the scheduling loop:

- the Auctioneer fetches the state of each Cell.  This includes information about the available capacity on the Cell and the Cell's stack
- The Auctioneer then distributes the Tasks across the Cells (in-memory).  During the process the Auctioneer optimizes for:
	+ an even distribution of memory and disk usage across Cells
- the Auctioneer then submits the allocated work to all Cells
	- any work that could not be allocated is carried over into the next batch
	- if the Cell responds saying that the work could not be performed, the auctioneer carries the failed work over into the next batch
	- if the Cell *fails to respond* (e.g. a connection timeout elapses), the auctioneer *does **not*** carry the work over into the next batch.  This is a case of partial failure and the auctioneer defers to the converger to figure out what to do
- any work carried over into the next batch is merged in with work that came in during the previous round of scheduling and the auction repeats the scheduling loop

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
- The Rep's poller will eventually notice that the task has been `COMPLETED` and will delete its container
	+ This is something that could be optimized via a message send to the Rep by the Receptor

The consumer then deletes the `COMPLETED` task.  It is an error to attempt to cancel a Task that is not in the `PENDING` or `RUNNING` states.

> There is a bit of a hole here.  A user could cancel then delete a Task and then request a new Task with the same `TaskGuid` *before* the Rep notices the container should be deleted.  This gap is narrowed by emitting a message to the Rep.  The only way to truly close the gap would be to introduce a `CANCELLING` state but this adds complexity.  (For CF this isn't super important as we have control over the `TaskGuid` and always generate a new one).

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
3. The converger will ask a random receptor to resolve a `COMPLETED` Task that has been in the `COMPLETED` state for too long
4. The converger will CAD a `COMPLETED` or `RESOLVING` Task that has been in the `COMPLETED` state for way too long
5. The converger will CAS a `RESOLVING` Task to the `COMPLETED` state and notify a Receptor if it has been `RESOLVING` for too long
	- Perhaps the resolving Receptor died?  If a Task is stuck `RESOLVING` for too long, the Converger gives it another change to resolve by moving it back to `COMPLETED`

### Harmonizing Tasks with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.  It does this periodically by fetching containers from the executor and taking actions.

Here is an outline of ow the Rep should act to reconcile the Tasks in the BBS with the set of containers.  In this example, α will represent the Cell performing the reconciliation and ω will represent some (any) other cell:

Container State | Task State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | No Task | Delete Container | The task has been cancelled, α should not act.
`RESERVED` | `PENDING` | Do Nothing | α merely has a reservation for the Task and should eventually CAS to `RUNNING on α` (if it doesn't the converger will eventually retry the Task).
`RESERVED` | `RUNNING on α` | CAS the Task to `COMPLETE + Failed` | BBS thinks α is running the Task but it isn't!
`RESERVED` | `RUNNING on ω` | Delete Container | Don't start running this task
`RESERVED` | `COMPLETED` | Delete Container | Don't start running this Task
`RESERVED` | `RESOLVING` | Delete Container | Don't start running this Task
`INITIALIZING|CREATED|RUNNING` | No Task | Delete Container | The task has been cancelled, α should stop running it.
`INITIALIZING|CREATED|RUNNING` | `PENDING` | CAS the Task to `RUNNING on α` | make sure BBS knows we're running the Task.
`INITIALIZING|CREATED|RUNNING` | `RUNNING on α` | Do Nothing | BBS is up-to-date
`INITIALIZING|CREATED|RUNNING` | `RUNNING on ω` | Delete Container (and log loudly) | Apparently this Task is running somewhere else!
`INITIALIZING|CREATED|RUNNING` | `COMPLETED` | Delete Container | The task is `COMPLETED` (perhaps it was cancelled) - delete the container
`INITIALIZING|CREATED|RUNNING` | `RESOLVING` | Delete Container | The task is `RESOLVING` (perhaps it was cancelled) - delete the container
`COMPLETED` | No Task | Delete Container | The Task has been cancelled - don't worry about it
`COMPLETED` | `PENDING` | CAS the Task to `COMPLETED` then Delete Container | make sure BBS knows we've completed the Task
`COMPLETED` | `RUNNING on α` | CAS the Task to `COMPLETED` then Delete Container | make sure BBS knows we've completed the Task
`COMPLETED` | `RUNNING on ω` | Delete Container | Apparently this Task is running somewhere else!
`COMPLETED` | `COMPLETED` | Delete Container | The task has already been marked `COMPLETED` - delete the container
`COMPLETED` | `RESOLVING` | Delete Container | The task has already been marked `RESOLVING` (perhaps it was cancelled) - delete the container
No Container | `RUNNING on α` | CAS to `COMPLETED` and `Failed` | Diego thinks α is running the instance, but it is not

Some notes:
- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell.  Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible.  However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence.  An exhaustive analysis of all possible combinations (such as this) ensures more safety around the Task lifecycle.
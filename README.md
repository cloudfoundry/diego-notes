# Diego Dev Notes

This describes in detail the Diego state machine for running Tasks and LRPs.

This document is not concerned with:

- The BBS API (documented [here](https://godoc.org/code.cloudfoundry.org/bbs#ExternalClient) and [here](https://godoc.org/code.cloudfoundry.org/bbs#Client)).
- Details around how the Executor runs Actions.

## Cell Organelles: The Rep-Executor(-Garden) Cycle

The Rep is responsible for representing the containers in Garden to BBS.
The Rep gets updates about container status by **polling** the executor and by listening for **events**.
In both cases, the Rep ends up with a container object that it must reconcile with the BBS.
It does this by either:

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
`CREATED` | A container has been created and the executor is starting to run the Setup/Action/Monitor actions. If there is no Monitor action the container will transition to `RUNNING` almost immediately. If there is a Monitor action it will only transition when the Monitor says the container is ready.
`RUNNING` | The container's Action action is running and will be actively monitored through the Monitor action.
`COMPLETED` | The container is no longer running. `container.Result` will include details around whether the work the container performed succeeded or failed, or if the container was explicitly stopped.

Allowed transitions:

- `RESERVED -> INITIALIZING`
- `INITIALIZING -> CREATED | COMPLETED`
- `CREATED -> COMPLETED | RUNNING`
- `RUNNING -> COMPLETED`

It is the Rep's responsibility to delete containers.
Containers can be deleted regardless of current state.

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

The rep should handle events exactly as it handles polling.
Given a container in a particular state, reconcile with the BBS.
The events are simply an optimization on top of polling.

#### Polling vs Events

The Rep keeps the BBS up-to-date via two mechanisms: polling and events.
This is complicated by the fact that events can come in while the poller is *fetching* state from its two sources (BBS & Executor).

As a result, when polling the Rep may have a stale copy of the containers and/or a stale copy of the BBS.
To be safe it should always validate a decision it is about to make.
This is *especially* important when deleting an `INITIALIZING/CREATED/RUNNING` container.

## LRPs

All things LRP. Here's an outline:

- **DesiredLRPs** discusses the DesiredLRP side of things.
- **ActualLRPs** discusses the various states that ActualLRPs can be in.
- **LRP Lifecycle** An overview
- **Computing ∆s** discusses the rules used to determine how Desired and Actual ought to be reconciled. The BBS does this.
- **Starting/Stopping ActualLRPs** discusses how start and stop requests are handled. The BBS can request stops. The BBS and Rep can request starts.
- **Distributing ActualLRPs** discusses how the Auctioneer distributes LRPs.
- **Crashes** discusses the lifecycle associated with a crashing ActualLRP. The Rep and BBS play a role here.
- **Evacuation** discusses how a Rep evacuates. The Rep and the BBS play a role here.
- **Harmonizing DesiredLRPs with Actual LRPs: BBS** discusses the BBS's responsibilities.
- **Harmonizing ActualLRPs with Container State: Rep** outlines (exhaustively) how the Rep keeps the ActualLRPs in the BBS in sync with the set of running containers on its Cell.
- **Detecting Cell Failure** describes the defenses Diego has in place to handle various types of Cell failure.
- **Pruning Corrupt/Invalid Data** describes how Diego deals with corrupt LRP data, or old data that has become invalid (this is relevant until we have stable versions for our data schema).

### DesiredLRPs

- `DesiredLRP`s can be created and deleted freely.
- `DesiredLRP`s can be updated freely, but only a subset of fields may be updated (`instances`, `routes`, `annotation`). This subset corresponds to changes that can be accommodated *without* restarting any ActualLRPs.
- Consumers can **only** modify `DesiredLRP`s through the BBS API. Therefore, only the BBS is allowed to manipulate a `DesiredLRP`.
- `DesiredLRP`s are organized into domains - arbitrary groupings for the consumer's convenience. Domains have a notion of "freshness" meaning that the Desired state is up-to-date. The BBS avoids taking incorrect destructive actions by only shutting down undesired ActualLRPs if the Desired state is "fresh" (i.e. up-to-date).

### ActualLRPs

Components modify ActualLRPs via calls to the BBS API; only the BBS may modify ActualLRLPs directly.
Diego attempts to keep one ActualLRP running per desired index.
To ensure this we use the BBS to walk an individual ActualLRP through a state machine.

The BBS stores the ActualLRP in the `actual_lrps` table.
The table has a composite primary key of three columns: identifiers `process_guid` and `index`, and an enum value called `presence` that describes the state of the cell hosting that ActualLRP.
`presence` can take on one of the three values below:

| Presence | Enum value | Description |
| -------- | ---------- | ----------- |
| `ORDINARY` (default) | 0 | The cell hosting the ActualLRP is operating normally. |
| `EVACUATING` | 1 | The cell hosting the ActualLRP has begun the process of either shutting down or getting updated, so all LRPs on the cell need to be restarted on another cell. Hence, the LRP in question is about to be moved. |
| `SUSPECT` | 2 | The cell's presence cannot be verified by the BBS. All LRPs present on that cell will be considered suspect until the rep can re-establish connection with the BBS. |

In normal operation, the ActualLRP will be stored in the table with the `presence` field set to `ORDINARY`.
When the cell is evacuating, the BBS creates a duplicate ActualLRP in the table with the `presence` field set to `EVACUATING`.
When the cell is suspect, the BBS creates an unclaimed duplicate of the ActualLRP in the table and sets the `presence` field for the existing ActualLRP to `SUSPECT`. The suspect ActualLRP is dealt with during LRP convergence (see Convergence for details).

Indices are in the range `0..N-1` where `N` is the number of desired instances specified in the DesiredLRP.

Here is the ActualLRP state machine:

State | Meaning
------|--------
`UNCLAIMED`| The ActualLRP is being scheduled by an auction
`CLAIMED` | The ActualLRP has been assigned to a Cell and is being started
`RUNNING` | The ActualLRP is running on a Cell and is ready to receive traffic/work.
`CRASHED` | The ActualLRP has crashed and is no longer on a Cell. It should be restarted (eventually).

In addition, the ActualLRP includes two pieces of data: `CrashCount` (keeping track of the number of crashes) and `Since` (keeping track of the time when `State` was last updated).
These are used to implement a backoff policy when restarting crashes.

An evacuating ActualLRP will always be in the `RUNNING` state. So will the suspect ActualLRP.

### LRP Lifecycle

Here is a happy path overview of the LRP lifecycle, just to set the stage.
Many more details will follow.

- A DesiredLRP is created/modified by an incoming request to the BBS
- The BBS compares the DesiredLRP to the Actual state and computes a ∆
- Starts are requested and sent to the Auctioneer
	- The Auctioneer commits a given cell to start an ActualLRP
	- The Rep starts the ActualLRP and updates its state in the BBS
- When stops are sent, they are sent directly to the Reps
	- The Rep stops the ActualLRP and updates its state in the BBS

### Computing ∆s

The BBS needs to compute ∆s between Desired and Actual state.
It does this when a new/updated DesiredLRP comes in, and during the convergence process.
Here's how ∆s are computed:

Given a DesiredLRP that requests `N` instances.

And a set of ActualLRPs associated with that DesiredLRP.

- If an ActualLRP has `index` > `N-1`: Stop the ActualLRP (see below)
- If there is no ActualLRP for some `index` `< N`: Create and start the ActualLRP (see below)

Note: the *state* of the ActualLRP is irrelevant.

### Starting/Stopping ActualLRPs

The following actions can be taken to start/stop an ActualLRP.

#### Starting ActualLRPs

When an ActualLRP needs to be started or restarted (in the case of crashes/evacuations):

- if starting a missing/new ActualLRP:
	- an UNCLAIMED ActualLRP is CREATED *(if this fails: do nothing if the record already exists, there's already an ActualLRP there which means Diego is on it. Otherwise the convergence bulk process will eventually attempt to recreate the record)*.
- if restarting a crashed ActualLRP:
	- Update the ActualLRP record from CRASHED to UNCLAIMED *(if this fails: do nothing if the record already exists, it means the ActualLRP is already being restarted)*.
- in both cases: upon success, a start is sent to the Auctioneer.
- the Auctioneer picks a Rep *(if this fails: the Auctioneer updates the `PlacementError` on the ActualLRP and another start will be issued by the BBS)*.
- the Rep is told to start the ActualLRP.
- the Rep creates a container reservation *(if this fails: the Auctioneer updates the `PlacementError` on the ActualLRP and another start request will be issued by the BBS)*.
- upon success, the Rep responds to the Auctioneer successfully (this ensure the Auctioneer isn't kept waiting).
- the Rep then tells the BBS to update the ActualLRP from `UNCLAIMED` to `CLAIMED`  *(upon failure: the Rep aborts and cleans up the reservation and we rely on the BBS's convergence process to try again)*.
- upon success, the Rep then starts running the container *(upon failure: the Rep aborts, cleans up the reservation, and deletes the ActualLRP. We rely on the BBS's convergence process to try again)*.
- eventually a `ContainerRunningEvent` causes the Rep to update the LRP from CLAIMED to RUNNING.

#### Stopping ActualLRPs

When an ActualLRP should be stopped:

- for `CLAIMED` or `RUNNING` ActualLRPs:
	- a stop is sent to the corresponding Rep.
	- the Rep deletes the container *(if this fails: the rep aborts - convergence will pick this up later)*
	- the Rep then removes the ActualLRP from the BBS *(if this fails: the rep aborts - the rep will retry later in its polling loop)*
- for `UNCLAIMED` or `CRASHED` ActualLRPs:
 	- delete the ActualLRP *(if this fails: convergence will pick it up later)*

### Distributing ActualLRPs: Auctioneer

The Auctioneer is responsible for distributing ActualLRPs optimally and efficiently.
There is only one Auctioneer ever handling requests.
Requests are handled on demand: when a request to start work (an ActualLRP or Task) arrives the Auctioneer immediately fetches state from all Cells, picks the optimal placement, and then tells the corresponding Cell to perform its assigned work.

Doing this for each individual piece of work would incur a large communication overhead.
As such the entire flow of information around starting Tasks and LRPs supports batching.
There are two notions of "batch":

1. When the BBS knows it must request many starts, it submits a batch request to the Auctioneer. For example, if the BBS needs to scale an application to 100 instances it submits a single batched request to the Auctioneer. Similarly if the BBS needs to start a number of missing LRPs, it sends a single batch request to the Auctioneer. The Auctioneer in turn submits work to Reps in batches.
2. Work sent to the Auctioneer is added to the next batch of work. The entire batch is scheduled on the next scheduling loop. Batches are heterogeneous and include both ActualLRPs and Tasks. Batches are de-duped before being scheduled -- so identical units of work in a single batch will not be scheduled multiple times.

Here are some details around the scheduling loop.
When a new ActualLRP arrives:

- the Auctioneer fetches the state of each Cell.  This includes information about the available capacity on the Cell, the Cell's stack, and the set of ActualLRPs currently running on the Cell.
- the Auctioneer then distributes the ActualLRPs across the Cells (in-memory).  It ensures that ActualLRPs are only placed on Cells with matching `stack`s that have sufficient resources to host the ActualLRP.  Once the Auctioneer identifies the set of Cells matching this criteria it choses the winning Cell by optimizing for (in increasing priority):
	- an even distribution of memory, disk, and container usage across Cells (all with equal weighting)
	- minimizing colocation of ActualLRP instances of a given DesiredLRP on the same Cell (takes precedence over the distribution of memory/disk/etc.).
	- minimizing colocation of ActualLRP instances of a given DesiredLRP in the same Availability Zone
- the Auctioneer then submits the allocated work to all Cells
	- any work that could not be allocated is carried over into the next batch
	- if the Cell responds saying that the work could not be performed, the auctioneer carries the failed work over into the next batch
	- if the Cell *fails to respond* (e.g. a connection timeout elapses), the auctioneer *does **not*** carry the work over into the next batch.  This is a case of partial failure and the auctioneer defers to the BBS to figure out what to do.
- any work carried over into the next batch is merged in with work that came in during the previous round of scheduling and the auction repeats the scheduling loop

#### Auction Work Prioritization

When the Auctioneer receives a batch of work it must decide the order in which the work must be distributed.
This is important.
When scheduling heterogeneous work the Auctioneer could, incorrectly, fill the Cells with small units of work before attempting to place large units of work.
As a result, these large units may fail to place even if - in principal - there is sufficient capacity in the cluster.
However, naively prioritizing large units of work over small units of work could lead to a situation in which, for example, a single large application fills the cells and prevents a smaller application from having *any* instances running.

To mitigate these issue the Auctioneer first sorts the batch of work it is operating on in the following order (high priority to low priority):

1. Index 0 LRPs
2. Tasks
3. Index 1 LRPs
4. Index 2 LRPs
5. etc...

Within each priority group, work is sorted in order of decreasing memory (so larger units of work are placed before smaller units of work).

It is important to note that this prioritization only applies within an Auctioneer's single batch of work.
Since applications can arrive in arbitrary order the effectiveness of this scheme is somewhat limited.
However, since communication around starts is generally distributed in batches, this prioritization will apply when (for example) a cell is evacuating or during convergence when the BBS is submitting a batch of work.

#### Communicating Fullness

When an ActualLRP cannot be placed because there are no resources to place it, the Auctioneer leaves the ActualLRP in the `UNCLAIMED` state and sets the `PlacementError` field.
Whenever the ActualLRP transitions out of the `UNCLAIMED` state the `PlacementError` should be cleared.

There are multiple reasons why an ActualLRP may fail to be placed; these should be communicated to the user.
For example, in the presence of placement pool rules it is possible that the Auctioneer simply cannot find appropriate host Cells (`PlacementError="found no compatible cells"`).
Alternatively, the Auctioneer may *find* appropriate Cells but all those Cells might be full (`PlacementError="insufficient resources"`).

Diego continues to attempt to schedule `UNCLAIMED` ActualLRPs.
Should an operator add spare capacity, Diego will automatically schedule the ActualLRPs.

### Crashes

When an ActualLRP instance crashes, Diego is responsible for restarting the instance.
There is a tension, however, between restarting a failed instance immediately and overloading the system with endless immediate restarts of instances that never succeed.

To strike this balance Diego immediately restarts a crashed instance up to 3 times.
We perform immediate restarts for a few reasons:
- they make for good demos (look it crashed -- look it's back!)
- they handle cases where a healthy well-written application has truly crashed and should be restarted ASAP.

Sometimes, however, an instance crashes because some external dependency is down.
In these cases it makes more sense for Diego to wait (and alleviate the strain of a thrashing instance hopping from one Cell to another) between restarts.
So for crashes 3 - 8 Diego applies an exponential backoff in the wait time between restarts.

Finally, we've learned from existing large public installations (PWS, Bluemix) that the vast majority of crashed instances in the system are poorly-written instances that never even manage to come up.
Instead of endlessly restarting them (which puts a substantial strain on the system) Diego will (eventually) give up on these instances.

All that remains is the question of how we reset the exponential backoff.
Instances that thrash and place a heavy load on the system typically crash quickly.
So we apply a simple heuristic: if an instance manages to stay running for > 5 minutes (defined below), we reset its crash count.

When the container associated with an ActualLRP enters the `COMPLETED` state the Rep takes actions to ensure the ActualLRP gets restarted.
Let's call this the `RepCrashDance`:

- If `CrashCount` < 3:
	-  Increment the `CrashCount` and bump `Since`, udpate the ActualLRP state to `UNCLAIMED`, and emit a start to the Auctioneer.
- If `CrashCount` > 3:
	- Increment the `CrashCount` and bump `Since`, update the ActualLRP state to `CRASHED`.

The `CRASHED` ActualLRP is eventually restarted by the BBS.
The wait time is computed like so:

- If `CrashCount < 8`
	- the BBS should restart the ActualLRP `N` seconds after `Since` (exponential backoff in [this story](https://www.pivotaltracker.com/story/show/83638710))
- If `CrashCount >= 8`
	- the BBS should restart the ActualLRP `MaxWaitTime = 16` minutes after `Since`
- If `CrashCount > 200`
	- the ActualLRP is never restarted

It is important that the `CrashCount` be reset eventually.
The BBS does this when the Rep marks an ActualLRP as crashed:

- If, at the time the crash occurs, the ActualLRP in the BBS is in the `RUNNING` state with a `Since` time that is `>= 5 minutes` ago:
	- Reset the `CrashCount` to 0 and the then do the `RepCrashDance`
- Otherwise
	- Do the `RepCrashDance`


> Note: the Rep is responsible for immediately restarting ActualLRPs that have `CrashCount < 3`. These ActualLRPs never enter the `CRASHED` state. The BBS is responsible for restarting ActualLRPs that are in the `CRASHED` state.

### LRP Evacuation

When a Cell must be evacuated its ActualLRPs must first transfer to another Cell.
Here's how this works.

#### The Rep's Role during Evacuation

After the rep is signaled to evacuate, it communicates the evacuating state to the rest of its components and periodically attempts to resolve its containers.
For the details of all the resolution cases, consult the ["Harmonizing during evacuation"](#harmonizing-during-evacuation) section below.
The rep shuts down once all of its containers have been destroyed or it reaches its evacuation timeout.

#### The BBS's Role during Evacuation

When queried for ActualLRPs the BBS follows the following rules:

- If there is no `evacuating` instance, always return the `instance` instance.
- If there *is* an `evacuating` instance, return it if:
	+ There is no `instance` instance
	+ The `instance` instance is `UNCLAIMED` or `CLAIMED` (to be clear: if `instance` is `RUNNING` or `CRASHED`, return `instance` instead of `evacuating`)

When performing convergence, the BBS pays no attention to evacuating ActualLRPs.

### Harmonizing DesiredLRPs with ActualLRPs: Convergence

Messages get lost.
Connections time out.
Components fail.
Bugs happen.

*Convergence* is the process of bringing Diego into eventual consistency in the face of these eventualities.
The BBS periodically performs convergence, striving to bring the actual state (the set of ActualLRPs) into consistency with the desired state (DesiredLRPs).

Here are it's responsibilities and the actions it takes:

1. *Reaping ActualLRPs on failed Cells:*
	- Cells periodically maintain their presence in the BBS.
	        - Missing Cell presence could be a result of network jitters or other temporary network failures.
		- A ActualLRP on a cell that has not maintained presence must be replaced.
			- In favor of keeping routability to an already running ActualLRP, the BBS marks the presence for an ActualLRP on a missing Cell as `SUSPECT`.
			- At the same time, the BBS starts a backup ActualLRP to take over the suspect ActualLRP in case the missing Cell never reappears.
	- In addition to polling periodically, the BBS actively watches for Cells disappearing. When a Cell goes away, convergence is triggered immediately. This was covered in #11.
2. *Starting missing ActualLRPs:*
	- If there is no ActualLRP in the BBS for a particular DesiredLRP index, it creates an `UNCLAIMED` ActualLRP and requests a start.
3. *Stopping extra ActualLRPs:*
	- If there is an ActualLRP (in any state) that does not correspond to a DesiredLRP index, the BBS stops the ActualLRP (see above).
	- This action is *only* taken if ActualLRP's Domain must be fresh.
4. *Restarting `CRASHED` ActualLRPs:*
	- It is the BBS's responsibility to restart `CRASHED` ActualLRPs (see above).
5. *Re-emitting start requests:*
	- The request to start the ActualLRP may have failed. This can be detected if the ActualLRP remains in the `UNCLAIMED` state for longer than some timescale. The BBS acts by resubmitting a start request.

### Harmonizing ActualLRPs based on Cell Presence: Suspect Cells

As mentioned in the previous section, when a Cell fails to register its presence with Locket, BBS marks ActualLRPs assigned to that Cell as `suspect`.
Rather than aggressively trying to prune out suspect ActualLRPs and potentially causing loss of routability to instances that may still be running, the BBS gives the missing Cell some time to recover its presence while working on a contingency plan for a replacement instance.
Here is how it works:

- During convergence, the BBS detects that a Cell has missed a heartbeat with the Locket.
- BBS fetches the list of ActualLRPs on the missing Cell and sets their `presence` to `SUSPECT`.
- For each suspect ActualLRP from the above, the BBS creates a new `UNCLAIMED` ActualLRP and requests Auction.
- While the new ActualLRP is auctioned and placed on a healthy Cell:
	- if the `suspect` Cell reappears and registers its presence with the BBS, the BBS picks up the reappeared Cell in the convergence loop, removes the replacement ActualLRP, and switches `presence` for the suspect ActualLRP from `SUSPECT` back to `ORDINARY`.
	- if a new Cell takes on the ActualLRP with the given `process_guid` and `index` and starts it, BBS removes the corresponding `suspect` ActualLRP.

The tables below shows how BBS handles transitions between the suspect ActualLRP and the replacement ActualLRP and what events are emitted.

_Notes for the Tables_:
- `i1` and `i2` refer to `instance_guids` of two ActualLRPs with identical `process_guid` and `index` and different `presence` types.
- `i1` and `i2` have containers running on two separate Cells.
- `i1` refers to the potentially suspect LRP, and `i2` refers to the replacement LRP.
- Cell `cell-1` runs the container corresponding to ActualLRP `i1`.
- Cell `cell-2` runs the container corresponding to ActualLRP `i2`.
- For each instance `i1` and `i2`, both its `State` and `Presence` are listed before and after a BBS harmonizing event.
- About Events:
  - ActualLRPCreatedEvent: This event gets triggered to signal the addition of a route to an ActualLRP
  - ActualLRPRemovedEvent: This events gets triggered in order to signal the redaction of a route to an ActualLRP
  - ActualLRPInstanceCreatedEvent: This event gets triggered to signal the creation of a new ActualLRP instance
  - ActualLRPInstanceChangedEvent: This event gets triggered whenever any ActualLRP instance changes its state or presence
  - ActualLRPInstanceRemovedEvent: This event gets triggered when the ActualLRP instance is removed


#### Expected Suspect ActualLRP Workflow

<table>
  <thead>
    <tr>
      <th>ActualLRP State & Presence</th>
      <th>Cell presence</th>
      <th>Event</th>
      <th>Resulting ActualLRP State/Presence and Group Events & Instance Events</th>
      <th>Reason</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
	<p><code>i1</code>: <code>RUNNING ORDINARY</code></p>
	<p><code>i2</code>: N/A</p>
      </td>
      <td>
	<p><code>cell-1</code>: <code>Present</code></p>
	<p><code>cell-2</code>: <code>N/A</code>, since <code>i2</code> is not running</p>
      </td>
      <td>As part of convergence, <code>cell-1</code> changes presence:<br/><code>Present</code> => <code>Missing</code></td>
      <td>
	<p><code>i1</code>: <code>RUNNING SUSPECT</code></p>
	<p><code>i2</code>: <code>UNCLAIMED ORDINARY</code>
	ActualLRPInstanceCreatedEvent(<code>i1</code>)</p>
      </td>
      <td>
	<p>
	  When the cell fails to establish presence, the BBS marks all instances
	  on that cell as <code>SUSPECT</code> because the cell may or may not be
	  functioning.
	</p>
	<p>
	  Replacement instances are put up for auction so
	  they can be started on other cells.
	</p>
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code><br><br><code>i2</code>: <code>UNCLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> : <code>Missing</code><br><br><code>cell-2</code>: <code>Present</code>
      </td>
      <td>
	<code>cell-2</code> calls <code>ClaimActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code><br><br> <code>i2</code>: <code>CLAIMED ORDINARY</code><br>
	ActualLRPInstanceChangedEvent(Before: <code>UNCLAIMED i2</code>, After: <code>CLAIMED i2</code>
      </td>
      <td>
	As part of the normal auction process, the available <code>cell-2</code>
	claims the replacement ActualLRP so that the <code>SUSPECT</code>
	ActualLRP can eventually be shut down. In the meantime, the
	<code>SUSPECT</code> instance on <code>cell-1</code> continues running to
	maintain routability.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code><br><br><code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> : <code>Missing</code><br><br><code>cell-2</code>: <code>Present</code>
      </td>
      <td>
	<code>cell-2</code> calls <code>StartActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>REMOVED</code><br/>
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>) <br/>
	ActualLRPInstanceRemovedEvent(<code>i1</code>)
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>,<br/>
	  ActualLRPCreatedEvent(Instance: <code>i2</code>, Evacuating: <code>nil</code>) <br/>
	  ActualLRPInstanceChangedEvent(Before: <code>CLAIMED i2</code>, After: <code>RUNNING i2</code>)
      </td>
      <td>
	Once <code>cell-2</code> starts a replacement ActualLRP, the
	<code>SUSPECT</code> instance can be removed from the missing cell and
	avoid any routing downtime.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>CLAIMED ORDINARY</code><br/><br/>
	<code>i2</code>: N/A</td>
      <td>
	<code>cell-1</code>: <code>Present</code><br/><br/>
	<code>cell-2</code>: <code>N/A</code>, since <code>i2</code> is not running</td>
	<td>As part of convergence, <code>cell-1</code> changes presence:<br><code>Present</code> => <code>Missing</code>
      </td>
      <td>
	<code>i1</code>: <code>CLAIMED SUSPECT</code>
	ActualLRPInstanceChangedEvent(Before: <code>ORDINARY i1</code>, After: <code>SUSPECT i1</code>)
	<br><br>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code>
	ActualLRPInstanceCreatedEvent(<code>i1</code>)
      </td>
      <td>
	When the cell fails to establish presence, the BBS mark all instances
	on that cell as <code>SUSPECT</code> because the cell may or may not be
	functioning.<br><br>Replacement instances are put up for auction so they
	can be started on other cells.
      </td>
    </tr>
  </tbody>
<table>


#### When `cell-1` Re-Establishes Presence

<table>
  <thead>
    <tr>
      <th>ActualLRP State & Presence</th>
      <th>Cell presence</th>
      <th>Event</th>
      <th>Resulting ActualLRP State/Presence and Group Events & Instance Events</th>
      <th>Reason</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code> or <br><code>CLIAMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code>: <code>Missing</code>
	<br/><br/>
	<code>cell-2</code>: <code>Present</code>
      </td>
      <td>
	As part of convergence, <code>cell-1</code> changes presence: <br/><code>Missing</code> => <code>Present</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING ORDINARY</code>
	ActualLRPInstanceChangedEvent(Before: <code>i1 SUSPECT</code>, After: <code>i1 ORDINARY</code>)
	<br/><br/>
	<code>i2</code>: <code>REMOVED</code>
	ActualLRPInstanceRemovedEvent(<code>i2</code>)
      </td>
      <td>The original cell re-establishes presence before the replacement instance has been started, so BBS can remove the replacement instance</td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code>: <code>Missing</code> 
	<br/><br/>
	<code>cell-2</code>: <code>Present</code>
      </td>
      <td>
	As part of convergence, <code>cell-1</code> changes presence: <br><code>Missing</code> => <code>Present</code>
      </td>
      <td>
	<code>i1</code>: <code>REMOVED</code>
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>)
	ActualLRPInstanceRemovedEvent(<code>i1</code>)
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>As above, the replacement instance takes over because it is successfully running.</td>
    </tr>
  </tbody>
</table>

#### Failing to Start the Replacement ActualLRP
In these scenarios, `cell-1` is `Missing` and `cell-2` is `Present`. Thus, `i1`, the ActualLRP running on `cell-1`, always has state `RUNNING SUSPECT`. The replacement instance `i2` may be either `UNCLAIMED ORDINARY` or `CLAIMED ORDINARY`.

<table>
  <thead>
    <tr>
      <th><code>i2</code> State & Presence</th>
      <th>Harmonizing Event</th>
      <th>Resulting ActualLRP State/Presence and Group Events & Instance Events</th>
      <th>Reason</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code>
      </td>
      <td>
	Auctioneer calls <code>FailActualLRP(i2)</code> because it cannot place <code>i2</code>
      </td>
      <td>
	<code>i1</code> : <code>RUNNING SUSPECT</code> 
	<br/><br/>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code>
      </td>
      <td>
	If the auctioneer fails to place the replacement instance, the BBS
	prefers to keep the <code>RUNNING SUSPECT</code> LRP, until a replacement
	instance is successfully started somewhere as a result of future BBS
	convergence.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-2</code> calls <code>CrashActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CRASHED ORDINARY</code>
	ActualLRPInstanceChangedEvent (<code>i2 CLAIMED</code>, <code>i2 CRASHED</code>)
      </td>
      <td>
	In this scenario, <code>cell-2</code> claimed the replacement
	ActualLRP, but the ActualLRP failed to come up.
	So, the <code>SUSPECT</code> ActualLRP should remain intact so
	traffic can be routed to it, and the replacement ActualLRP should be
	re-auctioned. The instance events reflect the change on the state of the crashed actual lrp. 
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-2</code> calls <code>RemoveActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code>
	ActualLRPInstanceRemovedEvent (<code>i2 CLAIMED</code>)
	ActualLRPInstanceCreatedEvent (<code>i2 UNCLAIMED</code>)
      </td>
      <td>
	In this scenario, <code>cell-2</code> claimed the replacement
	ActualLRP, but the ActualLRP needed to be
	stopped. So, the <code>SUSPECT</code> ActualLRP should remain intact so
	traffic can be routed to it, and the replacement ActualLRP should be
	re-auctioned. In this case a <code>ActualLRPInstanceRemovedEvent</code> is emitted when the
	replacement is removed. Also an <code>ActualLRPInstanceCreatedEvent</code> is emitted when a 
	replacement <code>UNCLAIMED</code> instance is created.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-2</code> calls <code>EvacuateClaimedActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
        <code>i2</code>: <code>UNCLAIMED ORDINARY</code>
	ActualLRPInstanceRemovedEvent (<code>i2 CLAIMED</code>)
	ActualLRPInstanceCreatedEvent (<code>i2 UNCLAIMED</code>)
      </td>
      <td>
	In this scenario, <code>cell-2</code> claimed the replacement
	ActualLRP, but the cell began evacuation before the replace instance
	could be started. So, the <code>SUSPECT</code> ActualLRP should remain
	intact so traffic can be routed to it, and the replacement ActualLRP
	should be re-auctioned.
      </td>
    </tr>
  </tbody>
</table>

#### BBS Convergence Edge Cases
If, during the normal convergence process, the BBS finds the ActualLRP or Cell presence in unexpected states, it handles those edge cases in the following way:

<table>
  <thead>
    <tr>
      <th>ActualLRP State & Presence</th>
      <th>Cell presence transition</th>
      <th>Resulting ActualLRP State/Presence and Group Events & Instance Events</th>
      <th>Reason</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code>: <code>Missing</code>
	<br/><br/>
	<code>cell-2</code>: <code>Present</code>
      </td>
      <td>
	<code>i1</code>: <code>REMOVED</code>
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>) 
	ActualLRPInstanceRemovedEvent(<code>i1</code>) 
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>
	Because the replacement instance has been successfully started, the
	original instance (which is still considered suspect) can be removed in
	favor of the functioning AcutualLRP instance.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>UNCLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code>: <code>Missing</code> 
	<br/><br/>
	<code>cell-2</code>: <code>Present</code> => <code>Missing</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code> 
	<br/><br/>
	<code>i2</code>: <code>REMOVED</code>
	ActualLRPInstanceRemovedEvent(<code>i2</code>) 
      </td>
      <td>
	If the cell hosting the replacement ActualLRP also goes missing, BBS
	will prefer the original <code>RUNNING SUSPECT</code> ActualLRP, because
	there's a chance it's still running (which is better than not running at
	all).
      </td>
    </tr>
  </tbody>
</table>

#### The missing `cell-1` making API calls to BBS
Even though the cell has failed to establish presence via `locket`, it may still issue API requests to the BBS.
In the following examples, you should assume that `cell-1` is missing and `cell-2` is present.
As such, the first instance `i1` is has state/presence `RUNNING SUSPECT` in each scenario described below.

<table>
  <thead>
    <tr>
      <th><code>i2</code> State & Presence</th>
      <th>Harmonizing Event</th>
      <th>Resulting ActualLRP State/Presence and Group Events & Instance Events</th>
      <th>Reason</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> calls <code>StartActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br><br> 
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	Even though the cell has made an API call to BBS, if the cell has failed
	to establish presence via <code>locket</code>, it should continue to
	treat the ActualLRP on <code>cell-1</code> as suspect. As such, it should
	continue allowing the replacement ActualLRP to proceed being placed on a
	new rep.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> calls <code>StartActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code> 
	<br><br>
	<code>i2</code>: <code>RUNNING ORDINARY</code>
      </td>
      <td>
	BBS returns <code>ErrorActualLRPCannotBeStarted</code> because
	<code>cell-2</code> is already running the ActualLRP. <code>i1</code>
	remains <code>RUNNING SUSPECT</code>, but will be removed later as part
	of destroying the instance.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> calls <code>CrashActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>REMOVED</code>, 
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>) 
	ActualLRPInstanceRemovedEvent (<code>i1</code>)
	<br><br> 
	<code>i2</code>: <code>CLAIMED ORDINARY</code>,
	ActualLRPCreatedEvent(Instance: <code>i2</code>, Evacuating: <code>nil</code>)
      </td>
      <td>
	When the cell reports the ActualLRP has crashed, there are no longer any
	running ActualLRPs on either cell. Because <code>cell-1</code> is still
	considered suspect, the BBS can move forward with emitting a created
	event for the replacement in the <code>CLAIMED</code> state since it must
	take over from the failed instance. Not that we do not emit any extra 
	<code>ActualLRPInstanceCreatedEvent</code> for <code>i2</code> because there 
	is no change in the state of <code>i2</code> and we already emitted a create
	event for the instance when the <code>UNCLAIMED</code> instance was created.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
      <code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
      <code>cell-1</code> calls <code>EvacuateRunningActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>RUNNING EVACUATING</code>,
	ActualLRPCreatedEvent(Instance: <code>nil</code>, Evacuating: <code>i1</code>), 
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>)
	ActualLRPInstanceChangedEvent (<code>i1 RUNNING</code>, <code>i1 EVACUATING</code>)
	<br><br> 
	<code>i2</code>: <code>CLAIMED ORDINARY</code>, 
	ActualLRPCreatedEvent(Instance: <code>i2</code>, Evacuating: <code>nil</code>)
      </td>
      <td>
	When the suspect cell reports that a running ActualLRP needs to
	evacuate, the BBS needs to do a few things to maintain routability and
	a smooth transition to a replacement instance. The suspect instance is
	transitioned to evacuating and the appropriate created and removed
	events are emitted. This should keep routing traffic to the instance,
	because the ActualLRP may still be running. The BBS also, emits an
	<code>ActualLRPCreated</code> event for the replacement
	<code>CLAIMED</code> instance so that it can take over from the now
	evacuating one as per the usual <a
	href="#lrp-evacuation">evacuation</a> strategy.
      </td>
    </tr>
    <tr>
      <td>
	<code>i1</code>: <code>RUNNING SUSPECT</code>
	<br/><br/>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	<code>cell-1</code> calls <code>EvacuateCrashedActualLRP</code> or <code>EvacuateStoppedActualLRP</code>
      </td>
      <td>
	<code>i1</code>: <code>REMOVED</code>, 
	ActualLRPRemovedEvent(Instance: <code>i1</code>, Evacuating: <code>nil</code>)
	ActualLRPInstanceRemovedEvent (<code>i1</code>)
	<br><br>
	<code>i2</code>: <code>CLAIMED ORDINARY</code>
      </td>
      <td>
	When the cell reports that a crashed or stopped ActualLRP needs to
	evacuate, the BBS learns that the ActualLRP that was previously known
	to be <code>RUNNING</code> is now <code>STOPPED</code> or
	<code>CRASHED</code> instead, which means that the instance is not
	running or routable. Thus, the BBS can simply remove the ActualLRP
	(rather than try to evacuate it).
      </td>
    </tr>
  </tbody>
</table>

### Harmonizing ActualLRPs with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.
It does this by periodically fetching containers from the executor and taking actions.
Here is an outline of how the Rep should act to reconcile the ActualLRPs in the BBS with the set of containers.

When performing reconciliation, the Rep compares the container associated with a `ProcessGuid` and `Index` with the entry in the BBS for that `ProcessGuid` and `Index`.
It then reconciles the state of the container (`RESERVED`, `INITIALIZING`, `CREATED`, `RUNNING`, `COMPLETED`) with the state of the `ActualLRP` (`UNCLAIMED, CLAIMED, RUNNING, CRASHED`).
In addition it must reconcile the `InstanceGuid` and `CellID` associated with the container and the BBS (this covers cases where (e.g.) the ActualLRP at a given index ends up running on two different Cells, or (e.g.) a single Cell has multiple ActualLRPs running on it).

In the following table we describe the actions the Rep must take to bring the BBS and the containers on a given Cell into harmony.
This table is from the vantage point of a BBS running on `CellID=A` with a container with `InstanceGuid=A` -- we'll call the `(InstanceGuid=A,CellID=A)` pair `α`.
It's possible that the Rep will see some other pair of identifiers in the BBS.
We call such a pair `ω`: this could be any other pairing including `(InstanceGuid B, Cell B)`, `(InstanceGuid B, Cell A)` or `(InstanceGuid A, Cell B)` -- the same reconciliation logic applies to either case.

Container State | ActualLRP State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | No ActualLRP | Delete Container | **Conceivable**: This ActualLRP is no longer desired (maybe because the DesiredLRP was scaled down or deleted)
`RESERVED` | `UNCLAIMED` | Update to `CLAIMED α` **THEN** Run Container | **Expected**: α has a reservation for this LRP and should claim it and then run it in the container.
`RESERVED` | `CLAIMED α` | Run Container | **Conceivable**: We already claimed it and should make sure we've started running the container
`RESERVED` | `CLAIMED ω` | Delete Container | **Conceivable**: Don't run this container, since a different rep got farther than us
`RESERVED` | `RUNNING α` | Update to `CLAIMED α` **THEN** Run Container | **Inconceivable**: We shouldn't have been able to get to this state, but if we have correct the BBS and run anyway
`RESERVED` | `RUNNING ω` | Delete Container | **Conceivable**: Don't run this container, since a different rep got farther than us
`RESERVED` | `CRASHED` | Delete Container | **Conceivable**: Don't run this container
`INITIALIZING/CREATED` | No ActualLRP | Delete Container | **Conceivable**: This ActualLRP is no longer desired (maybe because the DesiredLRP was scaled down or deleted)
`INITIALIZING/CREATED` | `UNCLAIMED` | Update to `CLAIMED α` | **Inconceivable**: We should have claimed the LRP before running it, but correct the BBS if this happens
`INITIALIZING/CREATED` | `CLAIMED α` | Do Nothing | **Expected**: The Cell is starting this ActualLRP, no need to write to the BBS
`INITIALIZING/CREATED` | `CLAIMED ω` | Delete Container | **Conceivable**: The ActualLRP is starting as ω (probably on some other Cell), stop starting it on this Cell.
`INITIALIZING/CREATED` | `RUNNING α` | Update to `CLAIMED α` | **Inconceivable**: This should not be possible, but the BBS should be made to reflect the truth
`INITIALIZING/CREATED` | `RUNNING ω` | Delete Container | **Conceivable**: The ActualLRP is running elsewhere, stop starting it on this Cell
`INITIALIZING/CREATED` | `CRASHED` | Delete Container | **Conceivable**: This Cell is incorrectly starting the instance - some other Cell will pick this up later.
`RUNNING` | No ActualLRP | CREATE `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, let Diego know so it can take action appropriately (we don't allow blank ActualLRPs to shut down containers as this would lead to catastrophic fail should the BBS be accidentally purged).
`RUNNING` | `UNCLAIMED` | Update to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, no need to start it elsewhere
`RUNNING` | `CLAIMED α` | Update to `RUNNING α` & Delete `/e` | **Expected**: This Cell is running the ActualLRP. Delete the evacuating instance if any
`RUNNING` | `CLAIMED ω` | Update to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP, no need to start it elsewhere
`RUNNING` | `RUNNING α` | Do Nothing | **Expected**: This Cell is running the ActualLRP, no need to write to the BBS
`RUNNING` | `RUNNING ω` | Delete Container | **Conceivable**: The ActualLRP is running elsewhere, stop running it on this Cell
`RUNNING` | `CRASHED` | Update to `RUNNING α` | **Conceivable**: This Cell is running the ActualLRP. It's not crashed and need not be restarted.
`COMPLETED (crashed)` | No ActualLRP | Perform `RepCrashDance` then Delete Container | This Cell just saw a crash but the BBS is empty.  Perhaps BBS was accidentally purged?  In that case: update it with what we know to be true.
`COMPLETED (crashed)` | `UNCLAIMED` | Delete Container | Instance will be scheduled elsewhere
`COMPLETED (crashed)` | `CLAIMED α` | Perform `RepCrashDance` then Delete Container | Instance crashed on this Cell while starting
`COMPLETED (crashed)` | `CLAIMED ω` | Delete Container | Instance is starting elsewhere, leave it be
`COMPLETED (crashed)` | `RUNNING α` | Perform `RepCrashDance` (see Crash section above for details around resetting the CrashCount) then Delete Container | Instance crashed on this Cell while running
`COMPLETED (crashed)` | `RUNNING ω` | Delete Container | Instance is running elsewhere, leave it be
`COMPLETED (crashed)` | `CRASHED` | Delete Container | The crash has already been noted
`COMPLETED (shutdown)` | No ActualLRP | Delete Container | **Conceivable**: Nothing to be done, this Cell was asked to shut the ActualLRP down.
`COMPLETED (shutdown)` | `UNCLAIMED` | Delete Container | **Conceivable**: Nothing to be done
`COMPLETED (shutdown)` | `CLAIMED α` | Delete ActualLRP then Delete Container | **Conceivable**: This Cell was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `CLAIMED ω` | Delete Container | **Conceivable**: The Instance is starting elsewhere, leave it be
`COMPLETED (shutdown)` | `RUNNING α` | Delete ActualLRP then Delete Container | **Expected**: This Cell was told to stop and should now clean up the BBS
`COMPLETED (shutdown)` | `RUNNING ω` | Delete Container | **Conceivable**: Instance is running elsewhere, leave it be
`COMPLETED (shutdown)` | `CRASHED` | Delete Container | **Conceivable**: Nothing to do
No Container | `CLAIMED α` | Delete ActualLRP | **Conceivable**: There is no matching container, delete the ActualLRP and allow the BBS to determine whether it is still desired
No Container | `RUNNING α` | Delete ActualLRP | **Conceivable**: There is no matching container, delete the ActualLRP and allow the BBS to determine whether it is still desired

Some notes:

- `COMPLETED` comes in two flavors. `crashed` implies the container died unexpectedly. `shutdown` implies the container was asked to shut down (e.g. the Rep was sent a stop). This can be determined by looking at `RunResult.Stopped`.
- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell. Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible. However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence. An exhaustive analysis of all possible combinations (such as this) ensures eventual consistency... eventually.
- When the Action described in this table fails, the Rep should log and do nothing. In this way we defer to the next polling cycle to retry actions.

Alternate view of table above:

BBS | Reserved | I/C | Running | Shutdown | Crashed | No Container
---|---|---|---|---|---|---
Missing | Do nothing | Delete Container | Create Running | Delete Container | RCD + Delete Container |
Unclaimed | Do Nothing | Update Claimed | Update Running | Delete Container | Delete Container |
Claimed-α | Do Nothing | Do nothing | Update Running | Delete ActualLRP + Delete Container | RCD + Delete Container | Delete ActualLRP
Claimed-ω | Do Nothing | Delete Container | Update Running | Delete Container | Delete Container |
Running-α | Do Nothing | Update Claimed | Do Nothing | Delete ActualLRP + Delete Container | RCD + Delete Container | Delete ActualLRP
Running-ω | Do Nothing | Delete Container | Delete Container | Delete Container | Delete Container |
Crashed | Do Nothing | Delete Container | Update Running | Delete Container | Delete Container |

```
I/C = Initializing/Created
RCD = RepCrashDance
```

#### Harmonizing During Evacuation

- the Rep is told to evacuate.
- the Rep subsequently refuses to take on any new work:
	+ when the auctioneer requests the Rep's State, the Rep informs the Auctioneer that it is evacuating which takes it out of the pool
	+ if any work comes in from the auctioneer, the rep immediately returns it all as failed work
- the Rep then periodically attempts to resolve its remaining containers according to their container states:

##### When the container is in the Running state

Assuming a `RUNNING` container on `α`, the α-rep performs these actions, always ensuring that records with `evacuating` set have their TTL set to the evacuation timeout.
`β` and `ω` represent other cells in the cluster.
`ε` indicates that the UNCLAIMED ActualLRP has a placement error set.

```
TA(Process Guid, Instance Index, Instance Guid, CellID)
EvacuateRunningActualLRP(TA) {
  OA := lookupOrdinaryByProcessGuidAndIndex(TA.ProcessGuid, TA.Index)
  EA := lookupEvacuatingByProcessGuidAndIndex(TA.ProcessGuid, TA.Index)
  ... do something ...
}
```
TA - Target ActualLRP - LRP described by the request args
OA - Ordinary ActualLRP
EA - Evacuating ActualLRP

Evacuating ActualLRP (EA) state | Target ActualLRP (TA) state | Action | Reason
`-` | Evacuating | CREATE EA: `RUNNING-α` |
`-` | Ordinary | CREATE EA: `RUNNING-α` |

Ordinary ActualLRP (OA) state | Evacuating ActualLRP (EA) state | Action | Reason
---|---|---|---
`UNCLAIMED` | - | CREATE EA: `RUNNING-α` | **Inconceivable?**: Ensure routing to our running instance
`UNCLAIMED+ε` | - | Do Nothing | **Inconceivable?**: No one won the auction, so the evacuating container stays put while the auctioneer has another go.
`CLAIMED-α` | - | CREATE EA: `RUNNING α`, Update OA: `UNCLAIMED` | **Conceivable**: α has a RUNNING container but didn't get to update the BBS to `RUNNING` yet
`CLAIMED-ω` | - | CREATE EA: `RUNNING α` | **Inconceivable?**: Ensure routing to our running instance
`RUNNING-α` | - | CREATE EA: `RUNNING α`, Update OA: `UNCLAIMED` | **Expected**: This is the initial action during evacuation
`RUNNING-ω` | - | Delete container | **Conceivable**: The ActualLRP is now running elsewhere but the EA was removed when the instance transitioned to running state
`CRASHED` | - | Delete container | **Conceivable**: The ActualLRP is now running elsewhere but the EA was somehow lost
`-` | - | Delete container | **Conceivable**: The ActualLRP is now running elsewhere but the EA was somehow lost

`UNCLAIMED` | `RUNNING-α` | Do Nothing | **Expected**: Waiting for other rep to win auction
`UNCLAIMED` | `RUNNING-β` | Delete container | **Conceivable**: β ran our evacuated instance, then evacuated itself
`UNCLAIMED+ε` | `RUNNING-α` | Do Nothing | **Conceivable**: No one won the auction, so the evacuating container stays put while the auctioneer has another go.
`CLAIMED-α` | `RUNNING-α` | Update OA: `UNCLAIMED` | **Conceivable**: α failed to update the BBS to UNCLAIMED while evacuating
`CLAIMED-α` | `RUNNING-β` | Update EA: `RUNNING α`, Update OA: `UNCLAIMED` | **Conceivable**: β evacuated the container, α CLAIMED it, ran it and began evacuating but hasn't yet updated the BBS to `RUNNING`
`CLAIMED-ω` | `RUNNING-α` | Do Nothing | **Expected**: Waiting for ω to start running instance
`CLAIMED-ω` | `RUNNING-β` | Delete container | **Conceivable**: β ran our evacuated instance, then evacuated itself
`RUNNING-α` | `RUNNING-α` | Update OA: `UNCLAIMED` | **Conceivable**: α failed to update the BBS to UNCLAIMED while evacuating
`RUNNING-α` | `RUNNING-β` | Update EA: `RUNNING α`, Update OA: `UNCLAIMED` | **Conceivable**: β evacuated the container, α CLAIMED it, ran it, and then began evacuating
`RUNNING-ω` | `RUNNING-α` | Delete EA && Delete container | **Expected**: Cleanup after successful evacuation
`RUNNING-ω` | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, ω CLAIMED it, ran it, and then began evacuating, and then α noticed
`CRASHED` | `RUNNING-α` | Delete EA && Delete container | **Expected**: Cleanup after successful evacuation (but then the new instance crashed)
`CRASHED` | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, some rep CLAIMED it, ran it, `CRASHED` it, and then α noticed
`-` | `RUNNING-α` | Delete EA && Delete container | **Expected**: Cleanup after scaling down during evacuation
`-` | `RUNNING-β` | Delete container | **Conceivable**: β evacuated the container, the ActualLRP was scaled down, and then α noticed

##### When the container is not Running

When the instance ActualLRP changes to the `UNCLAIMED` state, the BBS will also request a new auction for it.

- In container states that are not `RUNNING` or `COMPLETED`, the rep destroys the container, deletes the evacuating ActualLRP if is `RUNNING-α`, and requests a new auction for the instance ActualLRP by updating its state to `UNCLAIMED`
- In a `COMPLETED (SHUTDOWN)` container state, the rep destroys the container and deletes the evacuating ActualLRP if is `RUNNING-α`, and deletes the instance ActualLRP as usual.
- In a `COMPLETED (CRASHED)` container state, the rep destroys the container and deletes the evacuating ActualLRP if is `RUNNING-α`, and performs the `RepCrashDance` on the instance ActualLRP as usual.

- the Rep shuts down when either all containers have been destroyed OR an evacuation timeout is exceeded:
	+ in either case, the Rep ensures that any evacuating ActualLRPs associated with it are removed and that any tasks it still has running transition to `COMPLETED`.

### Detecting Cell Failures

Cells can fail for a variety of reasons and in a variety of ways.
When a Cell fails Diego must respond by:

1. rescuing LRPs that were running on the Cell
2. ensuring that no new work gets scheduled on the failed Cell

We support handling two failure modes:

1. *When a Cell disappears.*

  Cells periodically heartbeat their `CellPresence` into Consul. If a Cell fails to report in time it is considered lost and the Cell's LRPs are rescheduled. This responsibility is handled by the BBS, which is notified immediately by Consul (via a watch) if a Cell disappears.

2. *When a Cell fails to create new containers.*

  We have seen issues crop up where a Cell may enter a bad state and consistently fail to create containers. These have typically been bugs in Garden. When this occurs existing LRPs on the Cell are likely OK, however new LRPs should not be placed on the Cell. This is a catastrophic failure mode when the Cell is relatively empty - as the auctioneer will consistently place work on the busted Cell.
  To avoid this, the Rep periodically runs a self-test health check that verifies that containers and network bridges can be created. If this self-test health check fails repeatedly, the Rep marks it self as unavailable to perform work and informs the Auctioneer of this when providing its State.

### Pruning Corrupt/Invalid Data

During convergence, the BBS fetches all ActualLRP and DesiredLRP data and pre-processes it before determining what convergence actions to take.
As part of this process, it prunes any data that has become corrupt (e.g. the raw data cannot be deserialized into an object) or invalid (an unversioned BBS schema change may make existing data invalid).
Currently this is done as part of the main convergence loop, but it is **not** critical to convergence behaviour.

## Tasks

All things Tasks.
Here's an outline:

### Task States

The lifecycle of a Task in Diego is quite different from that of an LRP.
Tasks are guaranteed to run to completion at most once: they don't "crash", they don't move during an evacuation.
Instead Tasks `COMPLETE`.
Once in the `COMPLETE` state the Tasks have a notion of whether they have failed or not.
Failed Tasks include a `FailureReason`; succesful Tasks (may) include a `Result` (the contents of a file requested by the user).

Tasks however can be retried if they get rejected. Rejection for tasks happens either if auctioning the task fails, or if the task fails during container creation but prior to the actual running of the task.
Rejections cause BBS to retry the tasks at most `task.max_retries` number of times. This value is configurable in BBS. If `task.max_retries` is reached, the task completes in a failure state.

Here are the states for the Tasks:

State | Meaning
------|--------
PENDING | The Task is being scheduled by the Auctioneer
RUNNING | The Task is running on a Cell
COMPLETED | The Task is complete
RESOLVING | The Task is being resolved by the BBS

### Task Lifecycle

Here's a happy path overview of the Task lifecycle:

- When a new Task is created by the BBS it:
	+ Creates a Task in the `PENDING` state *(if this fails: the BBS returns an unhappy status code)*.
	+ Sends a start request to the Auctioneer *(if this fails: the BBS does nothing; it will eventually resend the message during convergence)*.
- The Auctioneer picks a Cell to run the Task *(if this fails: the auctioneer tells the BBS; the BBS will either try to place the task again during the next convergence run, or mark it as `COMPLETED` and `Failed`, depending on the BBS configuration and the number of attempts to place this task)*.
- The Rep creates a container reservation (and then ACKs the Auctioneer's request) *(if this fails: the Rep tells the Auctioneer that the Task could not be started and the auctioneer adds the Task to the next batch)*.
- In its processing loop, the Rep update the Task from `PENDING` to `RUNNING` *(if this fails: the Rep deletes the reservation -- the BBS will eventually see the `PENDING` task and ask the auctioneer to try again)*.
- Upon success the Rep starts the container running *(if this fails: the Rep marks the Task `COMPLETED` and failed)*.
- When the container completes (succesfully or otherwise) the Rep update the Task to `COMPLETED` *(if the update fails, the Rep will try again in its polling cycle)*.
- The BBS updates the Task to `RESOLVING` and calls the `CompletionCallback` if present *(if the update fails it does not call the `CompletionCallback`)*.
- Upon success BBS and Rep deletes the Task *(if the delete fails the BBS does nothing: during convergence it will eventually demote the Task to `COMPLETE` and we'll try again)*.

### Distributing Tasks: Auctioneer

Tasks are distributed much like ActualLRPs.
In fact the batch of work performed by the Auctioneer includes both Tasks and ActualLRPs.
The only differences are around how Tasks are optimally placed (unsurprisingly, they are simpler).
For Tasks the Auctioneer only:

- ensures the Task lands on a Cell with matching `stack`
- ensures an even distribution of memory and disk usage across Cells

#### Communicating Fullness

When a Task cannot be allocated the Auctioneer rejects the task and the BBS retries placement of the task up to `task.max_retries` times. Once `task.max_retries` is reached, the BBS updates the Task from the `PENDING` state to the `COMPLETED` state, marking it as `Failed` and including a `FailureReason` that explains that the cluster has no capacity for the task.
It is up to the consumer to then retry the Task.

### Resolving Completed Tasks

When a Task is `COMPLETED` it is up-to the consumer to delete the Task (though the BBS will clean up Tasks that have been `COMPLETED` for a lengthy period of time).
Consumers can either poll Tasks to see them enter the `COMPLETED` state or can register a `CompletionCallback` to be told the Task has been completed:

- When polling:
	+ consumers instruct the BBS to delete the Task. This updates `COMPLETED => RESOLVING` then deletes `RESOLVING`.
- When handling the `CompletionCallback`
	+ the BBS updates `COMPLETED => RESOLVING` to indicate that it is handling resolving the Task. This is necessary to ensure that no two BBSes attempt to call the `CompletionCallback`.
	+ when the `CompletionCallback` returns, the BBS deletes the Task.

It is an error to attempt to delete a Task that is *not* in the `COMPLETED` state.

> There are a few details around retrying the `CompletionCallback` -- these are documented in the [BBS API docs](https://github.com/cloudfoundry/bbs/tree/master/doc).

### Cancelling Tasks

Tasks in the `PENDING` and `RUNNING` state can be cancelled via the BBS at any time:

- If the Task is `PENDING` or `RUNNING`:
	+ The BBS sets the Task to `COMPLETED` and `Failed` with `FailureReason = "cancelled"`.
	+ The BBS then sends a cancellation message to the Rep in question.
- The Rep's poller will react to the cancel message and delete its container.

The consumer then deletes the `COMPLETED` task.
It is an error to attempt to cancel a Task that is not in the `PENDING` or `RUNNING` states.

> There is a bit of a hole here. A user could cancel then delete a Task and then request a new Task with the same `TaskGuid` *before* the Rep notices the container should be deleted. This gap is narrowed by emitting a message to the Rep. One option would be to generate a unique InstanceGuid so that the Rep can converge correctly, but this is a relatively minor issue that is unlikely to impact CF.

### Evacuation

Tasks never migrate from one Cell to another.
They effectively block Cell evacuation.
Here's what happens when a Cell must be evacuated:

- the Rep is told to evacuate.
- the Rep subsequently refuses to take on any new work
- the Rep shuts down when either all containers have been destroyed OR an evacuation timeout is exceeded:
	- in either case, the Rep ensures that any `RUNNING` Tasks associated with it are updated to the `COMPLETED` state. These should be marked `Failed` with the `FailureReason = "timed out during cell evacuation"`

### Maintaining Consistency: Convergence

Since tasks are guaranteed to run to completion at most once, the BBS never attempts to "restart" Tasks after they have finished.
Instead its role is:

1. The BBS will update the database records of Tasks that are `RUNNING` on failed Cells.
	- Cells periodically maintain their presence in the BBS. If a Cell disappears the BBS will notice and set `RUNNING` Tasks associated with the missing Cell to `COMPLETED` and `Failed`.
2. The BBS will re-emit start requests for Tasks stuck in the `PENDING` state.
	- If a Task is stuck in the `PENDING` state and has been there for a long period of time a start request is re-emitted to the auctioneer.
3. The BBS will delete the record of a `COMPLETED` Task that has first entered the `COMPLETED` state too long ago (2 minutes ago).
4. The BBS will ask the BBS to resolve a `COMPLETED` Task that has been in the `COMPLETED` state for too long (30 seconds).
5. The BBS will set a `RESOLVING` Task to the `COMPLETED` state and notify the BBS if it has been `RESOLVING` for too long.
	- Perhaps the resolving BBS died. If a Task is stuck at `RESOLVING` for too long, the BBS gives it another change to resolve by moving it back to `COMPLETED`.

### Harmonizing Tasks with Container State: Rep

The Rep is responsible for ensuring that the BBS is kept up-to-date with what is running.
It does this periodically by fetching containers from the executor and taking actions.

Here is an outline of how the Rep should act to reconcile the Tasks in the BBS with the set of containers.
In this example, α will represent the Cell performing the reconciliation and ω will represent a different Cell (possibly empty):

Container State | Task State | Action | Reason
----------------|-----------------|--------|-------
`RESERVED` | No Task | Delete Container | **Conceivable**: The task has been cancelled, α should not act.
`RESERVED` | `PENDING` | Start Task **THEN** Run Container | **Expected**: α has a reservation for this task and should start the task and run it in the container.
`RESERVED` | `RUNNING on α` | Do Nothing | **Expected**: The container is likely about to be initialized (the Rep tells the BBS running right after it reserves). Things won't get stuck in this state because the reservation will either enter a new state, or be reaped by the executor.
`RESERVED` | `RUNNING on ω` | Delete Container | **Conceivable**: Don't start running this task
`RESERVED` | `COMPLETED on α` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `COMPLETED on ω` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `RESOLVING on α` | Delete Container | **Conceivable**: Don't start running this Task
`RESERVED` | `RESOLVING on ω` | Delete Container | **Conceivable**: Don't start running this Task
`INITIALIZING/CREATED/RUNNING` | No Task | Delete Container | **Conceivable**: The task has been cancelled, α should stop running it.
`INITIALIZING/CREATED/RUNNING` | `PENDING` | Update the Task to `RUNNING on α` | **Inconceivable**: Make sure BBS knows we're running the Task (but we should already have started the task before running the container).
`INITIALIZING/CREATED/RUNNING` | `RUNNING on α` | Do Nothing | **Expected**: BBS is up-to-date
`INITIALIZING/CREATED/RUNNING` | `RUNNING on ω` | Delete Container (and log loudly) | **Inconceivable**: Apparently this Task is running somewhere else!
`INITIALIZING/CREATED/RUNNING` | `COMPLETED on α` | Delete Container | **Conceivable**: The task is `COMPLETED` (perhaps it was cancelled) - delete the container
`INITIALIZING/CREATED/RUNNING` | `COMPLETED on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`INITIALIZING/CREATED/RUNNING` | `RESOLVING on α` | Delete Container | **Conceivable**: The task is `RESOLVING` (perhaps it was cancelled) - delete the container
`INITIALIZING/CREATED/RUNNING` | `RESOLVING on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`COMPLETED` | No Task | Delete Container | **Conceivable**: The Task has been cancelled - don't worry about it
`COMPLETED` | `PENDING` | Update the Task to `COMPLETED` then Delete Container | **Inconceivable**: We should already have started the task before running the container.
`COMPLETED` | `RUNNING on α` | Update the Task to `COMPLETED` then Delete Container | **Expected**: Make sure BBS knows we've completed the Task
`COMPLETED` | `RUNNING on ω` | Delete Container | **Inconceivable**: Apparently this Task is running somewhere else!
`COMPLETED` | `COMPLETED on α` | Delete Container | **Conceivable**: The task has already been marked `COMPLETED` - delete the container
`COMPLETED` | `COMPLETED on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
`COMPLETED` | `RESOLVING on α` | Delete Container | **Conceivable**: The task has already been marked `RESOLVING` (perhaps it was cancelled) - delete the container
`COMPLETED` | `RESOLVING on ω` | Delete Container | **Inconceivable**: Apparently this Task ran to completion somewhere else!
No Container | `RUNNING on α` | Update to `COMPLETED` and `Failed` | **Conceivable**: Diego thinks α is running the instance, but it is not
No Container | `COMPLETED on α` | Do Nothing | **Expected**: The client has not resolved the task
No Container | `RESOLVING on α` | Do Nothing | **Expected**: The client is resolving the task


Some notes:

- The "No Container" rows are necessary to ensure that the BBS reflects the reality of what is - and is *not* - running on the Cell. Note that "No Container" includes "No Reservation".
- In principal several of these combinations should not be possible. However in the presence of network partitions and partial failures it is difficult to make such a statement with confidence. An exhaustive analysis of all possible combinations (such as this) ensures more safety around the Task lifecycle.
- When the Action described in this table fails, the Rep should log and do nothing. In this way we defer to the next polling cycle to retry actions.
- If container creation fails, the rep completes the task but marks it as retryable (Retryable Complete in table below). If the failure happens after container creation, the rep marks it as NonRetryable but completes the task.

BBS | Reserved | I/C/Running | Retryable Completed | NonRetryable Completed | No Container
---|---|---|---|---|---
Missing     | Delete Container          | Do Nothing               | Delete Container          | Delete Container |
Pending     | Start Task, Run Container | Do Nothing               | Start Task, Run Container | Complete Task, Delete Container |
Running-α   | Do Nothing                | Reject Task              | Do Nothing                | Complete Task, Delete Container | Complete Task with failure
Running-ω   | Delete Container          | Do Nothing               | Delete Container          | Delete Container          |
Completed-α | Delete Container          | Do Nothing               | Delete Container          | Delete Container          | Do Nothing
Completed-ω | Delete Container          | Do Nothing               | Delete Container          | Delete Container          |
Resolving-α | Delete Container          | Do Nothing               | Delete Container          | Delete Container          | Do Nothing
Resolving-ω | Delete Container          | Do Nothing               | Delete Container          | Delete Container          |

### Pruning Corrupt/Invalid Data

The BBS fetches all Task data, and pre-processes it before determining what convergence actions to take.
As part of this process, it prunes any data that has become corrupt (e.g. the raw data cannot be de-serialized into an object) or invalid (an unversioned BBS schema change may make existing data invalid).
Currently this is done as part of the main convergence loop, but it is **not** critical to convergence behaviour.

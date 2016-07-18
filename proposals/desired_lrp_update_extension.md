# Proposal for zero downtime updates of DesiredLRPs

## What Can be Updated?

1) App Update - Droplets, env vars etc - RunInfo
1) App Update - Memory Quota, Disk Quota - ScheduleInfo
1) Update exposed Ports - RunInfo But see below
* Routing information can already be updated in a DesiredLRP.  But no versions.
* What does it mean to update Ports when some instances use port A others use Port B?
* How would the GORouter manage this traffic?
* How do we emit these new routes?  How does an app user go to different ports?  If we change ports does this not make old ports useless?
1) Asset URLs - RunInfo
1) Actions Updated - Start command, healthcheck etc - RunInfo
1) App SecurityGroups - RunInfo
1) Privileged Flag - RunInfo

## How do we version in the Model?

RunInfo is the part of DesiredLRP stored as a BLOB in SQL schema.

We could have multiple runInfos for a single desiredLRP - is this a refactor of the
desiredLRP SQL.

Have a RunInfo table and references to the desiredLRP.

Inside the desiredLRP we could have active, old1, old2 references to RunInfo entries


Link between actual LRP and the runInfo it was started with.

DLRP 1-> Many(3) RunInfos
ALRP 1-> 1 RunInfos
DLRP 1-> Many ALRPs

We may have to split some of the items that are in schedule info (disk, memory quota) stuff to be able to
version those pieces.

We should use locking and transactions to ensure we can update both the runInfo and the DesiredLRP and ActualLRP tables in a single step.

## Backwards Compatibility

Pulling the RunInfo out will have some backwards compatibility effects.   

1) Migration required to convert current BLOB to a single entry in the table.
1) API versioning.  Should be fairly simple to have the legacy functionality having a single RunInfo per DLRP.

## What manages Behaviour?

The behaviour is similar to evacuation.  We want to ensure a new version of an instance is up prior to taking down the old version of an instance.  We also want to serially roll all instances of an app.

Probably use the BBS to manage the overall flow of this for a particular app.

Can we reuse the evacuation flag in the actual LRP?  This may cause conflicts with how cells are working as well as how convergence is running and looking at evacuating LRPs.

## Events

What RunInfo is required for eventing.

1) Need to look into useage of RunInfo in route-emitter and nsync bulker to ensure this can work correctly.


## How do we cleanup unused?

New convergence could cleanup unused runInfos.  Any RunInfo not the "active" RunInfo for a DesiredLRP and not referenced by a running ActualLRP can be removed.

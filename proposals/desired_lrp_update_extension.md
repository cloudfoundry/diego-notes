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
 * How would this work with cf delete commands from the cloud controller? If the versioning is meant to be transparent to users, we might want one evacuate call that targets LRPs by app name and one that can target `{name, version}` tuples.

When the BBS and the Rep are in the process of switching a group of ActualLRP instances from the old version to the new version, the system is in a transition state. If the BBS receives a delete command while in this state, it should begin evacuating without trying to finish transitioning all the LRPs to the new version.

## Events

What RunInfo is required for eventing.

1) Need to look into usage of RunInfo in route-emitter and nsync bulker to ensure this can work correctly.

Looks like RunInfo does not directly get passed to route emitter or nsync. Instead, it is the scheduling info that is used in route-emitter to reconstruct the routing tables. The scheduling info has desiredLRP resources (Memory / Disk / Rootfs) as well as the Route information. The route information does not include port number though, just a string of routes.


## How do we cleanup unused?

New convergence could cleanup unused runInfos.  Any RunInfo not the "active" RunInfo for a DesiredLRP and not referenced by a running ActualLRP can be removed.

What happens if cleanup occurs before a new version launches any ActualLRPs? If we don't have any kind of ordered versioning, then maybe we should spare `UNCLAIMED` and `CLAIMED` ActualLRPs as well.


## Proposed API in bbs

UpdateDesiredLRP(logger lager.Logger, processGuid string, update *models.DesiredLRPUpdate) error.

models.DesiredLRPUpdate would now have the following information:

```go
type DesiredLRPUpdate string {
    Instances * int32
    Routes *Routes
    Annotation *string
    DesiredLRP *DesiredLRP
    UpdateIdentifier *string
}
```

To maintain backwards compatibility we will leave the Instances, Routes, and Annottaion fields as top level entries in the DesiredLRPUpdate.  The DesiredLRP field will be a fill DesiredLRP containing all the other information to be updated.
The UpdateIdentifier will be the named version of this DesiredLRP allowing for mapping of an actual LRP to a specific DesiredLRP version.   On create of a DesiredLRP we can create a tag for the initial version.

Nysnc will be able to create this easily as they aleady build up the new DesiredLRP structure using the receipe builder.  They will simply use the full desiredLRP to send to the endpoint.

We could also think of creating a NEW endpoint that just takes the full new DesiredLRP.

UpdateDesiredLRP(logger lager.Logger, processGuid string, update *models.DesiredLRP) error

This would be a NEW endpoint and make backwards compatibility easier.

## Proposed New model for DesiredLRP and ActualLRP

### DesiredLRP

Current:

```go
type DesiredLRP struct {
    ProcessGuid                   string
    Domain                        string
    RootFs                        string
    Instances                     int32
    EnvironmentVariables          []*EnvironmentVariable
    Setup                         *Action
    Action                        *Action
    StartTimeoutMs                int64
    DeprecatedStartTimeoutS       uint32
    Monitor                       *Action
    DiskMb                        int32
    MemoryMb                      int32
    CpuWeight                     uint32
    Privileged                    bool
    Ports                         []uint32
    Routes                        *Routes
    LogSource                     string
    LogGuid                       string
    MetricsGuid                   string
    Annotation                    string
    EgressRules                   []*SecurityGroupRule
    ModificationTag               *ModificationTag
    CachedDependencies            []*CachedDependency
    LegacyDownloadUser            string
    TrustedSystemCertificatesPath string
    VolumeMounts                  []*VolumeMount
    Network                       *Network
}
```

New:

```go
type LRPDefinition struct {
    DefinitionIdentifier          string
    RootFs                        string
    Instances                     int32
    EnvironmentVariables          []*EnvironmentVariable
    Setup                         *Action
    Action                        *Action
    StartTimeoutMs                int64
    DeprecatedStartTimeoutS       uint32
    Monitor                       *Action
    DiskMb                        int32
    MemoryMb                      int32
    CpuWeight                     uint32
    Privileged                    bool
    Ports                         []uint32
    Routes                        *Routes
    LogSource                     string
    LogGuid                       string
    MetricsGuid                   string
    EgressRules                   []*SecurityGroupRule
    CachedDependencies            []*CachedDependency
    LegacyDownloadUser            string
    TrustedSystemCertificatesPath string
    VolumeMounts                  []*VolumeMount
    Network                       *Network
}

type DesiredLRP struct {
    ProcessGuid                   string
    Domain                        string
    models.LRPDefinition
    Annotation                    string
    LRPDefinition2         *models.LRPDefinition
    LRPDefinition3         *models.LRPDefinition
}
```


### ActualLRP

Current:

```go
type ActualLRP struct {
    ActualLRPKey
    ActualLRPInstanceKey
    ActualLRPNetInfo
    CrashCount           int32
    CrashReason          string
    State                string
    PlacementError       string
    Since                int64
    ModificationTag      ModificationTag
}
```

New:

```go
type ActualLRP struct {
    ActualLRPKey
    ActualLRPInstanceKey
    ActualLRPNetInfo
    DesiredLRPDefinitionIdentifier string
    CrashCount           int32
    CrashReason          string
    State                string
    PlacementError       string
    Since                int64
    ModificationTag      ModificationTag
}
```

The main change here is to have a DefinitionIdentifier to link the ActualLRP to the definition for that DLRP

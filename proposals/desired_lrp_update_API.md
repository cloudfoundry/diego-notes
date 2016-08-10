# Proposal for zero-downtime updates of DesiredLRPs

## Introduction

Cloud Controller would like the Diego backend to perform best-effort zero-downtime deploys in certain cases that affect the container runtime environment:

- explicit app updates: droplet, env vars, memory quota, disk quota
- transitioning all app containers to run as unprivileged
- updating exposed port list
- updating asset URLs in LRP definitions
- updating actions with new definitions (start command, health check, SSH; action structure)
- updating application security group rules after new bindings applied to system or space
- updating anything else that may affect the environment or specification of the individual LRP instances

Ideally, we wouldn't create a brand new DesiredLRP in these circumstances, since we're changing the environment rather than the app itself. To this end, the Diego BBS API should support updating a DesiredLRP with a new definition. This would allow the Diego scheduler to preserve the identity of the LRP while transitioning the deployed app to the new definition.


## Proposed API in bbs

```go
UpdateDesiredLRP(logger lager.Logger, processGuid string, update
*models.DesiredLRPUpdate) error.
```

`models.DesiredLRPUpdate` would now have the following information:

```go
type DesiredLRPUpdate struct {
    Instances *int32
    Routes *Routes
    Annotation *string
    DesiredLRP *DesiredLRP
    UpdateIdentifier *string
}
```

The `UpdateDesiredLRP `endpoint will be the same as previously defined.  The
change is in the `DesiredLRPUpdate` model to specify the new update.  Since the
new fields in this Update are additive the endpoint will not have to change as
older clients will simply not provide the new fields.

When a new `DesiredLRP` is provided the `UpdateIdentifier` is required to indicate
the new definition identifier.

The API can return an error (`ErrUpdateInProgress`) if there are existing
`ActualLRP`s with different identifiers for the same `DesiredLRP`.  This will
eliminate the issue of rapidly updating of `DesiredLRP`s as there can only be a
single update in flight at one time.

There could be issues with the in flight update process if the new definition
is not good.  For instance, does the new definition cause the application to
crash.  When does the system complete the update to allow the user to update
again?  Will it be possible to rollback an update?

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
    Instances                     int32
    Routes                        *Routes
    Annotation                    string
    PreviousDefinitionIdentifier  string
}
```

The new model is identical to the old model with the additional fields of
`DefinitionIdentifier` and `PreviousDefinitionIdentifier`. They will only both
be returned if there is an update in progress.  The original requirements
discussed having more than 2 definitions.  The choice was made to only support
current and previous versions of the definitions for simplicity.  We should
only ever have `ActualLRP`s with either the current definition or the previous
definition as we limit an in-flight updates to 1.


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

The only change here is to have a `DefinitionIdentifier` to link the
`ActualLRP` to the definition for that `DesiredLRP`

## Client Interaction

### CloudController

1. **Stage a NEW application**

    Stager creates the Staging Task to stage the application
1. **Create a NEW `DesiredLRP`**

    `NSync` creates the application using `bbsClient.DesireLRP`
1. **Query a `DesiredLRP`**

    `NSync` queries the BBS using `bbsClient.DesireLRPByProcessGuid`.
    We could supply the `DefinitionIdentifier` and
    `PreviousDefinitionIdentifier` fields to specify the "ID" of the current
    `DesiredLRPDefinitions`.  We do not need to return the old definition
    information, just the identifier. We can add additional APIs to get old
    definition information if required.
    Old clients will not receive these three new fields, just the
    current Definition information.
1. **Stage an Update to a `DesiredLRP`**

    Stager creates the Staging Task to stage the application.
1. **Update the `DesiredLRP`**

    `NSync` calls `bbsCLient.UpdateDesiredLRP` with the new `DesiredLRP`
1. **Query a `DesiredLRP`**

    If a `DesiredLRP` is currently in the process of a Blue/Green deploy
    (completing an Update).  Then the BBS will return a `DefinitionIdentifier`
    and `PreviousDefinitionIdentifier`.  The `PreviousDefinitionIdentifier` is used
    to indicate there are existing updates to the `DesiredLRP` in flight.   If
    there are no Upgrades in flight then the `PreviousDefinitionIdentifier` will
    be nil or not returned.

### TPS

`TPS` is used to get status information from Diego as well as reporting crashed
`LRP`s

The crashing `LRP` reporting will not be effected as it just returns
information on which `LRP`s crash and the reason.

The status and state endpoints in `TPS` would change slightly in that they can
now report which definition (current / previous) the `ActualLRP` is linked to.
This can also assist `CloudController` or the user to view how the progress of
the blue/green deploy is going.

Note: Old clients will simply not get the `DesiredLRPDefinitionIdentifier` and
so it will appear as if there is only a single definition.


### Route Emitter

As routes are emitted based on running `ActualLRP`s and their current
configuration, our code will need to be updated to map the `ActualLRP` that is
running with the correct `DesiredLRPDefinition` (the scheduling info part of
networking).

The route emitter communicating with the updated client will get a
`DefinitionIdentifier` to assist with this mapping.   Older clients will not
have this but will assume the single current Definition.  This may cause some
problems with emitting incorrect ports during the blue/green deploy.  If this
is an issue the route emitter must be upgraded prior to the BBS.


### SSH Proxy

As above with the route emitter.  The SSH-Proxy does do correlation between
`ActualLRP`s and `DesiredLRP` Scheduling information.   The code will need to
be updated to take into account the possibility that `ActualLRP`s can have
different definitions for the same `ProcessGuid`.

The same caveat as for Route Emitter will be relevant with regard to possible
upgrade order.








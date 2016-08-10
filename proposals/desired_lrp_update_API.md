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

## Example Usage

```go
update := &DesiredLRPUpdate{
	LRPDefinition: &LRPDefinition{
		DefinitionID: "version-2"
	    // Complete definition of the LRP
	}
}

bbsClient.UpdateDesiredLRP(logger, processGuid, update)

// While the update is in progress, CancelUpdateLRP may be called to cancel to current update and
// roll all instances back to the previous state. If no update is in progress, this call will return 
// an error and do nothing.
bbsClient.CancelUpdateLRP(logger, processGuid)

// We found our mistake with the original update and we prepare 'version-3' of the LRP.
update2 := &DesiredLRPUpdate{
	LRPDefinition: &LRPDefinition{
		DefinitionID: "version-3"
	 	// Complete definition of the LRP
	}
}

bbsClient.UpdateDesiredLRP(logger, processGuid, update2)

// Later on, after the update to version-3 has completed, we realize we actually wanted "version-2" after all.
bbsClient.RollbackDesiredLRP(logger, processGuid, "version-2")
```

Details about these API calls can be found in the sections below.

## Proposed API in BBS

### Summary
```go
UpdateDesiredLRP(logger lager.Logger, processGuid string, update
*models.DesiredLRPUpdate) error
CancelUpdateDesiredLRP(logger lager.Logger, processGuid string) error
RollbackDesiredLRP(logger lager.Logger, processGuid, definitionID string) error
```

`models.DesiredLRPUpdate` would now have the following information:

```go
type DesiredLRPUpdate struct {
    Instances *int32
    Routes *Routes
    Annotation *string
    LRPDefinition *LRPDefinition
}
```

### `UpdateDesiredLRP`

Updates the [DesiredLRP](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRP) with the given process GUID.

#### BBS API Endpoint

POST a UpdateDesiredLRPRequest
to `/v1/desired_lrp/update`
and receive a DesiredLRPLifecycleResponse

In this proposal, the `UpdateDesiredLRPRequest` will be updated to match the model defined above. Otherwise, this call does not change. Since the new fields in this Update are additive the endpoint will not have to change as older clients will simply not provide the new fields.

#### Golang Client API

```go
UpdateDesiredLRP(logger lager.Logger, processGuid string, update *models.DesiredLRPUpdate) error
```

(Unchanged in this proposal)

##### Inputs

* `processGuid string`: The GUID for the DesiredLRP to update.
* `update *models.DesiredLRPUpdate`: DesiredLRPUpdate struct containing fields to update, if any.
  * `Instances *int32`: Optional. The number of instances.
  * `Routes *Routes`: Optional. Map of routing information.
  * `Annotation *string`: Optional. The annotation string on the DesiredLRP.
  * **[NEW]** `LRPDefinition *LRPDefinition`: Optional. If non-nil, everything in existing LRPDefinition will be completed replaced by this new LRPDefinition. If anything on the LRPDefinition is left undefined (i.e., nil), those values will be deleted.

##### Output

* `error`:  Non-nil if an error occurred.


##### Example

```go
client := bbs.NewClient(url)
instances := 4
annotation := "My annotation"
definition := models.LRPDefinition{
	DefinitionID: "version-42",
	[...] // All other values must be specified
}
err := client.UpdateDesiredLRP(logger, "some-process-guid", &models.DesiredLRPUpdate{
    Instances: &instances,
    Routes: &models.Routes{},
    Annotation: &annotation,
    LRPDefinition: &definition,
})
if err != nil {
    log.Printf("failed to update desired lrp: " + err.Error())
}
```

### **[NEW]** `CancelUpdateDesiredLRP`

Cancels an in-flight desired LRP update.  All instances that have already been updated will be reverted to the previous LRP definition.  If no update is in flight, this call will do nothing and return an error.

#### BBS API Endpoint

POST a CancelUpdateDesiredLRPRequest
to `/v1/desired_lrp/cancel_update`
and receive a [DesiredLRPLifecycleResponse](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPLifecycleResponse).

A `CancelUpdateDesiredLRPRequest` looks like:
```go
type CancelUpdateDesiredLRPRequest struct {
    ProcessGuid string
}
```

#### Golang Client API

```go
CancelUpdateDesiredLRP(logger lager.Logger, processGuid string) error
```

##### Inputs

* `processGuid string`: The GUID for the [DesiredLRP](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRP) whose update to cancel.

##### Output

* `error`:  Non-nil if an error occurred.

##### Example

```go
client := bbs.NewClient(url)
err := client.CancelUpdateDesiredLRP(logger, "some-process-guid")
if err != nil {
    log.Printf("failed to cancel desired lrp update: " + err.Error())
}
```

### **[NEW]** `RollbackDesiredLRP`

Rolls back an LRP to a previous LRP definition, provided the definition still exists. Stale LRP definitions will be periodically pruned from the database.

#### BBS API Endpoint

POST a RollbackDesiredLRPRequest
to `/v1/desired_lrp/rollback`
and receive a [DesiredLRPLifecycleResponse](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPLifecycleResponse).

A `RollbackDesiredLRPRequest` looks like:
```go
type RollbackDesiredLRPRequest struct {
    ProcessGuid string
    DefinitionID string
}
```

#### Golang Client API

```go
RollbackDesiredLRP(logger lager.Logger, processGuid, definitionID string) error
```

##### Inputs

* `processGuid string`: The GUID for the [DesiredLRP](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRP) to roll back.
* `definitionID string`: The definition ID to roll back to. This ID was created by the user when `UpdateDesiredLRP` was called.

##### Output

* `error`:  Non-nil if an error occurred. This may error if the definition is no longer stored.

##### Example

```go
client := bbs.NewClient(url)
err := client.RollbackDesiredLRP(logger, "some-process-guid", "some-version")
if err != nil {
    log.Printf("failed to rollback desired lrp: " + err.Error())
}
```


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

    `NSync` calls `bbsCLient.UpdateDesiredLRP` with the new `DesiredLRP`.
1. **Query a `DesiredLRP`**

    If a `DesiredLRP` is currently in the process of a blue/green deploy
    (completing an update), then the BBS will return a `DefinitionIdentifier`
    and `PreviousDefinitionIdentifier`. The `PreviousDefinitionIdentifier` is used
    to indicate there are `DesiredLRP` updates already in flight. If
    there are no updates in flight then the `PreviousDefinitionIdentifier` will
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








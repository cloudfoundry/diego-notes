# Proposal for zero-downtime updates of DesiredLRPs

## Introduction
This document serves a proposal to broaden the scope of what is supported in Long Running Process (LRP) updates in Diego.

There is an existing Diego (i.e., BBS) endpoint called UpdateDesiredLRP, which allows Diego clients to change the number of instances, the routes, and the annotations associated with an LRP. This proposal expands that API call so that any aspect of an LRP may be modified.

In this proposal, we suggest that each iteration or version of an LRP has a label associated with it. This is referred to below as the `DefinitionID`. This allows a Diego client to roll an LRP back to a previous definition if desired. We expect that Diego will periodically clear out this archive of old LRP definitions, however, so a rollback may not succeed if it definition hasn't been actively used in a while.

The process of restarting all running instances of the LRP with the new definition will take some time, so we are also proposing a cancellation endpoint to halt and revert an update that is currently in progress. 

## Example Usage

```go
update := &DesiredLRPUpdate{
	LRPDefinition: &LRPDefinition{
		DefinitionID: "version-2"
	    // Complete definition of the LRP
	}
}

// Note: error handling is left out of the example for brevity.
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

Details about these API calls can be found in the section below.

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

In this proposal, the `UpdateDesiredLRPRequest` will be updated to match the `DesiredLRPUpdate` model defined above. Other than that, this call looks the same as it currently does. Since the new fields in this Update are additive the endpoint will not have to change as older clients will simply not provide the new fields.

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

* `error`:  Non-nil if an error occurred. `ErrUpdateInProgress` is returned if an existing update is in progress.

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

### **[Updated]** `DesiredLRPSchedulingInfos`

Retrieves all the `DesiredLRPSchedulingInfo` that match the given DesiredLRPFilter.  The update is there could be two `DesiredLRPSchedulingInfo` for a single `DesiredLRP`.  The old clients will use the deprecated endpoint and thus will only get a single `DesiredLRPSchedulingInfo` per `DesiredLRP`.

### BBS API Endpoint

POST a [DesiredLRPsRequest](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPsRequest)
to `/v1/desired_lrp_scheduling_infos/list.r1`
and receive a [DesiredLRPSchedulingInfosResponse](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPSchedulingInfosResponse).

### Deprecated Endpoints

POST a [DesiredLRPsRequest](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPsRequest)
to `/v1/desired_lrp_scheduling_infos/list`
and receive a [DesiredLRPSchedulingInfosResponse](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPSchedulingInfosResponse).

### Golang Client API

```go
DesiredLRPSchedulingInfos(logger lager.Logger, filter models.DesiredLRPFilter) ([]*models.DesiredLRPSchedulingInfo, error)
```

#### Inputs

* `filter models.DesiredLRPFilter`: [DesiredLRPFilter](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPFilter) to restrict the DesiredLRPs returned.
  * `Domain string`: If non-empty, filter to only DesiredLRPs in this domain.

#### Output

* `[]*models.DesiredLRPSchedulingInfo`: List of [DesiredLRPSchedulingInfo](https://godoc.org/code.cloudfoundry.org/bbs/models#DesiredLRPSchedulingInfo) records.
* `error`:  Non-nil if an error occurred.


#### Example

```go
client := bbs.NewClient(url)
info, err := client.DesiredLRPSchedulingInfos(logger, &models.DesiredLRPFilter{
    Domain: "cf-apps",
})
if err != nil {
    log.Printf("failed to retrieve desired lrp scheduling info: " + err.Error())
}
```

## Proposed new model for DesiredLRP and ActualLRP

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

type DesiredLRPSchedulingInfo struct {
	DesiredLRPKey
	Annotation         string
	Instances          int32
	DesiredLRPResource 
	Routes             Routes
	ModificationTag    
	VolumePlacement    *VolumePlacement
}

type DesiredLRPRunInfo struct {
	DesiredLRPKey
	EnvironmentVariables          []EnvironmentVariable
	Setup                         *Action
	Action                        *Action
	Monitor                       *Action
	DeprecatedStartTimeoutS       uint32
	Privileged                    bool
	CpuWeight                     uint32
	Ports                         []uint32
	EgressRules                   []SecurityGroupRule
	LogSource                     string
	MetricsGuid                   string
	CreatedAt                     int64
	CachedDependencies            []*CachedDependency
	LegacyDownloadUser            string
	TrustedSystemCertificatesPath string
	VolumeMounts                  []*VolumeMount
	Network                       *Network
	StartTimeoutMs                int64
}


```

New:

```go
type LRPDefinition struct {
    DefinitionID                  string
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
    PreviousDefinitionID          string
    ModificationTag               *ModificationTag
}


type DesiredLRPSchedulingInfo struct {
	DesiredLRPKey
	Annotation         string
	Instances          int32
	DesiredLRPResource 
	Routes             Routes
	ModificationTag    
	VolumePlacement    *VolumePlacement
	DefinitionID       string
}

type DesiredLRPRunInfo struct {
	DesiredLRPKey
	EnvironmentVariables          []EnvironmentVariable
	Setup                         *Action
	Action                        *Action
	Monitor                       *Action
	DeprecatedStartTimeoutS       uint32
	Privileged                    bool
	CpuWeight                     uint32
	Ports                         []uint32
	EgressRules                   []SecurityGroupRule
	LogSource                     string
	MetricsGuid                   string
	CreatedAt                     int64
	CachedDependencies            []*CachedDependency
	LegacyDownloadUser            string
	TrustedSystemCertificatesPath string
	VolumeMounts                  []*VolumeMount
	Network                       *Network
	StartTimeoutMs                int64
	DefinitionID                  string
}

```

The new model is identical to the old model with the additional fields of `DefinitionID` and `PreviousDefinitionID`. They will only both be set if there is an update in progress.  The original requirements discussed having more than two definitions, but for simplicity we modified this so that there is only a reference to a current and a previous definition. Other, older definitions may still exist in a separate database, but the DesiredLRP itself will not have a reference to them.

Each LRP definition is specific to a particular DesiredLRP, and as such is uniquely identified by `{ProcessGuid, DefinitionID}`. This allows users to name LRP definitions according to a particular ordering scheme without worrying about colliding with another DesiredLRP's naming convention.

`DesiredLRPSchedulingInfo` and `DesiredLRPRunInfo` will have the additional field DefinitionID to differentiate them from the current and previous definitions.

#### Additional Methods

Two additional methods are required to return the Previous SchedulingInfo and RunInfo from an existing DesiredLRP
```go
func (d *DesiredLRP) PreviousDesiredLRPSchedulingInfo() DesiredLRPSchedulingInfo
func (d *DesiredLRP) PreviousDesiredLRPRunInfo(createdAt time.Time) DesiredLRPRunInfo
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
    CrashCount             int32
    CrashReason            string
    State                  string
    PlacementError         string
    Since                  int64
    ModificationTag        ModificationTag
    DesiredLRPDefinitionID string
}
```

The only change here is to have a `DesiredLRPDefinitionID` to link the `ActualLRP` to the `LRPDefinition` for which it was created. This ID should always match either the `DefinitionID` or `PreviousDefinitionID` in the corresponding `DesiredLRP`.

#### Events

Events will not change other than additional fields returned in the underlying `DesiredLRP` or `ActualLRP` structs in the event payload.   Those who are processing events will have to handle the new fields and the updates to the ScheduleInfo and/or RunInfo retrieved from them.

#### Legacy Records

As part of the migration to this new API, LRP definitions for old DesiredLRP and ActualLRP records would have to be backfilled.

1. `DesiredLRP.PreviousDefinitionID` would be set to nil.
2. `DesiredLRP.DefinitionID` would be set to `{GeneratedGUID}`.
3. `ActualLRP.DefinitionID` would be set to `{GeneratedGUID}`.

This database migration should be applied before any new endpoints are exposed.  `{GeneratedGUID}` must be the same on the `DesiredLRP.DefinitionID` and the associated `ActualLRP`s for the `DesiredLRP`

#### Legacy Clients

When a legacy client Desires an LRP it will not provide the DefinitionID.  If a `DefinitionID` is not provided the BBS will generate a unique ID for that LRPDefinition similar to how this is done in the Legacy Records migration described above.


## Client Interaction

This section describes how the various BBS clients would interact with this new feature.

### CloudController
NSync will need to be modified to use the new `DesiredLRPUpdate` model and fill in the details of the LRP that should be updated. There also could be new interactions to support canceling a current LRP update or rolling back to a previous LRP version.

1. **Stage a new application**

    Stager creates the staging task to stage the application.
1. **Create a new `DesiredLRP`**

    `NSync` creates the application using `bbsClient.DesireLRP`.
1. **Query a `DesiredLRP`**

    `NSync` queries the BBS using `bbsClient.DesireLRPByProcessGuid`.
    The BBS returns the `DefinitionID` field to specify which DesiredLRPDefinition is active, and if the `PreviousDefinitionID` field is also set, it means that an update is currently in progress.
    Old clients will not receive these three new fields, just the current Definition information.
1. **Stage an update to a `DesiredLRP`**

    Stager creates the staging task to stage the application.
1. **Update the `DesiredLRP`**

    `NSync` calls `bbsClient.UpdateDesiredLRP` with the new `LRPDefinition`, including a `DefinitionID` provided by `NSync`.

### TPS
`TPS` is used to get status information from Diego as well as reporting crashed `LRP`s.

The crashing `LRP` reporting will not be affected as it just returns information on which `LRP`s crash and the reason.

The status and state endpoints in `TPS` would change slightly in that they can now report which definition (current / previous) the `ActualLRP` is linked to. This can also assist `CloudController` or the user to view how the progress of the blue/green deploy is going.

Note: Old clients will simply not see the `DesiredLRPDefinitionID` on the `ActualLRP`, so it will appear as if there is only a single definition.


### Route Emitter
As routes are emitted based on running `ActualLRP`s and their current configuration, our code will need to be updated to map the `ActualLRP` that is running with the correct `DesiredLRPDefinition` (the scheduling info part of networking).

The route emitter communicating with the updated client will get a `DefinitionID` to assist with this mapping. Older clients will not have this but will assume the single current Definition.  This may cause some problems with emitting incorrect ports during the blue/green deploy.  Because of this, the route-emitter must be updated before the BBS introduces this change.


### SSH Proxy
As above with the route emitter.  The SSH-Proxy does do correlation between `ActualLRP`s and `DesiredLRP` Scheduling information.   The code will need to be updated to take into account the possibility that `ActualLRP`s can have different definitions for the same `ProcessGuid`.

The same caveat as for Route Emitter will be relevant with regard to possible upgrade order.








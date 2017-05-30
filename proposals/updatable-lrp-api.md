# BBS API for Updatable LRPs

## `LRPDeployment`

We propose that every existing `DesiredLRP` is promoted automatically to an `LRPDeployment` that admits rolling updates through adding a new `LRPDefinition` specification.

The current `DesiredLRP` model essentially has the following fields:

```go
type DesiredLRP struct {
  ProcessGuid string             // immutable identifier
  Domain      string             // immutable identifier
  Instances   uint               // mutable, affects scale
  Routes      map[string][]byte  // mutable, no effect on execution
  Annotation  string             // mutable, no effect on execution

  Definition  LRPDefinition      // immutable, affects container execution
}
```

The `LRPDefinition` type includes fields relevant both for scheduling LRP instances and for executing those instances on the assigned Diego cell.

The `DesiredLRPUpdate` model has the following fields:

```go
type DesiredLRPUpdate struct {
  Instances   *uint               // mutable, affects scale
  Routes      *map[string][]byte  // mutable, no effect on execution
  Annotation  *string             // mutable, no effect on execution
}
```

The BBS API admits essentially the following operations on the collection of DesiredLRPs, via [the ExternalDesiredLRPClient](https://godoc.org/code.cloudfoundry.org/bbs#ExternalDesiredLRPClient) interface:

- `DesiredLRPByProcessGuid(id string) DesiredLRP`
- `DesiredLRPs(ids []string) []DesiredLRP`
- `DesireLRP(lrp DesiredLRP)`
- `UpdateDesiredLRP(id string, lrpUpdate DesiredLRPUpdate)`
- `RemoveDesiredLRP(id string)`

These effects of these calls are evident to those familiar with the Diego BBS API.

We propose a new `LRPDeployment` model that admits several simultaneous LRP definitions:

```go
type LRPDeployment struct {
  ProcessGuid string             // immutable identifier
  Domain      string             // immutable identifier
  Instances   uint               // mutable, affects scale
  Routes      map[string][]byte  // mutable, no effect on execution
  Annotation  string             // mutable, no effect on execution

  Definitions         map[string]LRPDefinition
  ActiveDefinitionID  string
  HealthyDefinitionID string
}

type LRPDeploymentDefinition struct {
  ProcessGuid string             // immutable identifier
  Domain      string             // immutable identifier
  Instances   uint               // mutable, affects scale
  Routes      map[string][]byte  // mutable, no effect on execution
  Annotation  string             // mutable, no effect on execution

  DefinitionID string
  Definition   LRPDefinition
}

type LRPDeploymentUpdate struct {
  Instances   *uint               // mutable, affects scale
  Routes      *map[string][]byte  // mutable, no effect on execution
  Annotation  *string             // mutable, no effect on execution

  DefinitionID *string
  Definition   *LRPDefinition
}
```

This model comes with internal constraints:

- The `ActiveDefinitionID` value will always be a key in the Definitions map.
- The `HealthyDefinitionID` value will also always be a key in the Definitions map, or will be an empty string if no instance of the DesiredLRP has ever become healthy.


The BBS API admits the following operations for these LRPDeployment entities:

- `LRPDeploymentByProcessGuid(id string) LRPDeployment`
- `LRPDeployments(ids []string) []LRPDeployment`
- `CreateLRPDeployment(deployment LRPDeploymentDefinition)`
- `UpdateLRPDeployment(id string, update LRPDeploymentUpdate)`
- `ActivateLRPDeploymentDefinition(id, definition_id string)`
- `DeleteLRPDeployment(id string)`

`LRPDeploymentByProcessGuid` returns a single `LRPDeployment` by its process guid. `LRPDeployments` returns the collection of all `LRPDeployment` entities in the Diego cluster with the appropriate filter (domain string or set of process guids) applied.

### `CreateLRPDeployment`

The `CreateLRPDeployment` call behaves similarly to `CreateDesiredLRP`. It creates a new LRP deployment with the supplied definition, setting the `ActiveDefinitionID` to the supplied definition ID. For an `LRPDeploymentDefinition` with the following content,

```go
LRPDeploymentDefinition{
  ProcessGuid: p,
  ..., // existing LRP fields
  DefinitionID: id,
  Definition:   def,
}
```
the `LRPDeployment` that is created initially has the value

```go
LRPDeployment{
  ProcessGuid: p,
  ..., // existing LRP fields
  Definitions: map[string]LRPDefinition{
    id: def,
  },
  ActiveDefinitionID: id,
  HealthyDefinitionID: "",
}
```

The BBS then submits all the instances of the new LRP for placement. Once one of those instances transitions to the `RUNNING` state, the BBS sets the `HealthyDefinitionID` field on the LRPDeployment to `id`.


### `UpdateLRPDeployment`

The `UpdateLRPDeployment` call behaves similarly to `UpdateDesiredLRP` when updating the instance count, routes, or annotation on an existing LRP. When updating a definition, both the new definition and the desired identifier for it must be supplied, and the identifier must be distinct from existing identifiers on that LRPDeployment. For an existing `LRPDeployment` with value

```go
LRPDeployment{
  ProcessGuid: p,
  Definitions: map[string]LRPDefinition{
    id1: def1,
  },
  ActiveDefinitionID: id1,
  HealthyDefinitionID: id1,
}
```

and an `LRPDeploymentUpdate` supplying `def2` with `id2`, the `LRPDeployment` initially has the value

```
LRPDeployment{
  ProcessGuid: p,
  Definitions: map[string]LRPDefinition{
    id1: def1,
    id2: def2,
  },
  ActiveDefinitionID: id2,
  HealthyDefinitionID: id1,
}
```

with the active definition set to `id2` and `id2: def2` added to the LRPDeployment's `Definitions` map.

Suppose LRPDeployment `p` has a total of 3 instances, all in the `RUNNING` state when the update applies. Diego will then schedule an index-0 instance for `def2` to start, with definition id `id2`. If it does so successfully, then Diego updates LRPDeployment `p` to have its `HealthyDefinitionID` set to `id2`, stops the `def1` instance at index 0, and schedules a `def2` instance at index 1 to start. Once this instance starts, then Diego stops the existing `def1` instance at index 1 and proceeds to replace the index-2 instance in the same way, at which point it has finished the transition to the `id2` definition.

Once no more instances are running with the `id1` definition, Diego also removes it from the `LRPDefinition` map on LRPDeployment `p` within the next LRP convergence tick.

Instances that need to restart outside of the rolling-deploy sequence (because of a crash or an evacuation event) will use the definition specified in the `HealthyDefinitionID` field of the `LRPDeployment`.

### `ActivateLRPDeploymentDefinition`

The `ActivateLRPDeploymentDefinition` call behaves similarly to the `UpdateLRPDeployment` call, except that it initiates a transition to an existing definition in the definition map of the `LRPDeployment`.


### `DeleteLRPDeployment`

The `DeleteLRPDeployment` call behaves similarly to the `DeleteDesiredLRP` call: it stops all `RUNNING` instances of the LRP and deletes the `LRPDeployment` record and the corresponding `ActualLRP` records.

## Extensions to and interoperability with existing BBS API models

### Promotion of existing DesiredLRP entities to LRPDeployment entities

Diego carries out a natural promotion of existing `DesiredLRP` records to `LRPDeployment` records. For the following partial `DesiredLRP` record,

```go
LRPDeployment{
  ProcessGuid: p,
  LRPDefinition: def1,
}
```

if the LRP has at least one `RUNNING` ActualLRP, it is promoted to the following `LRPDeployment`:

```go
LRPDeployment{
  ProcessGuid: p,
  Definitions: map[string]LRPDefinition{
    p: def1,
  },
  ActiveDefinitionID: p,
  HealthyDefinitionID: p,
}
```

using the `ProcessGuid` value `p` as the identifier for the `LRPDefinition`. If it has no associated `RUNNING` ActualLRP records, it is promoted to the same `LRPDeployment` entity, except that its `HealthyDefinitionID` is unset.


### Projection of an LRPDeployment entity down to a DesiredLRP entity

Likewise, the `LRPDeployment` entity naturally projects down to a `DesiredLRP`: given the following `LRPDeployment` with multiple definitions,

```go
LRPDeployment{
  ProcessGuid: p,
  Definitions: map[string]LRPDefinition{
    id1: def1,
    id2: def2,
    id3: def3,
  },
  ActiveDefinitionID: id2,
  HealthyDefinitionID: id2,
}
```

the BBS API uses the current `ActiveDefinitionID` to project it down to the following `DesiredLRP`:

```go
LRPDeployment{
  ProcessGuid: p,
  LRPDefinition: def2,
}
```

Denoting the promotion operator as `I_D` and the projection operator as `P_D`, it is clear that the composition `P_DI_D` is the identity operator on DesiredLRPs.

### Corresponding changes to ActualLRPs

ActualLRP records also require an extension to reflect the additional layer of hierarchy that the `LRPDeployment` introduces. The current model is of the form

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

type ActualLRPKey struct {
  ProcessGuid string
  Index       int32
  Domain      string
}

type ActualLRPInstanceKey struct {
  InstanceGuid string
  CellId       string
}

type ActualLRPNetInfo struct {
  Address         string
  Ports           []*PortMapping
  InstanceAddress string
}
```

We add a `DefinitionID` string-valued field to the `ActualLRPKey` type:

```go
type ActualLRPKey struct {
  ProcessGuid  string
  DefinitionID string
  Index        int32
  Domain       string
}
```

For backwards compatibility, this `DefinitionID` field is optional in requests and responses. For the Cloud Controller-facing work, we focus primarily on the consequences for the [ExternalActualLRPClient](https://godoc.org/code.cloudfoundry.org/bbs#ExternalActualLRPClient) methods.

- For the `ActualLRPGroups` and `ActualLRPGroupsByProcessGuid` methods, which return their ActualLRPs in a `[]*ActualLRPGroup` slice, the BBS API will return one ActualLRPGroup per unique `(ProcessGuid, DefinitionId, Index)` tuple.

- For the `ActualLRPGroupByProcessGuidAndIndex` method, which returns a single `ActualLRPGroup`, the BBS API will return the `ActualLRPGroup` for the active definition on the corresponding `LRPDeployment`.

- For the `RetireActualLRP` method, which takes an `ActualLRPKey` as its principal argument, the BBS API will select the ActualLRP with the active definition ID if there is any ambiguity in the instance to retire.

As the Diego team determines the best way to implement this refinement to the LRP lifecycle policy, other modifications to the ActualLRP models and APIs may become desireable or necessary, with the caveat of needing to maintain backwards compatibility with existing API endpoints.


### Further examples of behavioral cases

The behaviors below assume starting with an existing LRPDeployment in a stable state, specifying N instances and a single LRPDefinition (`def1`, with `id1`), with N corresponding ActualLRPs in the `RUNNING` state.

#### Fully functional new LRP definition

In the case of a new LRP definition that generally starts correctly within the specified start timeout, the client will observe a rolling deploy through the N ActualLRPs sequentially, starting the new instance at index J successfully before stopping the old index-J instance.

Once the `def2` LRP definition proves viable after the successful start of the, if an instance crashes and is restarted out of the normal rolling-deploy order, it will be restarted based on the `def2` definition.


#### Failing new LRP definition

Suppose that the new LRPDefinition, `def2` with `id2`, consistently fails to start:

- Diego will keep attempting to start a `def2`-based instance at index 0 indefinitely.
- Diego will also continue to keep the N `def1`-based instances running.
- Since the `HealthyDefinitionID` remains set to `id1`, if any `def1`-based instance crashes and is restarted, it will be based on `def1`.


#### Flaky new LRP definition

Suppose that the new LRPDefinition, `def2` with `id2`, fails to start the majority of the time:

- Diego will keep attempting to start a `def2`-based instance at index 0.
- Diego will also continue to keep the N `def1`-based instances running.
- Once the `def2`-based instance at index 0 starts, the `HealthyDefinitionID` will change to `id`.
- Suppose that the index-1 `def2`-based instance crashes repeatedly on startup without entering the `RUNNING` state. Diego will eventually give up on the instance in the interest of making forward progress on the remainder of the LRPDeployment. We expect the timeout will initially be hard-coded in the BBS, and may eventually be configurable for operators or for app developers.


#### New LRP definition during existing rolling deploy

Suppose that Diego is managing the transition of the LRPDeployment from `def1` to `def2`, waiting on the `def2`-based instance at index 2 to start, when the BBS API client supplies a new definition, `def3`, with id `id3`.

- Diego will stop the `def2`-based instance at index 2 if it has not yet started, keeping the existing `def1`-instance running.
- Diego will then begin a new rolling-deploy sequence with `def3`, starting at index 0 to replace the `def2`-based instance at that index.
- Once `def3`-based instances have replaced the `def2`-based instances at index 0 and 1, the `id2: def2` entry will be pruned from the LRPDeployment definition map.


# Release Versioning

## Operational guarantees

We intend the BOSH release of Diego to have a simple policy regarding upgradability, communicated clearly through its version: operators will have a stable upgrade path through versions of Diego as long as they deploy every major version of the release. This stability includes keeping application instances running through upgrades and maintaining the integrity of data submitted through Diego's BBS API.

To signal to operators the ramifications of upgrading their deployments to new versions of the release, we will follow `MAJOR.MINOR.PATCH`-style [semantic versioning](http://semver.org) with the following conventions:

- A **major version increment** may include the removal of deprecated APIs or functionality from a previous major release version. In accordance with our goal of allowing operators to upgrade seamlessly across major versions, we will typically remove functionality only after it is deprecated for a full major version.

- A **minor version increment** will indicate the addition of new functionality to the release, typically in the form of new APIs or behavior.

- A **patch version increment** will indicate no new addition of functionality, and the introduction only of bugfixes and refinements to behavior of existing APIs.

We call a pair of versions `(V1, V2)` *upgradable* if `V1 <= V2` lexicographically and if `0 <= MAJOR(V2) - MAJOR(V1) <= 1`. An upgradable version pair is *strict* if `V1 < V2`.  

## Timelines

We have yet to set a precedent for publishing major versions of Diego, and to a large extent our cadence there will depend on the scope and future direction of the Diego project within the CF ecosystem. That said, we understand that there is a natural tension between our desire to keep the Diego codebase and features clean and coherent, and the needs of the CF ecosystem and community to have a sufficiently stable set of APIs on which to build our platform. To that end, we expect to release major versions of Diego no more frequently than every three months, and more realistically approximately every six months to a year.


## Testing strategies

For a given upgradable version pair `(V1, V2)`, we intend to conduct a battery of tests to ensure that deployments with different combinations of components at `V1` and `V2` function correctly. These tests should include being able to run, scale up, and scale down existing apps created in the `V1` deployment, and to stage and run apps created in the mixed deployment. It will be most important to verify that `V1` cells are compatible with `V2` inputs and vice-versa, so we will test these operations against the following combinations of components:


| Configuration  | BBS  | Cells | Brain/Bridge/CF |
|----------------|------|-------|-----------------|
| `C0` (Initial) | `V1` | `V1`  | `V1`            |
| `C1`           | `V2` | `V1`  | `V1`            |
| `C2`           | `V2` | `V2`  | `V1`            |
| `C3`           | `V2` | `V1`  | `V2`            |
| `C4`           | `V2` | `V2`  | `V2`            |


Rather than attempting these operations against a deployment in flight during a monolithic upgrade of the Diego deployment, we can progress through these combinations intentionally and deterministically by arranging the Diego deployment as four separate, coordinated deployments:

- `D1`: Database nodes
- `D2`: Brain, CC-Bridge, Route-Emitter, and Access nodes
- `D3`: Cell group 1
- `D4`: Cell group 2

The CF deployment will be separate, as it is today. The configurations above can then be achieved with the following sequence of BOSH operations: 

- `C1`: Deploy `V2` to `D1`
- `C2`: Deploy `V2` to `D3`, stop `D4`
- `C3`: Deploy `V2` to `D2` and CF, start `D4`, stop `D3`
- `C4`: Deploy `V2` to `D4`, start `D3`

Assuming that BOSH stopping a VM triggers draining, so that the Diego cells evacuate correctly, these operations should allow existing app instances to be routable continually.

We detail the tests for each one of these configurations below:

### `C0`: System at `V1`

With the system in its initial state, we seed it with an instance of Dora, Grace, or another test application asset and verify it is routable. For the sake of illustration below, assume that the app is Dora.


### `C1`: BBS at `V2`, Cells and Brain/Bridge at `V1`

During the upgrade of the BBS to `V2`, we verify that the Dora instance is always routable.

After upgrading the BBS to `V2`, we verify that the system functions correctly:

- The existing Dora scales up to 2 instances that are all routable, then back down to 1.
- Stage and run a new instance of Dora, verify that it is routable, scale it up and down, then delete it.


### `C2`: BBS and Cells at `V2`, Brain/Bridge at `V1`

During the upgrade of the `D3` cells to `V2` and the shutdown of the `D4` cells, we verify that the Dora instance is always routable.

With all the active Cells at `V2`, we verify that the system functions correctly:

- The existing Dora scales up to 2 instances that are all routable, then back down to 1.
- Stage and run a new instance of Dora, verify that it is routable, scale it up and down, then delete it.


### `C3`: BBS and Brain/Bridge/CF at `V2`, Cells at `V1`

During the upgrade of the Brain and CC-Bridge to `V2`, we verify that the Dora instance is always routable.

During the upgrade of the CF deployment to `V2`, the gorouter and HAproxy will likely be offline for some amount of time, so the Dora instance is not expected to be routable externally then.

During the re-start of the `D4` cells and the stop of the `D3` cells, the Dora instance should be routable, as it will evacuate from the `D3` cells to the `D4` cells.

With all the active Cells at `V1` and the rest of the system at `V2`, we verify that the system functions correctly:

- The existing Dora scales up to 2 instances that are all routable, then back down to 1.
- Stage and run a new instance of Dora, verify that it is routable, scale it up and down, then delete it.

> If we introduce changes in the RunInfo that cannot be represented appropriately in the `V1` BBS API, do we expect that `V1` cells will be able to run that work? If not, is there any way we can ensure that older, incompatible cells won't be assigned work they can't do?


### `C4`: BBS, Cells, and Brain/Bridge/CF all at `V2`

With the entire system at `V2`, we verify that the system functions correctly:

- The existing Dora scales up to 2 instances that are all routable, then back down to 1.
- Stage and run a new instance of Dora, verify that it is routable, scale it up and down, then delete it.


## Minimal Deployment on BOSH-Lite

In order to run this test suite effectively in CI, we propose that the target environment be an instance of BOSH-Lite running on AWS, which will be easy to provision on demand and fast to deploy `V1` and then to migrate to `V2`. The deployments above can then consist of the following groups of VMs:

- `D1`: 3-node database cluster (to ensure minimal downtime during the update to `V2`)
- `D2`: 1 node each of the Brain, CC-Bridge, Access, and Route-Emitter
- `D3`: 1 Cell
- `D4`: 1 Cell
- CF: Minimal CF, with no need for the DEAs or HM9000

We may even be able to capture an image of the BOSH-Lite instance after deploying `V1` and pushing the initial set of apps, and begin the test suite by restarting a copy of that instance and verifying that the system is still functional before proceeding (although with BOSH-Lite this may require that the instance always have the same IP).


## Selection of Versions for Testing

While we would ideally conduct these tests for every strict upgradable version pair `(V1, V2)`, as we generate new minor and patch versions of the release this will be prohibitively expensive to implement, and would likely have strongly diminishing returns. Instead, we intend to test upgrades where `V1` is the earliest supported version within its major version, and `V2` is the HEAD of the develop branch (after passing smoke tests or perhaps CATs/DATs on ketchup). In the near-term, these `V1` versions will be:

- Version 0: earliest supported stable Diego (0.1434.0)
- Version 1: 1.0.0

When we release Version 2 of Diego, we will cease testing upgrades from 0.1434.0, but continue to test upgrades from 1.0.0 to the latest develop branch.

We also recognize that the addition of significant new functionality in a minor version may also warrant its promotion to a `V1` against which we test upgrades as part of CI.

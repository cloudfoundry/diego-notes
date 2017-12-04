# Upgrade Testing V2<a id="sec-1" name="sec-1"></a>

## Updates from [the previous version](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md)<a id="sec-1-1" name="sec-1-1"></a>

We revisit release version testing with the following changes in mind:
-   The recommended way to deploy CloudFoundry is [cf-deployment](https://github.com/cloudfoundry/cf-deployment) instead of [cf-release](https://github.com/cloudfoundry/cf-release) & [diego-release](https://github.com/cloudfoundry/diego-release)
-   No more separate instance groups for each AZ
-   Desire to reduce the number of component that are outside the control of diego-release. Deploying cf-release for example has proven to be time consuming as breaking changes occur over time
-   Desire not to rely on BOSH in the test suite:
    1.  Doing the initial deploy and subsequent updates through BOSH is slow
    2.  A Ginkgo test suite similar to inigo allows us to iterate faster and could be structured in a way to allow developers to run more focused subset of the test suite.

## Things that remained from [the previous version](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md)<a id="sec-1-2" name="sec-1-2"></a>

-   Operational guarantees hasn't changed. Refer to [this section](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#operational-guarantees) in the original version of the document
-   We don't test every possible upgrade path (i.e. every V0 V1 pair) See [this section](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#selection-of-versions-for-testing) for the rationale

# Test configurations<a id="sec-2" name="sec-2"></a>

## BBS running with Consul (v1.0.0 -> develop)<a id="sec-2-1" name="sec-2-1"></a>

| Configuration | BBS | Diego Client | Router (2) | SSH proxy + Auctioneer | Cells (2) | Notes                         |
|---------------|-----|--------------|------------|------------------------|-----------|-------------------------------|
| C0            | v0  | v0           | v0         | v0                     | v0        | Initial configuration         |
| C1            | v1  | v0           | v0         | v0                     | v0        | Simulates upgrading diego-api |
| C2            | v1  | v1           | v0         | v0                     | v0        | Simulates API upgrading       |
| C3            | v1  | v1           | v1         | v0                     | v0        | Simulates Router upgrading    |
| C4            | v1  | v1           | v1         | v1                     | v0        | Simulates scheduler upgrading |
| C5            | v1  | v1           | v1         | v1                     | v1        | Simulates cell upgrading      |

## BBS running with locket (v1.25.2 -> develop)<a id="sec-2-2" name="sec-2-2"></a>

| Configuration | Locket | BBS | Diego Client | Router (2) | SSH proxy + Auctioneer | Cells (2) | Notes                         |
|---------------|--------|-----|--------------|------------|------------------------|-----------|-------------------------------|
| C0            | v0     | v0  | v0           | v0         | v0                     | v0        | Initial configuration         |
| C1            | v1     | v0  | v0           | v0         | v0                     | v0        | Simulates upgrading diego-api |
| C2            | v0     | v1  | v0           | v0         | v0                     | v0        |                               |
| C3            | v1     | v1  | v1           | v0         | v0                     | v0        | Simulates API upgrading       |
| C4            | v1     | v1  | v1           | v1         | v0                     | v0        | Simulates Router upgrading    |
| C5            | v1     | v1  | v1           | v1         | v1                     | v0        | Simulates scheduler upgrading |
| C6            | v1     | v1  | v1           | v1         | v1                     | v1        | Simulates cell upgrading      |

## Notes<a id="sec-2-3" name="sec-2-3"></a>

-   The above configuration have the added benefit of testing the contract between rep and BBS regarding cell presences (newer BBS can still parse old rep cell presence in both Consul or Locket)
-   Deploy only one instance for all instance groups except Router and Cell to ensure routability to the test app at all times
-   Cells are to be updated gracefuly (i.e. each cell evacuates before it stops) to ensure routability to the test app at all times
-   Configurations follow the order of instance group updates as of [this version of cf-deployment](https://github.com/cloudfoundry/cf-deployment/commit/9be2644da8de08540891e24856bbdb88f9a83f67)
-   We have to test different versions of BBS and Locket since different versions can exist during a rolling update
-   SSH proxy and auctioneer don't communicate with each other and exist on the same VM, hence we update them at the same time

# Tests to run<a id="sec-3" name="sec-3"></a>

## Long running tests<a id="sec-3-1" name="sec-3-1"></a>

-   Similar to current [DUSTs](https://github.com/cloudfoundry/diego-upgrade-stability-tests) there will always be an app pushed right after `C0` and constantly checked for routability

## Tests to run after each configuration update<a id="sec-3-2" name="sec-3-2"></a>

-   Run vizzini (v0 for Diego client v0 or v1.x for more recent Diego clients)
-   This could potentially be a separate Ginkgo `Context` each configuration which start the corresponding versions and run the smoke test

# Concerns<a id="sec-4" name="sec-4"></a>

-   Is HTTP routing sufficient ?
    -   we decided not to do anything since the routers upgrade before the cells
    -   it is the responsibility of the routing team to ensure the router/routing-api are backward compatible with the cells
-   How do we deal with locket upgrades ? locket was introduced in v1.6.2 of diego-release but was experimental until 1.25.2
    -   We ended up with separately testing a configuration that uses Consul vs one that uses Locket
-   does everyone follow the same upgrade order in cf-dep ? What about cell blocks with different placement tags for example
    -   We think that is something we should communicate with operators, i.e. cells need to upgrade last, instead of testing different upgrade orders
-   getting rid of bosh-lite removes all test coverage we had for bosh upgradability. for example not cleaning up pid files; which is something we ran into before
    -   we think we should have a separate test suite for that, i.e. deploy C0 then upgrade to C5 and make sure everything come up and is healthy
-   We had a regeression with quota enforcement due to a breaking change in rep how can we excercise these kind of changes in a generic way that doesn't require prior knowledge of the bug:
    -   run the entire vizzini test suite
-   How do we test the diego client
    -   run v0 or v1 of vizzini which will excercise the bbs api

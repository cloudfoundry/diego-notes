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


| Configuration | Locket | BBS | Diego Client | Router (2) | SSH proxy + Auctioneer | Cells (2) | Notes                         |
|---------------|--------|-----|--------------|------------|------------------------|-----------|-------------------------------|
| C0            | v0     | v0  | v0           | v0         | v0                     | v0        | Initial configuration         |
| C1            | v1     | v0  | v0           | v0         | v0                     | v0        | Simulates upgrading diego-api |
| C2            | v0     | v1  | v0           | v0         | v0                     | v0        |                               |
| C3            | v1     | v1  | v1           | v0         | v0                     | v0        | Simulates API upgrading       |
| C3            | v1     | v1  | v1           | v1         | v0                     | v0        | Simulates Router upgrading    |
| C4            | v1     | v1  | v1           | v1         | v1                     | v0        | Simulates scheduler upgrading |
| C5            | v1     | v1  | v1           | v1         | v1                     | v1        | Simulates cell upgrading      |

## Notes<a id="sec-2-1" name="sec-2-1"></a>

-   Deploy only one instance for all instance groups except Router and Cell to ensure routability to the test app at all times
-   Cells are to be updated gracefuly (i.e. each cell evacuates before it stops) to ensure routability to the test app at all times
-   Configurations follow the order of instance group updates as of [this version of cf-deployment](https://github.com/cloudfoundry/cf-deployment/commit/9be2644da8de08540891e24856bbdb88f9a83f67)
-   We have to test both combinations of different versions of BBS and Locket since different versions can exist during a rolling update
-   SSH proxy and auctioneer don't communicate with each other and exist on the same VM, hence we update them at the same time

# Tests to run<a id="sec-3" name="sec-3"></a>

## Long running tests<a id="sec-3-1" name="sec-3-1"></a>

-   Similar to current [DUSTs](https://github.com/cloudfoundry/diego-upgrade-stability-tests) there will always be an app pushed right after `C0` and constantly checked for routability

## Tests to run after each configuration update<a id="sec-3-2" name="sec-3-2"></a>

-   Again, we can adapt DUSTs smoke test for this
    -   push an app and make sure it is routable
    -   scale up the app and make sure the 2 instances are routable
-   Should also add ssh to the smoke test
-   May be run a stripped down version of inigo/vizzini (doesn't have to be the actual test suite, as long as it offers the same test coverage) ?
-   This could potentially be a separate Ginkgo `Context` each configuration which start the corresponding versions and run the smoke test

# Questions<a id="sec-4" name="sec-4"></a>

-   Is HTTP routing sufficient ?
    -   we don't need to do anything since the routers upgrade before the cells (**preferred solution**)
    -   it is the responsbility of the routing team to ensure the router/routing-api are backward compatible with an cells
-   does everyone follow the same upgrade order in cf-dep ? What about cell blocks with different placement tags for example
    -   We should communicate this requirement to operators, i.e. cells need to upgrade last (**preferred solution**)
-   getting rid of bosh-lite removes all test coverage we had for bosh upgradability. for example not cleaning up pid files; which is something we ran into before
    -   can we ask release-integration to own this ?
        - not sure we should. It seems reasonable for us to own our own integration with bosh upgradability. The minimal test suite C0 -> C5 should be sufficient.
    -   we can also have a more focused test suite, i.e. deploy C0 then upgrade to C5 and make sure everything come up and is healthy
-   Which subset of vizzini/inigo are we interested in ?
    -   run the entire vizzini suite, it is fast enough. This way we don't have to worry about client version, i.e. when the diego client is V0 run vizzini from diego-release-v0 (**preferred solution**). The only drawback is that we cannot add any tests to the v0 vizzini and we don't have access to the v0 bbs client
-   We had a regression with quota enforcement due to a breaking change in rep how can we exercise these kind of changes in a generic way that doesn't require prior knowledge of the bug:
    -   run the entire vizzini test suite (**preferred solution** +1)
    -   we should exercise the different capacity constraints and ensure we don't violate them, total disk & memory usage of apps never go above the limit
    -   More generically i think we need to test both good and happy paths
        - Does this mean both a good path and a happy path, or two paths which are each good and happy? Which paths are these?
-   How do we test the diego client
    -   run v0 or v1 of vizzini which will exercise the bbs api (**preferred solution** +1)
    -   gopkg.in/diego-client-v0 ? (use directly from the test suite)
    -   cfdot-v0 the only drawback is that we are not exercising the protobuf api directly
        - this is a significant drawback
    -   some other binary that can exercise the proto api compiled with v0 bbs client
-   Is locket downgradable?
    - We probably don't want to use dusts tests to ensure their downgradability, if we even care about it.
    - if not, we could deploy two instances of locket, L1 and L2, both at v0. On C1, we upgrade L1 -> v1. On C2, we test with L2 (which stays v2). On runs >=C3, we use L1 which is on v1.

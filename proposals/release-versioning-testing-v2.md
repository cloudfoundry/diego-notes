# Upgrade Testing V2<a id="sec-1" name="sec-1"></a>

## Updates from [the previous version](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md)<a id="sec-1-1" name="sec-1-1"></a>

We revisit release version testing and the current upgrade testing strategy (the DUSTs) with the following changes and considerations in mind:

- The recommended way to deploy Cloud Foundry is now [cf-deployment](https://github.com/cloudfoundry/cf-deployment) instead of the manifest-generation systems in [cf-release](https://github.com/cloudfoundry/cf-release) and [diego-release](https://github.com/cloudfoundry/diego-release), which will be removed soon.
- As a result, the instance groups are implicitly spread across AZs, instead of explicitly grouped by AZ, although some instance groups may still end up being parallelized.
- Breaking changes to other systems in a CF deployment have caused the DUSTs to fail much more often than an actual breaking change between the APIs or components under the control of the Diego team.
- Reliance on BOSH makes the initial deploy and subsequent updates slow. An inigo-style test suite written in Ginkgo could allow the Diego team to iterate faster and to run more focused subset of the test suite.

## Aspects remaining from [the previous version](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md)<a id="sec-1-2" name="sec-1-2"></a>

- We perceive there to be value in maintaining deployments at different heterogeneous combinations of versions and running an appropriate set of tests to validate functionality, instead of coincidentally running tests while proceeding through an upgrade of a single BOSH deployment.
- Operational guarantees have not changed regarding compatibility of versions of diego-release. Refer to [this section](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#operational-guarantees) in the original version of the document.
- We still do not require testing every possible upgrade path (i.e. every `(V0, V1)` pair) See [this section](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#selection-of-versions-for-testing) for the rationale.


# Tests to run<a id="sec-2" name="sec-2"></a>

## Long running tests<a id="sec-2-1" name="sec-2-1"></a>

- As in the current [DUSTs](https://github.com/cloudfoundry/diego-upgrade-stability-tests), there will always be an app pushed right after the initial configuration (`C0`) is deployed and constantly checked for routability, apart for during the upgrade of the Router itself.

## Tests to run after each configuration update<a id="sec-2-2" name="sec-2-2"></a>

- Run the vizzini test suite at the appropriate version to verify core functionality of the Diego BBS API. This test could be structured as its own isolated context, in which it starts the appropriate versions of the components and runs the vizzini tests, instead of coupling it to the sequential upgrade of the components.

# Test configurations<a id="sec-3" name="sec-3"></a>

We select two main configurations of importance, based on the initial major versions of the release and major architectural changes:

- Diego starting at v1.0.0, configured to use Consul and global route-emitters,
- Diego starting at v1.25.2, configured to use Locket with local route-emitters.


## BBS running with Consul (from v1.0.0 to develop)<a id="sec-3-1" name="sec-3-1"></a>

- Initial Diego version is v1.0.0.
- Infrastructure components: MySQL database, NATS, Consul.
- Route-emitter is configured in global mode, with Consul lock.
- Cell includes the Diego cell rep and garden-runc.

| Configuration | BBS | BBS Client (Vizzini) | Router     | SSH proxy + Auctioneer | Cells (2) | Notes                                                 |
|---------------|-----|--------------|------------|------------------------|-----------|-------------------------------------------------------|
| C0            | v0  | v0           | v0         | v0                     | v0        | Initial configuration                                 |
| C1            | v1  | v0           | v0         | v0                     | v0        | Simulates upgrading diego-api                         |
| C2            | v1  | v1           | v0         | v0                     | v0        | Simulates API upgrading                               |
| C3            | v1  | v1           | v1         | v0                     | v0        | Simulates Router upgrading                            |
| C4            | v1  | v1           | v1         | v1                     | v0        | Simulates scheduler upgrading                         |
| C5            | v1  | v1           | v1         | v1                     | v1        | Simulates cell upgrading: start v1 cell, stop v0 cell |


## BBS running with Locket (from v1.25.2 to develop)<a id="sec-3-2" name="sec-3-2"></a>

- Initial Diego version is v1.25.2.
- Infrastructure components: MySQL database, NATS.
- Route-emitter is configured in local mode, with Consul lock.
- Cell includes the Diego cell rep, garden-runc, grootfs, and local route-emitters.

| Configuration | Locket | BBS | BBS Client (Vizzini) | Router     | SSH proxy + Auctioneer | Cells (2) | Notes                                                     |
|---------------|--------|-----|--------------|------------|------------------------|-----------|-----------------------------------------------------------|
| C0            | v0     | v0  | v0           | v0         | v0                     | v0        | Initial configuration                                     |
| C1            | v1     | v0  | v0           | v0         | v0                     | v0        | Simulates upgrading diego-api                             |
| C2            | v0     | v1  | v0           | v0         | v0                     | v0        |                                                           |
| C3            | v1     | v1  | v1           | v0         | v0                     | v0        | Simulates API upgrading                                   |
| C4            | v1     | v1  | v1           | v1         | v0                     | v0        | Simulates Router upgrading                                |
| C5            | v1     | v1  | v1           | v1         | v1                     | v0        | Simulates scheduler upgrading                             |
| C6            | v1     | v1  | v1           | v1         | v1                     | v1        | Simulates cell upgrading: start v1 cell, evacuate v0 cell |


## Notes<a id="sec-3-3" name="sec-3-3"></a>

- The above configurations have the added benefit of testing the contract between rep and BBS regarding cell presences (namely, that the newer BBS can still parse old rep cell presences in both Consul or Locket).
- Deploy only one instance for all instance groups except the Cell to ensure routability to the test app at all times.
- Cells are to be updated gracefuly (that is, each cell evacuates before it stops) to ensure routability to the test app at all times.
- Configurations follow the order of instance group updates as of [this version of cf-deployment](https://github.com/cloudfoundry/cf-deployment/commit/9be2644da8de08540891e24856bbdb88f9a83f67).
- We should test both combinations of different versions of BBS and Locket since different versions can exist during a rolling update.
- The SSH proxy and auctioneer do not communicate with each other and exist on the same VM, hence we update them at the same time.


# Concerns<a id="sec-4" name="sec-4"></a>

- Routing: We think testing HTTP routing is sufficient, as that is the main routing tier of importance in CF at present. The Routing team is primarily responsible for ensuring interoperability of the clients and servers in the routing control plane, but because routing is of such key importance and because diego-release contains the route-emitter we do choose to test it in the DUSTs.
- Locket: We test with a separate configuration that includes Locket because of the importance of this architectural change.
- Cell update order: The most common update order in the deployment is to update the Diego control plane first (BBS, auctioneer, locket) and then the cells, so we focus on those configurations.
- BOSH upgradability: We do lose test coverage for ensuring that the components restart correctly, and we have encountered issues around pid-file management, but we could handle this with a different test suite.
- Quota enforcement: We did regress on enforcing instance quotas with a breaking change to the cell rep API, but there were vizzini tests that would have caught that regression.
- Diego client: Running the appropriate version of vizzini will provide sufficient coverage of the Diego BBS client functionality

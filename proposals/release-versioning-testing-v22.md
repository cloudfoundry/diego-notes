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

-   Operational guarantees hasn't changed. Refer to [this secion](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#operational-guarantees) in the original version of the document
-   We don't test every possible upgrade path (i.e. every V0 V1 pair) See [this secion](https://github.com/cloudfoundry/diego-dev-notes/blob/master/proposals/release-versioning-testing.md#selection-of-versions-for-testing) for the rationale

# Test configurations<a id="sec-2" name="sec-2"></a>

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="left" />

<col  class="left" />

<col  class="left" />

<col  class="left" />

<col  class="left" />

<col  class="left" />

<col  class="left" />

<col  class="left" />
</colgroup>
<tbody>
<tr>
<td class="left">Configuration</td>
<td class="left">Locket</td>
<td class="left">BBS</td>
<td class="left">Diego Client</td>
<td class="left">Router (2)</td>
<td class="left">SSH proxy + Auctioneer</td>
<td class="left">Cells (2)</td>
<td class="left">Notes</td>
</tr>


<tr>
<td class="left">C0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">Initial configuration</td>
</tr>


<tr>
<td class="left">C1</td>
<td class="left">v1</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">Simulates upgrading diego-api</td>
</tr>


<tr>
<td class="left">C2</td>
<td class="left">v0</td>
<td class="left">v1</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">&#xa0;</td>
</tr>


<tr>
<td class="left">C3</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">Simulates API upgrading</td>
</tr>


<tr>
<td class="left">C3</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v0</td>
<td class="left">v0</td>
<td class="left">Simulates Router upgrading</td>
</tr>


<tr>
<td class="left">C4</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v0</td>
<td class="left">Simulates scheduler upgrading</td>
</tr>


<tr>
<td class="left">C5</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">v1</td>
<td class="left">Simulates cell upgrading</td>
</tr>
</tbody>
</table>

## Notes<a id="sec-2-1" name="sec-2-1"></a>

-   Deploy only one instance for all instance groups except Router and Cell to ensure routability to the test app at all times
-   Cells are to be updated gracefuly (i.e. each cell evacuates before it stops) to ensure routability to the test app at all times
-   Configurations follow the order of instance group updates as of [this version of cf-deployment](https://github.com/cloudfoundry/cf-deployment/commit/9be2644da8de08540891e24856bbdb88f9a83f67)
-   We have to test different versions of BBS and Locket since different versions can exist during a rolling update
-   SSH proxy and auctioneer don't communicate with each other and exist on the same VM, hence we update them at the same time

# Tests to run<a id="sec-3" name="sec-3"></a>

## Long running tests<a id="sec-3-1" name="sec-3-1"></a>

-   Similar to current [DUSTs](https://github.com/cloudfoundry/diego-upgrade-stability-tests) there will always be an app pushed right after `C0` and constantly checked for routability

## Tests to run after each configuration update<a id="sec-3-2" name="sec-3-2"></a>

-   Again, we can adapt DUSTs smoke test for this
-   May be run a stripped down version of inigo/vizzini (doesn't have to be the actual test suite, as long as it offers the same test coverage) ?
-   This could potentially be a separate Ginkgo `Context` each configuration which start the corresponding versions and run the smoke test

# Questions<a id="sec-4" name="sec-4"></a>

-   Which subset of vizzini/inigo are we interested in ?
-   We had a regeression with quota enforcement due to a breaking change in rep how can we excercise these kind of changes in a generic way that doesn't require prior knowledge of the bug:
    -   we should excercise the different capacity constraints and ensure we don't violate them, total disk & memory usage of apps never go above the limit
    -   More generically i think we need to test both good and happy paths
    -   what else ?

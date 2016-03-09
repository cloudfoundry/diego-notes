Running everything from the same VM is hard. We had to implement semaphores to
get around a problem with having too many connections to the BBS in parallel.
We had satisfactory results with the CF-MySQL release even though they're
slightly worse than RDS. We think that once the real BBS schema is fully
flushed out we can optimize certain aspects of the complexity we have with
convergence, and the other bulkers.

One thing to note from the experiment is that there might be artificial
slowdowns because of the single VM acting as multiple components. We decided
that for now the complexity of devising a distributed benchmarks suite is too
costly to be justifiable.

## Experiment #5 (full benchmark run)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `500`
- `num_reps`: `1000`
- `desired_lrps`: `200000`
- Semaphore around all HTTP calls on the BBS client (25,000 resources)
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results:

```
Running Suite: Benchmark BBS Suite
==================================
Random Seed: 1457549669
Will run 4 of 4 specs

• [MEASUREMENT]
Fetching for Route Emitter
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/route_emitter_fetch_test.go:33
  data for route emitter
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/route_emitter_fetch_test.go:32

  Ran 10 samples:
  {FetchActualLRPsAndSchedulingInfos}
  fetch all actualLRPs:
    Fastest Time: 17.671s
    Slowest Time: 51.122s
    Average Time: 37.447s ± 15.995s
------------------------------
[====================================================================================================] 10000/10000
Done with all loops, waiting for queue to clear out...
[====================================================================================================] 2000000/2000000
• [MEASUREMENT]
Fetching for rep bulk loop
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:114
  data for rep bulk
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:113

  Ran 1 samples:
  {RepBulkFetching}
  rep bulk fetch:
    Fastest Time: 0.006s
    Slowest Time: 7.160s
    Average Time: 0.078s ± 0.269s
  {RepBulkLoop}
  rep bulk loop:
    Fastest Time: 0.008s
    Slowest Time: 7.163s
    Average Time: 0.089s ± 0.299s
  {RepStartActualLRP}
  start actual LRP:
    Fastest Time: 0.002s
    Slowest Time: 120.636s
    Average Time: 0.163s ± 2.066s
  {RepClaimActualLRP}
  claim actual LRP:
    Fastest Time: 0.007s
    Slowest Time: 120.522s
    Average Time: 0.434s ± 0.896s
------------------------------
• [MEASUREMENT]
Fetching for nsync bulker
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/nsync_bulker_fetch_test.go:29
  DesiredLRPs
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/nsync_bulker_fetch_test.go:28

  Ran 10 samples:
  {NsyncBulkerFetching}
  fetch all desired LRP scheduling info:
    Fastest Time: 14.950s
    Slowest Time: 15.450s
    Average Time: 15.127s ± 0.140s
------------------------------
• [MEASUREMENT]
Gathering
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/lrp_convergence_gather_test.go:33
  data for convergence
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/lrp_convergence_gather_test.go:32

  Ran 10 samples:
  {ConvergenceGathering}
  BBS' internal gathering of LRPs:
    Fastest Time: 21.318s
    Slowest Time: 28.017s
    Average Time: 26.993s ± 1.904s
------------------------------

Ran 4 of 4 Specs in 3514.024 seconds
SUCCESS! -- 4 Passed | 0 Failed | 0 Pending | 0 Skipped PASS

Ginkgo ran 1 suite in 58m35.914623967s
Test Suite Passed
```

### Conclusions:

We can make the HA MySQL release works assuming they solve the proxy/load
balancing issues. Convergence is a little slow still, but there's hope that
with a different schema we can really improve how we do convergence in general,
same holds true for nsync and route-emitter.

## Experiment #4 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `500`
- `num_reps`: `1000`
- `desired_lrps`: `200000`
- Semaphore around all HTTP calls on the BBS client (25,000 resources)
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results:

```
• [MEASUREMENT]
Fetching for rep bulk loop
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:114
  data for rep bulk
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:113

  Ran 1 samples:
  {RepBulkFetching}
  rep bulk fetch:
    Fastest Time: 0.005s
    Slowest Time: 3.409s
    Average Time: 0.060s ± 0.171s
  {RepBulkLoop}
  rep bulk loop:
    Fastest Time: 0.008s
    Slowest Time: 3.413s
    Average Time: 0.066s ± 0.175s
  {RepStartActualLRP}
  start actual LRP:
    Fastest Time: 0.002s
    Slowest Time: 121.628s
    Average Time: 0.116s ± 2.025s
  {RepClaimActualLRP}
  claim actual LRP:
    Fastest Time: 0.007s
    Slowest Time: 121.213s
    Average Time: 0.345s ± 0.979s
------------------------------
SS
Ran 1 of 4 Specs in 1820.280 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 3 Skipped PASS | FOCUSED
```

### Conclusions:

"Slowest" on claim/start is due to the artificial limits we have with the
semaphore on the BBS client. Could be mitigated by having a distributed
benchmark suite.  Going forward we should probably move that semaphore up to
the benchmark code, since it doesn't make too much sense for it to exist on the
BBS client (that many requests in parallel is not a realistic use case).

## Experiment #3 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `500`
- `num_reps`: `250`
- `desired_lrps`: `50000`
- Semaphore around all HTTP calls on the BBS client (25,000 resources)
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results:

```
------------------------------
• [MEASUREMENT]
Fetching for rep bulk loop
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:114
  data for rep bulk
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:113

  Ran 1 samples:
  {RepBulkFetching}
  rep bulk fetch:
    Fastest Time: 0.005s
    Slowest Time: 1.093s
    Average Time: 0.011s ± 0.048s
  {RepBulkLoop}
  rep bulk loop:
    Fastest Time: 0.007s
    Slowest Time: 1.096s
    Average Time: 0.014s ± 0.049s
  {RepStartActualLRP}
  start actual LRP:
    Fastest Time: 0.002s
    Slowest Time: 120.516s
    Average Time: 0.157s ± 0.609s
  {RepClaimActualLRP}
  claim actual LRP:
    Fastest Time: 0.007s
    Slowest Time: 3.162s
    Average Time: 0.028s ± 0.087s
------------------------------
S
Ran 1 of 4 Specs in 661.736 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 3 Skipped PASS | FOCUSED
```

Conclusions:

Semaphores work, let's move on to 200,000 again.

## Experiment #2 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `100`
- `num_reps`: `250`
- `desired_lrps`: `50000`
- Workpool of size 25,000 only around BBS client calls
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results:

**None**

### Conclusions:

We cancelled this experiment because our workpool implementation seems to have
issues with large pool sizes. The tests were taking way too long to finish and
we aborted them.  Next experiment will replace the workpool with a semaphore.

## Experiment #1 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `100`
- `num_reps`: `250`
- `desired_lrps`: `50000`
- No workpool

### Results:

```
• [MEASUREMENT]
Fetching for rep bulk loop
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:84
  data for rep bulk
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/rep_bulk_fetch_test.go:83

  Ran 1 samples:
  {RepBulkFetching}
  rep bulk fetch:
    Fastest Time: 0.005s
    Slowest Time: 1.610s
    Average Time: 0.016s ± 0.086s
  {RepBulkLoop}
  rep bulk loop:
    Fastest Time: 0.007s
    Slowest Time: 1.612s
    Average Time: 0.021s ± 0.088s
  {RepStartActualLRP}
  start actual LRP:
    Fastest Time: 0.003s
    Slowest Time: 4.093s
    Average Time: 0.054s ± 0.164s
  {RepClaimActualLRP}
  claim actual LRP:
    Fastest Time: 0.007s
    Slowest Time: 1.639s
    Average Time: 0.038s ± 0.128s
------------------------------
SSS
Ran 1 of 4 Specs in 661.140 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 3 Skipped PASS | FOCUSED
```

### Conclusions:

Given the success running the experiment at this size, we thought 50,000/250
would be a decent size to split the experiment on if we were to distribute it.
We thought about running multiple instances of the benchmark-bbs errand in
parallel, but that is not supported by BOSH.
The alternative would be making multiple deployments with the same errand with
different parameters and synchronizing the startup of those. Seems fragile.
We could also try to devise an smarter orchestrated test suite that spins up
tasks on a Diego deployment, but also seems hard. Trying with a workpool first.


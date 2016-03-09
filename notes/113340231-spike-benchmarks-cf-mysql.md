A lot has happened on this story before we started these notes. Current problem
is the fact that spinning up all the reps (1,000) on the same errand VM causes
them to try to `Start` all (200,000) LRP instances at the same time. That makes
the VM "run out" of ports, because of all the concurrent HTTP requests.

## Experiment #3 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `500`
- `num_reps`: `250`
- `desired_lrps`: `50000`
- Semaphore around all HTTP calls on the BBS client (25,000 resources)
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results

## Experiment #2 (focused on rep-bulk cycles)

### Config:

- CF-MySQL 3 nodes talking to one node directly
- `num_populate_workers`: `100`
- `num_reps`: `250`
- `desired_lrps`: `50000`
- Workpool of size 25,000 only around BBS client calls
- `MaxIdleConnsPerHost` set to 25,000 as well on the BBS client

### Results

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


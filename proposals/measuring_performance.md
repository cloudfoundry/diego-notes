# Missing metrics:

First add the following metrics

- Time to run a convergence loop in Rep
- Time to perform a sync in route-emitter
- Number of CellPresence's in etcd

# Set up the environment

We'll run experiments against Diego-1.
- We'll want to aggregate logs with papertrail
- We'll want to have a Datadog dashboard for Diego-1 to give us access to all our metrics.

# Experiments

For the following scales:
- 10 32-Gig Cells, 2 CCs, 2 dopplers
- 20 32-Gig Cells, 4 CCs, 4 dopplers
- 50 32-Gig Cells, 8 CCs, 8 dopplers
- 100 32-Gig Cells, 16 CCs, 16 dopplers

1. Deploy a Diego+CF environment to diego-1
2. Run the experiments 1 and 2
3. Let the cluster sit for a day
4. Run experiment 3

## Experiment 1:

Run Fezzik a few times.
Fezzik will continuously append to a reports.json file.  Save off the .json file in between runs and share it with everyone.  Fezzik will have tools to plot the data in the file.

## Experiment 2:

CF Push the following distribution of applications:

No. Applications | No. Instances/App | Memory | Details
-----------------|-------------------|--------|--------
N*54 | 1 | 128 | A simple Hello World application: not chatty (one log-line/second)
N*12 | 2 | 512 | An HA low-load microservice: moderately chatty logs (10 log-lines/second)
N*2  | 4 | 1024 | Web application: very chatty (20 log-lines/second)
N*10 | 1 | 128 | A perpetually crashing application (no logs).  This app should start, wait for 30 seconds, then crash.

Where N is the number of cells (so for the 10-Cell case we'd push 540 simple hello world apps).  All these applications should have the same disk limit (something low like 1 GB should be good enough).

This gives us 96 instances/Cell and uses 28GB of memory per Cell with ~10% of instances crashing.  These numbers are more agressive than the current prod workload by a factor of 2 (prod shows ~50 instances/cell and 5% of instances crashing).

We'll want to run several `cf push`es in parallel - probably from a number of boxes running on AWS.  I propose 2*N parallel CF pushes.  We'll want to note how long it takes to push all the applications.  We'll also want to note any failures that occur.  We should push Go applications and specify Go buildpacks -- this is more about load-testing Diego when its running many applications than it is about measuring the performance of staging.

Once the applications are up we'll want to collect a day's worth of datadog metrics and use them as a basis to figure out what we should investigate (either via papertrail logs or some other sort of deep dive).

## Experiment 3

After a day, kill N/10 cells and see how long it takes to recover the missing applications (datadog plots should be enough to tell us when we've recovered).
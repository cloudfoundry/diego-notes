# Set up the environment

We'll run experiments against Diego-1.
- We'll want to aggregate logs with papertrail
- We'll want to have a Datadog dashboard for Diego-1 to give us access to all our metrics.

# Experiments

For the following scales:
- 10 32-Gig Cells, 2 CCs, 2 dopplers
- 20 32-Gig Cells, 4 CCs, 4 dopplers
- 50 32-Gig Cells, 10 CCs, 10 dopplers
- 100 32-Gig Cells, 20 CCs, 20 dopplers

1. Deploy a Diego+CF environment to diego-1 (make sure we deploy an etcd cluster -- etcd is much faster when not running in clustered mode)
2. Run the experiments 1 and 2
3. Let the cluster sit for a day
4. Run experiment 3

## Experiment 1: Fezzik

Fezzik excercises the Receptor API and launches very many Tasks and LRPs in tandem.  These Tasks and LRPs are very lightweight so Fezzik is isolated to benchmarking scheduling Tasks/LRPs and spinning up/tearing down containers.  Fezzik runs off of just one machine -- this can be an artificial bottleneck.  Fezzik continuously appends to a reports.json file - I'm building some tooling to plot/analyze the contents of this file.

1. Spin up an AWS instance (e.g. add a new instance of something to BOSH)
2. Get on that instance and run Fezzik a few times.
3. Save off the reports.json file in between runs and share it with everyone.

## Experiment 2: Launching and running many CF applications

CF Push the following distribution of applications:

App Name | No. Applications | No. Instances/App | Memory | App-Bits Size | Details*
---------|------------------|-------------------|--------|---------------|-------
Westley | N*54 | 1 | 128 | ~1M | A simple Hello World application: not chatty (one log-line/second)
Max | N*12 | 2 | 512 | ~10M | An HA low-load microservice: moderately chatty logs (10 log-lines/second)
Princess | N*2  | 4 | 1024 | ~200M | Web application: very chatty (20 log-lines/second)
Humperdink | N*10 | 1 | 128 | ~1M | A perpetually crashing application (no logs).  This app should start, wait for 30 seconds, then crash.

(* After running this experiment at the 10-Cell scale, we found log chattiness put a lot of pressure on the Cells and slowed down many things, especially Garden Info calls.  In subsequent runs of this experiment, we removed the logging behaviour from all the apps.)

Where N is the number of cells (so for the 10-Cell case we'd push 540 simple hello world apps).  All these applications should have the same disk limit (something low like 1 GB should be good enough).

This gives us 96 instances/Cell and uses 28GB of memory per Cell with ~10% of instances crashing.  These numbers are more agressive than the current prod workload by a factor of 2 (prod shows ~50 instances/cell and 5% of instances crashing).

We'll want to run several `cf push`es in parallel.  I propose the following:

1. Spin up N/10 AWS instances.  Let's call these *pusher* instances.
2. On each *pusher* instance perform 40 rounds of parallel pushes:
  - Perform the following 10 times:
      - Round A: In parallel, push 13 Westley, 3 Max, 1 Princess, 3 Humperdink
      - Round B: In parallel, push 13 Westley, 3 Max, 0 Princess, 3 Humperdink
      - Round C: In parallel, push 14 Westley, 3 Max, 1 Princess, 2 Humperdink
      - Round D: In parallel, push 14 Westley, 3 Max, 0 Princess, 2 Humperdink
3. Each push should verify that we can route to the pushed applications.
4. We'll want to record the following timings and info:
   - how long each application took to push.
   - how long it takes to successfully curl each application.
   - CF_TRACE logs for all the pushes
   - applications logs during the push and curl of each application
   we'll want to aggregate these from all pushers.

Westley, Max, Princess, and Humperdink should all be Go applications and we should specify the Go buildpack to avoid the overhead of copying all the buildpacks in.

Once the applications are up we'll want to collect a day's worth of datadog metrics and use them as a basis to figure out what we should investigate (either via papertrail logs or some other sort of deep dive).

## Experiment 3: Fault-recovery

After a day, kill N/10 cells (in various zones) and see how long it takes to recover the missing applications.

We'll want:
- Datadog plots to see how long recovery takes
- Converger logs to analyze how long the converger takes to handle this scenario.

Leave those cells dead for the next experiment.

## Experiment 4: Tolerating catastrophic cell and etcd failure

Kill another N/10 cells (in various zone).  At this point, the workload will exceed capacity.  Anything running (except Humperdinks) should continue to run, and anything not running should continue to fail to be scheduled (unless it sneaks in after a Humperdink crashes).  See that this is the case.

Kill all the etcd nodes.  Make sure they (and the N/5 dead cells) are down for a while, long enough for several convergence ticks.

`bosh cck` to recover the missing Cells and etcd cluster.  How long does it take for us to recover?  How does the routing table handle this?

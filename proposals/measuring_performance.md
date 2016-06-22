# Performance Experimentation Plans On Diego [PEPOD]

**Table of Contents**

- [Events](#events)
  - [Tasks](#tasks)
  - [LRPs](#lrps)
- [Metrics](#metrics)
- [Experiments](#experiments)
- [Experiment 1: Fezzik](#experiment-1-fezzik)
- [Experiment 2: Launching and running many CF applications](#experiment-2-launching-and-running-many-cf-applications)
- [Experiment 3: Fault-recovery](#experiment-3-fault-recovery)
- [Experiment 4: Tolerating catastrophic cell and etcd failure](#experiment-4-tolerating-catastrophic-cell-and-etcd-failure)


## Events

Given the centralization of the logic on Diego, we've decided to focus our
efforts in analyzing logs straight from the BBS. Since every state transition
operation goes through it, we should have easy access to logging those and then
later processing those logs effectively. These should be accessible through the
BBS logs.

### Tasks

Running:
- Requested
- Started
- Completed

### LRPs

Running:
- Desired
- Claimed
- Running

Stopping:
- Stopping
- Stopped

On stable system, if no catastrophes happen, no events should be triggered.

These can be used to analyze both cold start smoke tests and full scale stress
tests.

## Metrics

We want to observe the running system using the following metrics. It is also
interesting to observe how they change as we're filling up the deployment.

- Desired LRPs
- Running LRPs
- Missing LRPs
- Request Latency
- Auction Time (\*)
- Bulk loop durations
  - converger
  - route-emitter
  - nsync
  - rep
- Cell State
- Route count
- BBS Master Elections (consul?)
- Container Creation

Success:

- Diego can run 250,000 LRPs in 1000 cells.
  - There is little or no degradation when pushing an app on an empty env as a full env.
  - The bulk loops perform within their loop interval at scale.
  - Cells can be upgrades when the system is at load with no loss of running applications.
    - Cell evacuation time on systems with X% full.
  - Diego recovers from data loss in a "reasonable" amount of time and currently running apps remain running.

Monitoring Points:

- bbs
- cells
- auctioneer
- route-emitter
- cc-bridge

## Experiments

We're assuming `cell`s are equivalent to an AWS `r3.xlarge`:
* 4 CPUs
* 30 GB RAM
* "Moderate" performance network
Plus:
* 1000 IOPS `io1` SSD for the ephemeral drive `/var/vcap/data`

We're then using a `c4.4xlarge` equivalent for the `database`:
* 16 CPUs
* 30 GB RAM
* "High" performance network

1. Deploy a Diego + CF environment
2. Run the experiments 1 and 2
3. Let the cluster sit for a day
4. Run experiment 3

## Experiment 1: Fezzik

Fezzik exercises the BBS API and launches very many Tasks and LRPs in tandem.
These Tasks and LRPs are very lightweight so Fezzik is isolated to benchmarking
scheduling Tasks/LRPs and spinning up/tearing down containers.

This is supposed to be a simple "start as many instances as fast as possible"
kind of test.  These instances should consist of very minimal workloads.
The workload in question should have little to no ongoing activity and should
be started up all at the same time to measure scheduling performance and find
general bottlenecks with basic functionality of the system.

For this experiment we plan to start (for `N` = number of cells in the deployment).

- `N * 10` Tasks
- `N * 10` LRPs
- `N * 50` Tasks
- `N * 50` LRPs
- `N * 100` Tasks
- `N * 100` LRPs
- `N * 200` Tasks
- `N * 200` LRPs


## Experiment 2: Launching and running many CF applications

Apps are defined as different modes of [this app](https://github.com/cloudfoundry-incubator/diego-stress-tests/tree/master/assets/apps/jim).

Initial mix of apps:

| Name     | Logs/s   | Req/s   | Crash?   | Instances/LRP   |
|----------|----------|---------|----------|-----------------|
| light    | 1        | 1       | no       | 13              |
| medium   | 5        | 2       | no       | 7               |
| heavy    | 7        | 3       | no       | 3               |
| crashing | 0        | 0       | yes      | 2               |

Fill 1,000 Cell with 250,000 LRPs/tasks (10,000 pushes of the above mix).

Once the system is up we would then want to simulate some regular load which
would push, stop, start, and crash apps.

Steady State mix of apps:

| Name     | Logs/s   | Req/s   | Crash?   | Instances/LRP   |
|----------|----------|---------|----------|-----------------|
| light    | 1        | 1       | no       | 13              |
| medium   | 5        | 2       | no       | 7               |
| heavy    | 7        | 3       | no       | 3               |
| crashing | 0        | 0       | yes      | 2               |

Process of steady state:

Push new apps (1,000 pushes of the above mix, total of 25,000 extra instances)

Continually:
- Delete app
- Push app
- Stop app

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

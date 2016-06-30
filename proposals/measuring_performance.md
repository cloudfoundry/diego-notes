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

### Extracting Metrics

We have previously used Loggregator to send metrics to Datadog via the
datadog-firehose-nozzle. We experienced frustration trying to generate custom
graphs that are more presentable and easier to understand outside of observing
a live deployment. For those reasons we want to have access to raw metrics
somehow as part of running the experiments described in this document.

We have a few options:

- Create a "JSON" firehose nozzzle that dumps all the metrics on disk on a JSON
  format. That can be later processed locally to draw graphs using a tool like
  [Grafana](http://grafana.org) to graph things after the fact while also
  sending metrics to Datadog so that we can observe the live system.
- Use the [graphite-firehose-nozzle](https://github.com/pivotal-cf/graphite-nozzle)
  to send metrics directly to the graphite time series data store and configure
  a Grafana instance to use that source to plot graphs live. Graphite should
  also allow us to query the data after the fact if we want.

Building a JSON firehose nozzle shouldn't be too difficult and it seems
valuable in general, than converting that data in a format that Grafana
understands also seems like a easily solvable problem.

Using the already existing graphite tools could save us some time in
development, but also seems like something not a lot of people use at this
time.

We can probably build the JSON nozzle separately and explore how that would
work before we make a decision. We also discussed having a "tee"-nozzle so that
we can minimize the amount of metrics traffic to nozzles if we want to have
both Datadog and JSON emission happening at the same time.

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
5. Run experiment 4

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

Apps are defined as different modes of [this app](https://github.com/cloudfoundry-incubator/diego-performance-app/tree/master/stress-test/stress-app).

The mix of apps can be found in the [manifest
template](https://github.com/cloudfoundry-incubator/diego-performance-app/blob/master/stress-test/stress-app/manifest.yml.template)
in the app's repository.

Initial mix of apps:

| Name          | Logs/s  | Req/s  | Crash?   | # of Apps  | Instances/App  | # of Instances  | Memory/Instance (MB) | Total Memory (MB) |
| -----------   | ------- | ------ | -------- | ---------- | -------------- | ----------------| -------------------- | ----------------- |
| light-group   | 1       | 1      | no       | 1          | 4              | 4               | 32                   | 128               |
| light         | 1       | 1      | no       | 9          | 1              | 9               | 32                   | 288               |
| medium-group  | 5       | 2      | no       | 1          | 2              | 2               | 128                  | 256               |
| medium        | 5       | 2      | no       | 7          | 1              | 7               | 128                  | 896               |
| heavy         | 7       | 3      | no       | 1          | 1              | 1               | 1024                 | 1024              |
| crashing      | 0       | 0      | 30s-360s | 2          | 1              | 2               | 128                  | 256               |
| ------------- | ------- | ------ | -------- | ---------- | -------------- | --------------- | -------------------- | ----------------- |
| total         |         |        |          | 21         |                | 25              |                      | 2848M             |

Fill 1000 Cells with 250,000 LRP instances from 210,000 LRPs (10,000 pushes of the above mix).
These instances will allocate a total of 28,480 GB of memory.
Non-crashing app instances account for 25,920 GB of this allocation.

Crashing apps will crash at a random period from 30s to 6 minutes to make their
crash count reset once in a while, so that we can have crashing apps even after
they would've been given up on.

Once the system is up we would then want to simulate some regular load which
would push, stop, start, and crash apps.

Push new apps (1000 pushes of the above mix, for a total of 25,000 extra instances).

Continually:
- Delete app
- Push app
- Stop app


## Experiment 3: Fault-recovery

After a day, kill N/10 cells across the AZs and see how long it takes to recover the missing applications.

We'll want:
- Plots to see how long recovery takes
- Convergence logs to analyze how long convergence takes to handle this scenario.

Leave those cells dead for the next experiment.

## Experiment 4: Tolerating catastrophic cell and database failure

Kill another N/10 cells, again roughly balanced across AZs.
At this point, the workload will exceed capacity.
Instances that are running should continue to run, except for
instances of crashing apps that have not yet crashed when these cells are killed.
Instances that are not running should not be re-placed,
except as some capacity opens up from instances that crash as configured.
Verify that we observe this behavior.

Kill all the database nodes.  Make sure they (and the N/5 dead cells) are down
for a while, long enough for several convergence ticks.

`bosh cck` to recover the missing Cells and Database VMs.  How long does it
take for us to recover?  How does the routing table handle this?

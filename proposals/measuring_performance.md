# Performance Experimentation Plans On Diego [PEPOD]

**Table of Contents**

- [Events](#events)
  - [Tasks](#tasks)
  - [LRPs](#lrps)
- [Metrics](#metrics)
- [Experiments](#experiments)
  - [Scaling the Deployment](#scaling-the-deployment)
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

We decided to go with InfluxDB and Grafana to solve those issues:

- A version of the influxdb-firehose-nozzle that works with current loggregator
  can be found at: https://github.com/luan/influxdb-firehose-nozzle. That fork
  updates loggregator libraries and code with most recent
  datadog-firehose-nozzle
- Grafana (http://grafana.org) can be either deployed with bosh
  (https://github.com/vito/grafana-boshrelease) or ran locally and pointed to a
  remote InfluxDB.
- InfluxDB can be deployed via bosh
  (https://github.com/vito/influxdb-boshrelease) its configuration is really
  minimal and should straightforward.

## Experiments

### Scaling the Deployment

`N` = Number of Cells

It's assumed that number of instances on the deployment is `N * 250`. The mix
is defined on the [next section](#experiment-2-apps-matrix) of this document.

The following numbers come from an estimate of total requests/s and logs/s on
the system (where logs = log and metric messages going through the loggregator
system).

#### CF Components Scaling

- Cloud Controller API:
  - Instances: `N / 5`
  - Size: `m3.large` - 7.5 GBs | 2 CPUs
- Cloud Controller Workers:
  - Instances: `N / 10`
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- Doppler:
  - Instances: `N / 10`
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- Traffic Controller:
  - Instances: `N / 40`
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- Router:
  - Instances: `N / 10`
  - Size: `c3.xlarge` - 7.5 GBs | 4 CPUs
- ETCD:
  - Instances: `3` (raft size)
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- Consul:
  - Instances: `3` (raft size)
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- NATS:
  - Instances: `2` (don't really need to scale this as it will actually make performance worse)
  - Size: `c3.large` - 3.75 GBs | 2 CPUs
- UAA:
  - Instances: `2` (don't think we need to scale this as it doesn't do that much work on our performance experiment)
  - Size: `m3.large` - 7.5 GBs | 2 CPUs

#### Diego Scaling

All Diego components besides the cell are singletons, that is, they are leader
elected and only have one active process at any given time. For that reason, it
doesn't make sense to have more than one per availability zone, since that will
not help with scaling at all. So it's assumed that we are running one of each
of the Diego VMs per AZ.

**Note**: The singleton VMs will scale vertically, so choosing instance sizes
will be dependent on the number of cells deployed.

- Cell:
  - `r3.xlarge` equivalent
  - 4 CPUs
  - 30 GB RAM
  - "Moderate" performance network
  - 1,000 IOPS `io1` SSD for the ephemeral drive `/var/vcap/data`

- Database VM (BBS):
  - `c4.4xlarge` equivalent
  - 16 CPUs
  - 30 GB RAM
  - "High" performance network

- Brain VM:
  - `m3.medium` equivalent
  - 1 CPU
  - 4 GB RAM
  - "Moderate" performance network

- CC Bridge VM:
  - `m3.medium` equivalent
  - 1 CPU
  - 4 GB RAM
  - "Moderate" performance network

- Access VM:
  - `m3.medium` equivalent
  - 1 CPU
  - 4 GB RAM
  - "Moderate" performance network

- Route Emitter VM:
  - `m3.medium` equivalent
  - 1 CPU
  - 4 GB RAM
  - "Moderate" performance network

### Experiment 1: Fezzik

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


### Experiment 2: Launching and running many CF applications

Apps are defined as different modes of [this app](https://github.com/cloudfoundry/diego-stress-tests/tree/master/assets/stress-app).

The mix of apps can be found in the [manifest
template](https://github.com/cloudfoundry/diego-stress-tests/tree/master/assets/stress-app/manifest.yml.template)
in the app's repository.

<span id="experiment-2-apps-matrix"></span>Initial mix of apps:

| Name           | Req/s  | Crash?   | # of Apps  | Instances/App  | # of Instances   | Memory/Instance (MB) | Total Memory (MB) |
| -----------    | ------ | -------- | ---------- | -------------- | ---------------- | -------------------- | ----------------- |
| *light-group*  | 0.5    | no       | 1          | 4              | 4                | 32                   | 128               |
| *light*        | 0.5    | no       | 9          | 1              | 9                | 32                   | 288               |
| *medium-group* | 1      | no       | 1          | 2              | 2                | 128                  | 256               |
| *medium*       | 1      | no       | 7          | 1              | 7                | 128                  | 896               |
| *heavy*        | 1.5    | no       | 1          | 1              | 1                | 1024                 | 1024              |
| *crashing*     | 0      | 30s-360s | 2          | 1              | 2                | 128                  | 256               |
| **Total**      | **17** |          | **21**     |                | **25**           |                      | **2848M**         |

For an N-cell deployment, push 10 * N batches of the above mix, for a total of 210 * N LRPs with a total of 250 * N instances.
These instances will allocate a total of 28.48 * N GB of memory, with non-crashing app instances accounting for 25.92 * N GB of this allocation.

Fill 1000 Cells with 250,000 LRP instances from 210,000 LRPs (10,000 pushes of the above mix).
These instances will allocate a total of 28,480 GB of memory.
Non-crashing app instances account for 25,920 GB of this allocation.

Crashing apps will crash at a random period from 30s to 6 minutes to make their
crash count reset once in a while, so that we can have crashing apps even after
they would've been given up on.

Once the system is filled to this initial baseline, we will then generate some
continual realistic load of pushing, starting, crashing, and deleting additional apps.
For N batches of the seeding mix of apps, continually push the apps in the batches,
wait for them to run or to crash, then delete them.

The pushing will be done by creation of a batch of master applications and
using the `cf copy-bits` to copy the compiled application bits to a set of
`dummy` deployed non-started applications.   Once the bits are copied a `cf
start` will be called on those cloned applications.

### Experiment 3: Fault-recovery

After a day, kill N/10 cells across the AZs and see how long it takes to recover the missing applications.

We'll want:
- Plots to see how long recovery takes
- Convergence logs to analyze how long convergence takes to handle this scenario.

Leave those cells dead for the next experiment.

### Experiment 4: Tolerating catastrophic cell and database failure

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

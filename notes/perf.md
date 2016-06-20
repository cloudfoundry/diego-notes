Initial mix of apps:
 | name     | logs/s   | req/s   | crash?   | instances/lrp   |
 | ------   | -------- | ------- | -------- | --------------- |
 | light    | 1        | 1       | no       | 15              |
 | medium   | 5        | 2       | no       | 7               |
 | heavy    | 7        | 3       | no       | 3               |
 | crashing | 0        | 0       | yes      | 0               |

Fill 1000 Cell with 225,000 LRPs/tasks.

Once the system is up we would then want to simulate some regular load which
would push, stop, start, and crash apps.

Steady State mix of apps:
 | name     | logs/s   | req/s   | crash?   | instances/lrp   |
 | ------   | -------- | ------- | -------- | --------------- |
 | light    | 1        | 1       | no       | 15              |
 | medium   | 5        | 2       | no       | 7               |
 | heavy    | 7        | 3       | no       | 3               |
 | crashing | 0        | 0       | yes      | 3               |

Process of steady state:

Push new apps (including crashing apps)

Continually
- Delete app (So restage will occur)
- Push app
- Stop app

# Events
A good abstraction for what en Event is:
```go
type Event struct{
  Category string // {Staging,Running}
  Name string
  AppGuid string
  Index int // -1 for staging task
  InstanceGuid string
  Timestamp float64
}

How about using the events from the BBS.  Write a simple watcher to get the information and log it somewhere.
We could enhance the event to add "timestamp" to the event and be able to better use these for per testing.
```

For a push from scratch these are events the we can observe:

Staging:
- [bbs] Requested
- [auctioneer] (?) Placed [this could be a metric on auction times]
- [bbs] Started
- [rep] Finished
- [bbs] Uploaded (a.k.a. Complete)
Running:
- [bbs] Desired
- [auctioneer] (?) Placed [this could be a metric on auction times]
- [bbs] Claimed
- [bbs] Running
- [bbs] Stopping
- [bbs] Stopped

On stable system, if no catastrophes happen, no events should be triggered.

# General Metrics
We want to observe the running system using the following metrics. It is also
interesting to observe how they change as we're filling up the deployment.

- Desired LRPs
- Running LRPs
- Missing LRPs
- Request Latency
- Bulk loop durations
  - converger
  - route-emitter
  - nsync
  - rep
- Cell State
- Route count
- BBS Master Elections (consul?)
- Container Creation

Metrics we want to see:

- How long on average does it take to start a new app.   When the system is X% full
- How long does it take to get the list of all containers (apps).

Success:

- Diego can run 250,000 LRPs in 1000 cells.
  - There is little or no degradation when pushing an app on an empty env as a full env.
  - The bulk loops perform within their loop interval at scale.
  - Cells can be upgrades When the system is at load with no loss of running applications.
  - Diego recovers from data loss in a "reasonable" amount of time and currently running apps remain running.

Monitoring Points

- BBS
- Cells
- Auctioneer
- Route-Emitter
- cc-bridge

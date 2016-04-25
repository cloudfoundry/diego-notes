We want to know how the CF-MySQL release can take advantage of consul for
service discovery.

The release ships with a proxy job that can be horizontally scaled for high
availability, their responsibility is to chose one "master" node from the
Galera cluster and only connect clients to that one until it's no longer
available.

There is no way to make the proxy HA, though. The CF-MySQL team suggests using
a Load Balancer like Amazon ELB or HAProxy but cautions against it since load
balancing across multiple proxies might cause clients to connect to different
"master" MySQL nodes.  That is a problem because it can cause "deadlock" errors
when writing consistently to multiple masters on Galera.

We first considered removing the proxies entirely and relying on consul
directly on the MySQL nodes. However, if a MySQL node becomes unavailable,
consul will only prevent new connections to the dead node. There's no way for
the consul agent to close existing client connections like the proxies do.
Another problem is when shutting down the master MySQL node the MySQL server
can get into an irrecoverable state that causes established connections from
the clients to hang forever. This seems to be a known issue with tcp
connections that never really got addressed, see
[this](http://www.evanjones.ca/tcp-stuck-connection-mystery.html) for more
details.

So we experimented with putting consul agents on the proxies instead of the
consul nodes so that only one proxy is being used at a time. It seems that a
single proxy node will consistently connect to one MySQL node (as long as that
node is up), so we shouldn't run into deadlocks.

The first few experiments were not properly tracked so we're starting from the
middle here.

## Experiment 1

### Config

* 2 MySQL Nodes
* 2 Proxies (1 with consul agent running)
* 1 Arbitrator (a fake MySQL node that serves as a tie breaker)
* Patched BBS with a retry logic on Deadlock errors

We stopped the master MySQL during seeding roughly after 20k insertions and
restarted it after 30k.

The results seem to indicate that this is a viable solution. There were some
deadlock errors when bringing the MySQL node back online but the retry logic
spiked on the BBS mitigated that.

### Results

```ruby
• [MEASUREMENT]
main benchmark test
/var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/benchmark_test.go:210
  data for benchamrks
  /var/vcap/packages/benchmark-bbs/src/github.com/cloudfoundry-incubator/benchmark-bbs/benchmark_test.go:209

  Ran 1 samples:
  map[MetricName:RepBulkFetching]
  rep bulk fetch:
    Fastest Time: 0.008s
    Slowest Time: 1.831s
    Average Time: 0.087s ± 0.159s
  map[MetricName:RepBulkLoop]
  rep bulk loop:
    Fastest Time: 0.014s
    Slowest Time: 1.837s
    Average Time: 0.099s ± 0.161s
  map[MetricName:RepStartActualLRP]
  start actual LRP:
    Fastest Time: 0.010s
    Slowest Time: 30.599s
    Average Time: 0.292s ± 1.438s
  map[MetricName:RepClaimActualLRP]
  claim actual LRP:
    Fastest Time: 0.018s
    Slowest Time: 1.738s
    Average Time: 0.118s ± 0.133s
  map[MetricName:FetchActualLRPsAndSchedulingInfos]
  fetch all actualLRPs:
    Fastest Time: 5.633s
    Slowest Time: 5.633s
    Average Time: 5.633s ± 0.000s
  map[MetricName:ConvergenceGathering]
  BBS internal gathering of LRPs:
    Fastest Time: 6.244s
    Slowest Time: 6.244s
    Average Time: 6.244s ± 0.000s
  map[MetricName:NsyncBulkerFetching]
  fetch all desired LRP scheduling info:
    Fastest Time: 2.002s
    Slowest Time: 2.002s
    Average Time: 2.002s ± 0.000s
------------------------------

Ran 1 of 1 Specs in 627.672 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 0 Skipped

Ginkgo ran 1 suite in 10m31.388780701s
```



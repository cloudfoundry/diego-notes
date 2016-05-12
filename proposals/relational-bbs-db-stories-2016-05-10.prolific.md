As a Diego operator, I should be able to use Postgres as a backing relational store for the Diego BBS

We have received feedback that some operators would prefer to use Postgres as their backing relational store for the BBS API, as they have operational familiarity with that database instead of MySQL. CC and UAA also provide top-level support for both MySQL and postgres, so supporting both for Diego's BBS database is reasonable.

### Acceptance

- I can follow documentation to install and configure postgres on my local workstation and then run BBS test suites against it.
- I can observe from test runs in CI that appropriate test coverage is run against Postgres.

L: bbs:relational

---

As a Diego PM, I expect that failures in the super-laser tests and etcd-to-MySQL DUSTs run should block delivery of diego-release

In order to guarantee support for Diego when backed by a relational store, we should arrange for test runs in CI against a relational store to gate delivery of diego-release.

BLOCKED on #117843841: running an etcd-to-MySQL DUSTs run.

L: bbs:relational, blocked

---

[RELEASE] Diego with relational BBS fully supported

L: bbs:relational

---

Update performance-testing protocol for 250K-instance, 1000+-cell end-to-end performance test

Update the [performance-testing protocol in the diego-dev-notes](https://github.com/cloudfoundry-incubator/diego-dev-notes/blob/master/proposals/measuring_performance.md) for the 250K-instance experiment we intend to run. Address the following points:

- Distribution of and resource limits for app instances: at only 1000 cells, container density will be 250 per cell, which is right at the default limit for garden-linux (and garden-runc?)
- Prescribed IOPS configuration for cell disk at that scale
- Consistency of app behavior (we seem to have turned off substantial app logging in later experiments?)
- Collection of logs and metrics from the environment: for example, we learned that Datadog is not ideal for later analysis of metrics

Let's have a separate story to add extra experiments that we think would be valuable to run.


### Acceptance

The performance testing protocol has explicit configuration parameters for the 250K-instance test and is up-to-date and consistent in its Diego terminology. It also should account for any deficiencies we observed in data processing capabilities after the 10K-instance test run.

L: bbs:relational, perf, perf:breadth

---

As a Diego operator, I expect to be able to upgrade my MySQL-backed deployment from the earliest supported version to the latest without downtime

We can ensure this by configuring CI to run the DUSTs from a MySQL-backed deployment on that earliest supported version to a MySQL-backed deployment on the latest candidate version.

BLOCKED on declaring support for a relational store in a future Diego final release.

L: bbs:relational, blocked, pipeline

---

[CHORE] Migrate ketchup to a relational store and destroy the super-laser environment

BLOCKED on declaring support for a relational store in a future Diego final release.

L: bbs:relational, blocked, pipeline

---

Placeholder: remaining relational-BBS topics (migration validation, especially on failure)

### Pre-support

- charter: investigate migration failure scenarios
- make sure BBS can run SQL migrations (need migration to drive this for sure, could use the timeout-units one)


L: bbs:relational, placeholder

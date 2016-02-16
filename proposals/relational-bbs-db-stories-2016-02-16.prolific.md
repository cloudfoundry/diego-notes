As a Diego developer, I expect the Diego BBS-benchmark test suite to include per-record reads and writes in the rep bulk loop

The rep bulk loop tests in the BBS benchmark suite should perform individual reads and some writes (perhaps proportional to the number of instances) after their bulk loop, to match the actual rep code more closely.

For 200K instances, can we do these requests all from the same ginkgo process, if its host has enough CPU/bandwidth/connections available, or do we need to distribute this test across several VMs to avoid resource bottlenecks at the test site?

Acceptance: Benchmark test suites running in CI include these per-record writes and reads in their rep bulk loops.

L: perf, perf:breadth

---

As a Diego developer, I can run BBS unit tests against MySQL on my local workstation

The BBS should be able to pass its unit tests when run locally against a MySQL database, as well as against etcd. For the component integration test, pass in the required MySQL connection information via flags.

This should enclude support for encrypting the parts of Task, DesiredLRP, and ActualLRP records that should be encrypted and managing the encryption key name in the MySQL database.

Also document the steps needed to install MySQL on a development workstation.

Acceptance: I can follow the MySQL installation steps in the CONTRIBUTING document in diego-release and then successfully run the BBS unit tests.

L: bbs:relational

---

As a Diego developer, I expect BBS unit tests to run against both MySQL and etcd in CI

Now that the BBS unit tests run against both MySQL and etcd, configure CI to run them both against etcd and MySQL in the 'units' concourse job. This also entails installing any new dependencies in the appropriate pipeline Docker image.

Acceptance: I can verify from the 'units' job output that the expected tests suites are run.

L: bbs:relational

---

As a Diego developer, I expect inigo and component integration test suites to run against both MySQL and etcd in CI

Extend MySQL support to the other non-BOSH-based integration tests that our components run against the BBS. This includes inigo, the 'cmd' integration tests in each component, and a few other test suites (such as the operation test suites in the rep).

Acceptance: I can verify from the 'units' and 'inigo' job output that the expected tests suites are run.

L: bbs:relational

---

As a Diego operator, I can run a CF+Diego deployment backed by a MySQL DB instance

diego-release should allow the BBS to be BOSH-configured to use a MySQL database outside the Diego deployment as its store, with etcd still the default. This option should be configurable in the spiff-based manifest generation through the property-overrides stub.

Provide instructions and an auto-generated manifest for deploying a singleton MySQL instance to BOSH-Lite and configuring Diego to use it as its backend.

Acceptance: I can follow instructions to deploy CF+Diego to BOSH-Lite using a MySQL DB and then run CATs and vizzini against it successfully.  

L: bbs:relational

---

As a Diego developer, I expect Diego CI to run CATs and vizzini against a 'catsup' AWS environment with the BBS backed by an RDS MySQL instance

Create a new 'catsup' CF+Diego deployment on AWS analogous to ketchup, with a MySQL RDS instance as its backing store. Set up a pipeline parallel to ketchup that runs CATs and vizzini against catsup, both on changes to diego-release or cf-release and periodically.

Acceptance: CATs and vizzini against catsup are green.

L: bbs:relational

---

[RELEASE] Diego with relational BBS functional but experimental

L: bbs:relational

---

As a Diego developer, I expect to run Diego BBS benchmarks against an AWS environment with the BBS backed by an RDS MySQL instance
Create a new CF+Diego deployment on AWS analogous to the 'diego-3' environment and set up the BBS benchmarks to run against it, both on changes to diego-release at the appropriate level of stability (release-candidate?) and periodically. Configure it to emit metrics to Datadog and to store the test-run artifacts in a separate S3 bucket.

Aim to support 200K instances across 1000 reps initially.

Acceptance: BBS benchmarks pass consistently at the appropriate scale, and results are recorded in S3.

L: bbs:relational

---

As a Diego developer, I expect 'catsup' and the relational-benchmarks deployment to be backed by HA MySQL deployments

Deploy the CF MySQL release in a suitably HA configuration to catsup and the new relational benchmarks environment and configure the Diego deployment to use that as its backing store instead of RDS. Avoid usage of anything specific to AWS, such as ELBs, to make the MySQL deployment HA.

Acceptance: Catsup and the relational benchmarks environment function without RDS instances provisioned.

L: bbs:relational

---

BBS communicates to the relational store over SSL

The BBS can connect to its relational store using SSL. Any additional relevant information should be configurable via BOSH properties. Do we need to be able to supply a separate CA for use with that connection, or should we rely on the globally trusted or BOSH-deployed trusted certificates?

Acceptance: catsup and the relational benchmarks connect to their relational stores over SSL.

L: bbs:relational, security

---

[RELEASE] Diego with relational BBS supports 200K instances

L: bbs:relational, perf, perf:breadth

---

Placeholder: remaining relational-BBS topics (scale, Postgres, migration from etcd, manifest generation)

- BBS can use either MySQL or Postgres as a BBS store (BBS units run against all 3 backends)
- Diego manifest generation supports relational stores (BOSH-Lite: CF's postgres; catsup: HA MySQL)
- Diego migrates existing BBS data from etcd to its relational store
- Diego CI runs another DUSTs suite in etcd-to-relational migration mode
- [RELEASE] Diego with relational BBS fully supported
- Diego CI runs a DUSTs suite with only MySQL
- ketchup migrates to relational BBS store, catsup obsolete


L: bbs:relational, placeholder

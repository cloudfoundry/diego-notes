As a Diego operator, I expect the BBS to migrate existing data from etcd to the relational store

If the BBS is configured with connection information and credentials for both etcd and a relational store, it will migrate existing data in etcd to the relational store and then serve data only from the relational store. The migration mechanism should also ensure that if a BBS server becomes master that understands only how to serve data from etcd, it should realize that it is not current and relinquish the BBS lock. It should also ensure that this migration happens exactly once, so that subsequent BBS servers that start with the dual 

This likely needs to be done using the versioning system already implemented in the etcd store, so that BBSes all the way back to 0.1434.0 will function correctly. Consider the `CurrentVersion`/`TargetVersion` rules implemented in [the existing BBS migration mechanism](https://github.com/cloudfoundry-incubator/diego-dev-notes/blob/master/accepted_proposals/bbs-migrations.md#the-bbs-migration-mechanism).

Acceptance:
- I can configure a multi-instance BBS Diego first to populate etcd with data, then redeploy it with a relational configuration and observe that data is migrated from etcd to the relational store correctly (preserved, migration happens only once). 
- I can arrange the deployment with BBS both in etcd and in etcd+relational configurations and observe that once the data is migrated to the relational store, the etcd-only BBSes do not retain the lock during BBS elections.
- I can cause the etcd-to-relational migration to fail on one BBS node, and observe that it does not prevent etcd-only BBSes from taking over and serving requests or other etcd-and-relational BBSes from performing the migration later.
- If relational credentials are supplied to the BBS but a BBS node cannot connect on start-up, the deploy should fail on that node.


L: bbs:relational

---

As a Diego operator, I should be able to generate a Diego manifest with connection info for a relational store

An operator should be able to supply connection information to connect to the relational store they have provisioned externally. This information should come in as parameters in a stub (either in property-overrides or a new optional stub argument).

Acceptance:
- I can follow documentation in diego-release to place the relational connection information in the appropriate stub as an input to manifest-generation, and then use that to deploy Diego against MySQL on BOSH-Lite.

L: bbs:relational

---

Explore Diego system behavior with several CF MySQL nodes to discover how deadlock errors and rollbacks affect Diego correctness

We'd like to know how Diego deals with consistency errors if it's communicating with several different CF MySQL nodes simultaneously. Core Services has said we'll get deadlocks, but they will roll back and manifest as a failure client-side. Investigate how this affects the correctness of the Diego system, and generate stories with recommendations of how to mitigate the problems in the BBS, if possible. If this is impossibly bad, we may need to change the switchboard proxy to do leader election via, say, consul locks.

Also determine if we can configure the mysql client to accept several IPs, or if it requires only a single hostname. If the latter, we should explore colocating the consul agent on the proxy nodes to register them via Consul DNS.

Timebox to 2 days initially.

L: bbs:relational, charter

---

As a Diego operator, I should be able to follow documentation to deploy an single-node standalone CF-MySQL cluster on my infrastructure

The diego-release documentation should explain how to produce a deployment manifest for a single-node standalone CF-MySQL cluster. Ideally, we can link to documentation in the cf-mysql release itself with a high-level explanation of the required deployment parameters (instance counts, job types). If the cf-mysql-release makes this real difficult, talk to Core Services about getting it changed in their release. (Spoiler: Luan says they need this help for a minimal+standalone deployment option.)

Acceptance:
- I can follow documentation to provision a single-node standalone CF-MySQL cluster on AWS and then successfully deploy Diego to use that as its datastore via the manifest-generation scripts.

L: bbs:relational

---

As a Diego operator, I should be able to follow documentation to provision an RDS MySQL instance to support my AWS Diego deployment

The diego-release AWS documentation should explain how to provision an RDS MySQL instance manually in the VPC created for the BOSH+CF+Diego stack, and then how to provide the connection information to the Diego manifest-generation scripts. This configuration should be optional for now.

Let's also make this manual for now, and consider how to provision this with CloudFormation/BOOSH/bbl later.

Acceptance:
- I can follow the AWS-specific documentation in diego-release to provision this RDS instance and connect my AWS Diego deployment to it.

L: bbs:relational

---

As a Diego operator, I should be able to follow documentation to deploy an HA standalone CF-MySQL cluster on my infrastructure

The diego-release documentation should explain how to produce a deployment manifest for an HA standalone CF-MySQL cluster. Ideally, we can link to documentation in the cf-mysql release itself with a high-level explanation of the required deployment parameters (instance counts, job types).

BLOCKED on outcome of the charter #117854575 to explore Diego behavior when interacting with multiple masters.

Acceptance:
- I can follow documentation to provision an HA standalone CF-MySQL cluster on AWS and then successfully deploy Diego to use that as its datastore via the manifest-generation scripts.

L: bbs:relational

---

As a Diego developer, CI should be configured to run the DUSTs to upgrade the BBS from Diego 0.1434.0 targeting etcd to latest targeting MySQL

This DUSTs run should not yet block delivery of diego-release.

L: bbs:relational

---

As a Diego operator, I should be able to generate a Diego manifest that uses only a relational store

As a Diego operator who either has finished migrating an existing etcd-backed deployment to MySQL or is starting a new deployment with MySQL only, I should be able to produce a Diego deployment manifest that does not require the etcd-release and does not set etcd deployment properties. The manifest-generation script should also require minimal relational configuration to be provided.

Acceptance:
- I can follow documentation to generate a deployment manifest intended to support only a relational store, and then deploy Diego successfully without an etcd release present on the BOSH director. The manifest should also not contain any extraneous etcd-specific parameters, and should not configure the BBS to communicate with etcd at all.

L: bbs:relational

---

Placeholder: remaining relational-BBS topics (pipeline work, scale validation, validation of robustness to failure)

- Validate Diego system correctness in face of MySQL deployment failures
- warp-drive, etcd-to-MySQL DUST failures block delivery
- basic end-to-end perf validation?
- [RELEASE] Diego with relational BBS fully supported
- Diego CI runs a DUSTs suite with only MySQL
- ketchup migrates to relational BBS store, catsup obsolete


L: bbs:relational, placeholder

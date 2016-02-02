# Relational Datastore for BBS DB

## Overview

As the Diego team works both to make Diego perform at massive scale and to refine and extend features, it has been increasingly difficult to work within the constraints of the v2 etcd datastore. Althought its strong consistency and distributed nature have been attractive features, its lack of transactions and lack of support for multiple indices on sets of data have proved awkward in working with what is essentially a few flat tables of information (DesiredLRPs, ActualLRPs, Tasks, Domains). Moreover, to operate a deployment with hundreds of thousands of apps and their instances, Diego will need to manage gigabytes of data in its database, and current versions of etcd seem incapable of handling that in a performant way. Version 3 of etcd promises better support for cross-key transactions and larger datasets, but it is not yet available for use.

Consequently, we propose adding support for relational datastores to the BBS, and eventually making that the only supported datastore option. Having multiple indices per table will make it trivial to support the cell reps' bulk loops by allowing the BBS DB to retrieve and serve only those ActualLRPs with a particular cell's ID. It will also enable the efficient retrieval of only scheduling info for other system-wide bulk loops (nsync, route-emitter, LRP convergence) without the awkward split of DesiredLRP info across two sibling keys. Relational databases already offer strong consistency guarantees, and recent clustering strategies, such as those made available by the CF MySQL BOSH release, provide fast failover in the event of a master node failure.

To limit scope, we intend initially to support only MySQL and compatible databases as a relational datastore. With the existence of the [CF MySQL release](https://github.com/cloudfoundry/cf-mysql-release) and even hosted MySQL databases in some infrastructures, such as Amazon's RDS service, we expect MySQL will provide a robust, flexible database solution all the way from unit and integration tests to production deployments with requirements for HA.


## BBS changes

### Data model

We expect to be able to organize the existing principal data entities in Diego into relational tables easily. DesiredLRPs and Tasks already have primary keys, their process guid and task guid, respectively, and Domains are essentially their own primary key. ActualLRPs have a composite primary key, consisting of their process guid, index, and instance type (instance or evacuating). In many cases the system uses compare-and-swap operations on individual records to ensure correctness across components. We expect that these operations can be performed as desired within the consistency model of the relational database, possibly assisted by a version column for optimistic locking of records.

Once imported into the relational database structure, there may be additional efficiencies to be realized by restructuring some of the data in these top-level entities, such as extracting the routing data from DesiredLRPs into its own table.

The etcd datastore also records some metadata information, including information about the current and target versions of the data and the identifier of the key used to encrypt records. These can be stored in additional system or metadata tables in a relational datastore.


### Events

In addition to CRUD-style operations on DesiredLRPs, ActualLRPs, and Tasks, the BBS API also exposes an event stream of all creates, updates, and deletions of these entities. The etcd datastore has made this event stream fairly easy to implement through its ability to set watches on entire keyspaces. Since all reads and writes to the BBS DB now go through a single master node, that master node can instead construct and emit these events on successful writes of individual records. This may require additional read operations on the database, although those may already be required by the details of the compare-and-swap or compare-and-delete operations on some records.


### Records with TTLs

One other feature of etcd that Diego has used is the ability to set a TTL on a key, after which time it expires from the keystore. None of the interactions in the system rely on the expiration of these keys generating an event, though, so we can obtain equivalent functionality by recording an expiration time on the records. Expired records will not be returned in API queries, and the database-specific part of periodic convergence operations will eventually clean them up.


### Configuration

The BBS will require information about the host and the database to connect to, and a user and password for that database. We expect this information to be uniform enough across choices of relational datastore to provide it in a reasonably generic interface that will not be tied tightly to MySQL or any other database implementation.


### Migrations

One of the more challenging aspects of this change in datastore will be to migrate data for existing deployments from etcd to the relational store. Assuming that our experiments to validate the relational store succeed, we will then require a migration pathway from etcd to a relational store for existing HA deployments.

We propose the following strategy for these migrations:

- After relational-store support is officially enabled in version `0.X` of Diego, relational-store configuration on the BBS is optional. The presence of such configuration triggers the BBS on startup to migrate data from etcd to the configured relational store. On success, the version on the etcd store will be set to an extremely high value, so that any BBS instances that have only etcd configuration will give up and let newer BBSes take over.
- For Diego `1.X` versions, relational-store configuration is required, and etcd configuration is optional. For operators upgrading from any `0.X` release, this guarantees that their data in etcd will still be migrated to the relational store. The forced nature of this migration will also mean that from `1.0` onward, the development team need support only the relational datastore, and not etcd.
- At Diego `2.0`, etcd configuration will be removed entirely, and the relational store will be the only supported data store.


## Deployment strategies

We will have to support relational datastore deployments for three principal cases: running test suites as part of local development, supporting a development BOSH deployment of Diego (including a canned one for BOSH-Lite), and supporting an HA production BOSH deployment of Diego. 

For development on individual components and in the inigo test suite, tests can run their own local instances of a MySQL database. This may require some additional infrastructure to invoke these instances, as we have established already for dependencies such as etcd or consul, or to interact with a system-installed MySQL without the risk of test pollution. We will also have to install MySQL in the Docker images we use to run the Diego unit and integration tests in CI, and include installation instructions in our developer-setup documentation.

For BOSH deployments that do not require HA, such as ones for development, we can deploy a single instance of the CF MySQL release's `mysql` job, configured with a pre-seeded database for the Diego deployment. Credentials and connection details will then be provided to the BBS jobs. This deployment configuration should suffice for BOSH-Lite as well, and can likely take advantage of existing manifest-generation tooling for the CF MySQL release. Alternately, since it is a minimally simple deployment, we expect it would be straightforward to incorporate it into the Diego deployment manifest generation as we have already done with etcd.

For BOSH deployments that do require HA, such as production-grade deployments, we would recommend a fully HA standalone (non-brokered) deployment of the CF MySQL release, with mulitple database nodes and the switchboard proxy layer handling traffic to the current database master.

(TODO: How do we route traffic through several switchboard proxies correctly? The MySQL release docs mention setting up a load-balancer, but if we're not on AWS with an ELB, what LB should we use? Can we configure the BBS to do client-side load-balancing? Is this a case for the proxies to advertise the active master proxy through consul-based service discovery?)

In either of these cases, if there are database services available from the underlying infrastructure provider, such as Amazon's RDS, they could also be provided to the BBS as its backing relational datastore. We should be sure to make the BBS configuration generic enough to be compatible with this deployment option as well.


## Security

### Encryption (Data at rest)

With the etcd datastore, the individual Diego records are monolithic entities, and are wholly encrypted before being stored in the etcd key-value nodes. Within a relational datastore, the fields on the records would stored in separate columns, and we would be more selective in which ones to encrypt. Specifically, we would deliberately not encrypt any columns which we would incorporate into indices on tables, including of course primary keys, and likely any field with a simple scalar value. On the other hand, we would encrypt the actions, top-level environment variables, egress rules, and routes information on Diego Tasks and LRPs.


### Transport (Data in flight)

We intend to be able to communicate with any supported relational datastore over TLS, which MySQL supports. We will have to investigate how the CF MySQL release and other MySQL providers are configured, but it is likely that the BBS will have to support either a CA supplied in its VMs default trust store (as part of the globally trusted set of CAs, or deployed by BOSH), or one explicitly provided in its deployment manifest.

Once a secure connection to the relational datastore is established, the BBS will present the database credentials as its authorization for access to the Diego DB.

(TODO: Does the MySQL release allow the operator to rotate the DB access credentials declaratively without downtime for clients?)


## Validation of Scale and Correctness

### BBS Benchmark Tests

Our first measure of validation will be to run the BBS benchmark tests against a BBS backed by a relational datastore. This will help us validate the degree to which the BBS and its datastore can handle the bulk load we expect to put on it when managing 200,000 DesiredLRPs and their instances. If the system performs adequately under that load, we should extend the BBS benchmark suite to include the individual read (and write?) operations that are also part of the rep's bulk loop interactions, and scale up the load until it no longer performs adequately.


### Vizzini and other Acceptance Tests

The vizzini test suite will provide another set of assurances that the behavior of the system is correct, although we do not expect it to function correctly until the BBS's event-stream API is supported with the relational store. The CATs targeted at Diego will likewise supply greater confidence in the correctness of the system, although we generally expect them to be a less exacting test of the fast paths through the system. We may also wish to run some larger scale tests on a realistic deployment for additional validation.


### Failure Tolerance

In addition to basic correctness of the system as it contends over records in the data store, we intend to evaluate how the system performs as we cause nodes in the CF MySQL deployment to fail. These failures will include both the MySQL database nodes and the switchboard proxy nodes. We may wish to develop some test suites to run during these failure events to ensure that the system remains appropriately responsive and recovers as expected.

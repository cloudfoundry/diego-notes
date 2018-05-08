[CHORE] Remove legacyBBS from the Auctioneer

The Auctioneer is still referencing `legacyBBS` (i.e. `runtime-schema`).  It does this to fetch the set of Cells.  This should be an API on the BBS (it can reach out to consul to get the relevant data).

L: versioning, diego:ga

---

Remove the Receptor

Acceptance:
- The access VM should not have a Receptor!

L: versioning, diego:ga

---

All access to the BBS should go through one, master-elected, BBS server

We set this up to drive out the rest of the migration work.

All slave BBS servers should redirect to the master.

Acceptance: 
- I get a redirect when I talk to a slave.

L: versioning, diego:ga

---

The master-elected BBS server should never allow multiple simultaneous LRP/Task Convergence runs

Currently, the endpoints that trigger convergence will always spin up a convergence goroutine.  Instead, if a convergence goroutine is still running we should queue up at most *one* additional convergence request to run.  (i.e. if during a given convergence run *multiple convergence requests come in* we should only queue up *one* additional convergence request).

L: versioning, diego:ga

---

BBS server should emit metrics

- Convergence-related metrics (these were lost when we gutted the converger)
- BBS requests (both # of requests so we can compute request rates, and request latencies)
- All metrics emitted by the metrics server could, and should, just be emitted by the BBS during the convergence loop.

L: versioning, diego:ga

---

After a BOSH deploy, all data in the BBS should be stored in base64 encoded protobuf format

This drives out (see https://github.com/onsi/migration-proposal#the-bbs-migration-mechanism):

- introducing a database version
- teaching the BBS server to run the migration on start
- teaching the BBS server to bump the database version upon a succesful migration
- removing the command line arguments that control the data encoding

Acceptance:
- After completing a BOSH deploy I see Task data in the BBS stored in base64 encoded protobuf format
- I can (manually) fetch a database version key from etcd

L: versioning, diego:ga, migrations

---

[BUG] If the BBS is killed mid-migration, bringing the BBS back completes the migration

This drives out managing the database version in the case of a failed migration.

Acceptance:
- while true; do
    - I fill etcd with many JSON-encoded tasks
    - I perform a BOSH deploy
    - I kill the BBS mid-migration
    - I bring the BBS back
    - I see protobuf-encoded tasks, only

L: versioning, diego:ga, migrations

---

During a migration, the BBS should return 503 for all requests

- CC-Bridge components should forward the 503 error along where appropriate.
- Route-Emitter should maintain the routing table and continue emitting it

Acceptance:
- I set up a long-running migration (perhaps I have a lot of data)
- I see requests to the BBS return 503 during the migration

L: versioning, diego:ga, migrations

---

If a migration fails, I should be able to BOSH deploy the previously deployed release and recover.

If we keep the `/vN` root node then this becomes trivial.  We simply don't delete the `/vN-1` root node until after the migration.

L: versioning, diego:ga, migrations

---

If the Rep repeatedly fails to mark its ActualLRPs as EVACUATING it should fail to drain and the BOSH deploy should abort.

This is a safety valve.  It ensures we don't catastrophically lose all the applications if the BBS is unavailable.  We should be somewhat generous and retry our evacuation requests repeatedly and perhaps require that all the requests fail or 503.  The Cell should exit EVACUATING mode if this happens.

L: versioning, diego:ga, migrations

---

DesiredLRP data should be split across separate records

This should bump the DB version.  All LRP data should get migrated via a BOSH deploy.

We do not modify the API, but - instead - do whatever work is necessary under the hood to maintain the existing API.

Acceptance:
- After a BOSH deploy I can see that all LRP data has been split in two.

L: perf, versioning, diego:ga

---

As a BBS client, I can efficiently get frequently accessed data for all DesiredLRPs in a domain

Add a new API endpoint

L: perf, versioning, diego:ga

---

NSYNC's bulker should fetch the minimal set of DesiredLRP data

L: perf, versioning, diego:ga

---

Route-Emitter's bulk loop should fetch the minimal set of DesiredLRP data

L: perf, versioning, diego:ga

---


All Diego components should communicate securely via mutually-authenticated SSL

L: versioning, diego:ga

---

Cut Diego 0.9.0

We draw a line in the sand.  All subsequent work should ensure a clean upgrade path from 0.9.0.

We call this 0.9.0 not to signify that all our GA goals have been met (performance is still somewhat lacking).  However this gives us a reference point to be disciplined about maintaining backward compatibility during subsequent deploys.

L: versioning, diego:ga

---

[CHORE] Diego has an integration suite/environment that validates that upgrades from 0.9.0 to HEAD satisfy the minimal-downtime requirement

This ensures that subsequent work migrates cleanly.

L: versioning

---

Set up a test suite that runs all migrations from 0.9.0 up for 100k records.

Test suite should fail if migration time exceeds some threshold (2 minutes?)

This can help us determine whether or not we want to implement background-migrations for things like rotating encryption keys.

L: perf

---

The converger should run convergence, not the BBS.

To do this we'll want to:

1. Separate Task & Desired/Actual LRP convergence from the database-cleanup convergence.
2. Have the BBS perform database-cleanup periodically.
3. Have the Converger *fetch* data from the BBS, harmonize it, then update the BBS as relevant.

L: versioning, cleanup

---

I can rollback a migration to an earlier release

We can either run a bosh errand that triggers down-migrations to the specified version.  Or (better?) we can write a drain script on the BBS that triggers the down-migration -- is there any way to specify the target version this way?  If we go that direction how do we prevent other BBSes from rerunning up migrations.

L: versioning, needs-discussion

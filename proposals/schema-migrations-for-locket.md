## Problem Statement

We want to be able to evolve the DB schema for locket over time subject to the
following constraints in such a way:

- all deployed locket instances will be running (no leader election)
- operator may have a heterogeneous Diego deployment for some indeterminate
  period of time, with locket instances from at worst adjacent major releases
  (initially, Diego v1.x through Diego v2.y)
- during or after deployment of "next" Diego version, operator may need to roll
  Diego version back to previous deployed version (but no further back)

## Proposals

### Proposal 1

**Note** This proposal was copied from [this doc](https://docs.google.com/document/d/1bhBMx_il6tissr8C8Q1x3EFjVlg_zXjJo-GBXuEAi1w/edit?usp=sharing)

The proposed migration process for locket is similar to both BBS and Cloud Controller. Some type of migration management should be enabled in order to systematically change the database schema. We can use the same code as the BBS to run the migrations, and ensure that the migration is run only on the first instance of the locket service deployed using the BOSH property `spec.bootstrap`.

To ensure backwards-compatibility, upgrades, and rollback potential, we should ensure any database schema changes are additive-only changes. By only adding columns/tables, both old and new locket services will be able to continue talking to the database and working properly, allowing for a zero downtime upgrades. This also enables rollbacks, since an operator could just deploy an older version of Diego and the schema should be compatible. There would potentially be some data loss in the rollback scenario, since newer services would not be using deprecated columns that the older version is looking at. Another necessity for rollbacks is that the locket service does not fail if it sees that migrations it does not know about are applied to the database schema.

Any obsolete columns/tables can be removed after 2 major versions, when we no longer support upgrading from code that uses them. E.g. If a table is deprecated in 1.20, the table can be dropped in 3.0.

#### Questions
1. How do we pick which locket service instance runs the migration?
   - The first instance
   - Use BOSH spec.bootstrap property
   - ~~Use pre-start script Probably not necessary for locket right now~~
   - [CC example](https://github.com/cloudfoundry/cloud_controller_ng/blob/95ff7c25cbeefdef1d595edda64a180268174dd4/bosh/jobs/cloud_controller_ng/templates/pre-start.sh.erb#L111-L116)
2. How do we ensure that both new and old locket code works with updated schema?
   - Expand-then-contract policy for migrations
     - Ensure backwards-compatibility by doing additive changes and allowing time for old instances to be updated
     - Contraction (e.g. removal of old schema properties) can only occur after supported upgrade versions have been passed.
   - DUSTs. Is another variation of DUSTs worth the effort?
3. How do we rollback?
   - Given only additive changes, rolling back would be as simple deploying older versions and keeping the updated schema
     - Should Locket fail if there is a migration it doesn't know about? That would disallow rollback of the Diego release.
   - Actively rolling back database migrations is fraught with danger and would
     surely result in downtime. There is no BOSH feature for rollbacks and that
     means it would have to be a manual process as well.
   - How do we ensure that rollbacks work correctly?
     - CI? Worth the effort?
   - When are we able to remove old columns from schema?
     - Supporting rollbacks from previously deployed versions means adjacent major releases, same as upgrades.
     - Can't remove columns until 2 major releases?
       - Supporting upgrades from 1.x to 2.y, if 1.x uses column A and a
         migration within the 1.x->2.y timeframe removed it, the old locket
         instances would start to fail.

#### Ideas

- Can/Should we reuse BBS migration code?
- BBS migration code potentially reusable
- Has no ability to perform a scripted rollback of a migration
- Might want to extract into separate repo than BBS

#### Stories

- As a diego developer, I can change the schema of the locket service in a backwards compatible way
- As a diego developer, I am confident that the team is making additive-only changes (Worth the effort to automate CI?)
- As a diego PM, I would like to see database upgrade and downgrade in the CI (DUSTs? Is this even worth the effort?)


### Proposal 2

**tl;dr;** this proposal is similar to having different code branches (a.k.a feature flags) that execute depending on a version in the database. By always doing upgrades in two steps:

1. upgrading the code and the schema
2. switching the version in the database to cause all locket servers to switch atomically to running the new code

#### Terms

- `S1`, `S2` are schema versions before and after the schema upgrade
- `L1`, `L2` are two versions of locket server, where `L1` is the old version currently deployed and `L2` is the new version to be deployed
- `M2` is a schema migration to update the database schema from `S1` to `S2`

#### Modififications

- Add a versions table
- Add a schema version column to the versions table
- Add a data version column to the versions table (more about that later)
- Add a data version manifest property to the locket servers (more about that later)

#### Properties of a Migration

- A migration has two parts a schema migration and a data migration
- Migrations are always adding new columns/tables
- Since Schema migration is additive we don't have to make them downgradable
- Data migrations run in a transaction and should atomically do the following:
  1. Update the data in the tables, e.g. move data from column C1 to column C1' which has a different sql type or populate guids in a newly added column C2
  2. Set the data version to the match the schema version
- Data migrations should be downgradable. if that's not possible they should restore the previous columns to reasonable defaults
- Migrations to delete deprecated columns only run with major version updates to delete deprecated columns from v-2
- Schema migrations are idempotent, i.e. migration M2 can run on schema S2 without modifying it
- We can relax this assumption slightly for some migrations, e.g. increasing the size of a text column can happen in place without doing an `Expand-Then-Contract` migration

#### Migrations will happen in 2 steps

Initially we assume that we have a cluster of L1 locket servers, then:

- Deploy L2. Once an L2 starts up it will run all pending schema migrations, e.g. S2.
  - This will cause the schema version in the database to be `2`
  - **note** schema migrations are not downgradable so it's a noop if the locket server version is behind the current schema version
  - **note** the locket servers will continue running the same code as before

At this point we have a cluster of L2 servers following the code paths of L1, then:

- The following deploy will set the data version manifest property to `2`. This will cause the following:
  - The first server to deploy will check the difference between the current data version in the db and the configured value. There are three cases:
    1. There is no difference, then don't do anything
    2. Configured value is higher, then run all `UP` migrations from current data version to configured data version
    3. Configured value is lower, then run all `DOWN` migrations from current data version to configured data version
  - once the data version is updated all other L2 servers should start operating in the new V2 mode

#### Questions

1. Do we need to make migrations minimal, i.e. one DDL change per migration ?
2. How do we structure the code to reduce branching ? I think we can punt on that question until we have better understanding of how frequent we need to update the schema and how do the code changes look like. that way we can implement the right abstraction
3. How easy it is to write idempotent migrations ? This can be easily tested in unit tests, by running the migrations more than once to ensure idempotency, but it's unclear to me whether it will always be possible to do that, e.g. how do we add a column idempotently. The easy solution is always create a new table and rely on `CREATE TABLE...IF NOT EXIST`, but that could get ugly. Somehow related to the previous question. How often will we do these migrations ?
4. The workflow in this proposal could be an overkill sometimes. we can short-circuit it if necessary, e.g. if we are increasing the column size (although that raises the question of how we can do that idempotently). should we do that when appriopriate or keep ourselves honest by always following the same workflow ?
5. What should be included in the upgradability tests ? I can think of the following test:
   1. Start a V0 locket server (using the same terminology we use in DUSTS, `V0` is the oldest version we maintain)
   2. A client acquires a lock
   3. Upgrade to V1 locket server (again using DUSTS terminology, `V1` is `develop`) but keep the data version at `V0`
   4. Make sure the client didn't loose the lock
   5. Set the data version to `V1`. This will force a data migration
   6. Make sure the client didn't loose the lock

#### Stories

- Reimplement [#141394289](https://pivotaltracker.com/stories/show/141394289) using this strategy to get a feeling of how difficult it is

### Resources

- [CAPI Migration Guidelines](https://github.com/cloudfoundry/cloud_controller_ng/wiki/CAPI-Migration-Style-Guide)
- [Expand-Then-Contract](https://martinfowler.com/bliki/ParallelChange.html)

### Proposal 3

In the migration strategy the locket service is aware of two schemas viz: the `migrating from` schema and the `migrating to` schema.

#### Requirements

- A management table with the schema: `locket_id, schema_version, ttl`
- A schema version table with the schema `current, desired`

#### Performing a Migration

For the purposes of this proposal, we will assume that we have 3 locket servers (L1, L2, L3) and 2 schema versions (S1, S2).

- BOSH deploy begins updating L1.
  L1 updates the desired column in the schema version table to S2.
  L1 creates a new table with the desired schema S2.
  L1 updates its registration to S2.
  At this point in time L1 will continue respecting both the original table with S1 and the new table with S2, writing/reading locks and presences to both.
  This ensures that L1 respects locks that are acquired through L2 and L3.

- BOSH deploy updates L2
  L2 updates its registration to S2.
  At this point in time L2 will continue respecting both the original table with S1 and the new table with S2, writing/reading locks and presences to both.
  This ensures that L2 respects locks that are acquired through L1 and L3.

- BOSH deploy updates L3
  L3 updates its registration to S2.
  At this point in time L3 will continue respecting both the original table with S1 and the new table with S2, writing/reading locks and presences to both.
  This ensures that L3 respects locks that are acquired through L1 and L2.

- One of L1, L2, or L3 recognizes that all locket registrations have moved to S2.
  Update the current version in the schema version table to S2.
  Drop the S1 table.
  Old locket servers will no longer be able to start, as the current version is past their latest known version.

- At this point all locket servers are writing/reading from only S2 and the migration is complete.

#### Questions

- Can we use a post deploy trigger from BOSH to update the `current` version?
- Do we need all of this orchestration?
- This strategy does not solve migrating data into the new table (for example if we used this for the BBS). Could it? Proposal 2 is probably more useful in this case.

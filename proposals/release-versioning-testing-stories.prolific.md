As a Diego developer, I can deploy CF v220 + Diego 0.1434.0 + GL 0.307.0 ('V0') as a specific collection of coordinated piecewise deployments

This CF + Diego deployment should consist of 5 separate BOSH deployments:

- CF
- `D1`: 1 database VM
- `D2`: 1 brain, 1 cc-bridge, 1 access, 1 route-emitter
- `D3`: 1 cell
- `D4`: 1 cell

Acceptance:

- I can run a script to generate these manifests that I can easily configure to produce BOSH-Lite-specific manifests.
- After uploading the designated releases, I can use the manifests to deploy CF + Diego at these versions to a BOSH-Lite, even if other releases exist on its BOSH director.
- I can validate that the deployment is correct by running DATs against it successfully.


L: pipeline:upgrade-stable

---

As a Diego developer, I can upgrade my piecewise V0 deployment to a given CF/Diego/Garden/ETCD version combination later than V0 (V-prime)

For a given collection of CF and Diego versions, I can use scripts to generate a set of manifests designed to upgrade the piecewise V0 deployment to the new CF/Diego versions.


Acceptance:

- After checking out cf-release and diego-release at later versions, generating releases, and uploading them, I can generate these CF and Diego manifests and use them to upgrade my piecewise V0 deployment to V-prime.


L: pipeline:upgrade-stable

---

As a Diego developer, I have an 'upgrade-from-stable' test suite to create a piecewise V0 deployment against a BOSH director

This test suite should take the piecewise V0 deployment and deploy CF and Diego to the target director. We can assume that the correct V0 releases are already uploaded to the director.

Acceptance:

- After uploading the V0 releases to my BOSH-Lite, I can run the test suite with the appropriate configuration to deploy the piecewise V0 deployment.
- The suite should fail if the deployments already exist or if the BOSH commands themselves fail.


L: pipeline:upgrade-stable

---

As a Diego developer, I expect that the 'upgrade-from-stable' suite upgrades my V0 deployment to V-prime

The suite should assume that the V-prime releases are also already uploaded to the target BOSH director. The upgrade should proceed in the following order:

- Upgrade `D1` (databases)
- Upgrade `D3` (cell bank 1)
- Upgrade `D2` (brain and pals)
- Upgrade CF
- Upgrade `D4` (cell bank 2)

Acceptance:

- I can run this test suite against a BOSH-Lite after uploading the required V0 and V-prime releases and verify that it deploys a piecewise V0 deployment and then upgrades it to V-prime.


L: pipeline:upgrade-stable

---

As a Diego developer, I expect that the 'upgrade-from-stable' suite runs smoke tests against my piecewise deployment at each step between upgrades from V0 to V-prime 

We also need to change the deployment strategy to stop and start groups of cells intentionally to ensure that the apps in the smoke tests are running on cells of the intended version.

Deployment changes:

- After upgrading `D3`, stop `D4`.
- After upgrading `D2`, start `D4` and stop `D3`.
- Before upgrading `D4`, start `D3`.

Smoke test to run after each set of deployment changes:
- Push 2 instances of new test app (e.g., Dora) with SSH enabled
- Verify all instances run, are routable, have visible logs, and are accessible over SSH (run some trivial command)
- Destroy the instances and their routes

Acceptance: I can still run the test suite against a local BOSH-Lite and verify from the suite output and inspecting the system during the run that the smoke tests are running.


L: pipeline:upgrade-stable

---

As a Diego developer, I expect that the 'upgrade-from-stable' suite pushes an app to my piecewise deployment after V0 is deployed and checks its scalability at each step between upgrades from V0 to V-prime

After the suite deploys V0, it pushes one instance of a test app (e.g., Dora) and verifies it is running and routable. Call it A0.

After each step in the deployment upgrade, the suite scales the app up to 2 instances, verifies they are both running and routable, and then scales the app back down to 1 instance and verifies the second instance is gone.

Acceptance: I can still run the test suite against a local BOSH-Lite and verify from the suite output that the A0 app is created and exercised throughout the test run.


L: pipeline:upgrade-stable

---

As a Diego developer, I expect the 'upgrade-from-stable' suite to check the initial V0 app for continual routability during Diego deployment upgrades (but not CF)

During all of the BOSH operations on the Diego deployment, the test suite should continually verify that the A0 app is routable. The BOSH operations from previous stories are already structured so that the instance is always able to evacuate to an active cell.

This routing verification should not be done during the CF deployment, as parts of the routing tier (HAproxy and/or the gorouter) may not be available then.

Acceptance: I can still run the test suite against a local BOSH-Lite and verify from the test output (and by tailing logs on the A0 app) that these routing tests are performed.


L: pipeline:upgrade-stable

---

As a Diego developer, I expect to run the 'upgrade-from-stable' suite in CI against a new BOSH-Lite instance provisioned on AWS

In this case, V-prime is the collection of releases in the candidate build in the pipeline.

A job in the Diego CI should provision a new BOSH-Lite instance on AWS, upload the V0 and V-prime releases to it, then generate the V0 and V-prime piecewise manifests and run the 'upgrade-from-stable' test suite. Once it is done, we should destroy the BOSH-Lite instance.

We should run this job as part of the main Diego pipeline, in parallel with CATs and DATs after the ketchup deployment and smoke tests succeed. If it fails, we fail to promote the diego-release candidate through the pipeline


L: pipeline:upgrade-stable


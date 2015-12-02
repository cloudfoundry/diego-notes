Reproduce slow app startup times during evacuation

We've observed evacuation taking longer than expected on systems with sufficiently high app-instance densities (say, 50 app-instances per cell or higher). Here's the behavior we've seen from the logs:

- Previously evacuated cell comes back up while next cell is evacuating
- Auction places lots of containers simultaneously on the new cell
- Container-creation time increases to about 30 seconds
- The cell rep downloads droplets for the new app instances
- The cell rep starts the apps and health-checks them aggressively
- The health-check processes take 3-5 seconds to complete instead of the usual 50ms
- The app processes also take longer to come up, as the system is under load, and some of them fail to come up within the default 60s start timeout and are crashed
- Container deletion time also takes a long time (up to 30s)
- Eventually the cell stabilizes, but this can take as long as 10 to 15 minutes

Let's try to reproduce this phenomenon in an isolated environment so we can explore ways to mitigate it. Here's a starting suggestion about how to make this reproduction environment realistic:

- Deploy a Diego cluster to AWS with 4 cells on r3.xlarge VMs. This can likely be a one-AZ deployment, but then let's arrange the deployment manifest so that the cell job updates serially (not in parallel with the CC-Bridge/Access/Route-Emitter VMs, as is the default today).
- Deploy separate CF apps to fill up the cluster to a density of 50 instances per cell. The apps should have only 1 instance each, so that new cells incur a realistic number of droplet downloads (which are cached downloads). A large enough proportion of these apps should have large droplets that take a long time to start up, such as Java apps (for example, spring-music, or some Spring Boot examples).
- Deploy a change that causes an update to the cells and therefore evacuation. Measure how long it take cells to evacuate, and how long it takes individual instances on the new cells to start up and become healthy. 

We could also potentially explore this behavior on a 1-cell deployment by starting a lot of already-staged apps simultaneously, but the time required for SSH-key generation might interfere too much.

Onsi's [analyzer](https://github.com/onsi/analyzer) may be helpful for analyzing the aggregate timing behavior of these starts, but we'll likely need some assistance to get started with it. [Cicerone](https://github.com/onsi/cicerone) may also help with timeline analysis on a per-instance basis.


Acceptance: We can demonstrate reproduction of the described slow evacuation phenomenon.


L: charter, perf, net-check-action, start-placement

---

Evaluate the effect of reduced health-check frequency on evacuation time

Once we can reproduce the slow evacuation times in an isolated environment, let's evaluate how much of an effect increasing the time between health checks for unhealthy instances has. It's currently 500ms, so let's try 1s, 2s, and 5s instead with varying instance densities (50/cell, 100/cell, 150/cell?). Do any of those changes reduce the instance start-up times significantly?

This timing parameter can already be BOSH-configured on the rep, so this evaluation shouldn't require any code changes.


Acceptance: report on the differences in evacuation recovery time with different unhealthy check intervals and different instance densities, as well as differences in other relevant system metrics (such as system load, disk I/O, and CPU utilization).


L: charter, perf, net-check-action

---

Evaluate the effect of a spiked-out native net-check action on evacuation time

Once we can reproduce the slow evacuation times in an isolated environment, let's evaluate what effect a native net-check action has on improving them. For this spike, we need implement only the `Port` field on the proposed `NetCheckAction`, as we're not concerned about backwards-compatibility, HTTP checks, or timeout configurability. Nsync should then also use it in its LRP generation.


Acceptance: report on the differences in evacuation recovery time and system metrics when using this new action compared to the old action.

L: charter, perf, net-check-action


---

As a BBS client, I want to be able to specify a NetCheckAction on my LRP Monitor action and have the executor run a net-check directly instead of invoking a container-side process

As described in the 'Tuning Health Checks proposal', the BBS should support a new `NetCheckAction` action with the following fields:

- `Port`
- `Endpoint`
- `TimeoutInMilliseconds`
- `FallbackAction`

See the proposal for the details about types, optionality, and intent.

The BBS should also support new endpoints for tasks and LRP run-infos that are capable of returning this new action. If an client requests an LRP or Task from an old endpoint, the BBS should collapse all `NetCheckAction`s to their fallback actions.

The executor should be updated to understand this action and to perform a port-connectivity or HTTP endpoint check on the container with its own in-process client, instead of invoking a separate process inside the container to perform this check. We do not want this action to log on every attempt.


Acceptance:

- I can modify vizzini to use the `NetCheckAction` and it still passes against both old and new cells as long as the BBS in the deployment is updated to understand it.


L: net-check-action


---

Nsync should use the native NetCheckAction in the Monitor action for LRPs with a port-based health-check


Acceptance:

- I can use veritas to verify that newly desired CF apps are created using the `NetCheckAction` with an appropriate fallback RunAction for compatibility with old cells.

L: net-check-action

---

The executor's Monitor step should log on health transitions

The monitor step should log to the instance's log stream on the following events:

- Starting the monitor step: "Starting health monitoring of container"
- Failing the monitor step because the container does not become healthy within the N-second start timeout: "Container failed to become healthy within N seconds"
- Verifying the instance is healthy: "Container became healthy"
- Detecting the instance is now unhealthy after having previously been healthy: "Container became unhealthy"

The log source for these log lines should be the top-level LogSource on the executor container.


Acceptance:

- I can observe these log lines in the log stream for CF app instances when they:
	- become healthy within the timeout
	- become healthy and then unhealthy
	- never become healthy within the start timeout


L: logging, net-check-action


# Tuning Health Checks

From running larger and more public deployments of Diego, the Diego team has observed a few issues with the current approach to health-checking:

- When starting many containers simultaneously on the same cell, invoking the port-based health-check every half-second puts the system under undue stress, and can cause containers to fail to be verified as healthy before their start timeout. This situation occurs naturally from cell evacuation during a rolling deploy of a Diego cluster with a sufficiently high density of LRP instances.
- App developers have reported that the frequent health log lines are more noisy than helpful, and can prevent them from seeing application logs. This is especially true during the aggressive initial health-checking before instance health is verified, when the volume of logs can overwhelm the loggregator system's buffer of recent log messages.
- Garden-Windows currently has some special one-off argument-processing logic to configure the port-based health-check correctly, which the Greenhouse team has expressed interest in removing.

In light of these points, we propose some changes to how Diego handles both the timing and the method of health-checking instances:

- Cell performs native net-check action without logging
- Cell reduces frequency of initial health-check
- Monitor action logs only health transitions
- BBS client can configure health-check timing parameters on the DesiredLRP

We will explain these in more detail below.


## Native Net-Check Actions

We currently perform network-based health checks of services run in containers by invoking a separate process inside the container to connect to the service's port over TCP. While appealing in its simplicity, especially in the context of the plugin model we use for the platform-  and app-lifecycle-specific details of performing staging tasks and running app instances, this network task can be done far more efficiently by the executor component itself. This is particularly true with the strong support for these network operations in the Golang standard library.

The `healthcheck` executable currently operates in two different modes:

- checking establishment of a TCP connection to a particular port, on the first detected non-loopback IP address,
- making an HTTP GET request to a particular endpoint on this address and port and checking that the response is successful and has a `200 OK` status code.

A configurable timeout applies to both modes, defaulting to 1s.

Consequently, we propose introducing a new executor action, `NetCheckAction`, to take on this functionality. It will support the following fields:

| Field      | Required? | Type       | Description |
|------------|-----------|------------|-------------|
| `Port`     | required  | `uint32`   | Container-side port on which to check connectivity. |
| `Endpoint` | optional  | `string`   | If present, endpoint against which to check for a successful HTTP response. |
| `TimeoutInMilliseconds`     | optional  | `uint32`   | If present, different timeout in milliseconds to apply to the connection attempt or request. If absent, the executor instead uses a default duration for the timeout (1 second, perhaps configurable on the executor). |


### Backwards Compatibility

With the introduction of this new action, we also require cells from previous stable releases to be able to run a sufficiently equivalent action in its place. Unfortunately, the only actions at our disposal are the previous `Download`, `Run`, and `Upload` actions, as well as the various combining and wrapping actions. Two possible options for backwards compatibility are as follows:

- Backfill an action provided by the BBS that cannot fail, such as a `TryAction` wrapping a `RunAction` that runs `echo "backfill: no-op action"`.
- Backfill an arbitrary action provided by the client, which can be customized to provide equivalent functionality for the desired NetCheckAction behavior in terms of, say, a RunAction.

We prefer the second option, as it ensures that older cells can perform a real health-check on their app instances correctly. This option results in the following additional field on the NetCheckAction:

| Field      | Required? | Type       | Description |
|------------|-----------|------------|-------------|
| `FallbackAction`     | required  | `Action`   | Fallback action for older BBS clients that do not understand this new action type. |


In any case, the presence of this new action will require either new BBS endpoints for complete DesiredLRP and Task records or a new, optional parameter to be sent in requests to existing endpoints to indicate that the client is capable of understanding this new action. The old endpoints or responses should instead substitute this fallback action for each `NetCheckAction` in the action tree. We will of course have to make sure that the BBS correctly replaces all `NetCheckAction` instances in the tree with their fallbacks, in case a client perversely uses a `NetCheckAction` within the fallback action for another `NetCheckAction`.


#### Deprecation Plan for `FallbackAction` field

Obviously, this `Fallback` field is a wart on this action that we would like to freeze off as soon as possible. We expect the following transition plan to deprecate and eliminate this field:

- Diego `0.N`:
	- Release in which Diego cells understand `NetCheckAction` natively.
	- New BBS endpoints are introduced to handle the new action, and older BBS endpoints are deprecated.
- Diego `0.X` (`X` &geq;&nbsp; `N`) and `1.X`:
	- BBS validation requires the `Fallback` field, as it must be present for deployed cells from releases prior to `0.N`. Later cells ignore it and perform the native net-check.
- Diego `2.0` and later:
	- BBS validation does not require the `Fallback` field, and the BBS now ignores it, as no support is guaranteed for cells from a Diego `0.X` release.
	- Previously deprecated BBS endpoints are removed from the API.
	- Clients whose upgrades are coordinated with the BBS upgrades (say, the CC-Bridge) may safely drop the `Fallback` field, as they expect the BBS API to be upgraded to version 2 before they are, given Diego deployment order constraints.


### Justification of Effort

Before we make these changes, we should validate that they mitigate the problematic behavior we have observed during evacuation at scale. We should first characterize this evacuation behavior generally via logs and metrics from production systems where it is observed, then independently reproduce it in an isolated environment. Once we have this control baseline, we can spike out the `NetCheckAction` to validate that it has a significant mitigation benefit before we proceed with the work to implement it in a backward-compatible, test-driven way. 


### Implementation Notes

We expect the implementation of this action to be straightforward. The executor already has access to the external IP and the port mappings of the container, which is all that it requires to connect to a given container-side port from outside the container. We should certainly validate that this network information is correct on Windows cells as well, but we expect it to be, as app instances on Windows cells publish this information to the routing tiers to receive external requests.

The executor step corresponding to this action will also have to be appropriately cancelable.


## Reduced Frequency of Initial Health-Check

We have also received feedback that the 500ms duration between health checks while the instance is initially unhealthy may be too aggressive. We intend to increase this duration to a longer 2 seconds by default, both in the executor default configuration and in the property spec for the rep job in the Diego BOSH release.


## Logging Only Health Transitions

In the absence of any logging from the `NetCheckAction` itself, we intend for the executor's Monitor step to provide a minimal amount of logging, and only at the following health transitions:

- when it starts monitoring the instance,
- when the monitor step times out before the instance is considered healthy,
- when the instance becomes healthy,
- when a healthy instance becomes unhealthy by failing the monitor action.

We expect this should provide an adequate level of visibility into the health status of the instance without overwhelming the log stream emanating from the instance.

One issue that may arise is with the LogSource to be used for this Monitor step logging. We would prefer that the current `HEALTH` log source be used for these logs as well, but it is currently set only on the `RunAction` provided as the LRP's Monitor action. There is a container-wide LogSource field, but it sets the LogSource for any logs emitted by cell or container actions unless explicitly overriden on an action, so it is unsuited for customizing the LogSource only on logs coming from the Monitor step. We may wish to introduce an optional LogSource identifier on the LRP to be used for this Monitor action logging, or with the context included in the Monitor step logs, it may suffice to use the current container-wide LogSource (currently set to `CELL` for CF app instances).


## Configurable Health-Check Timing

We also realize that the default parameters of the Monitor step may not apply universally to all long-running workloads, especially if CF app developers are allowed to customize the health-check action to, say, run a custom script instead of the default port check. We will therefore expose some of these monitoring timing parameters on the DesiredLRP itself, such as:

- the duration to wait before initiating health-checking (currently `0`),
- the duration to wait between health checks before the instance is healthy (currently `500ms`, proposed to increase to `2s` by default),
- the duration to wait between health checks after the instance is healthy (currently `30s`). If `0`, the health check is never performed after the instance is considered healthy.

Each one of these new DesiredLRP properties would be optional, and if not specified, would default to the configuration of the rep (or its internal executor). Corresponding changes would also be required on the executor's Container model.

As additional motivation, since some of these parameters are configurable on the rep, the Diego BBS client may wish to specify these parameters explicitly on the DesiredLRP it submits, rather than relying on configuration that may vary with the Diego deployment.


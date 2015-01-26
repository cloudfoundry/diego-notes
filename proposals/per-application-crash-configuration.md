# Per-Application Crash Configuration

A common feature request in CF is to have finer-grained control over how applications should be restarted in the face of crashes.

Diego can easily support this.

## Crash Restart Policy

A CrashRestartPolicy in Diego can be expressed as:

```
{
    NumberOfImmediateRestarts: 3,
    MaxTimeBetweenRestarts: 960,
    MaxRestartAttempts: 200,
}
```

This particular set of numbers constitutes Diego's default policy.  Here's what they mean:

- An application is immediately restarted for the first `NumberOfImmediateRestarts = 3` crashes.
- After that we wait 30s, 60s, 120s, up-to `MaxTimeBetweenRestarts = 960` between subsequent restarts.
- After 200 restart attempts we give up on the application.

> Note: the consumer cannot modify the minimum time between restarts (30s) or the fact that we impose an exponential backoff.  Also the consumer cannot modify the 5-minute rule (i.e. we reset the crash count if you've been running for 5 minutes).

### Allowed Values:

#### `NumberOfImmediateRestarts`

- Allowed to be 0 (means never restart immediately)
- Must be less than `MaxRestartAttempts`
- Must be less than 10

#### `MaxTimeBetweenRestarts`

- Must be greater then 30s

#### `MaxRestartAttempts`

- Allowed to be 0 (means never stop restarting)

## Setting the Crash Policy

The crash policy is part of the `DesiredLRP`:

```
{
   ProcessGuid:...,
   ...
   CrashRestartPolicy: {
        ...
    },
    ...
}
```

- The crash policy can be updated after-the fact and, therefore, is part of `DesiredLRPUpdateRequest`.
- `CrashRestartPolicy` is optional.  If unspecified, the default will be used (see below).

## Setting the Default Crash Policy

The `DefaultCrashRestartPolicy` is stored in the BBS and can be modified via the Receptor API on a *per-domain-basis* (note: this is not a BOSH property - the `DefaultCrashRestartPolicy` can be modified at runtime!)

Only `DesiredLRP`s with *no* `CrashRestartPolicy` use the `DefaultCrashRestartPolicy`.  If the `DefaultCrashRestartPolicy` is not specified via the Receptor API, Diego will use the hard-coded values listed above.

## Required CF Work

We propose that only CF admins/operators will be allowed to set/modify crash policies.  So:

- As a CF Admin/Operator I can use the CC API to set the `DefaultCrashRestartPolicy`.
- As a CF Admin/Operator I can set up a CrashRestartPolicy on an Org/Space/App level.

> Notes: in addition to flowing an event through to NSYNC's listeneer, we'll probably want the NSYNC bulker to periodically (re)set the `DefaultCrashRestartPolicy` to make sure it's up-to-date.
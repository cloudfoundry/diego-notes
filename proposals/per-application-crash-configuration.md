# Per-Application Crash Configuration

A common feature request in CF is to have finer-grained control over how applications should be restarted in the face of crashes.

Diego can easily support this.

## Crash Policy

A crash policy in Diego can be expressed as:

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
   CrashPolicy: {

    },
    ...
}
```

- The crash policy can be updated after-the fact and, therefore, is part of `DesiredLRPUpdateRequest`.
- `CrashPolicy` is optional.  If unspecified, the default will be used.  This default is bosh-configurable and can be modified at runtime via a redeploy by the operator.



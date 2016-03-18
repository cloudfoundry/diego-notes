# Logging Guidence in Diego

## Formatting
Logger session keys should be hyphenated and message data keys should be underscored (because it is going to be JSON).
```
logger.Info("message-to-show",lager.Data{"key_to_show":data})
```

## Which level?

Logger does not have all of the normal logging levels. It only has:

* Fatal
* Error
* Info
* Debug

### Fatal
Fatal will cause a `panic` this is reserved for the highest level of errors.

### Error
Error will not `panic`, but something important failed.

### Info
Info sends logs to the stdout.log. Contains `starting` and `ending` messages and other normal behavior.

### Debug
Debug is for boring things like pings, heartbeats, subscriptions, polling, and notifications.

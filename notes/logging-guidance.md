# Logging Guidance in Diego

## Formatting

Logger session names and messages should be hyphenated. Keys in a message data payload should use snake-case (`key_name`) instead of hyphens (`key-name`) or camel-case (`keyName`).

```
logger = outer_logger.Session("handling-request")
logger.Info("message-to-show", lager.Data{"key_to_show": data})
```

## Sinks

Typically each Diego component has only one logging sink registered, which writes all log messages to stdout.


## Levels

The `lager` Logger type supports the following logging levels:

* Fatal
* Error
* Info
* Debug


### Fatal

Log a message at the fatal level to cause the logger to panic, writing a log line with the stack trace at the panic site. Reserve this level only for errors on which it is impossible for the program to continue operating normally.


### Error

Log a message at the error level to indicate something important failed.


### Info

Log a message at the info level to indicate some normal but significant event, such as beginning or ending an important section of logic.


### Debug

Log a message at the debug level to trace ordinary or frequent events, such as pings, heartbeats, subscriptions, polling, and notifications.

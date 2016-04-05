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

Log a message at the fatal level to cause the logger to panic, writing a log line with the stack trace at the panic site. Reserve this level only for errors on which it is impossible for the program to continue operating normally. These should be used in place of `panic(err)` and `os.Exit(1)`.

### Error

Log a message at the error level to indicate something important failed.


### Info

Log a message at the info level to indicate some normal but significant event, such as beginning or ending an important section of logic. Info logs are also usually appropriate to trace the boundaries of various APIs. 


### Debug

Log a message at the debug level to trace ordinary or frequent events, such as pings, metrics, heartbeats, subscriptions, polling, and notifications. If it happens on a timer in a component that is usually driven by a client, internal or otherwise, it should probably log at the Debug level. 

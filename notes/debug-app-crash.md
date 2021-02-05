# Understanding App Crashes in diego

## Overview
When an App (ActualLRP) crashes, Rep will notify BBS of the event using
`/v1/actual_lrps/crash` endpoint. In turn, BBS will generate an
[ActualLRPCrashedEvent](https://github.com/cloudfoundry/bbs/blob/master/doc/events.md#actuallrpcrashedevent) that is then consumed by [tps](https://github.com/cloudfoundry/tps) to record and notify the clients.

## Logs
- Start by looking for a `bbs-start-actual-lrp` log in Rep for the application
  in question. This will tell us when the instance was started.
  ```json
  {"timestamp":"2021-02-05T17:45:36.282971030Z","level":"info","source":"rep","message":"rep.executing-container-operation.ordinary-lrp-processor.process-running-container.bbs-start-actual-lrp","data":....}
  ```
- The next thing to look for would be `run-step.process-exit` log line for the
  application. This will tell us when the process exited the container.
  ```json
  {"timestamp":"2021-02-05T17:46:03.751426218Z","level":"info","source":"rep","message":"rep.executing-container-operation.ordinary-lrp-processor.process-reserved-container.run-container.containerstore-run.node-run.action.run-step.process-exit","data":{"cancelled":true,...}}
  ```
- If `log_level: debug` is set for Rep, we would even see when the rep called
  the crash endpoint
  ```json
  {"timestamp":"2021-02-05T20:06:18.870303528Z","level":"debug","source":"rep",
  "message":"rep.executing-container-operation.ordinary-lrp-processor.process-c
  ompleted-container.do-request.complete","data":{.... ,"request_path":"/v1/actual_lrps/crash","session":"61.1.1.1"}}
  ```
- Next, we would want to look at the BBS logs and look for `crash-actual-lrp`. This will let us know when the BBS received notification and when it generated the event.
  ```json
  {"timestamp":"2021-02-05T18:56:32.419684831Z","level":"info","source":"bbs","message":"bbs.request.crash-actual-lrp.db-crash-actual-lrp.starting","data":{"crash_reason":"APP/PROC/WEB: Exited with status 2",....}}
  {"timestamp":"2021-02-05T18:56:32.428710037Z","level":"info","source":"bbs","message":"bbs.request.crash-actual-lrp.db-crash-actual-lrp.complete","data":{"crash_reason":"APP/PROC/WEB: Exited with status 2",...}}
  ```
- tps-watcher is the next components in line for retrieving the
  [ActualLRPCrashedEvent](https://github.com/cloudfoundry/bbs/blob/master/doc/events.md#actuallrpcrashedevent) and notifying CAPI.
  ```json
  {"timestamp":"2021-02-05T20:22:49.665873433Z","level":"info","source":"tps-watcher","message":"tps-watcher.watcher.app-crashed","data":{}}
  {"timestamp":"2021-02-05T20:22:49.666082678Z","level":"info","source":"tps-watcher","message":"tps-watcher.watcher.recording-app-crashed","data":{}}
  ```

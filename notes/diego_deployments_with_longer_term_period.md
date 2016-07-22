# Exploring the effects of changing termination timeout for the Run Step in Executor

### Experiment 1

#### Design

1. we updated the timeout for terminate signal to 30 seconds
1. we pushed [grace](https://github.com/onsi/grace) with `-catchTerminate=true`

#### Observations

* When sending a SIGTERM to the app, the logs in executor say `signalling-terminate-success`, however the success does not mean that it successfully terminated the app but instead it implies that it sent the TERM signal successfully (misleading).

* After the 30 second period, we noticed that SIGKILL was sent to the application, and in the logs we saw `signalling-kill-success`

* During the termination period, we pushed another application to CF (`dora`), and we noticed that `dora` deployed successfully right before a KILL signal was sent to `grace`

* During the termination period, we scaled the `grace` up and down and started and stopped the app a number of times, looks like the app responds properly to the set of commands and the final state of it was the same as expected.


### Experiment 2

#### Design

1. same Design as Experiment 1, but we scaled grace up to 40 instances (the most our bosh-lite would allow) after the push
1. ran `cf restart grace`

#### Observations

* New instances of the `grace` app were added but the command initially failed due to `insufficient resources`.

 - every time it took about three to four
    minutes for all the instances to be in the up-and-running state.
 - We observed the following behavior consistently and across all our tries:
    - 22 instances were up and running initially,
    - the remaining instances took 3-4 minutes to start up, i.e., it took ~3 minutes after all the old instances were deleted for the new instances to be up and running.


### Experiment 3

#### Design

1. same as Experiment 2, except:
  * increased the timeout for SIGTERM to 60s
* scaled instances to 40
* ran `cf restart grace`

#### Observations

* redeploying diego on top of an existing one:
  - A new redeploy and a followup scale up results in a lot of app crashes
  - logs from the rep seem to suggest that a redeploy fails to create containers (at this point not clear whether it was because of how we enforced the container to operate or if there is any other reason)
  - Here is the error log from the container:
  ```
  {"timestamp":"1469141083.442837954","source":"rep","message":"rep.executing-cont
ainer-operation.ordinary-lrp-processor.process-reserved-container.run-container.
failed-creating-container","log_level":2,"data":{"container-guid":"b703676b-6c00
-4414-55bc-a36a1aa487d3","container-state":"reserved","error":"mounting file: mo
unting file: exit status 2","guid":"b703676b-6c00-4414-55bc-a36a1aa487d3","lrp-i
nstance-key":{"instance_guid":"b703676b-6c00-4414-55bc-a36a1aa487d3","cell_id":"
cell_z1-0"},"lrp-key":{"process_guid":"0a58732e-287d-40ff-8555-0a26afa415ce-b5d1
0ec5-bff4-4c65-bb0a-94b9e560dbca","index":26,"domain":"cf-apps"},"session":"2971
.1.1.3"}}```
  - also the following logs:
  ```
  {"timestamp":"1469142073.313192606","source":"rep","message":"rep.executing-cont
ainer-operation.get-container.failed-to-get-container","log_level":2,"data":{"co
ntainer-guid":"40b92210-d56e-4329-4f0b-8cea96559081","error":"container not foun
d","guid":"40b92210-d56e-4329-4f0b-8cea96559081","session":"3122.1"}}
{"timestamp":"1469142073.313233852","source":"rep","message":"rep.executing-cont
ainer-operation.failed-fetch-container","log_level":1,"data":{"container-guid":"
40b92210-d56e-4329-4f0b-8cea96559081","error":"container not found","session":"3
122"}} ```

  - The output for the state of app instances
```
state     since                    cpu    memory         disk          details
#0    running   2016-07-21 04:38:53 PM   0.0%   6.1M of 256M   19.6M of 1G
#1    crashed   2016-07-21 04:56:37 PM   0.0%   0 of 256M      0 of 1G
#2    crashed   2016-07-21 04:56:41 PM   0.0%   0 of 256M      0 of 1G
#3    crashed   2016-07-21 04:56:38 PM   0.0%   0 of 256M      0 of 1G
#4    running   2016-07-21 04:39:01 PM   0.1%   6.1M of 256M   19.6M of 1G
#5    running   2016-07-21 04:38:27 PM   0.0%   5.9M of 256M   19.6M of 1G
#6    crashed   2016-07-21 04:56:09 PM   0.0%   0 of 256M      0 of 1G
#7    running   2016-07-21 04:38:20 PM   0.0%   5.8M of 256M   19.6M of 1G
#8    crashed   2016-07-21 04:56:38 PM   0.0%   0 of 256M      0 of 1G
#9    crashed   2016-07-21 04:56:12 PM   0.0%   0 of 256M      0 of 1G
#10   running   2016-07-21 04:39:11 PM   0.1%   5.9M of 256M   19.6M of 1G
#11   running   2016-07-21 04:38:13 PM   0.2%   7.9M of 256M   19.6M of 1G
#12   crashed   2016-07-21 04:56:13 PM   0.0%   0 of 256M      0 of 1G
#13   running   2016-07-21 04:39:05 PM   0.2%   5.8M of 256M   19.6M of 1G
#14   crashed   2016-07-21 04:56:39 PM   0.0%   0 of 256M      0 of 1G
#15   running   2016-07-21 04:39:07 PM   0.1%   5.8M of 256M   19.6M of 1G
#16   running   2016-07-21 04:38:31 PM   0.1%   6.2M of 256M   19.6M of 1G
#17   running   2016-07-21 04:39:09 PM   0.2%   5.9M of 256M   19.6M of 1G
#18   crashed   2016-07-21 04:56:42 PM   0.0%   0 of 256M      0 of 1G
#19   crashed   2016-07-21 04:56:11 PM   0.0%   0 of 256M      0 of 1G
#20   crashed   2016-07-21 04:56:12 PM   0.0%   0 of 256M      0 of 1G
#21   crashed   2016-07-21 04:56:11 PM   0.0%   0 of 256M      0 of 1G
#22   crashed   2016-07-21 04:56:07 PM   0.0%   0 of 256M      0 of 1G
#23   crashed   2016-07-21 04:56:42 PM   0.0%   0 of 256M      0 of 1G
#24   running   2016-07-21 04:39:00 PM   0.2%   6M of 256M     19.6M of 1G
#25   crashed   2016-07-21 04:56:09 PM   0.0%   0 of 256M      0 of 1G
#26   crashed   2016-07-21 04:56:43 PM   0.0%   0 of 256M      0 of 1G
#27   crashed   2016-07-21 04:56:10 PM   0.0%   0 of 256M      0 of 1G
#28   crashed   2016-07-21 04:56:14 PM   0.0%   0 of 256M      0 of 1G
#29   running   2016-07-21 04:38:59 PM   0.0%   5.9M of 256M   19.6M of 1G
#30   crashed   2016-07-21 04:56:42 PM   0.0%   0 of 256M      0 of 1G
#31   crashed   2016-07-21 04:56:44 PM   0.0%   0 of 256M      0 of 1G
#32   running   2016-07-21 04:38:45 PM   0.0%   6M of 256M     19.6M of 1G
#33   crashed   2016-07-21 04:56:43 PM   0.0%   0 of 256M      0 of 1G
#34   crashed   2016-07-21 04:56:13 PM   0.0%   0 of 256M      0 of 1G
#35   crashed   2016-07-21 04:56:39 PM   0.0%   0 of 256M      0 of 1G
#36   crashed   2016-07-21 04:56:44 PM   0.0%   0 of 256M      0 of 1G
#37   crashed   2016-07-21 04:56:40 PM   0.0%   0 of 256M      0 of 1G
#38   crashed   2016-07-21 04:56:08 PM   0.0%   0 of 256M      0 of 1G
#39   crashed   2016-07-21 04:56:08 PM   0.0%   0 of 256M      0 of 1G
```

  - the output from the `rep/state` endpint
```
{
  "VolumeDrivers": [],
  "RootFSProviders": {
    "preloaded": {
      "type": "fixed_set",
      "set": {
        "cflinuxfs2": {}
      }
    },
    "docker": {
      "type": "arbitrary"
    }
  },
  "AvailableResources": {
    "Containers": 237,
    "DiskMB": 65940,
    "MemoryMB": 12719
  },
  "TotalResources": {
    "Containers": 250,
    "DiskMB": 79252,
    "MemoryMB": 16047
  },
  "LRPs": [
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 17,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    },
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 24,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    },
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 16,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    },
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 10,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    },
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 15,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    },
    {
      "VolumeDrivers": null,
      "RootFs": "",
      "DiskMB": 1024,
      "MemoryMB": 256,
      "domain": "cf-apps",
      "index": 32,
      "process_guid": "fa91b3d8-2457-48b0-877c-9b1b09598ad2-7f736278-5b25-4a53-8bb7-365affc692be"
    }
    ....
    ]
```

  - from what the rep reports, there are enough containers to serve the instances of the application. Also resource constraints are satisfied. However, for some strange reason garden cannot allocate containers, it seems.

* fresh diego deploy
  - the restart resulted in 22 instances of the app to be in `running state` whereas the other 18 instances were erroring due to `insufficient resources`
  - it took around 2 minutes for 4 of the remaining instances to change to `running`, i.e. 1 minute after the stopped instances were sent the kill signal
  - it took another minute for the remaining 16 instances to change to a `running` state
  - overall it took 3 minutes for the restart to take effect
  - bascially, the behavior was the same as what we observed with 30sec as timeout. so the increased timeout had no effect on the start time of the instances.

  - During the termination period, we pushed another application to CF (`dora`), and we noticed

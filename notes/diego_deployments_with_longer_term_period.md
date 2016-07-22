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

  - similar to the previous experiment
  - the restart resulted in 22 instances of the app to be in `running state` whereas the other 18 instances were erroring due to `insufficient resources`
  - it took around 2 minutes for 4 of the remaining instances to change to `running`, i.e. 1 minute after the stopped instances were sent the kill signal
  - it took another minute for the remaining 16 instances to change to a `running` state
  - overall it took 3 minutes for the restart to take effect
  - bascially, the behavior was the same as what we observed with 30sec as timeout. so the increased timeout had no effect on the start time of the instances.

  - During the termination period, we pushed another application to CF (`dora`), and we noticed

### Experiment 4

#### Design

1. Deploy diego with termination timeout period of one minute.
1. Create org `o` with `default` quota:
```
name      total memory   instance memory   routes   service instances   paid plans   app instances   route ports
default   10G            unlimited         1000     100                 allowed      unlimited       unlimited
```
1. Create spaces `s` and `s2` under organization `o`, both with `whole` space-quota:
```
name    total memory   instance memory   routes   service instances   paid plans   app instances
whole   10G            unlimited         1000     100                 allowed      unlimited
```
1. Target `s` and push `grace`, scale `grace` to 40 instances.
1. Restart `grace`.
1. Target `s2` and push `grace2`, scale `grace2` to 40 instances (within the
   termination timeout period of space `s`'s restart of `grace`).

#### Observations
1. Contention of resources prevents us from starting `grace2`. The restarting of
   `grace` apps in space `s` is not successful until after three minutes.
   Repeatedly trying to start `grace2` on space `s2` (during the termination time of `grace` on `s`) fails due to hitting organization memory limit.

### Experiment 5

#### Design

1. Deploy diego with termination timeout period of one minute.
1. Create org `o` with `default` quota:
```
name      total memory   instance memory   routes   service instances   paid plans   app instances   route ports
default   10G            unlimited         1000     100                 allowed      unlimited       unlimited
```
1. Create spaces `s` and `s2` under organization `o`, both with `half` space-quota:
```
name    total memory   instance memory   routes   service instances   paid plans   app instances
half    5G             unlimited         1000     100                 allowed      unlimited
```
1. Target `s` and push `grace`, scale `grace` to 20 instances.
1. Restart `grace`.
1. Target `s2` and push `grace2`, scale `grace2` to 20 instances (all within
   the termination timeout period of space `s`'s restart of `grace2`). i.e.
`cf restart grace; cf target -s s2; cf push grace2; cf scale -i 20 grace2`

#### Observations
1. Contention of resources prevents us from starting all instances of `grace2`
   immediately. It takes bout three minutes since the restart of `grace` until all
   instances of `grace2` are up and running. These instances that take long to
   start up are in the `down` state and print out the same `insufficient
   resources` error before they start up.
About one and a half minutes after the `cf restart grace` call
```
      state      since                    cpu    memory          disk          details               
#0    running    2016-07-22 07:50:56 AM   0.0%   4.4M of 256M    19.6M of 1G                         
#1    running    2016-07-22 07:51:03 AM   0.0%   20.2M of 256M   19.6M of 1G                         
#2    running    2016-07-22 07:51:15 AM   0.5%   4.8M of 256M    19.6M of 1G                         
#3    running    2016-07-22 07:51:09 AM   0.0%   9.1M of 256M    19.6M of 1G                         
#4    starting   2016-07-22 07:51:00 AM   0.0%   940K of 256M    1.3M of 1G                          
#5    running    2016-07-22 07:51:11 AM   0.1%   6.9M of 256M    19.6M of 1G                         
#6    running    2016-07-22 07:51:26 AM   0.0%   1008K of 256M   19.6M of 1G                         
#7    starting   2016-07-22 07:51:00 AM   0.0%   876K of 256M    1.3M of 1G                          
#8    starting   2016-07-22 07:51:00 AM   0.0%   868K of 256M    1.3M of 1G                          
#9    running    2016-07-22 07:51:28 AM   0.0%   1M of 256M      1.3M of 1G                          
#10   running    2016-07-22 07:51:22 AM   0.6%   4.6M of 256M    19.6M of 1G                         
#11   running    2016-07-22 07:51:23 AM   0.0%   4.4M of 256M    19.6M of 1G                         
#12   running    2016-07-22 07:51:15 AM   0.0%   7M of 256M      19.6M of 1G                         
#13   running    2016-07-22 07:51:27 AM   0.0%   864K of 256M    19.6M of 1G                         
#14   starting   2016-07-22 07:51:00 AM   0.0%   0 of 256M       0 of 1G                             
#15   starting   2016-07-22 07:51:00 AM   0.0%   0 of 256M       0 of 1G                             
#16   running    2016-07-22 07:51:22 AM   0.0%   4.1M of 256M    19.6M of 1G                         
#17   starting   2016-07-22 07:51:00 AM   0.0%   1M of 256M      1.3M of 1G                          
#18   down       2016-07-22 07:51:00 AM   0.0%   0 of 256M       0 of 1G       insufficient resources  
#19   down       2016-07-22 07:51:00 AM   0.0%   0 of 256M       0 of 1G       insufficient resources
```

It seems we cannot end-user exhaust the resources of the system this way, but we can make the degrade system performance with the longer terminate time.
Repeated runs of this process produced the same result.

### Experiment 6

#### Design

Same as Experiment 5, but with a higher timeout (3 minutes).

#### Observations

Same result as experiment 5. We see this as the initial status of `grace2` apps, but eventually the system corrects itself.
```
      state      since                    cpu    memory         disk          details                
#0    running    2016-07-22 08:19:39 AM   0.0%   4.8M of 256M   19.6M of 1G                          
#1    running    2016-07-22 08:19:55 AM   0.0%   4.7M of 256M   19.6M of 1G                          
#2    starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#3    starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#4    running    2016-07-22 08:19:51 AM   0.0%   4.6M of 256M   19.6M of 1G                          
#5    starting   2016-07-22 08:19:46 AM   0.0%   916K of 256M   1.3M of 1G                           
#6    running    2016-07-22 08:19:51 AM   0.0%   4.8M of 256M   19.6M of 1G                          
#7    running    2016-07-22 08:19:54 AM   0.0%   4.5M of 256M   19.6M of 1G                          
#8    starting   2016-07-22 08:19:46 AM   0.0%   876K of 256M   1.3M of 1G                           
#9    running    2016-07-22 08:19:54 AM   0.0%   4.9M of 256M   19.6M of 1G                          
#10   running    2016-07-22 08:19:58 AM   0.0%   4.3M of 256M   19.6M of 1G                          
#11   starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#12   running    2016-07-22 08:19:53 AM   0.0%   4.9M of 256M   19.6M of 1G                          
#13   starting   2016-07-22 08:19:46 AM   0.0%   956K of 256M   1.3M of 1G                           
#14   starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#15   running    2016-07-22 08:19:56 AM   0.0%   4.7M of 256M   19.6M of 1G                          
#16   starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#17   starting   2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G                              
#18   down       2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G       insufficient resources 
#19   down       2016-07-22 08:19:46 AM   0.0%   0 of 256M      0 of 1G       insufficient resources
```

HOWEVER: I then however was able to break the system:

Instead of running this in one terminal window:
`cf restart grace; cf target -s s2; cf push grace2; cf scale -i 20 grace2`

I ran this: 
`cf restart grace`
Switch over to a new terminal window, then ran this:
`cf target -s s2; cf push grace2; cf scale -i 20 grace2`

I pushed `grace2`, scaled it, but was unable to get them to start.
Then when I tried to restart `grace` again on `s`, it failed due to `Insufficient Resources`.

Output from `cf start grace2`
```
FAILED                                                                                         
InsufficientResources                                                                          
                                                                                               
TIP: use 'cf logs grace2 --recent' for more information                                        
Scaling app grace2 in org o / space s2 as admin...                                             
OK
```

Output from `cf logs grace2 --recent`
```
2016-07-22T08:41:48.90-0700 [API/0]      OUT Created app with guid 0345910f-f005-4ac4-8bb6-868333eeef91                                                                                       
│2016-07-22T08:41:49.28-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"route"=>"54ce1237-6a7a-4dbf-b140-8d6e44e6c1e9", :verb=>"add", :relation=>:routes, :related_guid=>"54ce1237-6a7a-4dbf-b140-8d6e44e6c1e9"})                                     
│2016-07-22T08:41:55.06-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"state"=>"STARTED"})                                                                
│2016-07-22T08:41:55.23-0700 [API/0]      ERR Failed to stage application: insufficient resources                                                                                              
│2016-07-22T08:42:00.63-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"instances"=>20})                                                                   
│2016-07-22T08:42:31.67-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"name"=>"grace2"})                                                                  
│2016-07-22T08:42:37.43-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"state"=>"STARTED"})                                                                
│2016-07-22T08:42:37.50-0700 [API/0]      ERR Failed to stage application: insufficient resources                                                                                             
│2016-07-22T08:42:42.91-0700 [API/0]      OUT Updated app with guid 0345910f-f005-4ac4-8bb6-868333eeef91 ({"instances"=>20})
```

The instances of `grace` take about three minutes to be all up and running.

This behavior was erratic however. I was able to successfully push and scale
`grace2` two out of six times. The two times that I was able to push it and
scale it, the same behavior occurred where it took three minutes for all
`grace2` instances to be running.

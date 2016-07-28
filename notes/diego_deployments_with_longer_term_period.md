# Exploring the effects of changing termination timeout for the Run Step in Executor

## Summary

In order to gauge the impact on the system of longer grace times for shutting down containers, we conducted the following experiments. We experimented with increasing the timeout between when the executor sends the TERM signal and the KILL signal, to analyze the impacts on the overall behavior of diego. Descriptions of the individual experiments follow, but the main take-away is that it would give an app developer way too much control if we let them specify the timeout. With the current way diego is written, a malicious user could exhaust resources on a cell and cause apps in other orgs to not be able to start for an arbitrary amount of time.

We didn't run all of these experiments with a variety of termination timeouts because the conclusion seemed to be the same, just exacerbated by how big the timeout is.

The main problem results from the fact that the rep assumes resources become available as soon as the TERM signal is sent to a container. However, the resources are only available after the process receives a KILL signal. A longer time between TERM and KILL means that the rep will be incorrect for a longer period of time, and this can significantly affect auctions during this period.

A potential fix could be to modify how the rep handles container deletion, so that it only updates its knowledge of available resources after the KILL has been sent.

### Setup

* The experiments are done on bosh-lite
* For each experiment the number of instances are set in such a way that the
  deployment would max out the available space-quota allocated to the app.

### Experiment 1

#### Design

1. we updated the timeout for terminate signal to 30 seconds
1. we pushed [grace](https://github.com/onsi/grace) with `-catchTerminate=true`

#### Observations

* When sending a SIGTERM to the app, the logs in executor say `signalling-terminate-success`, however the success does not mean that it successfully terminated the app but instead it implies that it sent the TERM signal successfully (misleading).

* After the 30 second period, we noticed that SIGKILL was sent to the application, and in the logs we saw `signalling-kill-success`

* During the termination period, we pushed another application to CF (`dora`/, and we noticed that `dora` deployed successfully right before a KILL signal was sent to `grace`

* During the termination period, we scaled `grace` up and down and started and stopped the app a number of times, looks like the app responds properly to the set of commands and the final state of it was the same as expected.


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
  - basically, the behavior was the same as what we observed with 30sec timeout. So, the increased timeout had no effect on the start time of the instances.

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
1. Immediately target `s2` and push `grace2`, scale `grace2` to 40 instances (within the
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
1. Immediately target `s2` and push `grace2`, scale `grace2` to 20 instances (all within
   the termination timeout period of space `s`'s restart of `grace2`). i.e.
`cf restart grace; cf target -s s2; cf push grace2; cf scale -i 20 grace2`

#### Observations
1. Contention of resources prevents us from starting all instances of `grace2`
   immediately. It takes about three minutes since the restart of `grace` until all
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

Sequential execution of commands did not result in exhausting resources, but we see performance degradation with the longer terminate time.
Repeated runs of this process produced the same results.

### Experiment 6

#### Design

Same as Experiment 5, but with parallel app deployments

Instead of running this in one terminal window:
`cf restart grace; cf target -s s2; cf push grace2; cf scale -i 20 grace2`

We ran this:
`cf restart grace`
Switched over to a new terminal window, then ran this before the previous restart completed:
`cf target -s s2; cf push grace2; cf scale -i 20 grace2`

#### Observations

We pushed `grace2`, scaled it, but were unable to get app instances to start.
Restarting `grace` again on `s` also failed due to `Insufficient Resources`.

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

This behavior was erratic however. We were able to successfully push and scale
`grace2` two out of six times. The two times that We were able to push it and
scale it, the same behavior occurred where it took three minutes for all
`grace2` instances to be running.

Our hypothesis is that we we restart the app in a situation where they have to compete
on existing resources, whichever of the two apps manages to secure the required resources
first, takes precedence and leaves the other app in a failed state with insufficient resources.

Essentially the conclusion here is that longer termination timeout for Run Step would result in
apps from different spaces to cut into each others resources.

### Experiment 7

#### Design

1. Deploy diego with termination timeout period of one minute.
1. Create orgs `o` and `o2`, each with a `half-org` quota that matches half the available resources in bosh-lite:
```
name       total memory   instance memory   routes   service instances   paid plans   app instances   route ports
half-org   7.8G           unlimited         1000     100                 allowed      unlimited       unlimited
```

1. Create spaces `s` in `o` and `s2` in `o2`, both with `whole` space-quota, taking the entire space for the corresponding org:
```
name    total memory   instance memory   routes   service instances   paid plans   app instances
whole   7.8G           unlimited         1000     100                 allowed      unlimited
```
1. Target `s` and push `grace`, scale `grace` to 31 instances (31 is the max number of instances allowed in 7.8G if we use 256M per instance).
1. Target `s2` and push `grace2`, scale `grace2` to 31 instances
1. Wait for all app instances to be in the running state in both orgs.
1. Restart `grace`, then immediately also restart `grace2`.
1. Target `s2` and push `grace2`, scale `grace2` to 31 instances

#### Observations

1. Both apps in both spaces took about 1 minutes (before the termination time) to start again
1. `grace2` started immediately after 1 minutes and all instances changed to `running` statge
1. for `grace` only 6 instances started after the 1 minute termination time, it took about another minute for the remaining instances to start


### Experiment 8

#### Design

Similar to Experiment 7, except:

1. Target `s` and push `grace`, scale `grace` to 31 instances (31 is the max number of instances allowed in 7.8G if we use 256M per instance).
1. Target `s2` and push `grace2`, scale `grace2` to 31 instances, with `--no-start` flag
1. Wait for all `grace` app instances to be in the running state.
1. Restart `grace`
1. Target `s2` and start `grace2`

#### Observations

1. We noticed that restarting `grace` cuts in to the resources of `grace2` in such a way that, starting `grace2` immediately
after restarting `grace` would result in some instances of `grace2` to not start due to `insufficient resources` issue.
1. This is essentially problematic in that if we choose to make the termination timeout user-configurable, any malicious user can
cause significant system malfunction by simply setting the termination timeout to something really high and then consistenly
restarting an app. This behavior can cause issues even with only one instance of an app, so setting app instance limits would not help.


### Experiment 9

#### Design

1. Have a cf + diego deployment with a `cell_z1/0` vm deployed
1. Create an organization `o` and a space `s`
1. Target `s` and push `grace`, scale `grace` to 30 instances.
1. Wait for all `grace` app instances to be in the running state.
1. Modify the bosh deployment so that it removes `cell_z1/0` and adds `cell_z2/0` to cause an evacuation behavior in diego

#### Observations

1. rep did the evacuation properly and all the 30 instances got moved from `cell_z1/0` to `cell_z2/0` almost immediately
1. bosh continued to tear down the `cell_z1/0` vm
1. bosh failed when destroying the `cell_z1/0` vm due to reaching the one minute timeout which is in turn caused by the termination timeout being greater than the bosh timeout.
1. the bosh agent then becomes unresponsive on that vm and bosh fails to delete the vm
1. the bosh agent the recovers from the unresponsive state, which in turn results in having an extra cell be in the running state.
1. this supposedly deleted vm then becomes available as yet another cell in diego, and diego deploys app instances to it, if requests come in.


### Experiment 10

#### Design

1. Change the "rep.evacuation_timeout_in_seconds" property in the manifest to be 60, so the this experiment runs faster.
1. Have a cf + diego deployment with a `cell_z1/0` vm deployed
1. Create an organization `o` and a space `s`
1. Target `s` and push `grace`, scale `grace` to 5 instances.
1. Wait for all `grace` app instances to be in the running state.
1. Modify the bosh deployment to add a second instance to `cell_z1`, so now there is a `cell_z1/0` and a `cell_z1/1`.
1. Add a blank line to the rep template file, to cause the rep to evacuate and restart on the next bosh deploy.
1. Create, upload, and deploy the new diego release.

#### Observations

1. Evacuation went smoothly and the evacuating cell took 1 minute to drain (i.e., the evac timeout).
1. The containers on the evacuating instance get deleted when the rep is restarted (stale containers are reaped on rep start).
1. This means the instances only got 1 minute to clean up after a term (because 1 minute was our modified evacuation timeout), not the full 20 minutes they expected to have. This might be worth thinking about more, but at first glance this doesn't seem unreasonable, provided there is a high enough evacuation timeout in general. We don't want to hold up deployments for an indefinite amount of time because someone chose an extremely large value for the term->kill timeout.

# Diego and Consul Failure Modes

## Presence (Cell)

### Experiment 1
In this experiment, we performed a rolling deploy of a 3 node
consul cluster.  We took the consul nodes down one-by-one and noticed that once
we are in the middle of doing leader election, reps stop functioning.

We also noted that leader election may occasionally take longer than usual in
which case the system took even longer to get back to normal.A

## Stories to Create

1. If rep cannot set Presence do not indicate it is ready [#118500389](https://www.pivotaltracker.com/story/show/118500389)
1. It is possible to set duplicate presence similar to BBS locket error seen in
   [#117967437](https://www.pivotaltracker.com/story/show/117967437), [#118957221](https://www.pivotaltracker.com/story/show/118957221)


## Locks (route_emitter, bbs, auctioneer, tps-watcher, nsync-bulker, converger)

### Experiment 1
In this experiment, we performed a rolling deploy of a 3 node consul cluster.
We took the consul nodes down one-by-one and noticed that once we are in the
middle of doing leader election, the component currently holding the lock lost
the lock.  The components then attempt to reacquire the lock and during the
leader election they are not able to but will retry correctly and eventually
once consul has a new leader the lock can be acquired by one of the instances.

Impact is that the leaders may change and components will be down until the
leader election completes.  

## Stories to Create

1. Losing the lock during migrations of the BBS does not cause the migration to
   exit immediately [#118957301](https://www.pivotaltracker.com/story/show/118957301)

## Service registration

### Experiment 1
In this experiment, we performed a rolling deploy of a 3 node
consul cluster.  We took the consul nodes down one-by-one and noticed that once
we are in the middle of doing leader election the service registrations seem to
persist but are not available during the leader election.

Once a new leader is elected we can see the service registrations again.  

## Stories to Create

1. Service registration should reregister if consul drops the service
   registration [#118957335](https://www.pivotaltracker.com/story/show/118957335)

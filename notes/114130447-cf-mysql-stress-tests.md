We want to know how SQL behaves in golang when using the CF-MySQL HA release.
We're also interested in knowing what behavior the current proxy (switchboard)
presents in face of a cluster outage.

We devised scenarios to observe those behaviors. Those are documented
[here](https://github.com/luan/cf-mysql-proxy-stress-test).

## Grand Conclusion

The go sql library does the right thing: it fails existing connections when the
pipe is broken, and recreates them next time you try to use it. Switchboard
seems to react pretty quickly to a server going down, allowing the go sql
library to do its job.  No major conclusions as far as next steps, which is
good, looks like we don't have to do much or don't have a lot to complain about
to MySQL just now.

## Scenario 1

success on all cases

## Scenario 2
stop non leader: success

stop leader: flakes when you restart because the connection gets dropped, moves
on to succeeding since switchboard starts routing them to the new leader.

## Scenario 3
stop non leader: success

stop leader: 2 records lost

```
Kill the leader now and press ENTER2016/03/15 22:32:39 failed to commit Transaction: Error 1047: WSREP has not yet prepared node for application use
[mysql] 2016/03/15 22:32:39 packets.go:33: unexpected EOF
[mysql] 2016/03/15 22:32:39 packets.go:33: unexpected EOF
[mysql] 2016/03/15 22:32:39 packets.go:124: write tcp 10.10.22.11:37713->10.10.22.11:3306: write: broken pipe
[mysql] 2016/03/15 22:32:41 packets.go:33: unexpected EOF
2016/03/15 22:32:41 failed to begin transaction: driver: bad connection
[mysql] 2016/03/15 22:32:41 packets.go:33: unexpected EOF

2016/03/15 22:32:50 Expected
    <int>: 2799
    to equal
        <int>: 2801
```

## Scenario 4

stop non leader: success
restart non leader: success

stop leader: flakes when you restart because the connection gets dropped, moves
on to succeeding since switchboard starts routing them to the new leader.

## Scenario 5

stop non leader: lost 8 records (with 10 maxConnections) lost 98 records (with 100 maxConnections)

stop leader: lost ~1000 records
restart leader: Causes the VM to be stuck in monit hell. We cannot stop/start even after reloading the monit deamon. Possible bug in ctl script?

Often it drops n-1 connections and thus has that many fewer records. ex 10, 20 100
At 500, only 495 failed. it's great! When the db dies, the the error

```
2016/03/15 18:26:12 failed write: Error 1213: Deadlock found when trying to get lock; try restarting transaction
```

is given.  It doesn't matter which db is killed, the result is the same. Note
that this is consistent with the results from scenario 3, because =1, thus
n-1=0=number of errors that occured.

At 1000 connections, there is the following error:

```
[mysql] 2016/03/15 18:28:36 statement.go:27: invalid connection
[mysql] 2016/03/15 18:28:36 packets.go:33: unexpected EOF
```

## Scenario 6

same as above


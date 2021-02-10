# DISCLAIMER
**DO NOT USE ANY OF THIS IN A PRODUCTION ENVIRONMENT**

This repository contents are only intended for learning purposes.

# Contents
* **deployment** folder include files for a redis sentinel topology using kubernetes deployments
* **statefulSet** folder include files for a redis sentinel topology using kuberntetes statefulSets

# Links and documentation
* [Redis sentinel](https://redis.io/topics/sentinel)
* [Redis replication](https://redis.io/topics/replication)
 
# statefulSet
```
kubectl craete -f redisconf.yml
kubectl create -f redis-primary-replica.yml
kubectl create -f redis-sentinel.yaml
```

**redisconf.yml** contains a kubernetes config map holding the redis.conf file for each type of redis member: primary, replica or sentinel.

**redis-primary-replica** The first pod created, named redis-0, will be selected as the primary. Subsequenly created pods (redis-1, redis-2 ... ) will be created as replicas of the first one.

This is achieved using an init container. It executed a bash script which checks the ending number of the statefulSet pod name.

Using an initcontainer the appropiate redis.conf file is copied into the container. Using the `cp -n` option, the file is only copied the first time the pod is created. This is important, because redis regulary writes its status to the file, so it is written into a persistent volume. So it is not lost between a pod recreation. StatefulSets in kuberntes have their own PersistentVolumen which is reused between pods recreation.

## Playing around with Redis Sentinel

### Can I do a quorum?
Sentinels are configured to need a quorum of 2 members before taking any action. So with 2 or more members everything is OK.

```
127.0.0.1:26379> sentinel ckquorum myRedisExample
OK 3 usable Sentinels. Quorum and failover authorization can be reached
```
Now we scale down to only one sentinel.
```
kubectl scale statefulset redis-sentinel --replicas=1
```
Sentinels will not be able to reach a quorum
```
127.0.0.1:26379> sentinel ckquorum myRedisExample
(error) NOQUORUM 1 usable Sentinels. Not enough available Sentinels to reach the majority and authorize a failover
```
If a new sentinel is added we can see it in the other sentinels log
```
+sentinel-address-switch master myRedisExample 10.244.0.18 6379 ip 10.244.0.34 port 26379 for 5d29eed6ef812aca3dd86641a348f077d96da693
```

### How many replicas do I have?

redis-0 is set to be the primary. 
```
kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
redis-0            1/1     Running   0          15m
redis-1            1/1     Running   0          14m
redis-2            1/1     Running   0          14m
```
So we have 2 replicas.
```
127.0.0.1:26379> sentinel replicas myRedisExample
1)  1) "name"
    2) "10.244.0.20:6379"
    ...
2)  1) "name"
    2) "10.244.0.22:6379"
    ...
```
If we scale up
```
kubectl scale statefulset redis --replicas=5
```
We can now see 4 replicas using sentinel commands
```
1)  1) "name"
    2) "10.244.0.30:6379"
    ...
2)  1) "name"
    2) "10.244.0.20:6379"
    ...
3)  1) "name"
    2) "10.244.0.32:6379"
    ...
4)  1) "name"
    2) "10.244.0.22:6379"
    ...
```
We can also have a look at the logs in the sentinel.
```
+slave slave 10.244.0.30:6379 10.244.0.30 6379 @ myRedisExample 10.244.0.18 6379
+slave slave 10.244.0.32:6379 10.244.0.32 6379 @ myRedisExample 10.244.0.18 6379
```

### Scaling down replicas
From 5 to 3

```
kubectl scale statefulset redis --replicas=3
```
We can see this in sentinel logs
```
+sdown slave 10.244.0.32:6379 10.244.0.32 6379 @ myRedisExample 10.244.0.18 6379
+sdown slave 10.244.0.30:6379 10.244.0.30 6379 @ myRedisExample 10.244.0.18 6379
```
And this one in the primary logs
```
Connection with replica 10.244.0.32:6379 lost.
Connection with replica 10.244.0.30:6379 lost.
```

### Failover
redis-0 is our primary 
```
kubectl describe pod redis-0
Name:         redis-0
...
IP:           10.244.0.18
...
```
We can check that is correct from the sentinel

```
sentinel get-master-addr-by-name myRedisExample
1) "10.244.0.18"
2) "6379"
```
Let's change it
```
127.0.0.1:26379> sentinel failover myRedisExample
OK
```
We have a new primary 
```
127.0.0.1:26379> sentinel get-master-addr-by-name myRedisExample
1) "10.244.0.20"
2) "6379"
```

Logs from sentinel
```
Executing user requested FAILOVER of 'myRedisExample'
+new-epoch 1
+try-failover master myRedisExample 10.244.0.18 6379
+vote-for-leader 7a48cac6bcd5636b7d9c882e468acc4636766726 1
+elected-leader master myRedisExample 10.244.0.18 6379
+failover-state-select-slave master myRedisExample 10.244.0.18 6379
+selected-slave slave 10.244.0.20:6379 10.244.0.20 6379 @ myRedisExample 10.244.0.18 6379
+failover-state-send-slaveof-noone slave 10.244.0.20:6379 10.244.0.20 6379 @ myRedisExample 10.244.0.18 6379
+failover-state-wait-promotion slave 10.244.0.20:6379 10.244.0.20 6379 @ myRedisExample 10.244.0.18 6379
+promoted-slave slave 10.244.0.20:6379 10.244.0.20 6379 @ myRedisExample 10.244.0.18 6379
+failover-state-reconf-slaves master myRedisExample 10.244.0.18 6379
+slave-reconf-sent slave 10.244.0.30:6379 10.244.0.30 6379 @ myRedisExample 10.244.0.18 6379
```

Logs from the redis-0, our original replica. Now a replica
```
Connection with replica 10.244.0.20:6379 lost.
Connection with replica 10.244.0.22:6379 lost.
Before turning into a replica, using my own master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
REPLICAOF 10.244.0.20:6379 enabled (user request from 'id=28 addr=10.244.0.33:50162 fd=8 name=sentinel-215e0487-cmd age=10 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=199 qbuf-free=32569 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=61468 events=r cmd=exec user=default')
CONFIG REWRITE executed with success.
Connecting to MASTER 10.244.0.20:6379
MASTER <-> REPLICA sync started
```

Logs from the new primary, in this case redis-1. Changing from replica to primary
```
Connection with master lost.
Caching the disconnected master state.
Discarding previously cached master state.
Setting secondary replication ID to 69bce30a4ba37f9827fbc650f0336e7de1b1e1e1, valid up to offset: 457690. New replication ID is 920006be1fca92d155479f8a5dbc3ac60ec9963b
MASTER MODE enabled (user request from 'id=17 addr=10.244.0.27:45648 fd=8 name=sentinel-7a48cac6-cmd age=2298 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=188 qbuf-free=32580 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=61468 events=r cmd=exec user=default')
CONFIG REWRITE executed with success.
```

# Deployment
```
kubectl create -f primary
kubectl create -f replica
kubectl create -f sentinel
```
It contains files to deploy one pod for a redis primary, one pod for a replica, and one last pod for sentinel
## playing around
### Testing the replica is working
```
kubectl get pods 

NAME                                         READY   STATUS
redis-primary-deployment-766c7647f6-d7bh7    1/1     Running
redis-replica-deployment-75d4fd777f-j89qk    1/1     Running
redis-sentinel-deployment-8d587554-k2ghh     1/1     Running
```
```
kubectl exec --stdin --tty redis-primary-deployment-766c7647f6-d7bh7 -- /bin/bash

root@redis-primary-deployment-766c7647f6-d7bh7:/data# redis-cli
127.0.0.1:6379> set patient:1 123
OK
```
```
kubectl exec --stdin --tty redis-replica-deployment-75d4fd777f-j89qk -- /bin/bash

root@redis-replica-deployment-75d4fd777f-j89qk:/data# redis-cli
127.0.0.1:6379> KEYS *
1) "patient:1"
```

### Testing a failover
#### Manual failover
```
 kubectl get pods
NAME                                        READY   STATUS    RESTARTS   AGE
redis-primary-deployment-766c7647f6-scphw   1/1     Running   0          78s
redis-replica-deployment-75d4fd777f-z9rft   1/1     Running   0          77s
redis-sentinel-deployment-8d587554-md2qz    1/1     Running   0          76s

kubectl get svc
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S
kubernetes              ClusterIP   10.96.0.1       <none>        443/TCP
redis-primary-service   ClusterIP   10.96.155.166   <none>        6379/TCP
redis-replica-service   ClusterIP   10.96.194.50    <none>        6379/TCP
```

```
kubectl exec --stdin --tty redis-sentinel-deployment-8d587554-md2qz -- /bin/bash
root@redis-sentinel-deployment-8d587554-md2qz:/data# redis-cli -p 26379
127.0.0.1:26379> sentinel get-master-addr-by-name myRedisInstallation
1) "10.96.155.166"
2) "6379"
127.0.0.1:26379> sentinel failover myRedisInstallation
OK
127.0.0.1:26379> sentinel get-master-addr-by-name myRedisInstallation
1) "10.244.0.39"
2) "6379"
127.0.0.1:26379>
```

Now the replica is the primary. You can check the IPs

#### Automatic failover
```
kubectl scale deployment redis-replica-deployment --replicas=0
```

```
127.0.0.1:26379> sentinel get-master-addr-by-name myRedisInstallation
1) "10.96.155.166"
2) "6379"
```

```
kubectl logs redis-sentinel
+sdown master myRedisInstallation 10.244.0.39 6379
+odown master myRedisInstallation 10.244.0.39 6379 #quorum 1/1
+new-epoch 2
+try-failover master myRedisInstallation 10.244.0.39 6379
+vote-for-leader 8c85ffee350a30e0523c6dac89f28e3fdd3b3da6 2
+elected-leader master myRedisInstallation 10.244.0.39 6379
+failover-state-select-slave master myRedisInstallation 10.244.0.39 6379
+selected-slave slave 10.96.155.166:6379 10.96.155.166 6379 @ myRedisInstallation 10.244.0.39 6379
+failover-state-send-slaveof-noone slave 10.96.155.166:6379 10.96.155.166 6379 @ myRedisInstallation 10.244.0.39 6379
+failover-state-wait-promotion slave 10.96.155.166:6379 10.96.155.166 6379 @ myRedisInstallation 10.244.0.39 6379
+promoted-slave slave 10.96.155.166:6379 10.96.155.166 6379 @ myRedisInstallation 10.244.0.39 6379
+failover-state-reconf-slaves master myRedisInstallation 10.244.0.39 6379
+slave-reconf-sent slave 10.244.0.38:6379 10.244.0.38 6379 @ myRedisInstallation 10.244.0.39 6379
+slave-reconf-inprog slave 10.244.0.38:6379 10.244.0.38 6379 @ myRedisInstallation 10.244.0.39 6379
```
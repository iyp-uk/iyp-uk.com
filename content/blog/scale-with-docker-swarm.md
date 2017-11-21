---
title: "Scale With Docker Swarm"
date: 2017-11-21T12:17:29Z
tags: [ "docker", "swarm", "scalability", "performance" ]
---

## What's the plan

In this article, we will demonstrate how to scale your app with [docker swarm](https://docs.docker.com/engine/swarm/).

## Definitions

### Swarm

A [docker swarm](https://docs.docker.com/engine/swarm/key-concepts/#what-is-a-swarm) is a group of docker hosts operating in **swarm mode**.
Every docker host can be:

* A manager
    * Manages orchestration of the workers
* A worker
    * Run containers as part of a *service* (see below for service defintion)
* Both at the same time

### Node

A [docker node](https://docs.docker.com/engine/swarm/key-concepts/#nodes) is an instance of the docker engine.
Several nodes can run on a single machine, but they are usually distributed across various machines.

### Service

A [docker service](https://docs.docker.com/engine/swarm/key-concepts/#services-and-tasks) is the definition of the tasks to execute.
You then specify:

* which container image to use
* which commands to execute inside these running containers

A **replicated service** is distributed amongst multiple nodes.
A **global** service is present on *every* node of the swarm

A task carries a docker container and the commands to run against it.

### Load balancing

Swarm has a built-in [load balancer and DNS](https://docs.docker.com/engine/swarm/key-concepts/#load-balancing) to manage connectivity between the nodes and expose a port for the external world. 

> Read more about these concepts [on the docker documentation](https://docs.docker.com/engine/swarm/key-concepts/)

## Set up the swarm

OK, let's create a swarm locally with `docker-machine` and VirtualBox.

> Your machines could be anywhere. This is just a repeatable and easy enough way of doing it.

> Docker machine has [multiple drivers](https://docs.docker.com/machine/drivers/) which you could be interested in if you want to spin off machines elsewhere.

### Prerequisites

We will create 3 docker machines:

* `manager1`
* `worker1`
* `worker2`

```console
$ docker-machine create --driver virtualbox manager1
$ docker-machine create --driver virtualbox worker1
$ docker-machine create --driver virtualbox worker2
```

Check your machines:
```console
$ docker-machine ls                                
NAME       ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
manager1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.10.0-ce   
worker1    -        virtualbox   Running   tcp://192.168.99.101:2376           v17.10.0-ce   
worker2    -        virtualbox   Running   tcp://192.168.99.102:2376           v17.10.0-ce   
```

Take note of the manager's IP: `192.168.99.100`.

Also take note of the `SWARM` column, where currently nothing is reported.

> We don't specify here any option which could be sent while creating these machines as we just want a linux with docker engine and that's it.

### Set up the swarm

#### Initialise

```console
$ docker-machine ssh manager1
```

Then initialise the swarm from there:

```console
docker@manager1:~$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (x2b4hxuc2v9loe1l37rlzb62w) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3guuuh9hbealub7kxc6cu88ysdhqxtsg4bup3mx6ji5zs2de0n-926iqz8n6mn2cofydizrxf5fn 192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> Take note of the output here, as it provides the token for the other nodes to join the swarm.

Get some insights:
```console
docker@manager1:~$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.10.0-ce
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
Swarm: active
 NodeID: x2b4hxuc2v9loe1l37rlzb62w
 Is Manager: true
 ClusterID: zo29gayi3tefpzzb5951n26as
 Managers: 1
 Nodes: 1
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Number of Old Snapshots to Retain: 0
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
  Force Rotate: 0
 Autolock Managers: false
 Root Rotation In Progress: false
 Node Address: 192.168.99.100
 Manager Addresses:
  192.168.99.100:2377
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 06b9cb35161009dcb7123345749fef02f7cea8e0
runc version: 0351df1c5a66838d0c392b4ac4cf9450de844e2d
init version: 949e6fa
Security Options:
 seccomp
  Profile: default
Kernel Version: 4.4.93-boot2docker
Operating System: Boot2Docker 17.10.0-ce (TCL 7.2); HEAD : 34fe485 - Wed Oct 18 17:16:34 UTC 2017
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 995.8MiB
Name: manager1
ID: RXGS:CGIL:LUHM:4SU3:5RCX:LBXE:VZSR:DJSE:LALT:6XL5:BEBY:TMK7
Docker Root Dir: /mnt/sda1/var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 32
 Goroutines: 137
 System Time: 2017-11-21T14:12:50.411007549Z
 EventsListeners: 0
Registry: https://index.docker.io/v1/
Labels:
 provider=virtualbox
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false
```

Which nodes are in the swarm:
```console
docker@manager1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
x2b4hxuc2v9loe1l37rlzb62w *   manager1            Ready               Active              Leader
```

```console
docker@manager1:~$ exit
```

#### Add workers

We will now ssh into the 2 workers and add them to the swarm using the output from the `docker swarm init` command.

```console
$ docker-machine ssh worker1
```
Then:
```console
docker@worker1:~$ docker swarm join --token SWMTKN-1-3guuuh9hbealub7kxc6cu88ysdhqxtsg4bup3mx6ji5zs2de0n-926iqz8n6mn2cofydizrxf5fn 192.168.99.100:2377
This node joined a swarm as a worker.
```

You can get some info from a worker too,
```console
docker@worker1:~$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.10.0-ce
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
Swarm: active
 NodeID: npf2bl47rnkjeedlkx9in4dfy
 Is Manager: false
 Node Address: 192.168.99.101
 Manager Addresses:
  192.168.99.100:2377
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 06b9cb35161009dcb7123345749fef02f7cea8e0
runc version: 0351df1c5a66838d0c392b4ac4cf9450de844e2d
init version: 949e6fa
Security Options:
 seccomp
  Profile: default
Kernel Version: 4.4.93-boot2docker
Operating System: Boot2Docker 17.10.0-ce (TCL 7.2); HEAD : 34fe485 - Wed Oct 18 17:16:34 UTC 2017
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 995.8MiB
Name: worker1
ID: TT3N:MSWG:NRRX:PGDU:KQSM:WGU2:WT62:4BAS:I24S:HXWA:UJMC:QSUQ
Docker Root Dir: /mnt/sda1/var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 26
 Goroutines: 81
 System Time: 2017-11-21T14:24:28.952824445Z
 EventsListeners: 0
Registry: https://index.docker.io/v1/
Labels:
 provider=virtualbox
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false
```

but not list nodes, which only the manager nodes can see!
```console
docker@worker1:~$ docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
```

> Same thing for `worker2`.

So let's ssh into our `manager1` again, 
```console
$ docker-machine ssh manager1
```
and check the swarm status:
```console
docker@manager1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
x2b4hxuc2v9loe1l37rlzb62w *   manager1            Ready               Active              Leader
npf2bl47rnkjeedlkx9in4dfy     worker1             Ready               Active              
pf20a66rgtactwftxout1d3lx     worker2             Ready               Active              
```
All good, our nodes are ready!
We can now deploy a service to the swarm.

## Deploy a service

Still from the manager,

```console
docker@manager1:~$ docker service create --replicas 1 --name helloworld alpine ping docker.com
dju980v89ljrs5iuqv9qw26g2
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```
This creates a service called `helloworld`, running the command `ping docker.com` in an `alpine` container.

You can see all currently running services within the swarm with:
```console
docker@manager1:~$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
dju980v89ljr        helloworld          replicated          1/1                 alpine:latest       
```
Or, for more details, with the `docker service inspect` command:
```console
docker@manager1:~$ docker service inspect helloworld --pretty

ID:		dju980v89ljrs5iuqv9qw26g2
Name:		helloworld
Service Mode:	Replicated
 Replicas:	1
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		alpine:latest@sha256:d6bfc3baf615dc9618209a8d607ba2a8103d9c8a405b3bd8741d88b4bef36478
 Args:		ping docker.com 
Resources:
Endpoint Mode:	vip
```
We can also see which nodes are running the service:
```console
docker@manager1:~$ docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
dollmixrb8mt        helloworld.1        alpine:latest       manager1            Running             Running 3 minutes ago                       
```
Interestingly, it's running on the manager! YMMV here, so you could see it running on another node.

This is because managers can be assigned tasks by default.

## Scale a service

Ok, let's scale this service and get it to run on other nodes.

We will scale it to 5, which is more than the number of available nodes, so it will run multiple tasks per nodes.

```console
docker@manager1:~$ docker service scale helloworld=5
helloworld scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
``` 
Let's check it then:
```console
docker@manager1:~$ docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
dollmixrb8mt        helloworld.1        alpine:latest       manager1            Running             Running 8 minutes ago                            
dezui9q445ud        helloworld.2        alpine:latest       manager1            Running             Running about a minute ago                       
ym5m5zoq52oz        helloworld.3        alpine:latest       worker1             Running             Running about a minute ago                       
uz35a1k3c3ie        helloworld.4        alpine:latest       worker2             Running             Running about a minute ago                       
0jkmzx4g5m3i        helloworld.5        alpine:latest       worker2             Running             Running about a minute ago                       
```
We have now 5 times the task running:

* 2x on `manager1`
* 1x on `worker1`
* 2x on `worker2`

Total 5 :)

We can also scale **down** similarly:

```console
docker@manager1:~$ docker service scale helloworld=3
helloworld scaled to 3
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
docker@manager1:~$ docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
dezui9q445ud        helloworld.2        alpine:latest       manager1            Running             Running 3 minutes ago                       
ym5m5zoq52oz        helloworld.3        alpine:latest       worker1             Running             Running 3 minutes ago                       
uz35a1k3c3ie        helloworld.4        alpine:latest       worker2             Running             Running 3 minutes ago                       
```

## Delete a service

Still from a manager:

```console
docker@manager1:~$ docker service rm helloworld
helloworld
```
Verify it disappeared:
```console
docker@manager1:~$ docker service ps helloworld
no such service: helloworld
```

## Deploying rolling update

> Rolling updates allow you to deploy new versions of a service following continuous deployment principle and with essentially no downtime (See [rolling releases](https://en.wikipedia.org/wiki/Rolling_release)).

Let's deploy here a `redis` service on a given version. We will then deploy a newer version with a rolling update.

From a manager:
```console
docker@manager1:~$ docker service create \
>   --replicas 3 \
>   --name redis \
>   --update-delay 10s \
>   redis:3.0.6
1j06u71w8eay4jf7ahuzep68f
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
```  

Take note of the `--update-delay 10s` argument here. 
It is specified straight at service creation.

But let's see it in more details:
```console
docker@manager1:~$ docker service inspect --pretty redis

ID:		1j06u71w8eay4jf7ahuzep68f
Name:		redis
Service Mode:	Replicated
 Replicas:	3
Placement:
UpdateConfig:
 Parallelism:	1
 Delay:		10s
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		redis:3.0.6@sha256:6a692a76c2081888b589e26e6ec835743119fe453d67ecf03df7de5b73d69842
Resources:
Endpoint Mode:	vip
```

Take a look at the `UpdateConfig`:

* `Parallelism` is the amount of tasks that can be updated in parallel. In this case, one after the other (default behaviour, can be set with `--update-parallelism`).
* `Delay` is the amount of time between one service has successfully updated (task status is `RUNNING`) and the beginning of the next task update. 10s here, as we've instructed.
* `On failure` is the action to perform when a task fails updating (returns `FAILED`) instead of `RUNNING`
* `Update order` (“start-first”|”stop-first”): `stop-first` here indicates the current version of a task is stopped, then the new one is started.

Check where the `redis` service is running: 
```console
docker@manager1:~$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
dv67f24pag3h        redis.1             redis:3.0.6         worker1             Running             Running 15 minutes ago                       
8dhwsdopmp2d        redis.2             redis:3.0.6         worker2             Running             Running 15 minutes ago                       
vmoipyvlvh0s        redis.3             redis:3.0.6         manager1            Running             Running 15 minutes ago                       
```
Evenly distributed across our 3 nodes, lovely.

Let's update it now to a newer version:
```console
docker@manager1:~$ docker service update --image redis:3.0.7 redis
redis
```
This below gets updated while the update command above runs,
```console
overall progress: 1 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: preparing [=================================>                 ] 
3/3:   
```
It goes along with another way of seeing it:
```console
docker@manager1:~$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
rslfg2no15q0        redis.1             redis:3.0.7         worker1             Running             Running 9 seconds ago                         
dv67f24pag3h         \_ redis.1         redis:3.0.6         worker1             Shutdown            Shutdown 22 seconds ago                       
8dhwsdopmp2d        redis.2             redis:3.0.6         worker2             Running             Running 16 minutes ago                        
vmoipyvlvh0s        redis.3             redis:3.0.6         manager1            Running             Running 16 minutes ago                        
```

When the update is complete:
```console
docker@manager1:~$ docker service update --image redis:3.0.7 redis
redis
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
```
and
```console
docker@manager1:~$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
rslfg2no15q0        redis.1             redis:3.0.7         worker1             Running             Running 2 minutes ago                            
dv67f24pag3h         \_ redis.1         redis:3.0.6         worker1             Shutdown            Shutdown 3 minutes ago                           
ckmxpqsnypbt        redis.2             redis:3.0.7         worker2             Running             Running about a minute ago                       
8dhwsdopmp2d         \_ redis.2         redis:3.0.6         worker2             Shutdown            Shutdown 2 minutes ago                           
srp2rafc58l1        redis.3             redis:3.0.7         manager1            Running             Running 2 minutes ago                            
vmoipyvlvh0s         \_ redis.3         redis:3.0.6         manager1            Shutdown            Shutdown 2 minutes ago                
```

## Drain a node

If you need to take a node out of the swarm, to update it in particular, you can instruct it to not receive updates from the manager.

Inspect your swarm from a manager:
```console
docker@manager1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
x2b4hxuc2v9loe1l37rlzb62w *   manager1            Ready               Active              Leader
npf2bl47rnkjeedlkx9in4dfy     worker1             Ready               Active              
pf20a66rgtactwftxout1d3lx     worker2             Ready               Active              
```

We currently have one redis task running on each node.
Let's take the worker1 out:
```console
docker@manager1:~$ docker node update --availability drain worker1
worker1
```
Check:
```console
docker@manager1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
x2b4hxuc2v9loe1l37rlzb62w *   manager1            Ready               Active              Leader
npf2bl47rnkjeedlkx9in4dfy     worker1             Ready               Drain               
pf20a66rgtactwftxout1d3lx     worker2             Ready               Active              
```

Some more details:
```console
 docker node inspect --pretty worker1
ID:			npf2bl47rnkjeedlkx9in4dfy
Hostname:              	worker1
Joined at:             	2017-11-21 14:22:17.63509973 +0000 utc
Status:
 State:			Ready
 Availability:         	Drain
 Address:		192.168.99.101
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			1
 Memory:		995.8MiB
Plugins:
 Log:		awslogs, fluentd, gcplogs, gelf, journald, json-file, logentries, splunk, syslog
 Network:		bridge, host, macvlan, null, overlay
 Volume:		local
Engine Version:		17.10.0-ce
Engine Labels:
 - provider=virtualbox
TLS Info:
 TrustRoot:
-----BEGIN CERTIFICATE-----
somekeyhere
-----END CERTIFICATE-----

 Issuer Subject:	foo
 Issuer Public Key:	bar
```
Take note of the `Status`:

* `State` shows the machine is up
* `Drain` shows the machine is "off the swarm", it will not receive orders from a manager until its status returns to `ACTIVE`

Our `redis` service has reorganised itself within the remaining ndoes of the swarm:
```console
docker@manager1:~$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
x9uc3roz8lcv        redis.1             redis:3.0.7         manager1            Running             Running 7 minutes ago                         
rslfg2no15q0         \_ redis.1         redis:3.0.7         worker1             Shutdown            Shutdown 7 minutes ago                        
dv67f24pag3h         \_ redis.1         redis:3.0.6         worker1             Shutdown            Shutdown 18 minutes ago                       
ckmxpqsnypbt        redis.2             redis:3.0.7         worker2             Running             Running 17 minutes ago                        
8dhwsdopmp2d         \_ redis.2         redis:3.0.6         worker2             Shutdown            Shutdown 17 minutes ago                       
srp2rafc58l1        redis.3             redis:3.0.7         manager1            Running             Running 17 minutes ago                        
vmoipyvlvh0s         \_ redis.3         redis:3.0.6         manager1            Shutdown            Shutdown 18 minutes ago              
```
Or more readable:
```console
docker@manager1:~$ docker service ps -f desired-state=running redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
x9uc3roz8lcv        redis.1             redis:3.0.7         manager1            Running             Running 8 minutes ago                        
ckmxpqsnypbt        redis.2             redis:3.0.7         worker2             Running             Running 18 minutes ago                       
srp2rafc58l1        redis.3             redis:3.0.7         manager1            Running             Running 18 minutes ago     
```
See how `manager1` now runs two tasks.

Bring the `worker1` back in:
```console
docker@manager1:~$ docker node update --availability active worker1
worker1
```

Is it back?
```console
docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
x2b4hxuc2v9loe1l37rlzb62w *   manager1            Ready               Active              Leader
npf2bl47rnkjeedlkx9in4dfy     worker1             Ready               Active              
pf20a66rgtactwftxout1d3lx     worker2             Ready               Active          
```
Yes, it is!

Interestingly, it doesn't run anything (yet):
```console
docker@manager1:~$ docker service ps -f desired-state=running redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
x9uc3roz8lcv        redis.1             redis:3.0.7         manager1            Running             Running 11 minutes ago                       
ckmxpqsnypbt        redis.2             redis:3.0.7         worker2             Running             Running 21 minutes ago                       
srp2rafc58l1        redis.3             redis:3.0.7         manager1            Running             Running 22 minutes ago            
```

## Make a service available outside the swarm

> Read about [docker swarm networking](https://docs.docker.com/engine/swarm/ingress/).

To make a service available to the outside world, use:

```console
$ docker service create \
  --name <SERVICE-NAME> \
  --publish TARGET-PORT:CONTAINER-PORT \
  <IMAGE>
```
Where:

* `TARGET-PORT` is the port available to the outside world
* `CONTAINER-PORT` is the port the container listens on

Example with NGINX which [exposes port 80](https://github.com/dockerfile/nginx/blob/master/Dockerfile):

From a manager:
```console
docker@manager1:~$ docker service create --name my_web --replicas 3 --publish 8080:80 nginx
bk615pfjc0qg1gr32b5nl1lra
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
```

Now all nodes listen on port `8080` as `nginx` is on all of them:
```console
docker@manager1:~$ docker service ps my_web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
eh3q0xp0826s        my_web.1            nginx:latest        worker1             Running             Running 4 minutes ago                       
b1iugchsy2er        my_web.2            nginx:latest        worker2             Running             Running 4 minutes ago                       
jvswfbxt4zxl        my_web.3            nginx:latest        manager1            Running             Running 4 minutes ago                       
```

From your host (not on any node):
```console
$ curl -I $(docker-machine ip manager1):8080
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 16:01:03 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
Connection: keep-alive
ETag: "59baafbd-264"
Accept-Ranges: bytes

$ curl -I $(docker-machine ip worker1):8080 
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 16:01:40 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
Connection: keep-alive
ETag: "59baafbd-264"
Accept-Ranges: bytes

$ curl -I $(docker-machine ip worker2):8080
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 16:01:45 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
Connection: keep-alive
ETag: "59baafbd-264"
Accept-Ranges: bytes
```

Great! But that's not super convenient, we may want to have a **unique** entry point to all these.

You will need a load balancer to distribute traffic to all these nodes.

Example with [HAProxy](https://www.haproxy.com/), which we will run as a docker container for convenience.

Set the haproxy conf:
```
$ touch haproxy.cfg
```
And then fill this file with:

```ini
global
        debug

defaults
        log global
        mode    http
        timeout connect 5000
        timeout client 5000
        timeout server 5000

# Configure HAProxy to listen on port 80
frontend http_front
        bind *:80
        default_backend http_back

# Configure HAProxy to route requests to swarm nodes on port 8080
backend http_back
        balance roundrobin
        rspadd X-Backend-Server:\ node1 if { srv_id 1 }
        rspadd X-Backend-Server:\ node2 if { srv_id 2 }
        rspadd X-Backend-Server:\ node3 if { srv_id 3 }
        server node1 192.168.99.100:8080 check
        server node2 192.168.99.101:8080 check
        server node3 192.168.99.102:8080 check
```
> Notice we're adding the `X-Backend-Server` with the node's name. 
It's just for clarity here, not something you'd want to run in production or anything.

Now, start the container:

```console
$ docker run -d --name haproxy -p 8888:80 -v $(pwd):/usr/local/etc/haproxy:ro haproxy:1.7.9-alpine
haproxy
```

> HAProxy listens on port `80` by default, but there's probably something running on this port your host, so we're mapping it to port `8888` on host.

We can then query HAProxy and verify it sends traffic to the swarm:
```console
$ curl -I localhost:8888    
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 17:11:25 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
ETag: "59baafbd-264"
Accept-Ranges: bytes
X-Backend-Server: node1

$ curl -I localhost:8888
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 17:11:30 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
ETag: "59baafbd-264"
Accept-Ranges: bytes
X-Backend-Server: node2

$ curl -I localhost:8888
HTTP/1.1 200 OK
Server: nginx/1.13.6
Date: Tue, 21 Nov 2017 17:11:31 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 14 Sep 2017 16:35:09 GMT
ETag: "59baafbd-264"
Accept-Ranges: bytes
X-Backend-Server: node3
```

It's distributed and goes round the different nodes from the swarm.

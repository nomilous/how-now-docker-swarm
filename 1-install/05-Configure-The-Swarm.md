# Configure The Swarm

## Swarm Node

**Each docker host intending to provide container services in the swarm should run this**

`swarm join --help`

<pre>
Usage: swarm join [OPTIONS] <discovery>

join a docker cluster

Arguments:
   <discovery>    discovery service to use [$SWARM_DISCOVERY]
                   * token://<token>
                   * consul://<ip>/<path>
                   * etcd://<ip1>,<ip2>/<path>
                   * file://path/to/file
                   * zk://<ip1>,<ip2>/<path>
                   * [nodes://]<ip1>,<ip2>

Options:
   --advertise, --addr                          Address of the Docker Engine joining the cluster. Swarm manager(s) MUST be able to reach the Docker Engine at this address. [$SWARM_ADVERTISE]
   --heartbeat "60s"                            period between each heartbeat
   --ttl "180s"                                 sets the expiration of an ephemeral node
   --delay "0s"                                 add a random delay in [0s,delay] to avoid synchronized registration
   --discovery-opt [--discovery-opt option --discovery-opt option]  discovery options
</pre>

Start the swarm binary:

* Specify join to join the swarm.
* Point --advertise to the local docker daemon on this node that will be shared into the swarm.
* Specify location of consul swarm service registry where this node registers it's servicees into the cluster. `/swarm` is the key (swarm name)
* Note: swarm binary is in $PATH per previous step

```bash
PRIVATE_IP=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
swarm join --advertise $PRIVATE_IP:2375 consul://$PRIVATE_IP:8500/swarm
^c
```

### Create upstart service for swarm

So the `start|stop|restart swarm` calls to both swarm-node and swarm-manager

```bash
mkdir -p /var/log/swarm
vi /etc/init/swarm.conf
```
```
description "Docker Swarm"

# wait for docker which waited for consul
start on started docker
stop on runlevel [!2345]
```


### Create upstart service for swarm-node

One of these on each node

```bash
vi /etc/init/swarm-node.conf
```
```
description "Docker Swarm Node"

# wait for swarm which waited for docker
start on started swarm
stop on stopped swarm

respawn

script

  PATH=/usr/local/go/bin:$PATH
  PATH=/opt/godep/bin:$PATH

  BIND=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`

  exec swarm join \
    --advertise $BIND:2375 \
    consul://$BIND:8500/swarm >> /var/log/swarm/node.log 2>&1
end script
```

```bash
start swarm

# verify
ps aux | grep swarm\ join
tail /var/log/swarm/node.log
```

## Swarm Manager

**Only one host runs `swarm manage` to manage and provide access to the swarm**

**Not very resiliant on only one host - perhaps there's HA options - later**

`swarm manage --help`

<pre>
Usage: swarm manage [OPTIONS] <discovery>

Manage a docker cluster

Arguments:
   <discovery>    discovery service to use [$SWARM_DISCOVERY]
                   * token://<token>
                   * consul://<ip>/<path>
                   * etcd://<ip1>,<ip2>/<path>
                   * file://path/to/file
                   * zk://<ip1>,<ip2>/<path>
                   * [nodes://]<ip1>,<ip2>

Options:
   --strategy "spread"                                      placement strategy to use [spread, binpack, random]
   --filter, -f [--filter option --filter option]           filter to use [health, port, dependency, affinity, constraint]
   --host, -H [--host option --host option]                 ip/socket to listen on [$SWARM_HOST]
   --replication                                            Enable Swarm manager replication
   --replication-ttl "15s"                                  Leader lock release time on failure
   --advertise, --addr                                      Address of the swarm manager joining the cluster. Other swarm manager(s) MUST be able to reach the swarm manager at this address. [$SWARM_ADVERTISE]
   --tls                                                    use TLS; implied by --tlsverify=true
   --tlscacert                                              trust only remotes providing a certificate signed by the CA given here
   --tlscert                                                path to TLS certificate file
   --tlskey                                                 path to TLS key file
   --tlsverify                                              use TLS and verify the remote
   --engine-refresh-min-interval "30s"                      set engine refresh minimum interval
   --engine-refresh-max-interval "60s"                      set engine refresh maximum interval
   --engine-failure-retry "3"                               set engine failure retry count
   --engine-refresh-retry "3"                               deprecated; replaced by --engine-failure-retry
   --heartbeat "60s"                                        period between each heartbeat
   --api-enable-cors, --cors                                enable CORS headers in the remote API
   --cluster-driver, -c "swarm"                             cluster driver to use [swarm, mesos-experimental]
   --discovery-opt [--discovery-opt option --discovery-opt option]  discovery options
   --cluster-opt [--cluster-opt option --cluster-opt option]        cluster driver options
                                                         * swarm.overcommit=0.05            overcommit to apply on resources
                                                         * swarm.createretry=0              container create retry count after initial failure
                                                         * mesos.address=                   address to bind on [$SWARM_MESOS_ADDRESS]
                                                         * mesos.checkpointfailover=false   checkpointing allows a restarted slave to reconnect with old executors and recover status updates, at the cost of disk I/O [$SWARM_MESOS_CHECKPOINT_FAILOVER]
                                                         * mesos.port=                      port to bind on [$SWARM_MESOS_PORT]
                                                         * mesos.offertimeout=30s           timeout for offers [$SWARM_MESOS_OFFER_TIMEOUT]
                                                         * mesos.offerrefusetimeout=5s      seconds to consider unused resources refused [$SWARM_MESOS_OFFER_REFUSE_TIMEOUT]
                                                         * mesos.tasktimeout=5s             timeout for task creation [$SWARM_MESOS_TASK_TIMEOUT]
                                                         * mesos.user=                      framework user [$SWARM_MESOS_USER]

</pre>

Start the swarm binary:

* Specify manage to manage the swarm.
* Specify --host as where to listen for docker clients attempting to use the swarm, **note: cannot listen on 2375 because that is where the local docker daemon is listening**, this new port behave like a listening docker daemon except it provides access to a swarm instead of a single docker instance
* Specify location of consul swarm service registry where this node registers it's servicees into the cluster. `/swarm` is the key (swarm name)


```bash
PRIVATE_IP=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
swarm manage --host $PRIVATE_IP:4000 consul://$PRIVATE_IP:8500/swarm

# INFO[0000] Initializing discovery without TLS
# INFO[0000] Listening for HTTP                            addr=10.131.2.58:4000 proto=tcp
# INFO[0000] Registered Engine swarm-03.snowball.global at 10.131.2.221:2375
# INFO[0000] Registered Engine swarm-01.snowball.global at 10.131.2.58:2375
# INFO[0000] Registered Engine swarm-02.snowball.global at 10.131.2.131:2375
```

Access from another terminal.

```bash
PRIVATE_IP=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
docker -H tcp://$PRIVATE_IP:4000 info

# Containers: 6
# Images: 10
# Role: primary
# Strategy: spread
# Filters: health, port, dependency, affinity, constraint
# Nodes: 3
#  swarm-01.snowball.global: 10.131.2.58:2375
#   └ Status: Healthy
#   └ Containers: 2
#   └ Reserved CPUs: 0 / 2
#   └ Reserved Memory: 0 B / 2.053 GiB
#   └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-71-generic, operatingsystem=Ubuntu 14.04.3 LTS, # storagedriver=aufs
#   └ Error: (none)
#   └ UpdatedAt: 2016-01-24T11:53:45Z
#  swarm-02.snowball.global: 10.131.2.131:2375
#   └ Status: Healthy
#   └ Containers: 2
#   └ Reserved CPUs: 0 / 2
#   └ Reserved Memory: 0 B / 2.053 GiB
#   └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-71-generic, operatingsystem=Ubuntu 14.04.3 LTS, # storagedriver=aufs
#   └ Error: (none)
#   └ UpdatedAt: 2016-01-24T11:53:26Z
#  swarm-03.snowball.global: 10.131.2.221:2375
#   └ Status: Healthy
#   └ Containers: 2
#   └ Reserved CPUs: 0 / 2
#   └ Reserved Memory: 0 B / 2.053 GiB
#   └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-71-generic, operatingsystem=Ubuntu 14.04.3 LTS, # storagedriver=aufs
#   └ Error: (none)
#   └ UpdatedAt: 2016-01-24T11:53:54Z
# CPUs: 6
# Total Memory: 6.158 GiB
# Name: swarm-01.snowball.global

```

on swarm-01

### Create upstart service for swarm manager

```bash
vi /etc/init/swarm-manager.conf
```
```
description "Docker Swarm Manager"

# wait for swarm which waited for docker
start on started swarm
stop on stopped swarm

respawn

script

  PATH=/usr/local/go/bin:$PATH
  PATH=/opt/godep/bin:$PATH

  BIND=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
  PORT=4000

  exec swarm manage \
    --host $BIND:$PORT \
    consul://$BIND:8500/swarm >> /var/log/swarm/manager.log 2>&1
end script
```

```bash
restart swarm

# verify
ps aux | grep swarm\ manage
tail /var/log/swarm/manager.log
```

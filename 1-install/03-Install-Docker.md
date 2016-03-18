# Install Docker

**repeat on each node**

[https://docs.docker.com/engine/installation/ubuntulinux/](https://docs.docker.com/engine/installation/ubuntulinux/)

## Update docker for swarm configuration

usually docker only listens on local unix file socket, but in a swarm they need to see each other

```bash

vi /etc/default/docker
```
**Give each daemon a sequence number corresponding to hostname swarm-01, 02, ...** (used in cases where creating continers on specific node is needed)
```
# On first node
DOCKER_SEQ=01
PRIVATE_IP=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
CLUSTER_STORE="--cluster-store=consul://$PRIVATE_IP:8500 --cluster-advertise=$PRIVATE_IP:2375"
DOCKER_OPTS="-H=$PRIVATE_IP:2375 -H=unix:///var/run/docker.sock $CLUSTER_STORE --label seq=$DOCKER_SEQ"

# On all subsequent nodes (point com.docker.network.driver.overlay.neighbor_ip to eth1 on first node)
DOCKER_SEQ=02
PRIVATE_IP=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`
CLUSTER_STORE="--cluster-store=consul://$PRIVATE_IP:8500 --cluster-advertise=$PRIVATE_IP:2375"
DOCKER_OPTS="-H=$PRIVATE_IP:2375 -H=unix:///var/run/docker.sock $CLUSTER_STORE --label seq=$DOCKER_SEQ"

# And so on
DOCKER_SEQ=03
...


```

```bash
restart docker

# verify listening
netstat -an | grep 2375
```

## Update Docker wait for Consul

```bash
vi /etc/init/docker.conf

# replace start on
#### start on (local-filesystems and net-device-up IFACE!=lo)
start on started consul-server

```

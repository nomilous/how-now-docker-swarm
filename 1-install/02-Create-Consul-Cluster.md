# Create Consul Cluster

Docker swarm requires a key-value store. The key-value store holds information about the network state which includes discovery, networks, endpoints, IP addresses, and more. Docker supports Consul, Etcd, and ZooKeeper key-value stores.

Choosing consul.

Assuming ubuntu or similar.

**Get latest zip url from [https://www.consul.io/downloads.html](https://www.consul.io/downloads.html)**

Using latest as at now in setup steps

**repeat on each node**

```bash
# as root
cd /root && mkdir downloads && cd downloads/
wget https://releases.hashicorp.com/consul/0.6.3/consul_0.6.3_linux_amd64.zip
apt-get install unzip
#
# verify
# sha256sum consul_0.6.3_linux_amd64.zip
# https://releases.hashicorp.com/consul/0.6.3/consul_0.6.3_SHA256SUMS
#
unzip consul_0.6.3_linux_amd64.zip
cp consul /usr/local/bin/
consul # should display usage

# create user to run consul
adduser --system --no-create-home --group --disabled-login --disabled-password consul

# create server config dir
mkdir /etc/consul-server.d

# create config
vi /etc/consul-server.d/00-server.json
```
```json
{
  "datacenter": "snowball",
  "data_dir": "/opt/consul/server",
  "log_level": "INFO",
  "server": true,
  "client_addr": "10.131.2.58",
  "bootstrap_expect": 2,
  "retry_join": ["10.131.2.221", "10.131.2.131"]
}
```
**NB in above**
* **client_addr** is local private lan interface (eth1)
* **bootstrap_expect** - required minimum number of active nodes before a cluster master is chosen during boot, suggest more than half the cluster size
* **retry_join** - list of at least one other node addresses for cluste discovery (**using private network**)

```bash
# create directory for log
mkdir /var/log/consul
chown consul:consul /var/log/consul

# create directory for data
mkdir -p /opt/consul/server
chown -R consul:consul /opt/consul
```

```
# create upstart to start on boot
vi /etc/init/consul-server.conf
```
```sh
description "Consul Server"

start on (local-filesystems and net-device-up IFACE!=lo)
stop on runlevel [!2345]

respawn

script
  # Make sure to use all our CPUs, because Consul can block a scheduler thread
  export GOMAXPROCS=`nproc`

  # Get the private vlan IP
  BIND=`ifconfig eth1 | grep "inet addr" | awk '{ print substr($2,6) }'`

  exec /usr/local/bin/consul agent \
    -config-dir="/etc/consul-server.d" \
    -bind=$BIND \
    >>/var/log/consul/server.log 2>&1
end script
```
```
# start consul on all hosts
start consul-server

# consul no longer listening on default 127.0.0.1 will need custom env for client
vi /etc/profile.d/consul.sh

# add the following with the ip pointing to local eth1
export CONSUL_RPC_ADDR="10.131.2.58:8400"

# load it
. /etc/profile.d/consul.sh

# list cluster members
consul members

Node        Address             Status  Type    Build  Protocol  DC
swarm-01.snowball.global  10.131.2.58:8301   alive   server  0.6.3  2         snowball
swarm-02.snowball.global  10.131.2.131:8301  alive   server  0.6.3  2         snowball
swarm-03.snowball.global  10.131.2.221:8301  alive   server  0.6.3  2         snowball
```
# Secure The Swarm

***
***
***
***
***

SKIP THIS.

Can't get swarm nodes to attach to a secured manager.

So keeping the manager on private network.

## Create CA key and cert

Docker client (developer workstations) can only connect to the swarm if in possesion of a TLS key sighed by this CA

On the swarm manager host

```bash
# as root
cd /root && mkdir -p certs/swarm && cd certs/swarm

# create CA key
openssl genrsa -aes256 -out ca-key.pem 4096
chmod -v 0400 ca-key.pem

# create CA cert
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
chmod -v 0444 ca.pem
```

## Create and sign server key

On the swarm manager host

```bash
# as root
cd /root && mkdir -p certs/swarm && cd certs/swarm

# create key
openssl genrsa -out server-key.pem 4096
chmod -v 0400 server-key.pem

# create signing request
openssl req -subj "/CN=swarm.snowball.global" -sha256 -new -key server-key.pem -out server.csr

# also allow connections to
echo subjectAltName = IP:10.131.2.58,IP:127.0.0.1 > extfile.cnf

# sign key with CA
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf

chmod -v 0444 server-cert.pem
rm server.csr

```

## Create and sign a client key

On the swarm manager host (because thats where the CA key is, and osx does some incompatable stuff with keys)

```bash
# as root
cd /root && mkdir -p certs/swarm && cd certs/swarm
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
echo extendedKeyUsage = clientAuth > extfile.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile.cnf

chmod -v 0400 key.pem
chmod -v 0444 cert.pem
rm extfile.cnf
rm client.csr

# make package for client distribution
tar -zcf keys.tgz ca.pem key.pem cert.pem
chmod -v 0400 keys.tgz
```

**docker clients will need keys.tgz to connect to swarm**

## Restart swarm manager with keys

On the swarm manager host

swarm-01

```bash
stop swarm
vi /etc/init/swarm-manager.conf
```

Swing to public ip and docker TLS port and add tls config

```
description "Docker Swarm Manager"

# wait for swarm which waited for docker
start on started swarm
stop on stopped swarm

respawn

script

  PATH=/usr/local/go/bin:$PATH
  PATH=/opt/godep/bin:$PATH

  BIND=`ifconfig eth0 | grep "inet addr" | awk '{ print substr($2,6) }'`
  PORT=2376

  exec swarm manage \
    --host $BIND:$PORT \
    --tlsverify \
    --tlscacert=/root/certs/swarm/ca.pem \
    --tlscert=/root/certs/swarm//server-cert.pem \
    --tlskey=/root/certs/swarm//server-key.pem \
    consul://$BIND:8500/swarm >> /var/log/swarm/manager.log 2>&1
end script
```
```bash
start swarm
```


## Connect workstation client to swarm manager

```bash
mkdir -p ~/.docker/certs/snowball
cd ~/.docker/certs/snowball
scp root@swarm.snowball.global:certs/swarm/keys.tgz .
tar -zxf keys.tgz

export DOCKER_TLS_VERIFY=yes
export DOCKER_CERT_PATH=~/.docker/certs/snowball
export DOCKER_HOST=tcp://swarm.snowball.global:2376

docker -H tcp://swarm.snowball.global:2376 info

# Containers: 0
# Images: 0
# Role: primary
# Strategy: spread
# Filters: health, port, dependency, affinity, constraint
# Nodes: 0
# CPUs: 0
# Total Memory: 0 B
# Name: 9ad7f80baf43
```

***There are no nodes in swarm**

That is because swarm_controllers (on on each node) cannot connect to secured swarm manager.

## Recreate swarm_controllers to use keys

Repeat for each swarm node

Assuming previous instruction has resulted in keys.tgz in ~/.docker/certs/snowball on workstation

```bash
scp keys.tgz root@swarm-01.snowball.global:/opt/docker/swarm/controller/
scp keys.tgz root@swarm-02.snowball.global:/opt/docker/swarm/controller/
scp keys.tgz root@swarm-03.snowball.global:/opt/docker/swarm/controller/
```

```bash
csshX root@swarm-01.snowball.global root@swarm-02.snowball.global root@swarm-03.snowball.global

...
```
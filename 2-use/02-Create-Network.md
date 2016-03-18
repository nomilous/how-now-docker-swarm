# Create Network

Creating a named network in the swarm and pushing containers into the network allows for the containers to easily connect to one another by name.

**These instructions assume you are using the `swarm` alias created in [01-Workstation-Access](https://github.com/nomilous/how-now-docker-swarm/blob/master/2-use/01-Workstation-Access.md)**

## Create Named Network

```bash
# docker network create my_net
swarm network create my_net
fe66b8bbf7cd4b9c29fb9a3b349500bfe037b5e0b5ead87b4eadd511cf940338

swarm network ls

# NETWORK ID          NAME                              DRIVER
# 95ee2160e73f        swarm-01.snowball.global/host     host
# 2aefa41797ee        swarm-02.snowball.global/host     host
# ce8e5f291c2e        my_net                            overlay
# 82f4b974f372        swarm-01.snowball.global/bridge   bridge
# f0d41ef4046b        swarm-03.snowball.global/bridge   bridge
# d3fbc685a02c        swarm-01.snowball.global/none     null
# 4df24b069cfa        swarm-02.snowball.global/bridge   bridge
# 6fd1c2859aae        swarm-02.snowball.global/none     null
# 6daf104cf4fb        swarm-03.snowball.global/none     null
# 9e2564cd7e8c        swarm-03.snowball.global/host     host

# Note that 'my_net' belongs to no particular docker host (uses overlay driver)
# and exists in each individual docker instance

ssh swarm-02.snowball.global sudo docker network ls

# NETWORK ID          NAME                DRIVER
# 4df24b069cfa        bridge              bridge
# 6fd1c2859aae        none                null
# 2aefa41797ee        host                host
# ce8e5f291c2e        my_net              overlay

```

## Create containers in network

```bash
# -d - to detatch
# -e - to set environment variables in container - here specifying constraint - which swarm node to run on per 'seq' label assigned in [03-Install-Docker](https://github.com/nomilous/how-now-docker-swarm/blob/master/1-install/03-Install-Docker.md)
# ubuntu - name of image
# sleep 1000 - shell instruction - runs in container - here simply to keep the container running - most containers are designed to exit when 'done'

swarm run -d --name machine1 --net my_net -e constraint:seq==01 ubuntu sleep 10000
swarm run -d --name machine2 --net my_net -e constraint:seq==01 ubuntu sleep 10000
swarm run -d --name machine3 --net my_net -e constraint:seq==03 ubuntu sleep 10000

swarm ps
# CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
# e0698e957e1c        ubuntu              "sleep 10000"       18 seconds ago      Up 15 seconds                           # swarm-03.snowball.global/machine3
# 1290000bde6c        ubuntu              "sleep 10000"       28 seconds ago      Up 25 seconds                           # swarm-01.snowball.global/machine2
# 6c0571d4e1d5        ubuntu              "sleep 10000"       36 seconds ago      Up 34 seconds                           # swarm-01.snowball.global/machine1


# connect to running container with exec bash
# -t - allocate tty
# -i - keep stdin open

swarm exec -ti machine1 bash

# root@6c0571d4e1d5:/#
# root@6c0571d4e1d5:/# cat /etc/hosts   <-------------
# 10.0.0.2    6c0571d4e1d5
# 127.0.0.1   localhost
# ::1 localhost ip6-localhost ip6-loopback
# fe00::0 ip6-localnet
# ff00::0 ip6-mcastprefix
# ff02::1 ip6-allnodes
# ff02::2 ip6-allrouters
# 10.0.0.3    machine2                  <------------- other machines in network dynamically
# 10.0.0.3    machine2.my_net
# 10.0.0.4    machine3                  <------------- inserted into hosts file
# 10.0.0.4    machine3.my_net
# root@6c0571d4e1d5:/#
# root@6c0571d4e1d5:/# ping machine3
# PING machine3 (10.0.0.4) 56(84) bytes of data.
# 64 bytes from machine3 (10.0.0.4): icmp_seq=1 ttl=64 time=0.633 ms
# ^C
# --- machine3 ping statistics ---
# 1 packets transmitted, 1 received, 0% packet loss, time 0ms
# rtt min/avg/max/mdev = 0.633/0.633/0.633/0.000 ms
# root@6c0571d4e1d5:/#

# same goes for the others

swarm exec machine2 cat /etc/hosts
swarm exec machine3 cat /etc/hosts

```

## Clean up

```bash
swarm ps
swarm rm -f machine1 machine2 machine3
swarm network rm my_net
```
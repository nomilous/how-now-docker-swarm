# Custom Containers

ie. using Dockerfile

**These instructions assume you are using the `swarm` alias created in [01-Workstation-Access](https://github.com/nomilous/how-now-docker-swarm/blob/master/2-use/01-Workstation-Access.md)**

This repo has 4 custom containers in `./node_modules/swarm-*`

One of them is a webserver and requires access from the outside. Using `-p` argument to the run command can create a port mapping from the outside to a port on the container. But it is necessary to create that continer on a specific swarm node so that we know which server the mapped port is listening on.

**Selecting swarm-03 node for webserver**

## Create local hostname mapping for convenience

```bash
# on workstation

# get ip of swarm-03
host swarm-03.snowball.global
swarm-03.snowball.global has address 188.166.146.15

# add convenient host for resolution
vi /etc/hosts
```
```
188.166.146.15  test
```
```bash
ping test
```

## Build network and containers at swarm (from local definitions)

**assumes current directory is root of repo**

```bash
swarm network create test_net

# NOTE: to not unnecessarily copy node_modules with the build context
cat node_modules/swarm-api/.dockerignore

swarm build -t example/swarm-redis node_modules/swarm-redis
swarm build -t example/swarm-api node_modules/swarm-api
swarm build -t example/swarm-worker node_modules/swarm-worker
swarm build -t example/swarm-kue-dashboard node_modules/swarm-kue-dashboard
```

Changed containers need to be **rebuilt**.

## Run Containers in network

```bash

swarm run --net=test_net -d --name swarm-redis example/swarm-redis

# note assigned name 'swarm-redis' is used in workers to connect to redis
# node_modules/swarm-worker/index.js

swarm run --net=test_net -d --name swarm-worker1 example/swarm-worker
swarm run --net=test_net -d --name swarm-worker2 example/swarm-worker
swarm run --net=test_net -d --name swarm-worker3 example/swarm-worker
swarm run --net=test_net -d --name swarm-worker4 example/swarm-worker
swarm run --net=test_net -d --name swarm-worker5 example/swarm-worker

# start api (webserver) and kue-dashboard on swarm-03 (ping test)
# with ports mapped to the outside

swarm run --net=test_net -p 5000:8080 -d --name swarm-api -e constraint:seq==03 example/swarm-api
swarm run --net=test_net -p 5001:8081 -d --name swarm-kue-dashboard -e constraint:seq==03 example/swarm-kue-dashboard

swarm ps
```
```bash
curl test:5000/arbitrary/url

# {
#   "job_result": {
#     "worker_id": "kue:b0d8f15bfc91:16:request:1",
#     "job": {
#       "url": "/arbitrary/url"
#     }
#   }
# }
```

[http://test:5001](http://test:5001)

## Updating container

It seems that a full delete/rebuild/run is necessary.

```bash
swarm rm -f swarm-api
swarm build -t example/swarm-api node_modules/swarm-api
swarm run --net=test_net -p 5000:8080 -d --name swarm-api -e constraint:seq==03 example/swarm-api



#
#
# i have experienced the above still then running the old image... (perhaps this helps)

swarm images

# REPOSITORY                    TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
# example/swarm-api             latest              fc980cc7fcd5        7 minutes ago       669.8 MB
# example/swarm-redis           latest              77718e5487c0        17 minutes ago      151.3 MB
# example/swarm-kue-dashboard   latest              34ba2c3d325e        17 minutes ago      669.8 MB
# example/swarm-kue-dashboard   latest              dfd90dac1408        24 minutes ago      669.8 MB
# example/swarm-worker          latest              98e5072c5ccb        24 minutes ago      669.8 MB
# example/swarm-redis           latest              6a8794aa7735        25 minutes ago      151.3 MB
# example/swarm-kue-dashboard   latest              5b819f720a2f        About an hour ago   669.8 MB
# node                          4                   7e9a0f4cde80        3 days ago          642.7 MB
# redis                         latest              8bccd73928d9        2 weeks ago         151.3 MB

# delete image
swarm rmi -f fc980cc7fcd5
```

## Clear Up

```bash
swarm rm -f swarm-worker1 swarm-worker2 swarm-worker3 swarm-worker4 swarm-worker5 \
  swarm-api swarm-kue-dashboard swarm-redis
swarm network rm test_net
```
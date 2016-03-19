# Workstation Access

Configure workstation for access to swarm.

Add tunnel connection alias and DOCKER_HOST env to `.bash_profile`

```bash
export DOCKER_HOST=tcp://localhost:4000
alias swarm_connect="ssh -NL 4000:10.131.2.58:4000 swarm.snowball.global"
```

Use `swarm_connect` command to create tunnel, leave running.<br />
Use `docker` command normally (it connects to tunnel at localhost:4000 per DOCKER_HOST exported env).

```bash
docker info

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
#
#  ...
#  ...
#
#   └ Error: (none)
#   └ UpdatedAt: 2016-01-24T20:55:56Z
# CPUs: 6
# Total Memory: 6.158 GiB
# Name: swarm-01.snowball.global

## run webserver, specifically on swarm-01 using sequence from --label in docker opts /etc/default/docker

docker run -d -e constraint:seq==01 --name www -p 80:80 nginx
1c2dd4033a013015a3ff983c067cc76570be60c35b5cd5329cce21d320697f52

## is it there?

curl swarm-01.snowball.global

docker rm -f www

```
**Note: file volume mounts from container won't be to your workstation.**
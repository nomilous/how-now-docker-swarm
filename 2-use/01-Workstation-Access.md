# Workstation Access

Configure workstation for access to swarm.

Add aliases to `.bash_profile`

```bash
alias swarm_connect="ssh -NL 4000:10.131.2.58:4000 swarm.snowball.global"
alias swarm="docker -H tcp://localhost:4000"
```

Use `swarm_connect` command to create tunnel, leave running.<br />
Use `swarm` command exactly like docker command.

**NB: `swarm` command is an alias to `docker -H swarm-addr`... and bears no resemblance to `swarm` bin on server**

```bash
swarm info

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

swarm run -d -e constraint:seq==01 --name www -p 80:80 nginx
1c2dd4033a013015a3ff983c067cc76570be60c35b5cd5329cce21d320697f52

## is it there?

curl swarm-01.snowball.global

swarm rm -f www

```
**Note: file volume mounts from container won't be to your workstation.**
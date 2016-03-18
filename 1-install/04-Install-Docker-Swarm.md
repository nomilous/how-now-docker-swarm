# Install Docker Swarm

**repeat on each node**

Running swarm as root. (??)

Not running swarm in container.

## Install golang

```bash
mkdir -p /root/downloads && cd /root/downloads
wget https://storage.googleapis.com/golang/go1.5.3.linux-amd64.tar.gz
tar -zxf go1.5.3.linux-amd64.tar.gz
mv go /usr/local/go

vi /etc/profile.d/go.sh
```
```
export PATH=/usr/local/go/bin:$PATH
```
```bash
# load it
. /etc/profile.d/go.sh
```

## Install Godep (package manager (i think?))

Making systemwide (global) location for godep installed packages

```bash
mkdir /opt/godep

vi /etc/profile.d/godep.sh
```
```
export GOPATH=/opt/godep
export PATH=/opt/godep/bin:$PATH
```
```bash
. /etc/profile.d/godep.sh

go get github.com/tools/godep
# see godep installed into /opt/dogep/bin included in path
```

## Install Swarm

```bash
go get github.com/docker/swarm
swarm --help
```
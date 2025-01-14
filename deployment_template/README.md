# Deployment template

This sub-folder contains the configuration files which are required to deploy
this emulator to a Docker Swarm environment.

## Host system setup

The test is meant to be performed on 1 _manager_ node and 20 _worker_ nodes
running Ubuntu 20.04 LTS. Therefore, we require the following host systems:

- 1x 4 core CPU manager host with 32GB of memory
- 20x 1 core CPU worker hosts with 4GB of memory

### Shared setup

Every host system needs to be equipped with Docker. Here's a snippet on how
to do so, taken from https://dockerswarm.rocks:

```bash
# ssh into your instance
# ...

# Install the latest updates
apt-get update
apt-get upgrade -y

# Download Docker
curl -fsSL get.docker.com -o get-docker.sh
# Install Docker using the stable channel (instead of the default "edge")
CHANNEL=stable sh get-docker.sh
# Remove Docker install script
rm get-docker.sh
```

Worker nodes run 1 server and 1 client task each. In order to allow clients
to initiate 50,000 connections, we need to increase the port range.

The manager node needs to be able to open connections to the respective backend
servers. With each server accepting up to 50,000 connections, we have to
increase the port range on the manager node as well.

This can be done like so:

```bash
# 1. change to root
sudo su -
# 2. increase the port range
# we need 50,000 ports + a little extra; the manual recommends
# that one boundary be odd and the other even
sysctl -w net.ipv4.ip_local_port_range="14001 65000"
# 3. save changes and exit
sysctl -p
exit
# 4. now you are back in the user shell; you need to re-log
# from this shell as well for the changes to take effect
exit
```

Note that this configuration change doesn't persist upon re-boot.

### Manager-specific setup

The manager node runs HAProxy. In order for it to accept 1 million connections,
the number of open files limit has to be increased to roughly around 2 million.
To do so, run:

```bash
# 1. change to root
sudo su -
# 2. increase the open files limit
sysctl -w fs.nr_open=2010000
# 3. save changes and exit
sysctl -p
exit
# 4. now you are back in the user shell; you need to re-log
# from this shell as well for the changes to take effect
exit
```

The value of `nf_conntrack_max` also needs to be increased:

```bash
# 1. change to root
sudo su -
# 2. increase nf_conntrack_max
# suggested formula: CONNTRACK_MAX = RAMSIZE (in bytes) / 16384 / (ARCH / 32)
# So, for 32GB RAM and a 64 bit system: 32*1024^3/16384/2=1048576
sysctl -w net.netfilter.nf_conntrack_max=1048576
# 3. save changes and exit
sysctl -p
exit
# 4. now you are back in the user shell; you need to re-log
# from this shell as well for the changes to take effect
exit
```

Note that this configuration change doesn't persist upon re-boot.

## How to deploy

First, [create a new swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)
and [add nodes to it](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/).
All your nodes should be on the same private network.

Next, copy this folder's contents to a _manager_ node and edit `CS_HOST`
environment variable in `docker-compose.yml` to match the public IP of your
manager node.

Then run the following command on your manager node:

```bash
docker stack deploy --compose-file=docker-compose.yml hackathon-2021
```

## View deployment statistics report

Go to: `<PUBLIC_IP_OF_YOUR_MANAGER_NODE>:8404/stats`

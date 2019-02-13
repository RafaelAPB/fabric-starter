# Multi host deployment


## Prepare multi host deployment with Docker Machine

Docker Machine utility provides convenient command-line tool for managing multiple virtual machines (such as Virtualbox or VMs in clouds)
from one terminal and with automatic docker support. 

### Install docker-machine
Install [docker-machine](https://docs.docker.com/machine/get-started/).

### install Oracle Virtualbox
For local deployment also install [VirtualBox](https://www.virtualbox.org/wiki/Downloads).


# Quick start with virtual hosts on local dev machine

Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

Use script [network-create-docker-machine.sh](./network-create-docker-machine.sh) to create a network with an arbitrary 
number of member organizations each running in its own virtual host.
```bash
./network-create-docker-machine.sh org1 org2 org3
```

Of course you can override the defaults with env variables.
```bash
DOMAIN=mynetwork.org \
CHANNEL=a-b \
WEBAPP_HOME=/home/oleg/webapp \
CHAINCODE_HOME=/home/oleg/chaincode \
CHAINCODE_INSTALL_ARGS='example02 1.0 chaincode_example02 golang' \
CHAINCODE_INSTANTIATE_ARGS="a-b example02 [\"init\",\"a\",\"10\",\"b\",\"0\"] 1.0 collections.json AND('a.member','b.member')" \
./network-create-docker-machine.sh a b
```

The script [network-create-docker-machine.sh](./network-create-docker-machine.sh) combines
[host-create.sh](./host-create.sh) to create host machines with `docker-machine` and
[network-create.sh](./network-create.sh) to create container with `docker-compose`.

If you don't want to recreate hosts every time you can re-run `./network-create.sh` with the same arguments and 
env variables to recreate the network on the same remote hosts by clearing and creating containers.   

## Quick start with remote hosts on AWS EC2

Define `amazonec2` driver for docker-machine and open ports in `docker-machine` security group.
Make sure your AWS credentials are saved in env variables or `~/.aws/credentials` or passed as arguments
`--amazonec2-access-key` and `--amazonec2-secret-key`. 
More settings are described on the [driver page](https://docs.docker.com/machine/drivers/aws/).
```bash
DOCKER_MACHINE_FLAGS="--driver amazonec2 --amazonec2-open-port 80 --amazonec2-open-port 7050 --amazonec2-open-port 7051 --amazonec2-open-port 4000" \
./network-create-docker-machine.sh org1 org2 org3
```

## Quick start with remote hosts on Microsoft Azure

Define `azure` driver for docker-machine and open ports in network security groups. 
Give your subscription id (the one looking like `deadbeef-8bad-f00d-989d-5fbe969ccb9e`) and the script will prompt you
to login to your Microsoft account and authorize `Docker Machine for Azure` application to manage your Azure instances. 
More settings are described on the [driver page](https://docs.docker.com/machine/drivers/azure/).
```bash
DOCKER_MACHINE_FLAGS="--driver azure --azure-size Standard_A1 --azure-subscription-id <your subs-id> --azure-open-port 80 --azure-open-port 7050 --azure-open-port 7051 --azure-open-port 4000" \
./network-create-docker-machine.sh org1 org2
```

## Quick start with existing remote hosts

If you have already created remote hosts in the cloud or on premises you can connect docker-machine to these hosts and 
operate with the same scripts and commands.

Make sure the remote hosts have open inbound ports for Fabric network: 80, 4000, 7050, 7051 and for docker: 2376.

Connect via [generic](https://docs.docker.com/machine/drivers/generic/) driver 
to hosts *orderer*, *a* and *b* at specified public IPs with ssh private key `~/docker-machine.pem`.
```bash
docker-machine create --driver generic --generic-ssh-key ~/docker-machine.pem --generic-ssh-user ubuntu \
--generic-ip-address 34.227.123.456 orderer
docker-machine create --driver generic --generic-ssh-key ~/docker-machine.pem --generic-ssh-user ubuntu \
--generic-ip-address 54.173.123.457 a
docker-machine create --driver generic --generic-ssh-key ~/docker-machine.pem --generic-ssh-user ubuntu \
--generic-ip-address 54.152.123.458 b
```

Now the hosts are known to docker-machine and you can run `network-create.sh` script to create 
docker containers running the network and create organizations, channel and chaincode.
```bash
DOMAIN=mynetwork.org CHANNEL=a-b WEBAPP_HOME=/home/oleg/webapp CHAINCODE_HOME=/home/oleg/chaincode CHAINCODE_INSTALL_ARGS='example02 1.0 chaincode_example02 golang' CHAINCODE_INSTANTIATE_ARGS="a-b example02 [\"init\",\"a\",\"10\",\"b\",\"0\"] 1.0 collections.json AND('a.member','b.member')" \
./network-create.sh a b
```

## Drill down

To understand the script please read the below step by step instructions for the network 
of two member organizations org1 and org2.

### Create host machines

Create 3 hosts: orderer and member organizations org1 and org2.
```bash
docker-machine create orderer
docker-machine create org1
docker-machine create org2
```

### Create orderer organization

Tell the scripts to use extra multihost docker-compose yaml files.
```bash
export MULTIHOST=true
```

Copy config templates to the orderer host.
```bash
docker-machine scp -r templates orderer:templates
```

Connect docker client to the orderer host. 
The docker commands that follow will be executed on the host not local machine.
```bash
eval "$(docker-machine env orderer)"
```

Inspect created hosts' IPs and collect them into `hosts` file to copy to the hosts. This file will be mapped to the
docker containers' `/etc/hosts` to resolve names to IPs.
This is better done by the script.
Alternatively, edit `extra_hosts` in `multihost.yaml` and `orderer-multihost.yaml` to specify host IPs directly.

```bash
docker-machine ip orderer
docker-machine ip org1
docker-machine ip org2
```

Generate crypto material for the orderer organization and start its docker containers.
```bash
./generate-orderer.sh
docker-compose -f docker-compose-orderer.yaml -f orderer-multihost.yaml up -d
```

### Create member organizations

Open a new console. Use env variables to tell the scripts to use multihost config yaml and to name your organization.
```bash
export MULTIHOST=true && export ORG=org1
```

Copy templates, chaincode and webapp folders to the host.
```bash
docker-machine scp -r templates $ORG:templates && docker-machine scp -r chaincode $ORG:chaincode && docker-machine scp -r webapp $ORG:webapp
```

Connect docker client to the organization host.
```bash
eval "$(docker-machine env $ORG)"
```

Generate crypto material for the org and start its docker containers.
```bash
./generate-peer.sh
docker-compose -f docker-compose.yaml -f multihost.yaml up -d
```

To create other organizations repeat the above steps in separate consoles 
and giving them names by `export ORG=org2` in the first step.

### Add member organizations to the consortium

Return to the *orderer* console.

Now the member organizations are up and serving their certificates. 
The orderer host can download them to add to the consortium definition. 
```bash
./consortium-add-org.sh org1
./consortium-add-org.sh org2
```

### Create a channel and a chaincode

Return to the *org1* console.

The first organization creates channel *common* and joins to it.
```bash
./channel-create.sh common
./channel-join.sh common
```

And adds other organizations to channel *common*.
```bash
 ./channel-add-org.sh common org2
```

And installs and instantiates chaincode *reference*.
```bash
./chaincode-install.sh reference
./chaincode-instantiate.sh common reference
```

Test the chaincode by invoke and query.
```bash
./chaincode-invoke.sh common reference '["put","account","1","{\"name\":\"one\"}"]'
./chaincode-query.sh common reference '["list","account"]'
```

### Have other organizations join the channel

Return to *org2* console.

Join *common* and install *reference*.
```bash
./channel-join.sh common
./chaincode-install.sh reference
``` 

Test the chaincode by a query by *org2*.
```bash
./chaincode-query.sh common reference '["list","account"]'
```





# Manual docker-machine deployment

There are two approaches for creating virtual machines which are to be used as Nodes:
- utilizing docker-machine functionality for quick creation of virtual machines with docker support from one terminal.
- manual creation of virtual machines independently of each other.

Then we can [Deploy Blockchain network](#blockchain)

See also options for providing name-resolution of DNS names for Hyperledger Fabric components (peers, orderer, CAs) located on different servers:
- configuring extra-hosts parameter which is added by docker-composer to each container's /etc/hosts file
- configuring DNS service specific for particular blockchain network (either central or DNS-cluster)
- using Docker Swarm engine to have all components in one virtual sub-net (there are security issues with this approach
so it's recommended only for test\dev purposes)



### Create necessary amount of Virtual Machines
Using docker machine we can quickly create Virtual Machines either local with VirtualBox or remote in any cloud infrastructure (Azure, AWS, etc).
It's very convenient taht the VMs created by the docker-machine automatically already have the docker engine installed on it.

We then need create one VM for each orderer's and organization's node.

Create a local VM for `orderer` and `org1`:
```bash
docker-machine create --driver virtualbox orderer
docker-machine create --driver virtualbox org1
```

Create VM in Azure:
```bash
docker-machine create --driver azure --azure-subscription-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
--azure-location westeurope \
--azure-size Standard_F4s_v2  \
--azure-open-port 80 --azure-open-port 7050  \
--azure-ssh-user ubuntu \
orderer

docker-machine create --driver azure --azure-subscription-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
--azure-location westeurope \
--azure-size Standard_F4s_v2  \
--azure-open-port 80 --azure-open-port 7051 --azure-open-port 7054 \
--azure-ssh-user ubuntu \
org1
```

We can see the ip address of the created VMs:
```bash
docker-machine ls
```
Returns:
```
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
orderer   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.06.1-ce
org1      *        virtualbox   Running   tcp://192.168.99.101:2376           v18.06.1-ce
```


### Copy templates and chaincode files to remote VM
Copy local template files to all VMs to `orderer` home directory (`/home/docker` on VirtualBox or `/home/ubuntu` on AWS EC2).
```bash
docker-machine scp -r templates orderer:
docker-machine scp -r chaincode orderer:
docker-machine scp -r templates org1:
docker-machine scp -r chaincode org1:
```


### Set multihost-related environment variables
```bash
export MULTIHOST=1
export WORK_DIR=`docker-machine ssh orderer pwd`
```


After all necessary VMs have been created *docker-machine* allows to switch between them just by applying corresponded environment.
Connect to `orderer` machine by setting env variables:
```bash
eval "$(docker-machine env orderer)"
```


When the terminal is "connected" any `docker` commands will actually be executed on the remote VM and not in the local host.
So the administrator can perform necessary operations on each node in order to deploy the network.


## Prepare multi host environment for manual deployment

Manual deployment means that the administrator have to allocate hardware or virtual machine, install Operating System,
install docker and docker-compose utilities and connect to the machines by SSH.

This time you don't need to export MULTIHOST and WORK_DIR environment variables.


The further steps for deploying blockchain components generally don't depend on how are you connected to remote machine (by docker-machine or by SSH)
and just have to be performed in corresponded machine.



<a name="blockchain"></a>
# Deployment of Blockchain network

## Deploy orderer

Connect to `orderer` machine
Now template files are on the `orderer` host and we can run `./generate-orderer.sh` but with a flag
`COMPOSE_FLAGS` which adds an extra docker-compose file `orderer-multihost.yaml` on top of the base
`docker-compose-orderer.yaml`. This file redefines volume mappings from `${PWD}` local to your host machine to
directory on remote host specified by `WORK_DIR` (defaulted to VirtualBox's home `/home/docker`;
for AWS EC2 do `export WORK_DIR=/home/ubuntu`).





## Prepare multi host environment for manual deployment

On each node including orderer clone *fabric-starter* project and export multi-host related flag and
WORK_DIR variable specifying absolute path to fabric-starter folder:

```bash
export MULTIHOST=1
export WORK_DIR=/home/ubuntu/fabric-starter
```

Start Swarm manager and join nodes as described in [swarm.md](swarm.md)
If it's undesirable to use Swarm because of security reasons, configure DNS service(s) as described in [dns.md](dns.md)

Deploy multi-host network in multihost environment as described below.


## Preparing multi host deployment with docker-machine

Install [docker-machine](https://docs.docker.com/machine/get-started/).
For local deployment install [VirtualBox](https://www.virtualbox.org/wiki/Downloads).


### Create orderer machine

Create and start `orderer` docker host as a virtual machine.
For AWS EC2 use `--driver amazonec2` and `export WORK_DIR=/home/ubuntu`.
```bash
docker-machine create --driver virtualbox orderer
```

See the ip address of the created VM.
```bash
docker-machine ls
```
Returns
```
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
orderer   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.06.1-ce
```

Connect to `orderer` machine by setting env variables.
```bash
eval "$(docker-machine env orderer)"
```

Start docker swarm on `orderer` machine. Replace ip address `192.168.99.100` with the actual ip of the remote host interface.
```bash
docker swarm init --advertise-addr 192.168.99.100
```
will output a command to join the swarm by other hosts;

The other hosts will be join to the swarm by the following command (this is an example, use the actual token and ip):
```bash
docker swarm join --token SWMTKN-1-4fbasgnyfz5uqhybbesr9gbhg0lqlcj5luotclhij87owzd4ve-4k16civlmj3hfz1q715csr8lf 192.168.99.100:2377
```

Copy local template files to `orderer` home directory (`/home/docker` on VirtualBox or `/home/ubuntu` on AWS EC2).
```bash
docker-machine scp -r templates orderer:templates
```

Now template files are on the `orderer` host and we can run `./generate-orderer.sh` but with a flag
`COMPOSE_FLAGS` which adds an extra docker-compose file `orderer-multihost.yaml` on top of the base
`docker-compose-orderer.yaml`. This file redefines volume mappings from `${PWD}` local to your host machine to
directory on remote host specified by `WORK_DIR` (defaulted to VirtualBox's home `/home/docker`;
for AWS EC2 do `export WORK_DIR=/home/ubuntu`).

Later docker-machine allows to "connect" to created machines by invoking `docker-machine env` command for specified machine name (org1):
```bash
eval "$(docker-machine env org1)"
```

After that docker commands running in local console will actually be executed in the (remote) machine.


# Deploy network on multiple nodes

Connect to orderer machine (with *ssh* for manual or *docker-machine env* command for docker machine):
```bash
./generate-orderer.sh
```

Start *Orderer* containers on `orderer` machine.
```bash
docker-compose -f docker-compose-orderer.yaml -f orderer-multihost.yaml up -d
```

### Create org1 machine

For docker-machine deployment create and start `org1` docker host as a virtual machine with VirtualBox.
```bash
docker-machine create --driver virtualbox org1
```

and connect to `org1` machine by setting env variables.
```bash
eval "$(docker-machine env org1)"
```

Or connect to `org1` machine by ssh for manual mode.

Join swarm; this is an example, use the actual command output by `orderer`. In orderer console get the command:
```bash
docker swarm join-token worker
```
Execute the output command in `org1` console.
```bash
docker swarm join --token SWMTKN-1-4rzq6pyg3z2kw7eqhqdnc86isjpwo6t69t1q0ooy84csioieyc-ajcb0rdg3l15ronhytmldc59s 192.168.99.100:2377
```

Need to run a dummy container on this new host `org1` to force join overlay network `fabric-overlay`
created by docker-compose on `orderer` host.
```bash
docker run -dit --name alpine --network fabric-overlay alpine
```

Copy local template and chaincode files to `org1` machine.
```bash
docker-machine scp -r templates org1:templates

docker-machine scp -r chaincode org1:chaincode

docker-machine scp -r webapp org1:webapp
```

Generate crypto material for member organization *org1*.

```bash
./generate-peer.sh
```

Start *org1* containers on `org1` machine.
```bash
docker-compose -f docker-compose.yaml -f multihost.yaml up -d
```

### Add org1 to the consortium as Orderer

Now `org1` machine is up and running a web container serving root certificates of *org1*. The orderer can access it
via an overlay network, download certs and add *org1*.

Connect to `orderer` machine by setting env variables or by SSH.
```bash
eval "$(docker-machine env orderer)"
```

Run local script to add to the consortium.
```
./consortium-add-org.sh org1
```

### Create a channel and deploy chaincode as org1

Connect to `org1` machine by setting env variables.
```bash
eval "$(docker-machine env org1)"
```

Run local scripts to create and join channels.
```
./channel-create.sh common

./channel-join.sh common

./chaincode-install.sh reference

./chaincode-instantiate.sh common reference
```

### Create org2 machine

```bash
export ORG=org2
export ORGS='{"org1":"peer0.org1.example.com:7051","org2":"peer0.org2.example.com:7051"}' CAS='{"org2":"ca.org2.example.com:7054"}'
docker-machine create --driver virtualbox org2
eval "$(docker-machine env org2)"
docker swarm join --token SWMTKN-1-4fbasgnyfz5uqhybbesr9gbhg0lqlcj5luotclhij87owzd4ve-4k16civlmj3hfz1q715csr8lf 192.168.99.102:2377
docker run -dit --name alpine --network fabric-overlay alpine
docker-machine scp -r templates org2:templates
docker-machine scp -r chaincode org2:chaincode
./generate-peer.sh
docker-compose -f docker-compose.yaml -f multihost.yaml up
```

Return to `org1` machine to add *org2* to the channel.
```bash
eval "$(docker-machine env org1)"
./channel-add-org.sh common org2
```

Back to `org2` machine to join channel.
```bash
export ORG=org2
eval "$(docker-machine env org2)"
./channel-join.sh common
```

Install chaincode (no need to instantiate) and query.
```bash
./chaincode-install.sh reference
./chaincode-query.sh common reference '["list","account"]'
```

### Create machines for other organizations

The above steps are collected into a single script that creates a machine, generates crypto material and starts
containers on a remote or virtual host for a new org. This example is for *org3*; replace swarm token and manager ip
with your own from `docker swarm join-token worker`.
```bash
./machine-create-peer.sh SWMTKN-1-4fbasgnyfz5uqhybbesr9gbhg0lqlcj5luotclhij87owzd4ve-4k16civlmj3hfz1q715csr8lf 192.168.99.102 org3
```
